# Debugging Protocol — Systematic Bug Elimination

Debugging is a science, not intuition. Follow this protocol. Do not skip steps because the bug seems obvious.
The obvious explanation is wrong more often than not.

---

## TABLE OF CONTENTS
1. The Debugging Mindset
2. Phase 1 — Reproduction
3. Phase 2 — Isolation
4. Phase 3 — Hypothesis Formation
5. Phase 4 — Verification (Not Fixing — Verifying)
6. Phase 5 — Root Cause Analysis
7. Phase 6 — Fix Implementation
8. Phase 7 — Regression Prevention
9. Common Bug Patterns by Category
10. Debugging by Symptom
11. Language-Specific Debugging Traps

---

## 1. THE DEBUGGING MINDSET

```
The bug is NOT where you think it is.
The bug is NOT what you think it is.
The bug is ALWAYS exactly what the code says it is — not what you intended it to say.
```

**The Three Laws of Debugging**

1. **The Symptom Lies**: What you observe is rarely where the bug is. An NPE on line 47 may be caused by an incorrect assignment on line 12.

2. **Assumptions Are The Enemy**: Every "I know that X is..." is a hypothesis, not a fact. Until you've printed/logged X and confirmed its value, you do not know what X is.

3. **The Bug Has Always Been There**: If "it suddenly broke," something changed. Find what changed. Don't debug the code; debug the change.

---

## 2. PHASE 1 — REPRODUCTION

**Before doing anything else, you must be able to reproduce the bug reliably.**

If you cannot reproduce it, you cannot verify when you've fixed it.

### Reproduction Checklist
- [ ] Can you trigger the bug on demand?
- [ ] What exact inputs/conditions trigger it?
- [ ] Does it happen every time, or only sometimes?
- [ ] If intermittent: what percentage of the time? Under what conditions?
- [ ] Does it happen in all environments or just one?

### Intermittent Bugs
These are the hardest. Possible causes:
- **Race conditions**: Depends on timing between async operations
- **State contamination**: Depends on previous operations (run tests in isolation)
- **Resource exhaustion**: Only fails under load (memory, file descriptors, connections)
- **Date/time sensitivity**: Fails at midnight, month boundary, DST change, leap year
- **External dependency flakiness**: Third-party API, network, or DNS failure

**For intermittent bugs**: Add logging to capture state at the moment of failure before attempting to fix anything.

---

## 3. PHASE 2 — ISOLATION

Reduce the failing case to the minimum reproducible example.

### Binary Search Debugging
If you have a large codebase and don't know where the bug is:
1. Comment out half the code path. Does the bug still occur?
2. If yes: the bug is in the remaining half. If no: the bug is in what you commented out.
3. Repeat until you've isolated to ~10 lines.

### Input Minimization
If the bug requires complex input:
1. Start removing fields from the input until the bug stops occurring
2. The last field you removed is either the cause or part of the cause
3. Now start adding fields back until the bug reappears

### Environment Isolation
- Does it fail in production but not dev? → Check environment variables, data differences, dependency versions
- Does it fail with a specific user? → Check that user's data, permissions, account state
- Does it fail on specific hardware? → Check architecture differences (32/64-bit, byte order, float precision)

---

## 4. PHASE 3 — HYPOTHESIS FORMATION

**Before looking at code**: Form a hypothesis about root cause.

Write it down: "I believe the bug is caused by _______ because _______."

### Hypothesis Quality Test
A good hypothesis is:
- **Specific**: "The session token is null when the user has been inactive for >30 minutes" ✓
- Not vague: "There's something wrong with session handling" ✗
- **Testable**: You can verify or disprove it with a specific check
- **Falsifiable**: You can name a test that would prove it wrong

### Common Hypothesis Sources
- **Recent changes**: What was deployed/changed right before the bug appeared?
- **Error messages**: Don't just read the last line. Read the full stack trace from the top.
- **Logs**: What happened in the 10 seconds before the error? Look for warnings.
- **Data patterns**: Does the bug only affect certain IDs, dates, or data formats?

---

## 5. PHASE 4 — VERIFICATION (NOT FIXING)

**Verify your hypothesis before writing a fix. This is the most commonly skipped step and the source of 80% of bug regressions.**

### Verification Methods

**Adding Diagnostic Logging**
```python
# Don't modify behavior. Just observe.
def process_payment(user_id, amount):
    print(f"DEBUG process_payment: user_id={user_id!r}, amount={amount!r}, type={type(amount)}")
    user = get_user(user_id)
    print(f"DEBUG process_payment: user={user!r}, user.balance={user.balance!r}")
    # ... rest of function
```

**Assertion Probes**
```python
def process_payment(user_id, amount):
    user = get_user(user_id)
    assert user is not None, f"Expected user for id={user_id}, got None"
    assert isinstance(amount, Decimal), f"Expected Decimal, got {type(amount)}: {amount!r}"
```

**Interactive Debugging (Breakpoints)**
- Set a breakpoint BEFORE the suspected error location
- Inspect actual values — don't assume they match what you passed in
- Check every variable's type AND value

### Proving Your Hypothesis
"My hypothesis is confirmed if I can show that [variable X has value Y at point Z]."
Check it. If the variable has a different value, your hypothesis is wrong. Form a new one.

---

## 6. PHASE 5 — ROOT CAUSE ANALYSIS

After confirming the immediate cause, ask **WHY** five times.

```
Bug: NullPointerException on user.getEmail()
Why 1: user is null
Why 2: getUser(id) returned null
Why 3: The user was deleted but the session wasn't invalidated
Why 4: The logout handler only invalidates sessions on the current device
Why 5: Session management was built without considering account deletion

Root cause: Account deletion doesn't invalidate all sessions.
Fix: The deletion handler must invalidate ALL sessions for that user.
```

**Fixing "Why 1" without fixing "Why 5" means the bug will recur in a different form.**

---

## 7. PHASE 6 — FIX IMPLEMENTATION

### Fix Principles

**Fix the Root Cause, Not the Symptom**
```python
# SYMPTOM FIX (wrong) — hides the bug
def process_payment(user_id, amount):
    user = get_user(user_id)
    if user is None:  # Just skip deleted users silently
        return None   # Caller won't know payment didn't happen!

# ROOT CAUSE FIX (right) — fix the session invalidation on account deletion
# AND add defensive assertion here
def process_payment(user_id, amount):
    user = get_user(user_id)
    if user is None:
        raise UserNotFoundError(f"Cannot process payment: user {user_id} not found")
```

**Minimal Fix First**
The fix should be as small as possible. Changing 3 lines is safer than changing 30.
If fixing this properly requires a large refactor, do the minimal safe fix first, then refactor separately.

**Preserve Existing Behavior**
Unless the bug IS the existing behavior, your fix should change only what's broken.
Don't "clean things up" while fixing a bug — that's how you introduce regressions.

---

## 8. PHASE 7 — REGRESSION PREVENTION

After fixing, prevent recurrence:

### Write a Test That Would Have Caught This Bug
```python
def test_payment_raises_when_user_deleted():
    """Regression test: payment for deleted user should raise, not silently succeed."""
    user_id = create_test_user()
    delete_user(user_id)
    
    with pytest.raises(UserNotFoundError):
        process_payment(user_id, Decimal("100.00"))
```

### Post-Bug Questions
1. **How did this get through review?** → What review process improvement prevents this?
2. **How long was this in production?** → What monitoring would have caught it sooner?
3. **Are there similar bugs elsewhere?** → Do a targeted search for the same pattern.
4. **What structural issue allowed this?** → Can you make this class of bug impossible?

---

## 9. COMMON BUG PATTERNS BY CATEGORY

### Off-By-One Errors
```
Symptoms: First/last element incorrect, loop runs n-1 or n+1 times, "index out of bounds"
Classic patterns:
  - for i in range(len(arr)) vs range(len(arr) - 1)
  - slice[0:n] includes index n-1, not n
  - Pagination: page 1 with size 10 → OFFSET 0, not OFFSET 10
  
Verify: Log the actual index values at the boundaries. Check: first iteration, last iteration, empty array.
```

### Null/Undefined Propagation
```
Symptoms: NPE/TypeError deep in a call chain, "Cannot read property X of null"
Pattern: data is set to null in one function, used 5 functions later
Verify: Add null checks at EVERY function boundary, not just the crash site
Fix: Validate at input, fail loudly at source, not silently at use site
```

### Async Race Conditions
```
Symptoms: Works in isolation, fails under load; "usually works but sometimes wrong"
Pattern: Two concurrent requests share state and interleave at the wrong point
Classic code:
  count = get_count()          // Both threads read 5
  count += 1                   // Both threads compute 6
  set_count(count)             // Both threads set 6 — result should be 7
Verify: Add logging with timestamps and thread/request IDs. Look for interleaving.
Fix: Use atomic operations, database transactions, or mutexes
```

### Type Coercion Bugs
```
Symptoms: Comparison that "should" work fails; values that "look equal" aren't
Pattern: JavaScript == vs ===, Python 2 int/float division, string vs int comparison
Verify: Log type(x) AND x for every comparison variable
Fix: Use strict equality everywhere; explicit type casts at input boundaries
```

### Cache Invalidation
```
Symptoms: Stale data shown after update; "it updated in the database but the UI still shows old value"
Pattern: Cache TTL too long; update invalidates one cache key but not all related keys
Verify: Bypass cache and check if fresh data is returned
Fix: On any update, identify ALL cache keys that might contain this data and invalidate them
```

### Memory Leaks
```
Symptoms: Memory grows over time; eventual OOM or slowdown over hours/days
Pattern: Event listeners not removed; circular references; closures holding large objects
Verify: Memory profiler over time; heap snapshot comparison
Common causes:
  - addEventListener without removeEventListener in React useEffect
  - Global caches without eviction
  - Closures in setInterval/setTimeout that hold references
```

---

## 10. DEBUGGING BY SYMPTOM

### "It works on my machine"
```
1. Check environment variables (esp. NODE_ENV, DATABASE_URL)
2. Check dependency versions (package-lock.json differences)
3. Check OS-specific behavior (file paths, line endings, case sensitivity)
4. Check available memory/disk
5. Check database data differences (missing seed data, schema differences)
```

### "It suddenly broke, I didn't change anything"
```
1. What actually changed? (check git log, deployment history, dependency updates)
2. Did an external dependency change? (API, library, timezone database)
3. Did the data change? (did a new data pattern reach production for the first time?)
4. Is it time-related? (date boundary, certificate expiry, token expiry)
5. Is it load-related? (traffic increase, database growth slowing queries)
```

### "It works sometimes"
→ See Phase 1 — Intermittent Bugs section above.

### "The error message makes no sense"
```
1. Read the FULL stack trace. The top of the trace is the symptom. The bottom is near the cause.
2. Search for the error message verbatim — exact error messages often have known causes
3. Check: what is the actual value of every variable mentioned?
4. Try reproducing with added verbose/debug logging
```

### "The test passes but production fails"
```
1. Are the test and production environments truly equivalent?
2. Is the test mocking something that behaves differently in production?
3. Is there a data difference? (production has edge cases tests don't cover)
4. Is there a timing difference? (tests run synchronously, production is concurrent)
5. Is there a scale difference? (tests use 10 items, production has 10M)
```

---

## 11. LANGUAGE-SPECIFIC DEBUGGING TRAPS

### JavaScript / TypeScript
```javascript
// Trap: typeof null === 'object' (historical JS bug)
if (typeof x === 'object') // WRONG: true for null!
if (x !== null && typeof x === 'object') // correct

// Trap: Array.sort() sorts lexicographically by default
[10, 9, 2].sort()          // → [10, 2, 9] !!!
[10, 9, 2].sort((a,b) => a-b)  // → [2, 9, 10]

// Trap: for...in iterates prototype chain
for (const key in obj) // includes inherited keys
Object.keys(obj).forEach(key => ...) // own keys only

// Trap: Floating point
0.1 + 0.2 === 0.3 // false
Math.abs(0.1 + 0.2 - 0.3) < 1e-10 // correct comparison
```

### Python
```python
# Trap: Mutable default arguments
def append_to(element, to=[]):  # THIS LIST IS SHARED ACROSS ALL CALLS
    to.append(element)
    return to
# Use: def append_to(element, to=None): if to is None: to = []

# Trap: Late-binding closures in loops
fns = [lambda: i for i in range(5)]
fns[0]()  # returns 4, not 0! i is captured by reference
# Fix: lambda i=i: i

# Trap: is vs ==
a = 256
b = 256
a is b  # True (CPython caches small integers)
a = 257
b = 257
a is b  # False! Use == for value comparison

# Trap: Integer division Python 2 vs 3
5 / 2 == 2   # Python 2
5 / 2 == 2.5 # Python 3
5 // 2 == 2  # Both (explicit integer division)
```

### Go
```go
// Trap: nil interface vs nil pointer
var p *MyStruct = nil
var i interface{} = p
i == nil  // FALSE! Interface is not nil if its type is set

// Trap: goroutine variable capture
for _, v := range items {
    go func() {
        process(v)  // v is shared — all goroutines see the same v
    }()
}
// Fix:
for _, v := range items {
    v := v  // shadow with local copy
    go func() { process(v) }()
}

// Trap: map is not safe for concurrent use
// Use sync.Map or protect with sync.RWMutex
```

### SQL
```sql
-- Trap: NULL comparisons
WHERE status != 'active'  -- MISSES rows where status IS NULL

WHERE status != 'active' OR status IS NULL  -- correct

-- Trap: COUNT(*) vs COUNT(column)
COUNT(*)        -- counts all rows including NULLs
COUNT(column)   -- counts non-NULL values only

-- Trap: NOT IN with NULLs
WHERE id NOT IN (1, 2, NULL)  -- returns NOTHING (NULL comparison is always UNKNOWN)
-- Use NOT EXISTS or check for NULLs explicitly
```
