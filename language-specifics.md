# Language-Specific Pitfalls & Idioms

The most dangerous bugs are language-specific traps that look correct at a glance.
This reference covers critical pitfalls for each major language.

---

## TABLE OF CONTENTS
1. Python
2. JavaScript / TypeScript
3. Go
4. Rust
5. SQL
6. React / Frontend
7. Shell / Bash

---

## 1. PYTHON

### Critical Traps

**Mutable Default Arguments — The Most Common Python Bug**
```python
# BUG — The list is created ONCE and shared across all calls
def add_item(item, collection=[]):
    collection.append(item)
    return collection

add_item("a")  # ['a']
add_item("b")  # ['a', 'b']  ← WRONG — should be ['b']

# FIX
def add_item(item, collection=None):
    if collection is None:
        collection = []
    collection.append(item)
    return collection
```

**Late-Binding Closures in Loops**
```python
# BUG — All lambdas capture the same 'i' variable (its final value)
functions = [lambda: i for i in range(5)]
[f() for f in functions]  # [4, 4, 4, 4, 4] — NOT [0, 1, 2, 3, 4]

# FIX — Default argument captures value at definition time
functions = [lambda i=i: i for i in range(5)]
[f() for f in functions]  # [0, 1, 2, 3, 4]
```

**is vs == (Identity vs Equality)**
```python
# is: Same object in memory (identity)
# ==: Same value (equality)

a = [1, 2, 3]
b = [1, 2, 3]
a == b   # True  (same value)
a is b   # False (different objects)

# Trap: CPython caches small integers (-5 to 256) and short strings
x = 256; y = 256; x is y  # True (cached)
x = 257; y = 257; x is y  # False (not cached) — never rely on this!

# ALWAYS use == for value comparison, is only for None/True/False
if x is None:   # Correct idiom for None check
if x == None:   # Works but non-idiomatic and slower
```

**Generator Exhaustion**
```python
# Generators can only be iterated ONCE
gen = (x for x in range(10))
list(gen)  # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
list(gen)  # []  ← EXHAUSTED!

# If you need multiple passes, use a list
items = list(x for x in range(10))
```

**Dictionary Mutation During Iteration**
```python
# BUG — RuntimeError: dictionary changed size during iteration
for key in d:
    if should_remove(key):
        del d[key]

# FIX — Iterate over a copy
for key in list(d.keys()):
    if should_remove(key):
        del d[key]
```

**Exception Handling Scope**
```python
# BUG — 'e' is deleted after except block in Python 3
try:
    risky()
except Exception as e:
    pass
print(e)  # NameError: 'e' is not defined!

# FIX — Save it if you need it outside
try:
    risky()
except Exception as e:
    saved_error = e
print(saved_error)
```

**Async Traps**
```python
# BUG — asyncio.run() inside a running event loop
async def main():
    asyncio.run(another_coro())  # RuntimeError!

# FIX — use await
async def main():
    await another_coro()

# BUG — Fire-and-forget without tracking
async def main():
    asyncio.create_task(background_work())  # Task may be garbage collected!

# FIX — Keep a reference
tasks = set()
task = asyncio.create_task(background_work())
tasks.add(task)
task.add_done_callback(tasks.discard)
```

### Python Best Practices
- Use `dataclasses` or `pydantic` for structured data — never plain dicts for complex objects
- Use `pathlib.Path` instead of `os.path` string manipulation
- Use `contextlib.suppress` for known, ignorable exceptions (not bare `except: pass`)
- Type annotations are documentation AND enable mypy static analysis — use them everywhere
- Use `__slots__` for classes that create millions of instances (memory optimization)

---

## 2. JAVASCRIPT / TYPESCRIPT

### Critical Traps

**`this` Context Loss**
```javascript
class Timer {
    count = 0
    
    start() {
        // BUG: 'this' is undefined or window inside setTimeout callback
        setTimeout(function() {
            this.count++ // TypeError: Cannot read property 'count' of undefined
        }, 1000)
    }
    
    // FIX 1: Arrow function captures lexical this
    startFixed() {
        setTimeout(() => {
            this.count++ // Correct
        }, 1000)
    }
    
    // FIX 2: Bind
    startBound() {
        setTimeout(function() {
            this.count++
        }.bind(this), 1000)
    }
}
```

**Array Method Confusion**
```javascript
// find vs filter — different return types
const users = [{ id: 1 }, { id: 2 }]
users.find(u => u.id === 1)    // { id: 1 } or undefined
users.filter(u => u.id === 1)  // [{ id: 1 }] or []

// findIndex returns -1 on miss (not undefined/null/false)
const idx = arr.findIndex(x => x > 10)
if (idx !== -1) { /* found */ }  // Correct
if (idx) { /* found */ }         // BUG: idx 0 is falsy!

// sort mutates the original array!
const original = [3, 1, 2]
const sorted = original.sort()  // original is now [1, 2, 3] too!
// FIX: const sorted = [...original].sort()
```

**Event Loop & Promise Timing**
```javascript
// Promise callbacks run in microtask queue — before setTimeout
console.log('1')
setTimeout(() => console.log('2'), 0)
Promise.resolve().then(() => console.log('3'))
console.log('4')
// Output: 1, 4, 3, 2

// await doesn't make code synchronous — it suspends the async function
async function fetchUser(id) {
    const user = await db.getUser(id)  // Suspends here; other code runs
    return user
}
```

**Falsy Value Pitfalls**
```javascript
// JavaScript falsy values: false, 0, '', null, undefined, NaN
// This causes bugs with legitimate 0 values
const count = getCount()
if (count) { show(count) }  // BUG: count of 0 is falsy — won't show!
if (count !== undefined && count !== null) { show(count) }  // Correct
if (count != null) { show(count) }  // Shorthand: != null catches both null and undefined
```

**TypeScript `any` Escape Hatch**
```typescript
// 'any' defeats TypeScript's entire purpose
function process(data: any) {
    data.nonexistentMethod() // No error at compile time, crashes at runtime
}

// Use 'unknown' for truly unknown types — forces you to narrow before use
function process(data: unknown) {
    if (typeof data === 'string') {
        data.toUpperCase() // TypeScript knows it's string here
    }
}
```

**Async/Await Error Handling**
```javascript
// BUG — No error handling
const data = await fetchData()  // Throws → uncaught rejection

// BUG — try/catch doesn't catch errors thrown inside callbacks
try {
    someArray.forEach(async (item) => {
        await processItem(item)  // Error here escapes the try/catch!
    })
} catch (e) { /* never caught! */ }

// FIX — Use Promise.all with map for async iteration
try {
    await Promise.all(someArray.map(item => processItem(item)))
} catch (e) { /* properly caught */ }
```

### TypeScript Specifics

```typescript
// Non-null assertion (!) — only use when you're 100% certain and can explain why
const element = document.getElementById('app')!  // OK if you know the HTML
// Not: const value = map.get(key)!  // map.get returns undefined for missing keys!

// Type vs Interface — prefer interface for object shapes, type for unions/primitives
type Status = 'active' | 'inactive' | 'pending'  // Use type
interface User { id: number; name: string }       // Use interface

// Generic constraints
function first<T extends { length: number }>(arr: T): T[0] | undefined {
    return arr.length > 0 ? arr[0] : undefined
}

// Discriminated unions — the right way to model variants
type Result<T> =
    | { success: true; data: T }
    | { success: false; error: Error }

function handle(result: Result<User>) {
    if (result.success) {
        result.data  // TypeScript knows this is User
    } else {
        result.error // TypeScript knows this is Error
    }
}
```

---

## 3. GO

### Critical Traps

**nil Interface vs nil Pointer**
```go
// This is Go's most confusing gotcha
type MyError struct { msg string }
func (e *MyError) Error() string { return e.msg }

func getError() error {
    var err *MyError = nil  // nil pointer
    return err              // Returns non-nil interface! Interface has type *MyError, value nil
}

err := getError()
if err != nil {  // TRUE — the interface is not nil!
    log.Fatal(err)  // logs: <nil>
}

// FIX: Return untyped nil for errors
func getError() error {
    // ... 
    return nil  // Returns truly nil interface
}
```

**Goroutine Variable Capture in Loops**
```go
// BUG — All goroutines share the same 'v' variable
for _, v := range items {
    go func() {
        process(v)  // v has changed by the time this runs!
    }()
}

// FIX — Shadow with local variable
for _, v := range items {
    v := v  // Local copy
    go func() {
        process(v)  // Uses local copy
    }()
}

// OR: Pass as parameter
for _, v := range items {
    go func(v Item) {
        process(v)
    }(v)
}
```

**Slice Header vs Underlying Array**
```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]  // b is [2, 3], shares underlying array with a!
b[0] = 99
fmt.Println(a)  // [1, 99, 3, 4, 5] — a was modified!

// FIX: Copy if you don't want to share
b := make([]int, 2)
copy(b, a[1:3])
```

**Context Cancellation**
```go
// ALWAYS propagate context for cancellation and timeout
// BAD: Ignores context
func fetchUser(id string) (*User, error) {
    return db.QueryRow("SELECT * FROM users WHERE id = ?", id)
}

// GOOD: Context-aware
func fetchUser(ctx context.Context, id string) (*User, error) {
    return db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = ?", id)
}
```

**Error Wrapping**
```go
// BAD: Loses context
if err != nil {
    return err
}

// GOOD: Wrap with context
if err != nil {
    return fmt.Errorf("fetchUser(%s): %w", id, err)
}

// Check wrapped errors with errors.Is / errors.As
if errors.Is(err, sql.ErrNoRows) { ... }
```

**Map Concurrency**
```go
// BUG — Maps are NOT safe for concurrent use
var cache = map[string]string{}

func set(key, val string) {
    cache[key] = val  // Data race if called concurrently
}

// FIX 1: sync.RWMutex
var (
    mu    sync.RWMutex
    cache = map[string]string{}
)

func set(key, val string) {
    mu.Lock()
    defer mu.Unlock()
    cache[key] = val
}

// FIX 2: sync.Map (for high-contention cases)
var cache sync.Map
cache.Store(key, val)
val, ok := cache.Load(key)
```

---

## 4. RUST

### Ownership & Borrowing

**The Borrow Checker Is Right**
If the borrow checker rejects your code, your code has a bug in a higher-level language.
Don't fight it with `unsafe` — understand why it's rejecting and fix the design.

**Common Ownership Mistakes**
```rust
// Moving out of a loop variable
let strings = vec!["a".to_string(), "b".to_string()];
for s in strings {
    println!("{}", s);  // s is moved each iteration
}
// strings is now moved — can't use it after

// FIX: Iterate by reference
for s in &strings {
    println!("{}", s);
}

// String vs &str — know when to use each
fn takes_string(s: String) { }    // Takes ownership — use when you need to store
fn borrows_string(s: &str) { }    // Borrows — use for read-only operations
fn returns_owned() -> String { }  // Returns owned — use when creating new data
```

**Unwrap in Production Code**
```rust
// BUG — Panics in production on None or Err
let value = map.get("key").unwrap();
let num: i32 = "abc".parse().unwrap();

// GOOD — Explicit error handling
let value = map.get("key").ok_or(AppError::KeyNotFound)?;
let num: i32 = "abc".parse().map_err(|e| AppError::ParseError(e))?;
```

---

## 5. SQL

### Correctness Traps

**NULL Semantics**
```sql
-- NULL is UNKNOWN, not false
-- Any comparison with NULL yields UNKNOWN
SELECT * FROM users WHERE name != 'Alice'
-- This EXCLUDES rows where name IS NULL — usually not intended!
SELECT * FROM users WHERE name != 'Alice' OR name IS NULL

-- Aggregate functions ignore NULLs
SELECT AVG(score) FROM results  -- Ignores NULL scores, not treating as 0
SELECT AVG(COALESCE(score, 0)) FROM results  -- Treats NULL as 0

-- COUNT(*) vs COUNT(column)
COUNT(*)           -- Counts ALL rows including those with NULL
COUNT(column_name) -- Counts only non-NULL values in that column
```

**NOT IN with NULLs**
```sql
-- DANGEROUS: NOT IN returns nothing when subquery contains NULL!
SELECT * FROM orders WHERE user_id NOT IN (SELECT id FROM banned_users)
-- If banned_users has ANY row with id IS NULL, this returns ZERO rows!

-- SAFE: Use NOT EXISTS
SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM banned_users b WHERE b.id = o.user_id
)
```

**Transaction Isolation**
```sql
-- READ COMMITTED (default in PostgreSQL): Can read another transaction's committed changes
-- This enables non-repeatable reads — same SELECT returns different results in one transaction

-- For financial operations, use SERIALIZABLE isolation:
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT balance FROM accounts WHERE id = ?;
UPDATE accounts SET balance = balance - ? WHERE id = ?;
COMMIT;
```

**Implicit Type Conversion**
```sql
-- PostgreSQL might coerce types and bypass indexes
WHERE id = '123'        -- id is integer — coercion happens, index may not be used
WHERE id = 123          -- explicit integer — index used
WHERE created_at = NOW() -- type mismatch if column is DATE not TIMESTAMP
```

---

## 6. REACT / FRONTEND

**Stale Closures in useEffect**
```javascript
// BUG — count is stale (always 0)
function Counter() {
    const [count, setCount] = useState(0)
    
    useEffect(() => {
        const interval = setInterval(() => {
            setCount(count + 1)  // count is always 0 from initial render
        }, 1000)
        return () => clearInterval(interval)
    }, [])  // count not in deps — stale!
    
    // FIX 1: Add count to dependency array
    // FIX 2: Use functional update form
    useEffect(() => {
        const interval = setInterval(() => {
            setCount(prev => prev + 1)  // Always uses latest value
        }, 1000)
        return () => clearInterval(interval)
    }, [])
}
```

**Object/Array Equality in deps**
```javascript
// BUG — New array on every render causes infinite re-render
function Component({ userId }) {
    const options = { userId, limit: 10 }  // New object every render!
    
    useEffect(() => {
        fetchData(options)
    }, [options])  // Always "different" — infinite loop!
    
    // FIX: Memoize or use primitive deps
    useEffect(() => {
        fetchData({ userId, limit: 10 })
    }, [userId])  // Only re-run when userId changes
}
```

**Key Prop in Lists**
```javascript
// BUG — Using index as key causes subtle bugs on reorder/delete
items.map((item, index) => <Item key={index} {...item} />)

// CORRECT — Use stable unique ID
items.map(item => <Item key={item.id} {...item} />)
```

---

## 7. SHELL / BASH

**Unquoted Variables**
```bash
# BUG — Word splitting and glob expansion
FILE="my file with spaces.txt"
cat $FILE       # Treated as: cat my file with spaces.txt (4 arguments!)
cat "$FILE"     # Correct: cat "my file with spaces.txt"

# Always quote variables: "$VAR" not $VAR
```

**Error Handling**
```bash
#!/usr/bin/env bash
set -euo pipefail
# -e: Exit on error
# -u: Error on unset variables
# -o pipefail: Pipeline fails if any command fails (not just last)

# Without this, errors are silently swallowed
```

**Command Substitution**
```bash
# Old style (nesting is messy)
result=`command`

# Modern style (nestable, clear)
result=$(command)

# Preserve newlines
output="$(command)"  # Quoted — preserves newlines
output=$(command)    # Unquoted — strips trailing newlines
```

**Arithmetic**
```bash
# BUG — No arithmetic in [[ ]] without (( ))
if [[ $a > $b ]]; then  # String comparison! "9" > "10" is true (lexicographic)
if (( a > b )); then    # Arithmetic comparison — correct
```
