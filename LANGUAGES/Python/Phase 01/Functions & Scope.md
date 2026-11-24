In Python, **functions** are reusable blocks of code designed to perform a specific task, while **scope** refers to the textual region of a program where a specific variable is accessible. Understanding how variables interact within and outside of functions prevents unexpected bugs like data leaking or naming collisions.

**Without functions:**
```python
# Imagine an API with 50 endpoints
# Each endpoint needs to validate a user token
# Without functions, you copy-paste this 50 times:

token = request.headers.get("Authorization")
if not token:
    return {"error": "No token"}
if not token.startswith("Bearer "):
    return {"error": "Invalid format"}
jwt_token = token[7:]
# ... validate jwt ...
# ... 50 times across your codebase ...

# One day the validation logic changes.
# You must update 50 places.
# You miss 3 of them.
# Security vulnerability. You're fired.
```

**With functions:**
```python
# Write ONCE
def validate_token(token):
    if not token:
        return None
    if not token.startswith("Bearer "):
        return None
    return token[7:]

# Use everywhere
token = validate_token(request.headers.get("Authorization"))

# Logic changes? Update ONE place. Done.
```
**Functions are the foundation of maintainable code.** Watch [this](https://youtu.be/89cGQjB5R4M) for better understanding. Also [this](https://youtu.be/u-OmVr_fT4s).

---
## Setup
```bash
cd ~/projects
mkdir functions_scope
cd functions_scope
python3 -m venv venv
source venv/bin/activate
touch functions.py
code .
```

Or, with Poetry:
```bash
cd ~/projects
poetry new project_folder_name
cd project_folder_name
poetry config virtualenvs.in-project true
poetry install
source <(poetry env activate)
touch functions.py
```

---
## 1. Defining Functions
In Python, you define a function using the **`def` keyword**, followed by the **function name**, **parentheses `()`**, and a **colon `:`**. The code block inside the function must be indented.

```python title:functions.py
#   keyword  name      parameters
#     ↓       ↓         ↓
def greet(name, greeting):
    """This is a docstring — describes what the function does."""
    message = f"{greeting}, {name}!"
    return message          # ← return value
#   ↑
# sends value back to caller

# Calling the function
result = greet("Alice", "Hello")
print(result)   # Hello, Alice!

# You can also print directly
print(greet("Bob", "Hi"))   # Hi, Bob!
```

### Functions Without Return
```python
# If no return statement → returns None
def log_request(method, path, status):
    print(f"[{status}] {method} {path}")
    # no return statement

result = log_request("GET", "/users", 200)
print(result)   # None

# Functions that do something vs return something
# log_request → DOES something (side effect: prints)
# greet       → RETURNS something (produces a value)
# Both are valid — depends on what you need
```

### Return Multiple Values
```python
# Python functions can return multiple values
# They're packed as a TUPLE

def get_user_info(user_id):
    # Imagine this queries a database
    name = "Alice Smith"
    email = "alice@example.com"
    role = "admin"
    return name, email, role   # returns tuple (name, email, role)

# Unpack the tuple
name, email, role = get_user_info(1)
print(name)     # Alice Smith
print(email)    # alice@example.com
print(role)     # admin

# Or keep as tuple
info = get_user_info(1)
print(info)         # ('Alice Smith', 'alice@example.com', 'admin')
print(info[0])      # Alice Smith

# Real backend pattern:
def parse_auth_header(header):
    """Returns (token, error) — either token or error will be None"""
    if not header:
        return None, "No authorization header"
    if not header.startswith("Bearer "):
        return None, "Invalid token format"
    token = header[7:]
    if len(token) < 10:
        return None, "Token too short"
    return token, None   # success

token, error = parse_auth_header("Bearer eyJhbGc...")
if error:
    print(f"Auth failed: {error}")
else:
    print(f"Token: {token}")
```

---
## 2. Parameters & Arguments
In [Python](https://www.python.org/), the terms **parameters** and **arguments** are closely related and often used interchangeably, but they refer to two distinct stages of a function.

### The Difference
**Parameter** = variable in the function DEFINITION.
**Argument**  = actual value you PASS when calling.
```python
def greet(name):    # "name" is a PARAMETER
    return f"Hello, {name}"

greet("Alice")      # "Alice" is an ARGUMENT
```

### Default Arguments
Parameters with default values. Defaults must come AFTER non-default parameters.
```python
def create_user(name, email, role="user", is_active=True):
    return {
        "name": name,
        "email": email,
        "role": role,
        "is_active": is_active
    }

# Call with all arguments
user1 = create_user("Alice", "alice@test.com", "admin", True)

# Call with defaults (role="user", is_active=True)
user2 = create_user("Bob", "bob@test.com")

# Mix — provide some defaults
user3 = create_user("Charlie", "charlie@test.com", "moderator")

print(user1)
# {'name': 'Alice', 'email': 'alice@test.com', 'role': 'admin', 'is_active': True}

print(user2)
# {'name': 'Bob', 'email': 'bob@test.com', 'role': 'user', 'is_active': True}

print(user3)
# {'name': 'Charlie', 'email': 'charlie@test.com', 'role': 'moderator', 'is_active': True}
```

### The Mutable Default Argument Trap
ONE OF THE MOST COMMON PYTHON BUGS, NEVER use mutable objects as defaults.
```python
# WRONG — BUG!
def add_tag(tag, tags=[]):      # this list is created ONCE
    tags.append(tag)            # and shared across ALL calls
    return tags

print(add_tag("python"))    # ['python']
print(add_tag("fastapi"))   # ['python', 'fastapi'] ← bug!
print(add_tag("redis"))     # ['python', 'fastapi', 'redis'] ← bug!
```
Why? The default `[]` is created once when the function is DEFINED, not each time it's CALLED. All calls share the same list object.

```python
# CORRECT — use None as default
def add_tag(tag, tags=None):
    if tags is None:
        tags = []           # create NEW list each call
    tags.append(tag)
    return tags

print(add_tag("python"))    # ['python']
print(add_tag("fastapi"))   # ['fastapi'] ← correct!
print(add_tag("redis"))     # ['redis']   ← correct!
```
Same rule for dicts, sets, lists as defaults, Always use None and create inside the function.

### Keyword Arguments
Keyword arguments make code READABLE. You know what each value means without looking at the function.
```python
def create_api_response(data, status_code, message, success):
    return {
        "success": success,
        "status_code": status_code,
        "message": message,
        "data": data
    }

# Positional - ORDER matters
response = create_api_response({"id": 1}, 200, "OK", True)

# Keyword - ORDER doesn't matter
response = create_api_response(
    data={"id": 1},
    status_code=200,
    message="OK",
    success=True
)

# Mix positional and keyword
# Positional args MUST come before keyword args
response = create_api_response(
    {"id": 1},          # positional
    200,                # positional
    message="OK",       # keyword
    success=True        # keyword
)
```

---
## 3. `*args` and `**kwargs`
In Python, **`*args` and `**kwargs`** allow a function to accept an **arbitrary number of arguments**. This makes your functions flexible and capable of handling varying inputs without breaking the function signature.
Watch [this](https://youtu.be/Vh__2V2tXUM) for better understanding.
### `*args` - Variable Positional Arguments
When you don't know how many arguments will be passed. `*args` collects them all into a TUPLE.
```python
def add_numbers(*args):
    print(type(args))   # <class 'tuple'>
    print(args)
    return sum(args)

print(add_numbers(1, 2))            # 3
print(add_numbers(1, 2, 3, 4, 5))   # 15
print(add_numbers())                 # 0

# Mix regular and *args
# Regular params BEFORE *args
def log(*messages, level="INFO"):
    for msg in messages:
        print(f"[{level}] {msg}")

log("Server started")
log("Request received", "Processing...", "Done", level="DEBUG")

# Real backend use:
def bulk_create_users(*user_data):
    """Create multiple users at once"""
    created = []
    for data in user_data:
        # imagine saving to database
        created.append({"id": len(created) + 1, **data})
    return created

users = bulk_create_users(
    {"name": "Alice", "email": "alice@test.com"},
    {"name": "Bob", "email": "bob@test.com"},
    {"name": "Charlie", "email": "charlie@test.com"},
)
print(len(users))   # 3
```

### `**kwargs` - Variable Keyword Arguments
`**kwargs` collects keyword arguments into a DICT.
```python
def create_profile(**kwargs):
    print(type(kwargs))     # <class 'dict'>
    print(kwargs)
    return kwargs

profile = create_profile(
    name="Alice",
    email="alice@test.com",
    age=25,
    city="New York"
)
# {'name': 'Alice', 'email': 'alice@test.com', 'age': 25, 'city': 'New York'}

# Real backend use:
def build_query_filters(**filters):
    """Build SQL-like filter string from kwargs"""
    if not filters:
        return "No filters"

    conditions = []
    for column, value in filters.items():
        conditions.append(f"{column} = '{value}'")

    return "WHERE " + " AND ".join(conditions)

query = build_query_filters(role="admin", is_active=True, country="USA")
print(query)
# WHERE role = 'admin' AND is_active = 'True' AND country = 'USA'
```

### Combining Everything
**Order MUST be:**
`regular → *args → keyword-with-defaults → **kwargs`

```python
def ultimate_function(required, *args, keyword_only="default", **kwargs):
    print(f"required: {required}")
    print(f"args: {args}")
    print(f"keyword_only: {keyword_only}")
    print(f"kwargs: {kwargs}")

ultimate_function(
    "hello",           # required
    1, 2, 3,           # *args
    keyword_only="hi", # keyword only
    name="Alice",      # **kwargs
    age=25             # **kwargs
)
# required: hello
# args: (1, 2, 3)
# keyword_only: hi
# kwargs: {'name': 'Alice', 'age': 25}

# Real world backend function:
def paginate_query(
    queryset,           # required: what to paginate
    *filters,           # extra filter conditions
    page=1,             # keyword: page number
    page_size=10,       # keyword: items per page
    **sort_options      # extra sort options
):
    start = (page - 1) * page_size
    end = start + page_size

    print(f"Page {page}, size {page_size}")
    print(f"Filters: {filters}")
    print(f"Sort options: {sort_options}")

    return queryset[start:end]
```

---
## 4. Scope - The LEGB Rule
Scope = where a variable is visible/accessible. Watch [this](https://youtu.be/38uGbVYICwg).
Python has 4 levels of scope:
```
L — Local      (inside the current function)
E — Enclosing  (inside outer function, for nested functions)
G — Global     (module level — top of the file)
B — Built-in   (Python's own names: print, len, range...)
```
Python looks for variables in this order: `L → E → G → B`

### Local Scope
```python
def calculate_tax(price):
    # tax_rate is LOCAL — only exists inside this function
    tax_rate = 0.08
    tax = price * tax_rate
    return tax

print(calculate_tax(100))   # 8.0

# print(tax_rate)  ← NameError: tax_rate not defined
# print(tax)       ← NameError: tax not defined
# Local variables DIE when the function ends
```

### Global Scope
```python
# Global variable — defined at module (file) level
APP_NAME = "MyBackendApp"
DEBUG = True
MAX_CONNECTIONS = 100

def show_config():
    # Can READ global variables from inside a function
    print(f"App: {APP_NAME}")
    print(f"Debug: {DEBUG}")

show_config()

# But CANNOT modify global without declaring it
count = 0

def increment():
    # count = count + 1  ← UnboundLocalError!
    # Python sees assignment → thinks count is local
    # But local count doesn't exist yet → error
    pass

# To modify a global variable (use rarely):
def increment():
    global count        # declare intent to modify global
    count = count + 1

increment()
increment()
print(count)    # 2

# ⚠️ Avoid global variables in real code
# They make code hard to test and reason about
# Use function parameters and return values instead
# Constants (ALL_CAPS) as globals are fine
```

### Enclosing Scope (Closures Preview)
```python
# When a function is defined INSIDE another function
# the inner function can access the outer function's variables

def outer_function(message):
    # 'message' is in the ENCLOSING scope for inner_function

    def inner_function():
        # Accessing 'message' from enclosing scope
        print(f"Message: {message}")

    inner_function()    # call inner from outer

outer_function("Hello from outer")
# Message: Hello from outer

# inner_function has access to message even though
# message is not local to inner_function
# This is called a CLOSURE — more in section 1.2.5
```

### Built-in Scope
Python's built-in names are always available - `len, print, range, type, int, str, list, dict` etc.
```python
print(len("hello"))     # 5  — len is built-in
print(type(42))         # <class 'int'>  — type is built-in
print(range(5))         # range(0, 5)  — range is built-in

# Don't shadow built-ins by using their names as variables
# This is a common beginner mistake
```

```python
# BAD
list = [1, 2, 3]        # you just overwrote the built-in list!
print(list([4, 5, 6]))  # TypeError — list is now your variable, not the type
```

```python
# GOOD
my_list = [1, 2, 3]
numbers_list = [1, 2, 3]
# Never name variables: list, dict, set, type, id, input, print, etc.
```

### LEGB in Action
```python
x = "global"        # Global scope

def outer():
    x = "enclosing" # Enclosing scope

    def inner():
        x = "local" # Local scope
        print(x)    # L: local → found! prints "local"

    inner()
    print(x)        # L: not here → E: enclosing → found! prints "enclosing"

outer()
print(x)            # L: not here → E: not here → G: global → found! prints "global"

# Output:
# local
# enclosing
# global
```

---
## 5. Closures
A closure is a function that REMEMBERS the variables from its enclosing scope, even after that scope is gone.
Like a backpack - the inner function carries variables from the outer function wherever it goes. Watch [this](https://youtu.be/tNSOaA1z6Uo).
```python
def make_multiplier(factor):
    # 'factor' lives in make_multiplier's scope

    def multiply(number):
        # multiply CLOSES OVER 'factor'
        # it remembers 'factor' even after make_multiplier returns
        return number * factor

    return multiply     # return the FUNCTION, not the result

# Create specialized multipliers
double = make_multiplier(2)     # factor=2 is "closed over"
triple = make_multiplier(3)     # factor=3 is "closed over"
times_ten = make_multiplier(10) # factor=10 is "closed over"

print(double(5))        # 10  (5 * 2)
print(triple(5))        # 15  (5 * 3)
print(times_ten(5))     # 50  (5 * 10)

# make_multiplier has FINISHED running but 'factor' is still alive inside double, triple, times_ten
```

### Real Backend Use Cases
```python
# Use case 1: Rate limiter factory
def make_rate_limiter(max_calls_per_minute):
    calls = []  # closed over — remembered between calls

    def check_rate_limit(user_id):
        import time
        now = time.time()
        # Remove calls older than 1 minute
        recent_calls = [t for t in calls if now - t < 60]
        calls.clear()
        calls.extend(recent_calls)

        if len(calls) >= max_calls_per_minute:
            return False    # rate limit exceeded

        calls.append(now)
        return True         # allowed

    return check_rate_limit

# Create different limiters for different endpoints
api_limiter = make_rate_limiter(100)    # 100 calls/minute
auth_limiter = make_rate_limiter(5)     # 5 calls/minute (stricter)

print(api_limiter("user_123"))   # True
print(auth_limiter("user_123"))  # True

# Use case 2: Configuration-based validator
def make_validator(min_length, max_length, required=True):

    def validate(value, field_name="field"):
        if required and not value:
            return f"{field_name} is required"
        if value and len(value) < min_length:
            return f"{field_name} must be at least {min_length} characters"
        if value and len(value) > max_length:
            return f"{field_name} must be at most {max_length} characters"
        return None     # None = valid

    return validate

# Create specific validators
validate_username = make_validator(3, 30)
validate_password = make_validator(8, 100)
validate_bio = make_validator(0, 500, required=False)

print(validate_username("ab"))              # username must be at least 3 characters
print(validate_username("alice"))           # None (valid)
print(validate_password("short"))           # password must be at least 8 characters
print(validate_password("SecurePass123!")) # None (valid)
```

### Inspecting Closures
```python
def make_counter(start=0):
    count = start

    def increment(by=1):
        nonlocal count      # tell Python: modify enclosing 'count'
        count += by
        return count

    return increment

counter = make_counter(10)
print(counter())    # 11
print(counter())    # 12
print(counter(5))   # 17

# nonlocal vs global:
# global  → modifies module-level variable
# nonlocal → modifies enclosing function's variable
```

---
## 6. Lambda Functions
Lambda = anonymous (no-name) function written in ONE line for SIMPLE operations. Watch [this](https://youtu.be/IljPHDyBRog).

**Syntax:** `lambda parameters: expression`
```python
# Regular function
def square(x):
    return x ** 2

# Same thing as lambda
square = lambda x: x ** 2

print(square(5))    # 25

# Multiple parameters
add = lambda x, y: x + y
print(add(3, 4))    # 7

# With condition
classify = lambda x: "even" if x % 2 == 0 else "odd"
print(classify(4))  # even
print(classify(7))  # odd
```

### When to Use Lambdas
Lambdas shine as ONE-TIME functions passed as arguments to other functions.
```python
users = [
    {"name": "Charlie", "age": 30, "score": 85},
    {"name": "Alice", "age": 25, "score": 92},
    {"name": "Bob", "age": 35, "score": 78},
    {"name": "Dave", "age": 28, "score": 92},
]

# Sort by age
by_age = sorted(users, key=lambda u: u["age"])
print([u["name"] for u in by_age])
# ['Alice', 'Dave', 'Charlie', 'Bob']

# Sort by score descending
by_score = sorted(users, key=lambda u: u["score"], reverse=True)
print([u["name"] for u in by_score])
# ['Alice', 'Dave', 'Charlie', 'Bob']

# Sort by multiple fields (score desc, then name asc)
by_multi = sorted(users, key=lambda u: (-u["score"], u["name"]))
print([u["name"] for u in by_multi])
# ['Alice', 'Dave', 'Charlie', 'Bob']

# Real backend use:
api_endpoints = [
    {"path": "/users", "method": "GET", "response_time": 45},
    {"path": "/posts", "method": "POST", "response_time": 120},
    {"path": "/auth/login", "method": "POST", "response_time": 30},
    {"path": "/products", "method": "GET", "response_time": 85},
]

# Find slowest endpoints
slowest = sorted(api_endpoints, key=lambda e: e["response_time"], reverse=True)
print(slowest[0])
# {'path': '/posts', 'method': 'POST', 'response_time': 120}
```

### Lambda Limitations
Lambdas can only have ONE expression
They cannot have:
- Multiple lines
- Statements `(if/else as statement, for, while)`
- `return` keyword (it's implicit)

**This doesn't work:**
```python
process = lambda x:
    y = x * 2    # can't have statements
    return y
```

**Use a regular function for anything complex:**
```python
def process(x):
    y = x * 2
    return y
```
**Rule:** if lambda feels complicated → use `def` instead.

---
## 7. Higher-Order Functions
A higher-order function either: Takes a function as an argument, OR Returns a function as output (like closures). Watch [this](https://youtu.be/xZtTIm3fpfA).

Functions are "first-class" in Python:
- Can be stored in variables
- Can be passed as arguments
- Can be returned from functions

```python
# Storing functions in variables
def greet(name):
    return f"Hello, {name}!"

say_hello = greet           # no () — not calling, just referencing
print(say_hello("Alice"))   # Hello, Alice!
print(say_hello)            # <function greet at 0x...>

# Storing in data structures
operations = {
    "add": lambda x, y: x + y,
    "subtract": lambda x, y: x - y,
    "multiply": lambda x, y: x * y,
}

result = operations["add"](10, 5)
print(result)   # 15

# Real backend use — route dispatch:
def handle_get(request):
    return {"method": "GET", "data": []}

def handle_post(request):
    return {"method": "POST", "created": True}

def handle_delete(request):
    return {"method": "DELETE", "deleted": True}

routes = {
    "GET": handle_get,
    "POST": handle_post,
    "DELETE": handle_delete,
}

def dispatch(method, request):
    handler = routes.get(method)
    if not handler:
        return {"error": "Method not allowed"}
    return handler(request)

print(dispatch("GET", {}))      # {'method': 'GET', 'data': []}
print(dispatch("POST", {}))     # {'method': 'POST', 'created': True}
print(dispatch("PATCH", {}))    # {'error': 'Method not allowed'}
```

### `map()` - Apply Function to Every Item
The **`map()` function** in Python applies a specified function to every item of an iterable (like a list, tuple, or string) and returns a **lazy iterator** (a map object). It allows you to process data element-by-element without using explicit `for` loops, making your code shorter and cleaner.
Watch [this](https://youtu.be/OXfcmRFne0Q).
```python
# map(function, iterable)
# applies function to EACH item
# returns a map object (lazy - use list() to see values)

numbers = [1, 2, 3, 4, 5]

# Without map:
squared = []
for n in numbers:
    squared.append(n ** 2)

# With map:
squared = list(map(lambda n: n ** 2, numbers))
print(squared)  # [1, 4, 9, 16, 25]

# With regular function:
def celsius_to_fahrenheit(c):
    return (c * 9/5) + 32

temperatures_c = [0, 20, 37, 100]
temperatures_f = list(map(celsius_to_fahrenheit, temperatures_c))
print(temperatures_f)   # [32.0, 68.0, 98.6, 212.0]

# Real backend use:
raw_users = [
    {"name": "  alice  ", "email": "ALICE@TEST.COM"},
    {"name": "  bob  ", "email": "BOB@TEST.COM"},
]

def normalize_user(user):
    return {
        "name": user["name"].strip().title(),
        "email": user["email"].lower()
    }

clean_users = list(map(normalize_user, raw_users))
print(clean_users)
# [{'name': 'Alice', 'email': 'alice@test.com'},
#  {'name': 'Bob', 'email': 'bob@test.com'}]
```

### `filter()` - Keep Items That Pass a Test
The **`filter()` function** in Python is a built-in tool used to extract elements from an iterable (like a list, tuple, or set) that satisfy a specific condition. It is a memory-efficient higher-order function because it returns an **iterator (a filter object)** instead of creating a new list immediately. To view or use the results as a list, you must explicitly convert it using a constructor like `list()`.
Watch [this](https://youtu.be/kosZevWurtE).
```python
# filter(function, iterable)
# keeps items where function returns True

numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# Without filter:
evens = []
for n in numbers:
    if n % 2 == 0:
        evens.append(n)

# With filter:
evens = list(filter(lambda n: n % 2 == 0, numbers))
print(evens)    # [2, 4, 6, 8, 10]

# Real backend use:
users = [
    {"id": 1, "name": "Alice", "is_active": True, "role": "admin"},
    {"id": 2, "name": "Bob", "is_active": False, "role": "user"},
    {"id": 3, "name": "Charlie", "is_active": True, "role": "user"},
    {"id": 4, "name": "Dave", "is_active": True, "role": "admin"},
]

# Get active admins
active_admins = list(filter(
    lambda u: u["is_active"] and u["role"] == "admin",
    users
))
print([u["name"] for u in active_admins])   # ['Alice', 'Dave']

# filter with None removes falsy values
mixed = [0, 1, "", "hello", None, False, True, [], [1, 2]]
truthy_only = list(filter(None, mixed))
print(truthy_only)  # [1, 'hello', True, [1, 2]]
```

### `reduce()` - Collapse to Single Value
The **`reduce()` function** in Python applies a specified function cumulatively to the items of an iterable, processing them from left to right, to eventually **reduce the entire sequence to a single value**.

Unlike `map()` or `filter()`, `reduce()` is not a built-in function and must be imported from the **`functools` module**. Watch [this](https://youtu.be/mH_hLmZs1h8).
```python
from functools import reduce

# reduce(function, iterable)
# applies function cumulatively
# collapses entire iterable to ONE value

numbers = [1, 2, 3, 4, 5]

# How reduce works:
# Step 1: function(1, 2) = 3
# Step 2: function(3, 3) = 6
# Step 3: function(6, 4) = 10
# Step 4: function(10, 5) = 15

total = reduce(lambda x, y: x + y, numbers)
print(total)    # 15

# Same as sum(), but shows how reduce works
# More useful for complex operations:

# Find maximum without max()
maximum = reduce(lambda x, y: x if x > y else y, numbers)
print(maximum)  # 5

# Build a string from list
words = ["Python", "is", "awesome"]
sentence = reduce(lambda acc, word: acc + " " + word, words)
print(sentence)     # Python is awesome

# Real backend use:
# Merge multiple dicts (Python 3.9+ has | operator, but this shows reduce)
configs = [
    {"debug": True, "host": "localhost"},
    {"port": 8000, "workers": 4},
    {"log_level": "INFO"},
]

merged = reduce(lambda a, b: {**a, **b}, configs)
print(merged)
# {'debug': True, 'host': 'localhost', 'port': 8000,
#  'workers': 4, 'log_level': 'INFO'}

# Practical note:
# In real code, list comprehensions are often clearer than map/filter
# Use whichever is more readable in context
```

---
## 8. Recursion
A function that calls ITSELF. Watch [this](https://youtu.be/ivl5-snqul8).
Every recursive function needs:
1. BASE CASE - when to stop (no more calling itself)
2. RECURSIVE CASE - call itself with a SMALLER problem

```python
# Classic example: factorial
# factorial(5) = 5 × 4 × 3 × 2 × 1 = 120

# Iterative version:
def factorial_iterative(n):
    result = 1
    for i in range(1, n + 1):
        result *= i
    return result

# Recursive version:
def factorial(n):
    # BASE CASE — stop here
    if n == 0 or n == 1:
        return 1

    # RECURSIVE CASE — call itself with smaller problem
    return n * factorial(n - 1)

print(factorial(5))     # 120
print(factorial(0))     # 1
print(factorial(1))     # 1

# How it executes:
# factorial(5)
#   = 5 * factorial(4)
#   = 5 * 4 * factorial(3)
#   = 5 * 4 * 3 * factorial(2)
#   = 5 * 4 * 3 * 2 * factorial(1)
#   = 5 * 4 * 3 * 2 * 1
#   = 120
```

### Recursion with Data Structures
```python
# Real backend use: traverse nested data

# Flatten nested categories (e.g., from a database)
categories = {
    "name": "Electronics",
    "children": [
        {
            "name": "Phones",
            "children": [
                {"name": "Smartphones", "children": []},
                {"name": "Feature Phones", "children": []}
            ]
        },
        {
            "name": "Laptops",
            "children": [
                {"name": "Gaming Laptops", "children": []},
                {"name": "Business Laptops", "children": []}
            ]
        }
    ]
}

def get_all_category_names(category):
    """Recursively collect all category names"""
    names = [category["name"]]  # add current

    for child in category["children"]:
        names.extend(get_all_category_names(child))  # recurse

    return names

all_names = get_all_category_names(categories)
print(all_names)
# ['Electronics', 'Phones', 'Smartphones', 'Feature Phones',
#  'Laptops', 'Gaming Laptops', 'Business Laptops']

# Another real use: build nested comments (Reddit-style)
def build_comment_tree(comments, parent_id=None):
    tree = []
    for comment in comments:
        if comment["parent_id"] == parent_id:
            children = build_comment_tree(comments, comment["id"])
            comment["replies"] = children
            tree.append(comment)
    return tree

flat_comments = [
    {"id": 1, "parent_id": None, "text": "Great article!"},
    {"id": 2, "parent_id": 1, "text": "I agree!"},
    {"id": 3, "parent_id": 1, "text": "Me too!"},
    {"id": 4, "parent_id": 2, "text": "Same here."},
    {"id": 5, "parent_id": None, "text": "Interesting..."},
]

tree = build_comment_tree(flat_comments)
print(tree[0]["text"])              # Great article!
print(tree[0]["replies"][0]["text"])# I agree!
```

### Recursion Limits
```python
import sys

# Python has a recursion limit (default 1000)
print(sys.getrecursionlimit())  # 1000

# If you exceed it: RecursionError: maximum recursion depth exceeded

# ⚠️ Infinite recursion (missing base case):
def infinite(n):
    return infinite(n - 1)  # no base case!
# infinite(5)  ← RecursionError after 1000 calls

# Can increase limit (rarely needed):
sys.setrecursionlimit(10000)

# For very deep recursion, use iteration instead
# Python doesn't optimize tail recursion (unlike some languages)
```

---
## 9. `Docstrings` & Type Hints
**Docstrings and type hints** are complementary tools used in Python to document your code, improve readability, and prevent bugs. Together, they turn your source code into a clean, self-documenting system that Modern IDEs and static analysis tools can leverage.

**Comparison Overview:**

| Feature             | Docstrings                                                       | Type Hints                                              |
| ------------------- | ---------------------------------------------------------------- | ------------------------------------------------------- |
| **Primary Purpose** | Explains _what_ a block of code does, its context, and behavior. | Explicitly defines _which data types_ are allowed.      |
| **Syntax**          | Multiline strings (`"""..."""`) placed right below definitions.  | Inline text using colons `:` and arrow indicators `->`. |
| **Enforcement**     | Not enforced. Purely human-readable.                             | Checked by static analysis tools (like `mypy`).         |
| **Runtime Access**  | Accessible via `obj.__doc__` or `help(obj)`.                     | Accessible via `obj.__annotations__` or `inspect`.      |

### `Docstrings`
In Python, a **docstring** (documentation string) is a special string literal placed immediately after the definition of a module, class, function, or method to **describe its purpose and behavior**. Unlike regular comments, docstrings are parsed by the Python interpreter and can be accessed at runtime. Watch [this](https://youtu.be/0YUdYk5E-w4).
```python
# Single line docstring
def add(a, b):
    """Add two numbers and return the result."""
    return a + b

# Multi-line docstring (Google style — most popular)
def create_user(name, email, role="user"):
    """
    Create a new user account.

    Args:
        name (str): The user's full name.
        email (str): The user's email address. Must be unique.
        role (str): The user's role. Defaults to "user".
                    Options: "user", "admin", "moderator"

    Returns:
        dict: A dictionary containing the new user's data.

    Raises:
        ValueError: If email is not a valid email address.
        DuplicateError: If email already exists in the database.

    Example:
        >>> user = create_user("Alice Smith", "alice@example.com")
        >>> print(user["name"])
        Alice Smith
    """
    # ... implementation
    return {"name": name, "email": email, "role": role}

# Access docstring programmatically
print(create_user.__doc__)

# VS Code shows docstrings when you hover over function
# This is why writing good docstrings matters
```

### Type Hints
**Type hints** in Python are formal annotations that allow you to declare the expected data types of variables, function arguments, and return values. Introduced in **Python 3.5 via PEP 484**, type hints make your code self-documenting, dramatically improve IDE autocomplete, and enable static tools to catch bugs early.

Importantly, **Python does not enforce type hints at runtime**. The Python interpreter completely ignores them during execution, acting as a dynamic language. To catch type mismatches, you must use a separate static type checker like [mypy](https://docs.python.org/3/library/typing.html). Watch [this](https://youtu.be/79zeCq9raY0).
```python
# Type hints tell you WHAT TYPE each parameter expects
# and WHAT TYPE the function returns
# Python doesn't enforce them at runtime
# But tools (mypy, VS Code) use them for checking

# Basic type hints
def greet(name: str) -> str:
    return f"Hello, {name}!"

def add(a: int, b: int) -> int:
    return a + b

def get_price(item_id: int) -> float:
    return 19.99

def process(data: dict) -> None:
    print(data)

# With complex types (import from typing)
from typing import Optional, Union, List, Dict, Tuple, Any

# Optional — value can be the type OR None
def find_user(user_id: int) -> Optional[dict]:
    # returns dict if found, None if not found
    if user_id == 1:
        return {"id": 1, "name": "Alice"}
    return None

# Union — can be one of several types
def parse_id(user_id: Union[int, str]) -> int:
    return int(user_id)

# List, Dict, Tuple
def get_users() -> List[dict]:
    return [{"name": "Alice"}, {"name": "Bob"}]

def get_config() -> Dict[str, str]:
    return {"host": "localhost", "port": "8000"}

def get_coordinates() -> Tuple[float, float]:
    return (40.7128, -74.0060)

# Python 3.10+ — cleaner syntax with |
def find_user(user_id: int) -> dict | None:   # instead of Optional[dict]
    pass

def parse_id(user_id: int | str) -> int:      # instead of Union[int, str]
    pass

# Real backend function with full type hints and docstring:
from typing import Optional, List

def paginate(
    items: List[dict],
    page: int = 1,
    page_size: int = 10
) -> Dict[str, Any]:
    """
    Paginate a list of items.

    Args:
        items: List of items to paginate.
        page: Page number (1-indexed). Defaults to 1.
        page_size: Number of items per page. Defaults to 10.

    Returns:
        Dict containing:
            - data: List of items for current page
            - total: Total number of items
            - page: Current page number
            - pages: Total number of pages
            - has_next: Whether there's a next page
            - has_prev: Whether there's a previous page
    """
    total = len(items)
    pages = (total + page_size - 1) // page_size
    start = (page - 1) * page_size
    end = start + page_size

    return {
        "data": items[start:end],
        "total": total,
        "page": page,
        "pages": pages,
        "has_next": page < pages,
        "has_prev": page > 1
    }

# Test it
all_items = [{"id": i} for i in range(1, 26)]  # 25 items
result = paginate(all_items, page=2, page_size=10)
print(result)
# {
#   'data': [{'id': 11}, ..., {'id': 20}],
#   'total': 25,
#   'page': 2,
#   'pages': 3,
#   'has_next': True,
#   'has_prev': True
# }
```

---
## 10. Putting It All Together
Real Backend Functions:
```python
# A realistic set of utility functions you'd write
# in an actual backend project

from typing import Optional, List, Dict, Any
from functools import reduce

# ─────────────────────────────────────────
# Input validation utilities
# ─────────────────────────────────────────

def validate_email(email: str) -> bool:
    """Basic email validation."""
    if not email or not isinstance(email, str):
        return False
    parts = email.strip().split("@")
    if len(parts) != 2:
        return False
    local, domain = parts
    if not local or not domain:
        return False
    if "." not in domain:
        return False
    return True

def validate_password(password: str) -> Optional[str]:
    """
    Validate password strength.
    Returns error message if invalid, None if valid.
    """
    if not password:
        return "Password is required"
    if len(password) < 8:
        return "Password must be at least 8 characters"
    if not any(c.isupper() for c in password):
        return "Password must contain at least one uppercase letter"
    if not any(c.isdigit() for c in password):
        return "Password must contain at least one number"
    return None     # valid

# ─────────────────────────────────────────
# Data transformation utilities
# ─────────────────────────────────────────

def sanitize_user_input(data: Dict[str, Any]) -> Dict[str, Any]:
    """Clean and normalize user input data."""
    sanitized = {}
    for key, value in data.items():
        if isinstance(value, str):
            sanitized[key] = value.strip()
        else:
            sanitized[key] = value
    return sanitized

def mask_sensitive_data(user: Dict[str, Any]) -> Dict[str, Any]:
    """Remove sensitive fields before sending to client."""
    sensitive_fields = {"password", "password_hash", "secret_key", "token"}
    return {
        key: value
        for key, value in user.items()
        if key not in sensitive_fields
    }

# ─────────────────────────────────────────
# Response builders
# ─────────────────────────────────────────

def success_response(
    data: Any,
    message: str = "Success",
    status_code: int = 200
) -> Dict[str, Any]:
    """Build a standardized success response."""
    return {
        "success": True,
        "status_code": status_code,
        "message": message,
        "data": data
    }

def error_response(
    message: str,
    status_code: int = 400,
    errors: Optional[List[str]] = None
) -> Dict[str, Any]:
    """Build a standardized error response."""
    response = {
        "success": False,
        "status_code": status_code,
        "message": message,
    }
    if errors:
        response["errors"] = errors
    return response

# ─────────────────────────────────────────
# Using it all together
# ─────────────────────────────────────────

def register_user(raw_data: Dict[str, Any]) -> Dict[str, Any]:
    """
    Process user registration.

    Args:
        raw_data: Raw input from registration form.

    Returns:
        Standardized success or error response.
    """
    # Step 1: Sanitize input
    data = sanitize_user_input(raw_data)

    # Step 2: Validate
    validation_errors = []

    if not data.get("name"):
        validation_errors.append("Name is required")

    if not validate_email(data.get("email", "")):
        validation_errors.append("Valid email is required")

    password_error = validate_password(data.get("password", ""))
    if password_error:
        validation_errors.append(password_error)

    if validation_errors:
        return error_response(
            "Validation failed",
            status_code=422,
            errors=validation_errors
        )

    # Step 3: Create user (simulated)
    new_user = {
        "id": 1,
        "name": data["name"],
        "email": data["email"].lower(),
        "password_hash": "hashed_" + data["password"],  # never store plain!
        "role": "user",
        "is_active": True
    }

    # Step 4: Return safe response (no password)
    safe_user = mask_sensitive_data(new_user)
    return success_response(safe_user, "User registered successfully", 201)

# Test it
print(register_user({
    "name": "  Alice Smith  ",
    "email": "Alice@TEST.com",
    "password": "SecurePass1"
}))

print(register_user({
    "name": "",
    "email": "not-an-email",
    "password": "weak"
}))
```

---
## Visual Summary

```
┌──────────────────────────────────────────────────────────────┐
│                    FUNCTIONS & SCOPE                         │
│                                                              │
│  FUNCTION ANATOMY:                                           │
│  def name(param, *args, kwarg=default, **kwargs) -> type:    │
│      """docstring"""                                         │
│      # body                                                  │
│      return value                                            │
│                                                              │
│  SCOPE (LEGB):                                               │
│  ┌─────────────────────────────────┐                         │
│  │ Built-in (print, len, range...) │                         │
│  │  ┌───────────────────────────┐  │                         │
│  │  │ Global (module level)     │  │                         │
│  │  │  ┌─────────────────────┐  │  │                         │
│  │  │  │ Enclosing (outer fn)│  │  │                         │
│  │  │  │  ┌───────────────┐  │  │  │                         │
│  │  │  │  │ Local (inner) │  │  │  │                         │
│  │  │  │  └───────────────┘  │  │  │                         │
│  │  │  └─────────────────────┘  │  │                         │
│  │  └───────────────────────────┘  │                         │
│  └─────────────────────────────────┘                         │
│  Python searches: L → E → G → B                              │
│                                                              │
│  KEY CONCEPTS:                                               │
│  *args    → variable positional args → tuple                 │
│  **kwargs → variable keyword args   → dict                   │
│  lambda   → anonymous one-line function                      │
│  closure  → function that remembers outer scope              │
│  map()    → apply function to each item                      │
│  filter() → keep items that pass test                        │
│  reduce() → collapse to single value                         │
└──────────────────────────────────────────────────────────────┘
```

---
## Knowledge Check

1. What is the difference between a parameter and an argument?
2. What is the mutable default argument trap? How do you fix it?
3. What does `*args` give you inside a function?
4. What does `**kwargs` give you inside a function?
5. What is the LEGB rule?
6. What is the difference between `global` and `nonlocal`?
7. What is a closure and when would you use one?
8. What does `lambda` return implicitly?
9. What does `map()` do? What does `filter()` do?
10. What is a base case in recursion?
11. What is the difference between `return None` and no return?
12. What are type hints and do Python enforce them at runtime?

---
## Practice Exercises
Exercise 1:
```
- Write a function make_greeting(greeting)
- that returns a function that greets a person
- greet_hello = make_greeting("Hello")
- greet_hi = make_greeting("Hi")
- print(greet_hello("Alice"))  → "Hello, Alice!"
- print(greet_hi("Bob"))       → "Hi, Bob!"
```

Exercise 2:
```
- Write a function that takes a list of dicts (users)
- and returns only users who:
  - are active
  - have a valid email (contains @ and .)
  - have age >= 18
- Use filter() and a helper validation function
```

Exercise 3:
```
- Write a recursive function that sums all numbers
- in a NESTED list (list of lists of lists...)
- nested = [1, [2, 3], [4, [5, 6]], 7]
- sum_nested(nested) → 28
```

Exercise 4:
```
- Write a function with *args and **kwargs
- that builds an HTTP request log string
- log_request("GET", "/users", status=200, duration=45)
- → "[GET] /users | status=200 | duration=45ms"
```

---
## Phase 1.2 Complete!
**You now know:**
```
✅ Defining functions and return values
✅ All parameter types (positional, default, *args, **kwargs)
✅ The mutable default argument trap
✅ Scope and the LEGB rule
✅ Closures and nonlocal
✅ Lambda functions
✅ Higher-order functions (map, filter, reduce)
✅ Recursion
✅ Docstrings and type hints
```

---