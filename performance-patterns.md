# Performance Patterns — From Correct to Fast

Performance optimization without measurement is guessing. This guide covers how to think about
performance, common patterns, and the ordered approach to optimization.

---

## TABLE OF CONTENTS
1. The Performance Hierarchy
2. Measurement First
3. Complexity Analysis
4. Database Performance
5. Caching Strategies
6. API & Network Performance
7. Memory Management
8. Concurrency & Parallelism
9. Common Anti-Patterns
10. Performance Checklist

---

## 1. THE PERFORMANCE HIERARCHY

Optimize in this order. Do not skip levels.

```
Level 1 — Algorithm:      O(n log n) vs O(n²) — 1000x difference at scale
Level 2 — Data structures: HashMap vs List lookup — 100x difference
Level 3 — Database:        Index vs full scan — 10,000x difference
Level 4 — Network:         Batch vs N+1 requests — 100x difference
Level 5 — Caching:         Memory vs DB — 100x difference
Level 6 — Code:            Micro-optimizations — 10% difference

Never start at Level 6. The 10% win at Level 6 is invisible next to
the 1000x loss you haven't fixed at Level 1.
```

---

## 2. MEASUREMENT FIRST

**Profile before you optimize. Your intuition about bottlenecks is wrong.**

### What to Measure
- **Wall clock time**: End-to-end latency (what the user experiences)
- **CPU time**: Compute-bound operations
- **Memory**: Peak usage, allocation rate, GC pressure
- **I/O**: DB query count, query time, network calls
- **Throughput**: Requests/second under load

### How to Measure

**Backend API**
```python
# Add timing middleware — measure every request
import time

@app.middleware("http")
async def add_timing(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration_ms = (time.perf_counter() - start) * 1000
    response.headers["X-Response-Time"] = f"{duration_ms:.2f}ms"
    # Log slow requests
    if duration_ms > 500:
        logger.warning(f"Slow request: {request.method} {request.url.path} took {duration_ms:.0f}ms")
    return response
```

**Database Queries**
```sql
-- PostgreSQL: Find slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Enable EXPLAIN ANALYZE for suspicious queries
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders WHERE user_id = 123;
```

### Defining "Fast Enough"
Before optimizing, define targets:
- P50 latency: ___ms (median user experience)
- P95 latency: ___ms (95th percentile — what most users see)
- P99 latency: ___ms (tail latency — what your worst users see)
- Throughput target: ___req/sec

Optimizing beyond "fast enough" is waste. Stop when targets are met.

---

## 3. COMPLEXITY ANALYSIS

### Big-O Quick Reference

| Complexity | n=100 | n=10,000 | n=1,000,000 | Acceptable? |
|---|---|---|---|---|
| O(1) | 1 | 1 | 1 | Always |
| O(log n) | 7 | 14 | 20 | Always |
| O(n) | 100 | 10,000 | 1,000,000 | Usually |
| O(n log n) | 700 | 140,000 | 20,000,000 | Often |
| O(n²) | 10,000 | 100,000,000 | 10^12 | n<1000 only |
| O(2ⁿ) | 10^30 | — | — | Never (in practice) |

### Identifying O(n²) Code

```python
# OBVIOUS O(n²) — nested loops over same collection
for item in items:
    for other in items:
        if item.id == other.parent_id:
            ...

# HIDDEN O(n²) — loop with a linear search inside
for user in users:
    if user.id in [order.user_id for order in orders]:  # list 'in' is O(n)!
        ...

# FIX — Build lookup set first: O(n) total
order_user_ids = {order.user_id for order in orders}  # O(n) once
for user in users:
    if user.id in order_user_ids:  # O(1) lookup
        ...
```

### When O(n²) Is Acceptable
- n is guaranteed to be small and bounded (n < 100)
- The algorithm is much simpler and you've profiled that it's fast enough
- State this explicitly with a comment: `# O(n²) — n is bounded by BATCH_SIZE (max 50)`

---

## 4. DATABASE PERFORMANCE

### The N+1 Problem — The Most Common Performance Bug

```python
# N+1 QUERY — fetches 1 order list + N user queries
orders = Order.find_all()
for order in orders:
    print(f"Order by {order.user.name}")  # Queries DB for each user!
    # 100 orders = 101 DB queries

# SOLVED — Eager loading
orders = Order.find_all(include=["user"])  # 2 DB queries total
for order in orders:
    print(f"Order by {order.user.name}")  # Uses preloaded data
```

**Detecting N+1**: In Django, use `django-debug-toolbar`. In Rails, use `bullet`. Log slow requests.
Any endpoint making >5 DB queries is suspicious. >20 is almost certainly N+1.

### Index Strategy

```sql
-- ALWAYS index:
--   - Foreign key columns
--   - Columns used in WHERE clauses
--   - Columns used in ORDER BY (esp. with LIMIT)
--   - Columns used in JOIN conditions

-- Single column index
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Composite index (order matters!)
-- Supports: WHERE status = ? AND created_at > ?
-- Supports: WHERE status = ?  (prefix match)
-- Does NOT support: WHERE created_at > ?  (non-prefix)
CREATE INDEX idx_orders_status_created ON orders(status, created_at);

-- Partial index (smaller, faster for filtered queries)
CREATE INDEX idx_active_orders ON orders(user_id) WHERE status = 'active';

-- Covering index (includes extra columns — avoids table lookup)
CREATE INDEX idx_orders_user_status_total ON orders(user_id, status) INCLUDE (total);
```

### Query Optimization Patterns

```sql
-- BAD: Function on indexed column destroys index usage
WHERE LOWER(email) = 'user@example.com'
-- GOOD: Store normalized, or use functional index
WHERE email = LOWER('user@example.com')
-- OR: CREATE INDEX idx_email_lower ON users(LOWER(email));

-- BAD: SELECT * (transfers unnecessary data, prevents covering indexes)
SELECT * FROM orders WHERE user_id = ?
-- GOOD: Select needed columns only
SELECT id, total, status, created_at FROM orders WHERE user_id = ?

-- BAD: OFFSET for pagination at high page numbers (scans all rows)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000
-- GOOD: Cursor-based pagination
SELECT * FROM orders WHERE created_at < ? ORDER BY created_at DESC LIMIT 20

-- BAD: Subquery executed for every row
SELECT *, (SELECT COUNT(*) FROM order_items WHERE order_id = orders.id) as item_count
FROM orders
-- GOOD: Aggregation with JOIN
SELECT orders.*, COALESCE(item_counts.count, 0) as item_count
FROM orders
LEFT JOIN (
    SELECT order_id, COUNT(*) as count FROM order_items GROUP BY order_id
) item_counts ON item_counts.order_id = orders.id
```

### Connection Pooling
```python
# NEVER — Create connection per request
def get_user(user_id):
    conn = psycopg2.connect(DATABASE_URL)  # Expensive! ~10ms per connection
    # ...

# ALWAYS — Use a connection pool
from psycopg2 import pool
connection_pool = pool.ThreadedConnectionPool(5, 20, DATABASE_URL)

def get_user(user_id):
    conn = connection_pool.getconn()  # Reuse existing connection
    try:
        # ...
    finally:
        connection_pool.putconn(conn)
```

---

## 5. CACHING STRATEGIES

### What to Cache

```
GOOD cache candidates:
  - Expensive computations that rarely change (reports, aggregations)
  - External API responses (weather, exchange rates, user profiles)
  - Database queries for relatively static data (categories, config, permissions)
  - Rendered page fragments
  - Session data

BAD cache candidates:
  - User-specific data that changes frequently
  - Data where stale reads would cause incorrect behavior
  - Data with complex invalidation requirements (risk of inconsistency)
  - Very cheap data (caching overhead exceeds query cost)
```

### Cache Invalidation Strategies

```python
# Strategy 1: TTL-based (simple, eventually consistent)
cache.set("popular_products", products, ttl=300)  # stale for up to 5 min

# Strategy 2: Event-based (consistent, more complex)
def update_product(product_id, data):
    db.update(product_id, data)
    cache.delete(f"product:{product_id}")                    # Direct invalidation
    cache.delete(f"category:{data['category_id']}:products") # Related list

# Strategy 3: Write-through (always consistent)
def update_product(product_id, data):
    db.update(product_id, data)
    cache.set(f"product:{product_id}", data)  # Update cache simultaneously

# Strategy 4: Cache-aside (lazy loading)
def get_product(product_id):
    cached = cache.get(f"product:{product_id}")
    if cached:
        return cached
    product = db.find(product_id)
    cache.set(f"product:{product_id}", product, ttl=600)
    return product
```

### Cache Stampede Prevention
When cache expires, many concurrent requests all miss and all hit the DB simultaneously.

```python
import threading

_locks = {}

def get_with_stampede_protection(key, fetch_fn, ttl):
    cached = cache.get(key)
    if cached:
        return cached
    
    # Acquire lock per key — only one request fetches
    if key not in _locks:
        _locks[key] = threading.Lock()
    
    with _locks[key]:
        # Check again — another thread may have fetched while we waited
        cached = cache.get(key)
        if cached:
            return cached
        
        result = fetch_fn()
        cache.set(key, result, ttl=ttl)
        return result
```

---

## 6. API & NETWORK PERFORMANCE

### Request Batching
```javascript
// BAD — N separate API calls
const users = await Promise.all(
  userIds.map(id => fetch(`/api/users/${id}`))
)

// GOOD — One batched call
const users = await fetch('/api/users/batch', {
  method: 'POST',
  body: JSON.stringify({ ids: userIds })
})
```

### Response Compression
Always enable gzip/brotli compression for API responses. Typically reduces payload by 70-80%.

### HTTP/2 and Connection Reuse
HTTP/1.1: 6 concurrent connections per domain (browser limit)
HTTP/2: Multiplexes all requests over single connection — eliminates need for domain sharding

### GraphQL vs REST for Performance
- REST with field selection: `GET /users?fields=id,name,email`
- GraphQL: Naturally prevents over-fetching
- Key: Design responses around use cases, not data schemas

### CDN for Static Assets
- All static assets (JS, CSS, images, fonts) behind CDN
- Immutable caching with content hashing: `app.abc123.js` with `Cache-Control: max-age=31536000, immutable`
- Dynamic content: CDN edge caching with short TTL + stale-while-revalidate

---

## 7. MEMORY MANAGEMENT

### Memory Leak Detection

```javascript
// Common leak: Event listener not removed
class Component {
  mount() {
    this.handleResize = () => this.resize()
    window.addEventListener('resize', this.handleResize)
  }
  
  unmount() {
    window.removeEventListener('resize', this.handleResize)  // MUST remove
  }
}

// React: useEffect cleanup
useEffect(() => {
  window.addEventListener('resize', handleResize)
  return () => window.removeEventListener('resize', handleResize)  // cleanup
}, [])
```

### Streaming vs Buffering
```python
# BAD — Load entire file into memory
def process_large_file(path):
    content = open(path).read()  # 2GB file = 2GB RAM
    return process(content)

# GOOD — Stream in chunks
def process_large_file(path):
    with open(path, 'r') as f:
        for chunk in f:  # Process line by line
            process_line(chunk)
```

### Object Pooling for Hot Paths
If you're creating and discarding thousands of small objects per second (game loops, trading systems),
object pooling avoids GC pressure.

---

## 8. CONCURRENCY & PARALLELISM

### When to Parallelize

```
I/O-bound work: ALWAYS parallelize
  - Multiple API calls
  - Multiple DB queries (when independent)
  - File reads

CPU-bound work: Parallelize carefully
  - True parallelism requires multiple processes (Python GIL)
  - Use worker threads/processes with proper task distribution
  - Overhead may exceed benefit for small tasks
```

### Async Patterns

```python
# BAD — Sequential I/O
async def get_dashboard_data(user_id):
    user = await db.get_user(user_id)        # Wait 5ms
    orders = await db.get_orders(user_id)    # Wait 10ms
    stats = await compute_stats(user_id)     # Wait 15ms
    return {user, orders, stats}             # Total: 30ms

# GOOD — Concurrent I/O
async def get_dashboard_data(user_id):
    user, orders, stats = await asyncio.gather(
        db.get_user(user_id),
        db.get_orders(user_id),
        compute_stats(user_id),
    )
    return {user, orders, stats}             # Total: 15ms (longest operation)
```

### Worker Queues for Heavy Tasks
Never do heavy work (image resizing, PDF generation, email sending, report generation) in the request path.

```python
# BAD — Block the request
@app.post("/upload")
async def upload(file: UploadFile):
    image = await process_image(file)  # Takes 2 seconds — request hangs!
    return {"url": image.url}

# GOOD — Queue it
@app.post("/upload")
async def upload(file: UploadFile):
    job_id = await queue.enqueue("process_image", file_path=save_temp(file))
    return {"jobId": job_id, "status": "processing"}  # Returns immediately
```

---

## 9. COMMON PERFORMANCE ANTI-PATTERNS

### Anti-Pattern 1: Synchronous Operations in Async Context
```javascript
// BAD: Blocking the event loop
app.get('/data', async (req, res) => {
  const data = fs.readFileSync('large-file.json')  // Blocks ALL requests!
  res.json(JSON.parse(data))
})

// GOOD
app.get('/data', async (req, res) => {
  const data = await fs.promises.readFile('large-file.json')
  res.json(JSON.parse(data))
})
```

### Anti-Pattern 2: Serializing Concurrent Work
```python
# BAD: Sequential when parallel is safe
for user_id in user_ids:
    await send_notification(user_id)  # Wait for each one

# GOOD: Concurrent (with rate limiting to avoid overwhelming the service)
await asyncio.gather(*[send_notification(uid) for uid in user_ids])
```

### Anti-Pattern 3: Loading More Data Than Needed
```python
# BAD: Load all users to find admins
all_users = User.find_all()
admins = [u for u in all_users if u.role == 'admin']

# GOOD: Filter at the database
admins = User.find_all(role='admin')
```

### Anti-Pattern 4: String Concatenation in Loops
```python
# BAD: O(n²) string building
result = ""
for item in items:
    result += str(item) + ","  # Creates new string object each time

# GOOD: O(n) with join
result = ",".join(str(item) for item in items)
```

---

## 10. PERFORMANCE CHECKLIST

Before shipping performance-critical code:

**Database**
- [ ] N+1 queries eliminated (eager loading for all related data)
- [ ] All WHERE/JOIN columns have indexes
- [ ] LIMIT on all list queries
- [ ] Connection pooling configured
- [ ] Slow query log enabled

**Caching**
- [ ] Cache invalidation strategy defined
- [ ] TTL set appropriately for data change frequency
- [ ] Cache stampede protection for expensive operations

**API**
- [ ] Response compression enabled
- [ ] Unnecessary data excluded from responses
- [ ] Concurrent operations parallelized with Promise.all / asyncio.gather

**Code**
- [ ] No O(n²) algorithms for n>100
- [ ] No unnecessary object creation in hot paths
- [ ] Heavy work offloaded to background queues
- [ ] Memory streams used instead of full-file reads for large data

**Measurement**
- [ ] Performance targets defined (P50, P95, P99)
- [ ] Load testing done against targets
- [ ] Monitoring/alerting set up for performance regression
