# Code Beauty Standards — The Craft of Elegant Code

Beautiful code is not decoration. It is the clearest possible expression of intent.
Code that is hard to read is code that is hard to trust.

---

## TABLE OF CONTENTS
1. The Philosophy of Beautiful Code
2. Naming as Communication
3. Function Design
4. Structure & Organization
5. Comments & Documentation
6. Error Handling as UX
7. The Refactoring Mindset
8. Language-Specific Elegance
9. Code Review Checklist for Quality

---

## 1. THE PHILOSOPHY OF BEAUTIFUL CODE

```
Code is read 10x more than it is written.
Write for the reader, not the machine.
The machine doesn't care. Your teammates do.
You will be a stranger to your own code in 6 months.
Write for that stranger.
```

**The Three Properties of Beautiful Code**

1. **Clarity**: The intent is immediately obvious. No mental translation required.
2. **Simplicity**: The minimal solution that correctly solves the problem. Not simpler.
3. **Consistency**: Following the same patterns throughout. No surprises.

**The Single Most Important Principle**
> Each function, variable, class, and module does exactly one thing and communicates that thing perfectly in its name.

---

## 2. NAMING AS COMMUNICATION

### The Name Must Answer: What Is This? What Is It For?

If a name requires a comment to explain it, the name has failed.

**Variable Naming**
```python
# Bad — What is d? What are n? What is r?
d = datetime.now()
n = get_n()
r = n * 0.0825

# Good — Self-documenting
invoice_date = datetime.now()
invoice_subtotal = get_invoice_subtotal()
sales_tax_amount = invoice_subtotal * SALES_TAX_RATE
```

**Boolean Variable Names**
```python
# Bad — What does 'flag' mean?
flag = check_user(user)
if flag: ...

# Good — Reads like English
is_email_verified = user.email_verified_at is not None
has_active_subscription = user.subscription and user.subscription.is_active
can_post_comment = is_email_verified and has_active_subscription

if can_post_comment:
    allow_comment()
```

**Function Naming**
```python
# Commands (do something) — verb + noun
calculate_monthly_revenue(month, year)
send_password_reset_email(user)
invalidate_user_sessions(user_id)

# Queries (return something) — descriptive noun or is/has/get prefix
monthly_active_users(month)      # Returns a count
is_premium_subscriber(user)      # Returns bool
get_user_by_email(email)         # Returns User or None
find_overdue_invoices(cutoff_date)  # Returns list

# Avoid vague verbs: do, process, handle, manage, update, run, execute
# They communicate nothing about what actually happens
```

**Collection Naming**
```python
# Bad — What does 'data' contain?
data = get_data()
for item in data:
    process(item)

# Good — Noun describes the content
active_subscriptions = fetch_active_subscriptions()
for subscription in active_subscriptions:
    notify_upcoming_renewal(subscription)
```

**Parameter Naming**
```python
# Bad — What is 'x', what is 'y'?
def calculate(x, y, z=False):
    ...

# Good — Parameters document themselves
def calculate_discounted_price(original_price: Decimal, discount_rate: float, include_tax: bool = False) -> Decimal:
    ...
```

### Naming Anti-Patterns to Eliminate

| Anti-Pattern | Example | Why Bad | Fix |
|---|---|---|---|
| Generic nouns | data, info, item, obj, thing | Describes nothing | Name the domain concept |
| Index variables | i, j, k in nested loops | Lost in nesting | user_idx, row_idx |
| Abbreviations | usr, mgr, cfg, tmp | Saves 3 keystrokes, wastes 10 of reading | user, manager, config, temp |
| Type in name | userList, nameString | Redundant with type annotations | users, name |
| Negated booleans | isNotActive, notDisabled | Double negation in conditionals | isActive, isEnabled |
| Numbered series | data1, data2, data3 | No semantic distinction | user_data, product_data |
| Single letters | n, s, b (outside math) | Meaningless | count, status, buffer |

---

## 3. FUNCTION DESIGN

### The Single Responsibility Principle in Practice

```python
# BAD — This function does four things
def process_order(order_data):
    # Validate
    if not order_data.get('user_id'):
        raise ValueError("Missing user_id")
    
    # Calculate
    total = sum(item['price'] * item['qty'] for item in order_data['items'])
    tax = total * 0.0825
    
    # Save to DB
    order = db.execute("INSERT INTO orders ...", (order_data['user_id'], total + tax))
    
    # Send email
    send_email(order.user.email, "Order Confirmation", render_template('order_confirm', order))
    
    return order

# GOOD — Four responsibilities, four functions
def create_order(user_id: int, items: list[OrderItem]) -> Order:
    validate_order_items(items)
    pricing = calculate_order_pricing(items)
    order = persist_order(user_id, pricing)
    schedule_confirmation_email(order)
    return order

def validate_order_items(items: list[OrderItem]) -> None:
    if not items:
        raise InvalidOrderError("Order must contain at least one item")
    for item in items:
        if item.quantity <= 0:
            raise InvalidOrderError(f"Item {item.sku} has invalid quantity: {item.quantity}")

def calculate_order_pricing(items: list[OrderItem]) -> OrderPricing:
    subtotal = sum(item.price * item.quantity for item in items)
    tax = round(subtotal * SALES_TAX_RATE, 2)
    return OrderPricing(subtotal=subtotal, tax=tax, total=subtotal + tax)
```

### Function Length Guidelines
- **Ideal**: 5-15 lines
- **Acceptable**: Up to 30 lines
- **Needs splitting**: 30-50 lines
- **Wrong**: 50+ lines — this is multiple functions

### Command-Query Separation
A function either DOES something (command) or RETURNS something (query). Not both.

```python
# VIOLATION — Gets a value AND has a side effect
def get_and_log_user(user_id):
    user = db.find(user_id)
    log_access(user_id)  # Side effect!
    return user

# BETTER — Separate concerns; caller decides what to do
def get_user(user_id):
    return db.find(user_id)

# Caller:
user = get_user(user_id)
log_user_access(user_id)  # Explicit, caller-controlled
```

### The Principle of Least Surprise
A function should do exactly what its name implies — nothing more, nothing less.
If you can't name a function accurately, the function is wrong.

---

## 4. STRUCTURE & ORGANIZATION

### File Organization Principles

```
Within a file, order by dependency:
1. Imports (external, internal)
2. Constants / configuration
3. Types / interfaces / classes
4. Private helper functions
5. Public API functions / exports

Within a class, order by visibility and importance:
1. Public interface (what callers use)
2. Constructor
3. Public methods
4. Private methods
5. Private helpers
```

### Module Cohesion
Modules should have a clear, single-sentence purpose.
"This module handles user authentication" — cohesive.
"This module handles users and products and emails" — not a module, it's a dumping ground.

### The Dependency Rule
High-level modules should not depend on low-level modules.
Both should depend on abstractions.

```python
# BAD — Business logic depends on Stripe directly
class OrderService:
    def charge_customer(self, order):
        stripe.PaymentIntent.create(amount=order.total, currency='usd')

# GOOD — Business logic depends on an abstraction
class OrderService:
    def __init__(self, payment_provider: PaymentProvider):
        self.payment_provider = payment_provider
    
    def charge_customer(self, order):
        self.payment_provider.charge(amount=order.total, currency='usd')

# StripePaymentProvider implements PaymentProvider
# Easy to test with MockPaymentProvider, easy to switch providers
```

---

## 5. COMMENTS & DOCUMENTATION

### The Ratio Rule
**Code explains HOW. Comments explain WHY.**

If you feel the urge to comment WHAT the code does, that's a signal to rename something.

```python
# BAD — Comment restates the code
user_count += 1  # Increment user count

# BAD — Comment explains obvious logic
if age >= 18:  # Check if user is 18 or older
    allow_access()

# GOOD — Comment explains non-obvious WHY
# Multiply by 100 before rounding to preserve 2 decimal places
# without floating point precision errors
price_in_cents = round(price * 100)

# GOOD — Comment explains business rule context
# Free tier users are capped at 10 projects per BILLING CYCLE not per month.
# This aligns with how we calculate overage charges.
if user.project_count >= FREE_TIER_PROJECT_LIMIT:
    raise ProjectLimitExceeded()

# GOOD — Comment flags a future risk
# TODO(2024-12): Remove this once all users have migrated to v2 auth.
# Can safely delete when user_auth_version = 1 count reaches 0.
if user.auth_version == 1:
    apply_legacy_auth_rules(user)
```

### When Comments Are Required
- Non-obvious algorithms (include a reference if applicable)
- Business rules that aren't self-evident from the domain
- Workarounds for known library/language bugs (cite the issue)
- Performance optimizations that sacrifice readability
- TODOs with deadline and owner: `// TODO(alice, 2024-Q2): Replace with stream processing`

### Docstrings / JSDoc for Public APIs
Public functions, classes, and modules need documentation.
Internal helpers usually don't (names should be sufficient).

```python
def calculate_compound_interest(
    principal: Decimal,
    annual_rate: float,
    years: int,
    compounds_per_year: int = 12
) -> Decimal:
    """
    Calculate compound interest using the standard formula: A = P(1 + r/n)^(nt)
    
    Args:
        principal: Initial investment amount in the account's currency
        annual_rate: Annual interest rate as a decimal (e.g., 0.05 for 5%)
        years: Investment duration in years
        compounds_per_year: How many times interest compounds per year (default: monthly)
    
    Returns:
        Total amount after interest (principal + accumulated interest)
    
    Raises:
        ValueError: If principal or annual_rate is negative, or years <= 0
    
    Example:
        >>> calculate_compound_interest(Decimal('1000'), 0.05, 10)
        Decimal('1647.01')
    """
```

---

## 6. ERROR HANDLING AS UX

Error messages are user-facing communication. Write them with care.

### Error Message Quality

```python
# BAD — Useless error messages
raise ValueError("Invalid input")
raise Exception("Error occurred")
raise RuntimeError("Failed")

# GOOD — Actionable, specific error messages
raise ValueError(
    f"Invalid email format: '{email}'. Expected format: user@domain.com"
)
raise ResourceNotFoundError(
    f"User with ID {user_id} not found. "
    f"They may have been deleted or the ID may be incorrect."
)
raise RateLimitExceededError(
    f"Rate limit exceeded for endpoint {endpoint}. "
    f"Limit: {limit} requests per {window}s. "
    f"Reset at: {reset_time.isoformat()}"
)
```

### Error Types as Documentation

```python
# Define specific exception types — they ARE documentation
class InsufficientFundsError(PaymentError):
    """Raised when an account balance is insufficient for the requested operation."""
    def __init__(self, account_id: int, required: Decimal, available: Decimal):
        self.account_id = account_id
        self.required = required
        self.available = available
        super().__init__(
            f"Insufficient funds in account {account_id}: "
            f"required {required}, available {available}"
        )
```

### Error Handling Layers

```
Layer 1 — Input Validation (at the boundary):
    Reject bad input immediately with descriptive errors.
    Don't let invalid data into your system.

Layer 2 — Business Logic (at the decision point):
    Throw domain-specific exceptions for business rule violations.
    These are expected exceptional conditions, not bugs.

Layer 3 — Infrastructure (at I/O):
    Wrap infrastructure errors with context.
    "Database connection failed" → "Failed to load user profile: Database unavailable"

Layer 4 — API/UI (at the interface):
    Translate technical errors to user-facing messages.
    Never expose stack traces or internal details to end users.
```

---

## 7. THE REFACTORING MINDSET

### The Boy Scout Rule
Leave the code better than you found it.
Every file you touch should have at least one small improvement.

### Safe Refactoring Sequence
1. Write tests for current behavior (if they don't exist)
2. Make the smallest possible change
3. Verify tests still pass
4. Commit (separate commit from feature work)
5. Repeat

### When to Refactor

**Refactor when:**
- You're about to add a feature to complex code (simplify first)
- You're fixing a bug that was hard to find (make it easier next time)
- Code review reveals a systemic problem
- You've understood the domain better and the old abstraction is wrong

**Don't refactor when:**
- You're under deadline pressure
- You don't have test coverage for the code you're changing
- You're about to ship
- The code works and won't change

---

## 8. CODE REVIEW CHECKLIST FOR QUALITY

Use this when reviewing your own code before delivery:

**Naming**
- [ ] Every variable, function, and class name communicates intent without a comment?
- [ ] No generic names (data, item, result, temp, obj)?
- [ ] Boolean names are positive (isActive, not isNotDeactivated)?
- [ ] Function names are verb phrases (commands) or noun phrases (queries)?

**Functions**
- [ ] Each function does exactly one thing?
- [ ] No function is longer than 30 lines?
- [ ] No function has more than 4 parameters?
- [ ] Return types and error types are explicit?

**Structure**
- [ ] Related code is grouped together?
- [ ] Dependencies flow in one direction (no circular dependencies)?
- [ ] Public interface is minimal and well-defined?

**Comments**
- [ ] Comments explain WHY, not WHAT?
- [ ] No commented-out code?
- [ ] All public API has docstrings?
- [ ] All complex algorithms are explained?

**Consistency**
- [ ] Same conventions used throughout the file?
- [ ] Same conventions used as the rest of the codebase?
- [ ] No "just this once" style violations?

**Pride Test**
- [ ] Would you be proud to have written this?
- [ ] Would a junior developer understand this with minimal explanation?
- [ ] Would this code make a reviewer smile, not wince?
