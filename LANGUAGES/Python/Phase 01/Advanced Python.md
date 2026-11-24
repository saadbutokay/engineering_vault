Python Basics → You can write Python;
Functions & Scope → You can organize Python into functions;
Object Oriented Programming → You can model real world problems with classes;
**Advanced Python** → You write Python the way professionals do;

This phase covers the features that:
```
- Senior engineers use DAILY
- Make code elegant and efficient
- Show up in EVERY production codebase
- Are asked about in TECHNICAL INTERVIEWS
- Separate junior from mid/senior engineers
```
**Every single topic here will appear in your backend work.**

---
## Setup
```bash
cd ~/projects
mkdir advanced_python
cd advanced_python
python3 -m venv venv
source venv/bin/activate
touch advanced.py
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
touch advanced.py
code .
```

---
## 1. Iterators & Generators
In Python, **iterators and generators are tools used for memory-efficient, lazy evaluation** of data streams. They allow you to process data one item at a time instead of loading entire datasets into memory at once.
### Iterators
In Python, an **iterator** is an object that allows you to traverse through all the elements of a collection (like a list, tuple, or dictionary) **one element at a time**. Technically, a Python iterator is any object that implements the **Iterator Protocol**, which consists of two special methods: `__iter__()` and `__next__()`. Watch [this](https://youtu.be/k0D3MQwLn7A).

```python title:iterator.py
# What happens when you write: for item in something
# Python calls iter() on it → gets an iterator
# Then calls next() repeatedly → gets each item
# When items run out → StopIteration raised → loop stops

# Any object with __iter__ and __next__ is an iterator
numbers = [1, 2, 3]

# Python does this under the hood:
iterator = iter(numbers)  # calls numbers.__iter__()
print(next(iterator))     # 1      calls iterator.__next__()
print(next(iterator))     # 2
print(next(iterator))     # 3
try:
    print(next(iterator)) # StopIteration!
except StopIteration:
    print("Iterator exhausted")

# Build your own iterator
class CountUp:
    """Counts from start to end."""
    def __init__(self, start: int, end: int):
        self.current = start
        self.end = end

    def __iter__(self):
        return self  # iterator returns itself

    def __next__(self):
        if self.current > self.end:
            raise StopIteration
        value = self.current
        self.current += 1
        return value

counter = CountUp(1, 5)
for n in counter:
    print(n, end=" ")  # 1 2 3 4 5
print()

# Can also use next() manually
counter
2 = CountUp(1, 3)
print(next(counter2))   # 1
print(next(counter2))   # 2
print(next(counter2))   # 3
```

### Generators
In Python, a **generator** is a special type of function that returns an iterable **generator object**. Unlike regular functions that calculate an entire dataset and return it all at once, generators use the **`yield` keyword** to produce values **one at a time, on demand**.

Because they generate data dynamically instead of storing the entire sequence in memory, they are incredibly **memory-efficient** and are the industry-standard choice for handling large files, data streams, or infinite loops. Watch [this](https://youtu.be/G1lJeEIl05o).

```python title:generators.py
# Generators are functions that YIELD values one at a time
# instead of computing everything and returning a list

# They are LAZY — compute values only when needed
# They are MEMORY EFFICIENT — don't store all values in memory

# ─────────────────────────────────────────
# GENERATOR FUNCTION — uses yield
# ─────────────────────────────────────────
def count_up(start: int, end: int):
    current = start
    while current <= end:
        yield current  # ← pause here, send value, resume later
        current += 1

# Using it
gen = count_up(1, 5)
print(type(gen))     # <class 'generator'>
print(next(gen))     # 1
print(next(gen))     # 2
print(next(gen))     # 3

# Or use in a for loop
for n in count_up(1, 5):
    print(n, end=" ")  # 1 2 3 4 5
print()

# ─────────────────────────────────────────
# HOW YIELD WORKS (execution flow)
# ─────────────────────────────────────────
def explain_yield():
    print("Step 1: Before first yield")
    yield 10  # ← pauses here
    print("Step 2: Before second yield")
    yield 20  # ← pauses here
    print("Step 3: Before third yield")
    yield 30  # ← pauses here
    print("Step 4: Function ends")

gen = explain_yield()

print("Calling next() first time:")
value = next(gen)  # runs until first yield
print(f"Got: {value}")

print("\nCalling next() second time:")
value = next(gen)  # resumes from first yield
print(f"Got: {value}")

print("\nCalling next() third time:")
value = next(gen)  # resumes from second yield
print(f"Got: {value}")

print("\nCalling next() fourth time:")
try:
    next(gen)  # runs "Step 4" then StopIteration
except StopIteration:
    print("Generator done")
```

### Why Generators?
Memory Efficiency.
```python
import sys
# ─────────────────────────────────────────
# THE MEMORY PROBLEM without generators
# ─────────────────────────────────────────
def get_million_numbers_list():
    return [i for i in range(1_000_000)]  # stores ALL in memory

# ─────────────────────────────────────────
# THE SOLUTION with generators
# ─────────────────────────────────────────
def get_million_numbers_gen():
    for i in range(1_000_000):
        yield i  # one at a time

# Memory comparison
numbers_list = get_million_numbers_list()
numbers_gen = get_million_numbers_gen()

print(sys.getsizeof(numbers_list))  # ~8,697,456 bytes (~8MB)
print(sys.getsizeof(numbers_gen))   # 112 bytes (!!!)

# The generator holds NO data — just knows HOW to produce it

# ─────────────────────────────────────────
# REAL BACKEND USE: Processing large datasets
# ─────────────────────────────────────────
def read_large_file(filepath: str):
    """Read a huge log file line by line — never loads full file."""
    with open(filepath, "r") as f:
        for line in f:
            yield line.strip()  # one line at a time

def read_database_in_chunks(query: str, chunk_size: int = 1000):
    """Read database results in chunks — not all at once."""
    offset = 0
    while True:
        # Simulating: SELECT * FROM logs LIMIT 1000 OFFSET offset
        chunk = [{"id": i} for i in range(offset, offset + chunk_size)]
        if not chunk:
            break
        for record in chunk:
            yield record
        offset += chunk_size
        if offset >= 5000:  # pretend database has 5000 records
            break

# Process 5000 records without loading all into memory
for record in read_database_in_chunks("SELECT * FROM logs"):
    pass  # process each record
# e.g., send to analytics, transform, save somewhere

# ─────────────────────────────────────────
# GENERATOR EXPRESSIONS — inline generators
# ─────────────────────────────────────────
# List comprehension → stores everything in memory
squares_list = [x ** 2 for x in range(1000000)]  # 8MB

# Generator expression → lazy, memory efficient
squares_gen = (x ** 2 for x in range(1000000))   # 112 bytes

# Look identical but: [ ] = list, ( ) = generator

# Use with sum, max, min, any, all
total = sum(x ** 2 for x in range(1000))
print(total)  # 332833500

# Check if ANY user is admin
users = [{"name": "Alice", "role": "user"}, {"name": "Bob", "role": "admin"}]
has_admin = any(u["role"] == "admin" for u in users)
print(has_admin)  # True

# Check if ALL users are active
all_active = all(u.get("is_active", True) for u in users)
print(all_active)  # True
```

### `yield` Advanced: `send()` and Two-Way Communication
In Python, the `yield` keyword is not just for pulling data out of a generator; it can also **push data back in**. By using the `generator.send()` method, you turn a standard generator into a **coroutine**, enabling dynamic, two-way communication.

When you call `next(gen)` or `gen.send(None)`, the generator runs until it hits a `yield` statement.

The secret to two-way communication is assigning the `yield` expression to a variable:
```python
received_value = yield expression
```
- **The Pause:** The generator yields `expression` to the caller and pauses.
- **The Resume:** The caller uses `gen.send(value)`.
- **The Injection:** The `value` sent by the caller replaces the paused `yield` expression and is assigned to `received_value`.

```python
# Generators can also RECEIVE values via send()
# This is less common but good to know
def running_average():
    """Generator that maintains a running average."""
    total = 0
    count = 0
    average = 0
    while True:
        value = yield average  # yield sends out average, receives value
        if value is None:
            break
        total += value
        count += 1
        average = total / count

gen = running_average()
next(gen)  # prime the generator (reach first yield)

print(gen.send(10))   # send 10, get average back: 10.0
print(gen.send(20))   # 15.0
print(gen.send(30))   # 20.0
print(gen.send(40))   # 25.0

# ─────────────────────────────────────────
# yield from — delegate to sub-generator
# ─────────────────────────────────────────
def inner_gen():
    yield 1
    yield 2
    yield 3

def outer_gen():
    yield "start"
    yield from inner_gen()  # ← delegate to inner
    yield "end"

for item in outer_gen():
    print(item, end=" ")  # start 1 2 3 end
print()
```

---
## 2. Decorators
A decorator is a function that WRAPS another function adding behavior BEFORE and/or AFTER it runs.
```
Like a sandwich: bread (before code) + your function + bread (after code)
```
**Used for:** logging, authentication, caching, timing, validation, rate limiting.
Watch [this](https://youtu.be/U-G-mSd4KAE).

### Understanding Functions as Objects First
```python
# Functions are objects — they can be passed around
def shout(text):
    return text.upper()

def whisper(text):
    return text.lower()

def speak(func, message):  # takes a FUNCTION as argument
    return func(message)

print(speak(shout, "hello"))    # HELLO
print(speak(whisper, "HELLO"))  # hello
```

### Building a Decorator Step by Step
```python
# ─────────────────────────────────────────
# STEP 1: A function that wraps another function
# ─────────────────────────────────────────
def my_decorator(func):
    def wrapper():
        print("Before the function runs")
        func()
        print("After the function runs")
    return wrapper

def say_hello():
    print("Hello!")

# Manually wrapping
decorated = my_decorator(say_hello)
decorated()
# Before the function runs
# Hello!
# After the function runs

# ─────────────────────────────────────────
# STEP 2: Using @ syntax (syntactic sugar)
# ─────────────────────────────────────────
@my_decorator  # same as: say_hello = my_decorator(say_hello)
def say_hello():
    print("Hello!")

say_hello()
# Before the function runs
# Hello!
# After the function runs
```

### Real Decorator with Arguments and Return Values
```python
import functools
import time
from typing import Callable, Any

# ─────────────────────────────────────────
# IMPORTANT: functools.wraps
# Preserves the original function's metadata
# ─────────────────────────────────────────
def timer(func: Callable) -> Callable:
    """Measure how long a function takes to run."""
    @functools.wraps(func)  # ← always use this!
    def wrapper(*args, **kwargs) -> Any:
        start = time.perf_counter()
        result = func(*args, **kwargs)  # call original function
        end = time.perf_counter()
        duration = (end - start) * 1000
        print(f"[TIMER] {func.__name__} took {duration:.2f}ms")
        return result  # return original result
    return wrapper

@timer
def process_data(items: list) -> list:
    time.sleep(0.1)  # simulate work
    return [item * 2 for item in items]

result = process_data([1, 2, 3, 4, 5])
print(result)
# [TIMER] process_data took 100.23ms
# [2, 4, 6, 8, 10]

# Without @functools.wraps, name would be "wrapper" not "process_data"
print(process_data.__name__)  # process_data ✅ (wraps preserves this)

# ─────────────────────────────────────────
# LOGGING DECORATOR — extremely common
# ─────────────────────────────────────────
def log_calls(func: Callable) -> Callable:
    """Log every call to a function with args and return value."""
    @functools.wraps(func)
    def wrapper(*args, **kwargs) -> Any:
        args_repr = [repr(a) for a in args]
        kwargs_repr = [f"{k}={v!r}" for k, v in kwargs.items()]
        signature = ", ".join(args_repr + kwargs_repr)
        print(f"[LOG] Calling {func.__name__}({signature})")
        try:
            result = func(*args, **kwargs)
            print(f"[LOG] {func.__name__} returned {result!r}")
            return result
        except Exception as e:
            print(f"[LOG] {func.__name__} raised {type(e).__name__}: {e}")
            raise
    return wrapper

@log_calls
def create_user(name: str, email: str, role: str = "user") -> dict:
    return {"name": name, "email": email, "role": role}

create_user("Alice", "alice@test.com", role="admin")
# [LOG] Calling create_user('Alice', 'alice@test.com', role='admin')
# [LOG] create_user returned {'name': 'Alice', ...}
```

### Decorators with Arguments
```python
# When the decorator itself needs parameters
# You need THREE levels of nesting
def retry(max_attempts: int = 3, delay: float = 1.0):
    """Retry a function if it raises an exception."""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            last_exception = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    print(f"[RETRY] Attempt {attempt}/{max_attempts} failed: {e}")
                    if attempt < max_attempts:
                        time.sleep(delay)
            print(f"[RETRY] All {max_attempts} attempts failed")
            raise last_exception
        return wrapper
    return decorator

# With arguments
@retry(max_attempts=3, delay=0.5)
def connect_to_database(url: str) -> str:
    import random
    if random.random() < 0.7:  # 70% chance of failure
        raise ConnectionError(f"Failed to connect to {url}")
    return f"Connected to {url}"

try:
    result = connect_to_database("postgresql://localhost/mydb")
    print(result)
except ConnectionError:
    print("Database connection ultimately failed")

# ─────────────────────────────────────────
# REQUIRE AUTH decorator (real backend use)
# ─────────────────────────────────────────
def require_role(*allowed_roles: str):
    """Check user has required role before running function."""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            # In real FastAPI, you'd get user from request context
            # Here we simulate with a global current_user
            current_user = kwargs.get("current_user", {})
            user_role = current_user.get("role", "guest")
            if user_role not in allowed_roles:
                raise PermissionError(
                    f"Role '{user_role}' not allowed. "
                    f"Required: {allowed_roles}"
                )
            return func(*args, **kwargs)
        return wrapper
    return decorator

@require_role("admin", "moderator")
def delete_post(post_id: int, current_user: dict) -> str:
    return f"Post {post_id} deleted by {current_user['name']}"

@require_role("admin")
def ban_user(user_id: int, current_user: dict) -> str:
    return f"User {user_id} banned by {current_user['name']}"

# Test
try:
    result = delete_post(
        42, current_user={"name": "Alice", "role": "moderator"}
    )
    print(result)  # Post 42 deleted by Alice
except PermissionError as e:
    print(e)

try:
    result = ban_user(
        10, current_user={"name": "Bob", "role": "moderator"}
    )
except PermissionError as e:
    print(e)  # Role 'moderator' not allowed. Required: ('admin',)
```

### Stacking Decorators
```python
# Decorators stack from BOTTOM to TOP (closest first)
@log_calls   # applied second (outer)
@timer       # applied first (inner)
def heavy_computation(n: int) -> int:
    return sum(range(n))

# Same as: heavy_computation = log_calls(timer(heavy_computation))

result = heavy_computation(1000000)

# Execution order:
# log_calls wrapper runs first
#   → timer wrapper runs
#     → actual heavy_computation runs
#   ← timer logs duration
# ← log_calls logs return value
```

### Class-Based Decorators
```python
# Decorators can also be CLASSES
# Useful when decorator needs to maintain STATE
class RateLimiter:
    """Limit how often a function can be called."""
    def __init__(self, max_calls: int, period: float):
        self.max_calls = max_calls
        self.period = period
        self.calls = []

    def __call__(self, func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            # Remove old calls outside the time period
            self.calls = [t for t in self.calls if now - t < self.period]
            if len(self.calls) >= self.max_calls:
                raise Exception(
                    f"Rate limit exceeded: {self.max_calls} calls "
                    f"per {self.period}s"
                )
            self.calls.append(now)
            return func(*args, **kwargs)
        return wrapper

@RateLimiter(max_calls=3, period=60)  # 3 calls per minute
def send_email(to: str, message: str) -> str:
    return f"Email sent to {to}"

print(send_email("alice@test.com", "Hello"))    # works
print(send_email("bob@test.com", "Hello"))      # works
print(send_email("charlie@test.com", "Hello"))  # works
try:
    print(send_email("dave@test.com", "Hello")) # Rate limit exceeded!
except Exception as e:
    print(e)
```

---
## 3. Context Managers
Context managers handle SETUP and TEARDOWN automatically. They guarantee cleanup even if an error occurs.
In Python, a **context manager** is an object that allocates and releases resources precisely when you need them. They are most commonly used via the **`with` statement**, which guarantees that setup and cleanup actions are performed automatically, even if errors or exceptions occur inside the block. Watch [this](https://youtu.be/Lv1treHIckI).

The most classic example is opening a file:
```python
with open("example.txt", "w") as file:
    file.write("Hello, World!")
# The file is automatically closed here, even if write() raises an exception!
```

The `with` statement uses context managers.
```
Most common use:
  - Opening/closing files
  - Opening/closing database connections
  - Acquiring/releasing locks
  - Starting/stopping timers
  - Managing transactions
```

### Why Use Context Managers?
Without a context manager, you have to remember to close resources manually. This requires writing verbose `try...finally` blocks:
```python
# The manual way (boilerplate heavy)
file = open("example.txt", "w")
try:
    file.write("Hello, World!")
finally:
    file.close()  # Ensures the file closes no matter what
```

### The `with` Statement
The `with` statement completely eliminates this boilerplate code, preventing resource leaks, file corruption, and broken connections.
```python
# Without context manager — DANGEROUS
file = open("test.txt", "w")
file.write("Hello")
# If an error happens here, file never gets closed!
# File stays locked. Memory leak.
file.close()

# With context manager — SAFE
with open("test.txt", "w") as file:
    file.write("Hello")
# File ALWAYS closed here, even if exception occurs

# The with block calls:
#   file.__enter__() → sets up, returns the resource
#   file.__exit__()  → teardown (always runs)
```

### Building Context Managers with Classes
To build a custom context manager using a class in Python, your class must implement two dunder (double underscore) methods: **`__enter__()`** and **`__exit__()`**. This pair of methods governs the setup and teardown phases of a resource when used inside a `with` statement.
```python
class CustomContext:
    def __init__(self, resource_target):
        # 1. Initialize properties here
        self.resource_target = resource_target

    def __enter__(self):
        # 2. Setup the resource (e.g., open a file, connect to a database)
        print("Setting up resource...")
        # Whatever is returned here is assigned to the variable after 'as'
        return self.resource_target 

    def __exit__(self, exc_type, exc_value, traceback):
        # 3. Teardown/clean up the resource
        print("Cleaning up resource...")
        
        # Handle exceptions if needed
        # Return True to suppress an internal exception, or False to let it propagate
        return False
```

**More Example:**
```python
class DatabaseConnection:
    """Simulate a database connection with proper lifecycle."""
    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        self.connection = None
        self.transaction_active = False

    def __enter__(self):
        """Setup: open connection, start transaction."""
        print(f"[DB] Connecting to {self.connection_string}")
        self.connection = f"Connection to {self.connection_string}"
        self.transaction_active = True
        print("[DB] Transaction started")
        return self  # returned as 'as' variable

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Teardown: commit or rollback, close connection."""
        if exc_type is None:  # No exception occurred
            print("[DB] Committing transaction")
        else:  # Exception occurred
            print(f"[DB] Rolling back due to: {exc_val}")
        self.transaction_active = False
        self.connection = None
        print("[DB] Connection closed")
        return False  # False = don't suppress the exception
        # True = suppress the exception (rarely want this)

    def execute(self, query: str) -> list:
        if not self.connection:
            raise RuntimeError("No active connection")
        print(f"[DB] Executing: {query}")
        return [{"id": 1, "name": "Alice"}]  # simulated

# Normal execution
with DatabaseConnection("postgresql://localhost/mydb") as db:
    users = db.execute("SELECT * FROM users")
    print(users)
# [DB] Connecting to postgresql://localhost/mydb
# [DB] Transaction started
# [DB] Executing: SELECT * FROM users
# [DB] Committing transaction
# [DB] Connection closed

print()

# Exception during transaction
try:
    with DatabaseConnection("postgresql://localhost/mydb") as db:
        users = db.execute("INSERT INTO users ...")
        raise ValueError("Something went wrong!")  # simulated error
        db.execute("UPDATE ...")  # never reaches here
except ValueError as e:
    print(f"Caught: {e}")
# [DB] Connecting...
# [DB] Transaction started
# [DB] Executing: INSERT INTO users ...
# [DB] Rolling back due to: Something went wrong!
# [DB] Connection closed
# Caught: Something went wrong!
```

### Building Context Managers with `@contextmanager`
The **`@contextmanager` decorator** from the standard library's `contextlib` module allows you to build custom context managers using a simple **generator function** instead of writing a full class with `__enter__` and `__exit__` methods. To create a context manager with this decorator, your function must use a `try...finally` block (or `try...except...finally`) and **`yield` exactly once**.

- **Before the `yield`**: Code executes when entering the `with` block (Setup).
- **The `yield` value**: This is what gets assigned to the variable after the `as` keyword.
- **After the `yield`**: Code executes when exiting the `with` block (Teardown/Cleanup)

**Basic Example:**
```python
from contextlib import contextmanager

@contextmanager
def my_context_manager():
    # 1. Setup code goes here (e.g., open a resource, start a timer)
    print("Entering the context...")
    
    try:
        # 2. Yield control (and optionally, a resource) to the with-block
        yield "Resource Object"
    finally:
        # 3. Teardown code goes here (guaranteed to run)
        print("Exiting the context and cleaning up...")
```

**Complete Example:**
```python
from contextlib import contextmanager

# Easier way to write context managers
# Use yield to separate setup from teardown

@contextmanager
def timer_context(operation_name: str):
    """Time a block of code."""
    print(f"[TIMER] Starting: {operation_name}")
    start = time.perf_counter()
    try:
        yield  # code inside "with" block runs here
    except Exception as e:
        print(f"[TIMER] {operation_name} failed: {e}")
        raise
    finally:
        end = time.perf_counter()
        duration = (end - start) * 1000
        print(f"[TIMER] {operation_name} completed in {duration:.2f}ms")

with timer_context("database query"):
    time.sleep(0.05)  # simulate query
    data = [1, 2, 3]
# [TIMER] Starting: database query
# [TIMER] database query completed in 50.12ms

@contextmanager
def managed_transaction(db_session):
    """Manage a database transaction."""
    try:
        yield db_session  # give the session to the with block
        db_session.commit()    # success → commit
    except Exception:
        db_session.rollback()  # error → rollback
        raise
    finally:
        db_session.close()     # always close

@contextmanager
def temp_file(filename: str, content: str):
    """Create a temp file, yield it, then delete it."""
    import os
    with open(filename, "w") as f:
        f.write(content)
    try:
        yield filename  # caller uses the file
    finally:
        if os.path.exists(filename):
            os.remove(filename)
            print(f"Cleaned up {filename}")

with temp_file("temp_data.txt", "hello world") as fname:
    with open(fname) as f:
        print(f.read())  # hello world
# Cleaned up temp_data.txt

# ─────────────────────────────────────────
# contextlib utilities
# ─────────────────────────────────────────
from contextlib import suppress

# Suppress specific exceptions cleanly
with suppress(FileNotFoundError):
    open("nonexistent_file.txt")  # silently ignored
# No try/except needed!

from contextlib import ExitStack

# Manage MULTIPLE context managers dynamically
with ExitStack() as stack:
    files = [
        stack.enter_context(open(f"file{i}.txt", "w"))
        for i in range(3)
    ]
    for f in files:
        f.write("hello")
# All 3 files closed automatically
```

### When to use `@contextmanager` vs Class-based approach

|Metric|`@contextmanager` (Generator)|Class-Based (`__enter__`/`__exit__`)|
|---|---|---|
|**Boilerplate**|Very low; utilizes a single function.|Higher; requires a class blueprint.|
|**Readability**|High for simple resource setups.|High for complex, multi-method structures.|
|**State Tracking**|Difficult to track complex state across steps.|Easy; values can be stored inside `self` attributes.|

---
## 4. Exception Handling
**Exception handling in Python** allows a program to deal with unexpected runtime errors without crashing. It provides a controlled pathway to detect errors, manage them gracefully, and let your program resume execution. Python uses **four primary blocks (`try`, `except`, `else`, `finally`)** and the **`raise` keyword** to manage exceptions. Watch [this](https://youtu.be/j_q6NGOwDJo) and, [this](https://youtu.be/V_NXT2-QIlE).
### `try`, `except`, `else`, `finally`
Here is how the four main keywords work together:
```python
# ─────────────────────────────────────────
# BASIC STRUCTURE
# ─────────────────────────────────────────
try:
    # Code that might raise an exception
    result = 10 / 0
except ZeroDivisionError:
    # Runs if ZeroDivisionError is raised
    print("Can't divide by zero")
except (TypeError, ValueError) as e:
    # Catches multiple exception types
    print(f"Type or value error: {e}")
except Exception as e:
    # Catches ANY exception (be careful — don't overuse)
    print(f"Unexpected error: {e}")
else:
    # Runs ONLY if no exception occurred
    print(f"Result: {result}")
finally:
    # ALWAYS runs — exception or not
    print("Cleanup code here")

# ─────────────────────────────────────────
# BUILT-IN EXCEPTIONS you'll use constantly
# ─────────────────────────────────────────
# ValueError     → wrong value (int("abc"))
# TypeError      → wrong type (1 + "a")
# KeyError       → dict key not found (d["missing"])
# IndexError     → list index out of range (l[999])
# AttributeError → object has no attribute (None.split())
# NameError      → variable not defined (print(undefined))
# FileNotFoundError → file doesn't exist
# PermissionError → no permission to access resource
# ConnectionError → network/database connection failed
# TimeoutError   → operation timed out
# StopIteration  → iterator exhausted
# RuntimeError   → general runtime error
# NotImplementedError → abstract method not implemented
# ZeroDivisionError  → dividing by zero
```

### Custom Exceptions
To create a **custom exception** in Python, you need to **define a new class that inherits from the built-in `Exception` class**. Custom exceptions allow you to capture domain-specific errors and give them clear, descriptive names to improve code readability and debugging.
#### Minimal Custom Exception
If you only need a descriptive error name without modifying how the error behaves, use a simple `pass` statement.
```python
class ValueTooLowError(Exception):
    """Raised when the input value is below the allowed threshold."""
    pass

# Usage
def check_age(age):
    if age < 18:
        raise ValueTooLowError("Age must be at least 18.")

try:
    check_age(15)
except ValueTooLowError as e:
    print(f"Caught an error: {e}")
```

#### Custom Exception with Extra Attributes
If you want your exception to carry extra context—such as an error code, status code, or the faulty system's data—override the `__init__` method. Always call `super().__init__()` to preserve default exception functionality.
```python
class APIError(Exception):
    def __init__(self, message, status_code):
        # Pass the main error message to the parent Exception class
        super().__init__(message)
        # Store custom data
        self.status_code = status_code

# Usage
try:
    raise APIError("User authentication failed", status_code=401)
except APIError as e:
    print(f"Error Message: {e}")
    print(f"HTTP Status Code: {e.status_code}")
```

#### Creating an Exception Hierarchy
When developing complex applications or packages, it is a best practice to create a base custom exception class for your app. Then, inherit from that base class to create more specific errors.
```python
# Base application error
class AppError(Exception):
    pass

# Specific module errors inheriting from the base error
class DatabaseConnectionError(AppError):
    pass

class QueryExecutionError(AppError):
    pass

# Usage
try:
    # Simulating a database problem
    raise DatabaseConnectionError("Could not connect to replica database.")
except DatabaseConnectionError:
    print("Retrying connection...")
except AppError:
    print("A general application error occurred.")
```

#### All Exceptions

```python
# Always inherit from Exception (or a more specific exception)

# ─────────────────────────────────────────
# Base exception for your application
# ─────────────────────────────────────────
class AppException(Exception):
    """Base exception for all application errors."""
    def __init__(self, message: str, status_code: int = 500):
        super().__init__(message)
        self.message = message
        self.status_code = status_code

    def to_dict(self) -> dict:
        return {
            "error": self.__class__.__name__,
            "message": self.message,
            "status_code": self.status_code
        }

# ─────────────────────────────────────────
# Specific exceptions
# ─────────────────────────────────────────
class ValidationError(AppException):
    """Raised when input data is invalid."""
    def __init__(self, message: str, field: str = None):
        super().__init__(message, status_code=422)
        self.field = field

    def to_dict(self) -> dict:
        d = super().to_dict()
        if self.field:
            d["field"] = self.field
        return d

class NotFoundError(AppException):
    """Raised when a resource is not found."""
    def __init__(self, resource: str, id: int):
        super().__init__(f"{resource} with id {id} not found", status_code=404)
        self.resource = resource
        self.resource_id = id

class AuthenticationError(AppException):
    """Raised when authentication fails."""
    def __init__(self, message: str = "Authentication required"):
        super().__init__(message, status_code=401)

class PermissionDeniedError(AppException):
    """Raised when user lacks permission."""
    def __init__(self, action: str):
        super().__init__(f"Permission denied: cannot {action}", status_code=403)

class DuplicateError(AppException):
    """Raised when trying to create a duplicate resource."""
    def __init__(self, resource: str, field: str, value: str):
        super().__init__(
            f"{resource} with {field}='{value}' already exists",
            status_code=409
        )

# ─────────────────────────────────────────
# Using custom exceptions in a service
# ─────────────────────────────────────────
class UserService:
    def __init__(self):
        self._users = {}
        self._next_id = 1

    def create_user(self, name: str, email: str) -> dict:
        # Validate
        if not name or len(name) < 2:
            raise ValidationError("Name must be at least 2 characters", field="name")
        if "@" not in email:
            raise ValidationError("Invalid email format", field="email")

        # Check duplicate
        for user in self._users.values():
            if user["email"] == email:
                raise DuplicateError("User", "email", email)

        # Create
        user = {"id": self._next_id, "name": name, "email": email}
        self._users[self._next_id] = user
        self._next_id += 1
        return user

    def get_user(self, user_id: int) -> dict:
        user = self._users.get(user_id)
        if not user:
            raise NotFoundError("User", user_id)
        return user

    def delete_user(self, user_id: int, requesting_user: dict) -> bool:
        if requesting_user.get("role") != "admin":
            raise PermissionDeniedError("delete users")
        if user_id not in self._users:
            raise NotFoundError("User", user_id)
        del self._users[user_id]
        return True

# Using the service
service = UserService()

# Test various exceptions
try:
    user = service.create_user("A", "invalid")  # name too short
except ValidationError as e:
    print(e.to_dict())
    # {'error': 'ValidationError', 'message': 'Name...', 'status_code': 422, 'field': 'name'}

try:
    alice = service.create_user("Alice", "alice@test.com")
    print(alice)
    service.create_user("Alice2", "alice@test.com")  # duplicate email
except DuplicateError as e:
    print(e.to_dict())

try:
    user = service.get_user(999)  # doesn't exist
except NotFoundError as e:
    print(e.to_dict())

# ─────────────────────────────────────────
# Exception chaining — raise from
# ─────────────────────────────────────────
def fetch_user_from_db(user_id: int):
    try:
        raise ConnectionError("Database unreachable")
    except ConnectionError as e:
        # Chain exceptions — shows original cause
        raise NotFoundError("User", user_id) from e

try:
    fetch_user_from_db(1)
except NotFoundError as e:
    print(e)                     # User with id 1 not found
    print(f"Caused by: {e.__cause__}")  # Database unreachable
```

#### Best Practices
- **Inherit from `Exception`, not `BaseException`:** `BaseException` is reserved for system-altering exceptions that you rarely want to catch, like `SystemExit` or `KeyboardInterrupt`.

- **Naming Conventions:** Always end your class name with `Error` (e.g., `ValidationError`, `InvalidTokenError`) to align with Python's naming standards.
- 
- **Organize Your Code:** For larger projects, keep all custom error classes together in a dedicated file like `errors.py` or `exceptions.py` to keep your codebase organized.

---
## 5. File Handling

```python
import json
import csv
from pathlib import Path

# ─────────────────────────────────────────
# TEXT FILES
# ─────────────────────────────────────────
# Write
with open("data.txt", "w") as f:
    f.write("Line 1\n")
    f.write("Line 2\n")
    f.writelines(["Line 3\n", "Line 4\n"])

# Read all at once
with open("data.txt", "r") as f:
    content = f.read()
    print(content)

# Read line by line (memory efficient for large files)
with open("data.txt", "r") as f:
    for line in f:
        print(line.strip())

# Read all lines into list
with open("data.txt", "r") as f:
    lines = f.readlines()  # ['Line 1\n', 'Line 2\n', ...]

# Append to file
with open("data.txt", "a") as f:
    f.write("Line 5\n")

# File modes:
# "r"  → read (default)
# "w"  → write (overwrites!)
# "a"  → append
# "rb" → read binary
# "wb" → write binary
# "r+" → read + write

# ─────────────────────────────────────────
# JSON FILES — you'll use this ALL the time
# ─────────────────────────────────────────
# Writing JSON
data = {
    "users": [
        {"id": 1, "name": "Alice", "email": "alice@test.com"},
        {"id": 2, "name": "Bob", "email": "bob@test.com"},
    ],
    "total": 2,
    "page": 1
}

with open("data.json", "w") as f:
    json.dump(data, f, indent=2)  # indent makes it pretty

# Reading JSON
with open("data.json", "r") as f:
    loaded = json.load(f)
    print(loaded["users"][0]["name"])  # Alice

# JSON to/from string (not file)
json_string = json.dumps(data, indent=2)  # dict → string
parsed = json.loads(json_string)          # string → dict

# Handling non-serializable types
from datetime import datetime

class DateTimeEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        return super().default(obj)

data_with_date = {"name": "Alice", "created_at": datetime.now()}
json_str = json.dumps(data_with_date, cls=DateTimeEncoder)
print(json_str)

# ─────────────────────────────────────────
# CSV FILES — common for data export/import
# ─────────────────────────────────────────
# Writing CSV
users = [
    {"name": "Alice", "email": "alice@test.com", "age": 25},
    {"name": "Bob", "email": "bob@test.com", "age": 30},
]

with open("users.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "email", "age"])
    writer.writeheader()
    writer.writerows(users)

# Reading CSV
with open("users.csv", "r") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(dict(row))

# ─────────────────────────────────────────
# pathlib — modern file path handling
# ─────────────────────────────────────────
from pathlib import Path

# Create path objects
base = Path(".")
data_dir = base / "data"       # / operator joins paths
log_file = data_dir / "app.log"

print(log_file)        # data/app.log
print(log_file.suffix) # .log
print(log_file.stem)   # app
print(log_file.parent) # data
print(log_file.name)   # app.log

# Create directories
data_dir.mkdir(exist_ok=True)  # create if doesn't exist

# Check existence
print(data_dir.exists())   # True
print(log_file.exists())   # False

# List directory contents
for path in Path(".").iterdir():
    if path.is_file():
        print(f"File: {path.name}")
    elif path.is_dir():
        print(f"Dir: {path.name}/")

# Find all Python files recursively
python_files = list(Path(".").rglob("*.py"))
print(python_files)

# Read/write with Path
log_file.write_text("Application started\n")
content = log_file.read_text()
print(content)

# Cleanup
import os
os.remove("data.txt")
os.remove("data.json")
os.remove("users.csv")
```

---

## 6. Regular Expressions

```python
import re

# Regex = pattern matching for strings
# Used for: validation, parsing, searching, replacing

# ─────────────────────────────────────────
# BASIC PATTERNS
# ─────────────────────────────────────────
# .    → any character except newline
# \d   → digit (0-9)
# \D   → not digit
# \w   → word char (a-z, A-Z, 0-9, _)
# \W   → not word char
# \s   → whitespace (space, tab, newline)
# \S   → not whitespace
# ^    → start of string
# $    → end of string
# []   → character class [abc] = a, b, or c
# [^]  → NOT in class [^abc] = not a, b, or c

# ─────────────────────────────────────────
# QUANTIFIERS
# ─────────────────────────────────────────
# *    → 0 or more
# +    → 1 or more
# ?    → 0 or 1 (optional)
# {n}  → exactly n
# {n,m}→ between n and m
# {n,} → n or more

# ─────────────────────────────────────────
# MAIN FUNCTIONS
# ─────────────────────────────────────────
text = "Contact us at support@example.com or sales@company.org"

# re.search → find FIRST match anywhere in string
match = re.search(r"\w+@\w+\.\w+", text)
if match:
    print(match.group())   # support@example.com
    print(match.start())   # 14 (start position)
    print(match.end())     # 33 (end position)

# re.findall → find ALL matches, return list
emails = re.findall(r"[\w.+-]+@[\w-]+\.\w+", text)
print(emails)  # ['support@example.com', 'sales@company.org']

# re.match → match at BEGINNING of string only
result = re.match(r"Contact", text)
print(result)  # Match object (found at beginning)

result = re.match(r"support", text)
print(result)  # None (not at beginning)

# re.fullmatch → entire string must match
result = re.fullmatch(r"\d{4}", "2024")
print(result)  # Match (entire string is 4 digits)

result = re.fullmatch(r"\d{4}", "2024abc")
print(result)  # None

# re.sub → replace matches
phone = "Call us: (555) 123-4567 or 555.987.6543"
# Remove all non-digit characters from phone number
digits_only = re.sub(r"[^\d]", "", "(555) 123-4567")
print(digits_only)  # 5551234567

# re.split → split by pattern
text = "one,two;three|four"
parts = re.split(r"[,;|]", text)
print(parts)  # ['one', 'two', 'three', 'four']

# ─────────────────────────────────────────
# GROUPS — capture parts of the match
# ─────────────────────────────────────────
# () creates a group
log_line = "2024-01-15 14:32:01 ERROR User 42 not found"
pattern = r"(\d{4}-\d{2}-\d{2}) (\d{2}:\d{2}:\d{2}) (\w+) (.+)"

match = re.search(pattern, log_line)
if match:
    print(match.group(0))  # entire match
    print(match.group(1))  # 2024-01-15
    print(match.group(2))  # 14:32:01
    print(match.group(3))  # ERROR
    print(match.group(4))  # User 42 not found
    print(match.groups())  # all groups as tuple

# Named groups (?P<name>pattern)
pattern = r"(?P<date>\d{4}-\d{2}-\d{2}) (?P<time>\d{2}:\d{2}:\d{2}) (?P<level>\w+)"
match = re.search(pattern, log_line)
if match:
    print(match.group("date"))   # 2024-01-15
    print(match.group("level"))  # ERROR
    print(match.groupdict())     # {'date': '...', 'time': '...', 'level': '...'}

# ─────────────────────────────────────────
# COMPILE — reuse patterns for performance
# ─────────────────────────────────────────
# When using same pattern many times, compile it once
EMAIL_PATTERN = re.compile(r"^[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}$")
PHONE_PATTERN = re.compile(r"^\+?1?\s*[\(]?\d{3}[\)]?[-\s.]?\d{3}[-\s.]?\d{4}$")
UUID_PATTERN = re.compile(
    r"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$",
    re.IGNORECASE
)

def validate_email(email: str) -> bool:
    return bool(EMAIL_PATTERN.match(email))

def validate_phone(phone: str) -> bool:
    return bool(PHONE_PATTERN.match(phone))

def validate_uuid(uuid: str) -> bool:
    return bool(UUID_PATTERN.match(uuid))

print(validate_email("alice@test.com"))           # True
print(validate_email("not-an-email"))             # False
print(validate_phone("+1 (555) 123-4567"))        # True
print(validate_uuid("550e8400-e29b-41d4-a716-446655440000"))  # True

# ─────────────────────────────────────────
# FLAGS
# ─────────────────────────────────────────
# re.IGNORECASE (re.I) → case insensitive
# re.MULTILINE (re.M)  → ^ and $ match line starts/ends
# re.DOTALL (re.S)     → . matches newlines too

text = "Hello\nWorld\nPython"
lines_starting_with = re.findall(r"^\w+", text, re.MULTILINE)
print(lines_starting_with)  # ['Hello', 'World', 'Python']
```

---

## 7. Collections Module

```python
from collections import Counter, defaultdict, OrderedDict, namedtuple, deque

# ─────────────────────────────────────────
# COUNTER — count occurrences
# ─────────────────────────────────────────
# From a list
words = ["python", "java", "python", "rust", "python", "java"]
counter = Counter(words)
print(counter)                           # Counter({'python': 3, 'java': 2, 'rust': 1})
print(counter["python"])                 # 3
print(counter["nonexistent"])            # 0 (not KeyError!)
print(counter.most_common(2))            # [('python', 3), ('java', 2)]

# From a string
char_count = Counter("hello world")
print(char_count)  # Counter({' ': 1, 'h': 1, 'e': 1, 'l': 3, ...})

# Counter arithmetic
c1 = Counter({"python": 3, "java": 1})
c2 = Counter({"python": 1, "rust": 2})
print(c1 + c2)  # Counter({'python': 4, 'rust': 2, 'java': 1})
print(c1 - c2)  # Counter({'python': 2, 'java': 1})
print(c1 & c2)  # intersection: Counter({'python': 1})
print(c1 | c2)  # union: Counter({'python': 3, 'rust': 2, 'java': 1})

# Real backend use: analytics
def analyze_api_calls(logs: list) -> dict:
    endpoints = Counter(log["endpoint"] for log in logs)
    methods = Counter(log["method"] for log in logs)
    status_codes = Counter(log["status"] for log in logs)
    return {
        "top_endpoints": endpoints.most_common(5),
        "methods": dict(methods),
        "status_codes": dict(status_codes)
    }

logs = [
    {"endpoint": "/users", "method": "GET", "status": 200},
    {"endpoint": "/posts", "method": "POST", "status": 201},
    {"endpoint": "/users", "method": "GET", "status": 200},
    {"endpoint": "/users", "method": "DELETE", "status": 403},
]
print(analyze_api_calls(logs))

# ─────────────────────────────────────────
# DEFAULTDICT — dict with default values
# ─────────────────────────────────────────
# Normal dict raises KeyError for missing keys
# defaultdict provides a default value automatically

# Group items by category
from collections import defaultdict

users = [
    {"name": "Alice", "role": "admin"},
    {"name": "Bob", "role": "user"},
    {"name": "Charlie", "role": "admin"},
    {"name": "Dave", "role": "user"},
    {"name": "Eve", "role": "moderator"},
]

# Normal dict — requires check
by_role_normal = {}
for user in users:
    role = user["role"]
    if role not in by_role_normal:
        by_role_normal[role] = []
    by_role_normal[role].append(user["name"])

# defaultdict — cleaner
by_role = defaultdict(list)
for user in users:
    by_role[user["role"]].append(user["name"])

print(dict(by_role))
# {'admin': ['Alice', 'Charlie'], 'user': ['Bob', 'Dave'], 'moderator': ['Eve']}

# defaultdict with int (for counting)
tag_counts = defaultdict(int)
posts = ["python fastapi", "python django", "fastapi tutorial"]
for post in posts:
    for tag in post.split():
        tag_counts[tag] += 1

print(dict(tag_counts))  # {'python': 2, 'fastapi': 2, 'django': 1, 'tutorial': 1}

# defaultdict with defaultdict (nested)
nested = defaultdict(lambda: defaultdict(list))
nested["users"]["admins"].append("Alice")
nested["users"]["regular"].append("Bob")
print(dict(nested["users"]))  # {'admins': ['Alice'], 'regular': ['Bob']}

# ─────────────────────────────────────────
# NAMEDTUPLE — tuple with named fields
# ─────────────────────────────────────────
# Lightweight alternative to dataclass for simple data

Point = namedtuple("Point", ["x", "y"])
User = namedtuple("User", ["id", "name", "email", "role"])
HTTPResponse = namedtuple("HTTPResponse", ["status_code", "body", "headers"])

# Creating
point = Point(10, 20)
user = User(1, "Alice", "alice@test.com", "admin")
response = HTTPResponse(200, {"data": []}, {"Content-Type": "application/json"})

# Access by name OR index
print(point.x)    # 10
print(point[0])   # 10
print(user.name)  # Alice
print(user.role)  # admin

# Unpack like a regular tuple
x, y = point
print(x, y)  # 10 20

# namedtuples are IMMUTABLE (like regular tuples)
try:
    user.name = "Bob"  # AttributeError
except AttributeError as e:
    print(e)

# _replace → create new tuple with some fields changed
admin_user = user._replace(role="user")
print(admin_user)  # User(id=1, name='Alice', email='alice@test.com', role='user')

# _asdict → convert to dict
print(user._asdict())  # {'id': 1, 'name': 'Alice', 'email': 'alice@test.com', 'role': 'admin'}

# ─────────────────────────────────────────
# DEQUE — double-ended queue
# ─────────────────────────────────────────
# Like a list but FAST at both ends: O(1) vs O(n) for list
from collections import deque

# Initialize
dq = deque([1, 2, 3, 4, 5])
dq_limited = deque(maxlen=3)  # max size 3 — auto-removes oldest

# Add/remove from BOTH ends efficiently
dq.append(6)        # add to right: [1,2,3,4,5,6]
dq.appendleft(0)    # add to left: [0,1,2,3,4,5,6]
dq.pop()            # remove from right: [0,1,2,3,4,5]
dq.popleft()        # remove from left: [1,2,3,4,5]

# Rotate
dq.rotate(2)       # rotate right by 2
print(list(dq))    # [4, 5, 1, 2, 3]
dq.rotate(-2)      # rotate left by 2
print(list(dq))    # [1, 2, 3, 4, 5]

# maxlen deque — keeps only last N items (sliding window)
recent_requests = deque(maxlen=5)
for i in range(10):
    recent_requests.append(f"request_{i}")
print(f"Recent: {list(recent_requests)}")  # Always keeps last 5 requests

# Real use: rolling log of last N events
# request_log, error_history, recent_searches
```

---

## 8. `Functools` Module

```python
from functools import lru_cache, partial, wraps, reduce, cached_property

# ─────────────────────────────────────────
# lru_cache — memoization (cache results)
# ─────────────────────────────────────────
# LRU = Least Recently Used
# Caches function results — same inputs → return cached result
# Huge performance boost for expensive repeated calls

@lru_cache(maxsize=128)  # cache up to 128 results
def fibonacci(n: int) -> int:
    """Calculate fibonacci number — slow without cache."""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

import time

# Without cache: fibonacci(35) takes ~several seconds
# With cache: fibonacci(35) is instant after first call
start = time.perf_counter()
print(fibonacci(35))  # 9227465
print(f"First call: {(time.perf_counter()-start)*1000:.2f}ms")

start = time.perf_counter()
print(fibonacci(35))  # 9227465 (from cache)
print(f"Second call: {(time.perf_counter()-start)*1000:.2f}ms")
# Second call: 0.001ms ← from cache!

# View cache stats
print(fibonacci.cache_info())  # CacheInfo(hits=34, misses=36, maxsize=128, currsize=36)

# Clear cache
fibonacci.cache_clear()

# Real backend use:
@lru_cache(maxsize=256)
def get_country_by_code(code: str) -> dict:
    """Look up country by ISO code — cached after first lookup."""
    # In reality: database or API call
    countries = {
        "US": {"name": "United States", "currency": "USD"},
        "GB": {"name": "United Kingdom", "currency": "GBP"},
        "DE": {"name": "Germany", "currency": "EUR"},
    }
    return countries.get(code.upper())

# ⚠️ lru_cache only works with HASHABLE arguments
# Can't use dicts or lists as arguments
# For methods on classes use cached_property

class UserProfile:
    def __init__(self, user_id: int):
        self.user_id = user_id

    @cached_property
    def permissions(self) -> list:
        """Computed once, cached for the lifetime of the object."""
        print("Computing permissions...")  # only runs once!
        return ["read", "write", "comment"]

profile = UserProfile(1)
print(profile.permissions)  # Computing permissions...
print(profile.permissions)  # (from cache — no print)

# ─────────────────────────────────────────
# partial — freeze some arguments
# ─────────────────────────────────────────
# Create a new function with some arguments pre-filled

def send_notification(recipient: str, message: str, channel: str = "email"):
    return f"[{channel.upper()}] to {recipient}: {message}"

# Pre-fill the channel
send_email = partial(send_notification, channel="email")
send_sms = partial(send_notification, channel="sms")
send_push = partial(send_notification, channel="push")

print(send_email("alice@test.com", "Your order shipped!"))
# [EMAIL] to alice@test.com: Your order shipped!
print(send_sms("+1-555-0123", "Order shipped!"))
# [SMS] to +1-555-0123: Order shipped!

# Real use: pre-configure functions for specific contexts
import json

# Pre-configured JSON dumper with indent
pretty_json = partial(json.dumps, indent=2, sort_keys=True)
compact_json = partial(json.dumps, separators=(",", ":"))

data = {"name": "Alice", "age": 25}
print(pretty_json(data))   # pretty formatted
print(compact_json(data))  # {"age":25,"name":"Alice"}

# ─────────────────────────────────────────
# reduce — already covered in 1.2
# ─────────────────────────────────────────
from functools import reduce

# Find total revenue from orders
orders = [
    {"amount": 99.99, "items": 3},
    {"amount": 149.50, "items": 1},
    {"amount": 24.99, "items": 2},
]
total = reduce(lambda acc, order: acc + order["amount"], orders, 0)
print(f"Total revenue: ${total:.2f}")  # Total revenue: $274.48

# Merge list of dicts
configs = [
    {"host": "localhost", "debug": True},
    {"port": 8000, "workers": 4},
    {"log_level": "INFO"},
]
merged = reduce(lambda a, b: {**a, **b}, configs)
print(merged)
```

---

## 9. `Itertools` Module

```python
import itertools

# ─────────────────────────────────────────
# INFINITE ITERATORS
# ─────────────────────────────────────────
# count → counts from n forever
counter = itertools.count(start=1, step=2)  # 1, 3, 5, 7...
for _ in range(5):
    print(next(counter), end=" ")  # 1 3 5 7 9
print()

# cycle → repeats sequence forever
colors = itertools.cycle(["red", "green", "blue"])
for _ in range(7):
    print(next(colors), end=" ")  # red green blue red green blue red
print()

# repeat → repeat value N times (or forever)
print(list(itertools.repeat("hello", 3)))  # ['hello', 'hello', 'hello']

# ─────────────────────────────────────────
# COMBINATORICS
# ─────────────────────────────────────────
items = ["A", "B", "C"]

# permutations — ordered arrangements
perms = list(itertools.permutations(items))
print(perms)  # [('A','B','C'), ('A','C','B'), ('B','A','C')...]
print(f"Total permutations: {len(perms)}")  # 6

# permutations of length 2
perms2 = list(itertools.permutations(items, 2))
print(perms2)  # [('A','B'), ('A','C'), ('B','A'), ('B','C')...]

# combinations — order doesn't matter, no repeats
combos = list(itertools.combinations(items, 2))
print(combos)  # [('A','B'), ('A','C'), ('B','C')]

# combinations_with_replacement — can repeat
combos_r = list(itertools.combinations_with_replacement(items, 2))
print(combos_r)  # [('A','A'), ('A','B'), ('A','C'), ('B','B')...]

# product — cartesian product
sizes = ["S", "M", "L"]
colors = ["red", "blue"]
variants = list(itertools.product(sizes, colors))
print(variants)
# [('S','red'), ('S','blue'), ('M','red'), ('M','blue'), ('L','red'), ('L','blue')]

# Real backend use: generate all test cases
http_methods = ["GET", "POST", "PUT", "DELETE"]
endpoints = ["/users", "/posts", "/comments"]
test_cases = list(itertools.product(http_methods, endpoints))
print(f"Total test cases: {len(test_cases)}")  # 12

# ─────────────────────────────────────────
# SLICING AND GROUPING
# ─────────────────────────────────────────
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# chain → combine multiple iterables
a = [1, 2, 3]
b = [4, 5, 6]
c = [7, 8, 9]
print(list(itertools.chain(a, b, c)))  # [1,2,3,4,5,6,7,8,9]

# chain.from_iterable
lists = [[1, 2], [3, 4], [5, 6]]
print(list(itertools.chain.from_iterable(lists)))  # [1,2,3,4,5,6]

# islice → slice an iterator (lazy)
gen = (x ** 2 for x in range(1000000))
first_five = list(itertools.islice(gen, 5))
print(first_five)  # [0, 1, 4, 9, 16]

# groupby → group consecutive items by key
# NOTE: data must be SORTED by the key first!
data = [
    {"name": "Alice", "dept": "Engineering"},
    {"name": "Bob", "dept": "Engineering"},
    {"name": "Charlie", "dept": "Marketing"},
    {"name": "Dave", "dept": "Marketing"},
    {"name": "Eve", "dept": "Engineering"},  # ← back to Engineering
]

# Sort first!
sorted_data = sorted(data, key=lambda x: x["dept"])
for dept, members in itertools.groupby(sorted_data, key=lambda x: x["dept"]):
    names = [m["name"] for m in members]
    print(f"{dept}: {names}")
# Engineering: ['Alice', 'Bob', 'Eve']
# Marketing: ['Charlie', 'Dave']

# takewhile / dropwhile
nums = [2, 4, 6, 7, 8, 10]
even_prefix = list(itertools.takewhile(lambda x: x % 2 == 0, nums))
print(even_prefix)  # [2, 4, 6] — stops at first odd

after_odd = list(itertools.dropwhile(lambda x: x % 2 == 0, nums))
print(after_odd)  # [7, 8, 10] — drops evens until first odd

# zip_longest — zip with fill for unequal lengths
a = [1, 2, 3, 4, 5]
b = ["a", "b", "c"]
print(list(itertools.zip_longest(a, b, fillvalue=None)))
# [(1,'a'), (2,'b'), (3,'c'), (4,None), (5,None)]
```

---

## 10. Typing Module

```python
from typing import (
    Optional, Union, List, Dict, Tuple, Set, Any,
    Callable, TypeVar, Generic, Literal, Final, ClassVar,
    Protocol, runtime_checkable
)
from typing import TYPE_CHECKING

# ─────────────────────────────────────────
# BASICS (you've seen these)
# ─────────────────────────────────────────
def get_user(user_id: int) -> Optional[Dict[str, Any]]:
    pass

def process(value: Union[int, str, float]) -> str:
    return str(value)

# Python 3.10+ cleaner syntax:
def get_user(user_id: int) -> dict | None:  # instead of Optional[dict]
    pass

def process(value: int | str | float) -> str:  # instead of Union[int, str, float]
    return str(value)

# ─────────────────────────────────────────
# LITERAL — restrict to specific values
# ─────────────────────────────────────────
from typing import Literal

Role = Literal["user", "admin", "moderator"]
HTTPMethod = Literal["GET", "POST", "PUT", "PATCH", "DELETE"]
Status = Literal["active", "inactive", "banned", "pending"]

def set_role(user_id: int, role: Role) -> None:
    print(f"Setting {user_id} to {role}")

set_role(1, "admin")  # fine
# set_role(1, "superuser")  # type checker will warn!

def route(method: HTTPMethod, path: str) -> None:
    pass

# ─────────────────────────────────────────
# TYPEVAR — generic functions
# ─────────────────────────────────────────
T = TypeVar("T")

def first_item(items: List[T]) -> Optional[T]:
    """Return first item of any list type."""
    return items[0] if items else None

# Type checker knows:
result1: int = first_item([1, 2, 3])     # T is int
result2: str = first_item(["a", "b"])    # T is str

# ─────────────────────────────────────────
# PROTOCOL — structural typing (duck typing)
# ─────────────────────────────────────────
@runtime_checkable
class Serializable(Protocol):
    """Any class with a to_dict method."""
    def to_dict(self) -> dict:
        ...

class User:
    def to_dict(self) -> dict:
        return {"name": "Alice"}

class Product:
    def to_dict(self) -> dict:
        return {"name": "Laptop", "price": 999}

class InvalidClass:
    pass  # no to_dict

def serialize(obj: Serializable) -> str:
    import json
    return json.dumps(obj.to_dict())

print(serialize(User()))     # {"name": "Alice"}
print(serialize(Product()))  # {"name": "Laptop", "price": 999}

# Runtime check with isinstance
print(isinstance(User(), Serializable))       # True
print(isinstance(InvalidClass(), Serializable))  # False

# ─────────────────────────────────────────
# CALLABLE — type hint for functions
# ─────────────────────────────────────────
# Callable[[arg_types], return_type]
from typing import Callable

def apply_twice(func: Callable[[int], int], value: int) -> int:
    return func(func(value))

def double(x: int) -> int:
    return x * 2

print(apply_twice(double, 3))  # 12

# Middleware type:
RequestHandler = Callable[[dict], dict]

def add_logging(handler: RequestHandler) -> RequestHandler:
    def logged_handler(request: dict) -> dict:
        print(f"Request: {request}")
        response = handler(request)
        print(f"Response: {response}")
        return response
    return logged_handler

# ─────────────────────────────────────────
# FINAL — value that shouldn't change
# ─────────────────────────────────────────
from typing import Final

MAX_CONNECTIONS: Final = 100
DATABASE_URL: Final[str] = "postgresql://localhost/mydb"
API_VERSION: Final = "v1"

# ─────────────────────────────────────────
# TYPE_CHECKING — avoid circular imports
# ─────────────────────────────────────────
# Sometimes you need types only for type hints
# not at runtime — use TYPE_CHECKING

if TYPE_CHECKING:
    from some_module import SomeClass  # only imported during type checking
    # not imported at runtime → no circular import issues
```

---

## 11. Enums

```python
from enum import Enum, IntEnum, Flag, auto

# ─────────────────────────────────────────
# BASIC ENUM
# ─────────────────────────────────────────
class UserRole(Enum):
    GUEST = "guest"
    USER = "user"
    MODERATOR = "moderator"
    ADMIN = "admin"

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"
    REFUNDED = "refunded"

# Using enums
role = UserRole.ADMIN
print(role)           # UserRole.ADMIN
print(role.name)      # ADMIN
print(role.value)     # admin

# Comparison
print(role == UserRole.ADMIN)  # True
print(role == "admin")          # False (not equal to string!)

# Get by value
role_from_db = UserRole("admin")  # get enum from string value
print(role_from_db)  # UserRole.ADMIN

# Get by name
role_by_name = UserRole["ADMIN"]
print(role_by_name)  # UserRole.ADMIN

# Iterate all values
print([r.value for r in UserRole])
# ['guest', 'user', 'moderator', 'admin']

# ─────────────────────────────────────────
# auto() — auto-assign values
# ─────────────────────────────────────────
class Priority(Enum):
    LOW = auto()       # 1
    MEDIUM = auto()    # 2
    HIGH = auto()      # 3
    CRITICAL = auto()  # 4

print(Priority.HIGH.value)  # 3

# ─────────────────────────────────────────
# IntEnum — enum that behaves like int
# ─────────────────────────────────────────
class StatusCode(IntEnum):
    OK = 200
    CREATED = 201
    BAD_REQUEST = 400
    UNAUTHORIZED = 401
    FORBIDDEN = 403
    NOT_FOUND = 404
    SERVER_ERROR = 500

# Can compare with integers directly
print(StatusCode.OK == 200)  # True (unlike regular Enum)
print(StatusCode.NOT_FOUND > 200)  # True
print(StatusCode.NOT_FOUND < 200)  # False

# ─────────────────────────────────────────
# Flag — combinable bit flags
# ─────────────────────────────────────────
class Permission(Flag):
    READ = auto()
    WRITE = auto()
    DELETE = auto()
    ADMIN = auto()

    # Combination
    EDITOR = READ | WRITE
    SUPER_ADMIN = READ | WRITE | DELETE | ADMIN

# Combine permissions
user_perms = Permission.READ | Permission.WRITE
print(user_perms)  # Permission.READ|WRITE

# Check if permission is included
print(Permission.READ in user_perms)      # True
print(Permission.DELETE in user_perms)    # False
print(Permission.EDITOR in user_perms)    # True (READ | WRITE)

# ─────────────────────────────────────────
# Enum with methods
# ─────────────────────────────────────────
class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

    def can_cancel(self) -> bool:
        """Check if order can be cancelled."""
        return self in (OrderStatus.PENDING, OrderStatus.CONFIRMED)

    def can_ship(self) -> bool:
        return self == OrderStatus.CONFIRMED

    @property
    def is_final(self) -> bool:
        return self in (OrderStatus.DELIVERED, OrderStatus.CANCELLED)

    def next_status(self) -> "OrderStatus":
        transitions = {
            OrderStatus.PENDING: OrderStatus.CONFIRMED,
            OrderStatus.CONFIRMED: OrderStatus.SHIPPED,
            OrderStatus.SHIPPED: OrderStatus.DELIVERED,
        }
        return transitions.get(self)

status = OrderStatus.PENDING
print(status.can_cancel())      # True
print(status.can_ship())        # False
print(status.is_final)          # False
print(status.next_status())     # OrderStatus.CONFIRMED

status = OrderStatus.DELIVERED
print(status.can_cancel())      # False
print(status.is_final)          # True
print(status.next_status())     # None
```

---

## 12. Python 3.10+ Features

```python
# ─────────────────────────────────────────
# WALRUS OPERATOR := (Python 3.8+)
# "assignment expression"
# ─────────────────────────────────────────
# Assigns AND uses a value in one expression

# Without walrus — read file and check length separately
data = [1, 2, 3, 4, 5]
length = len(data)
if length > 3:
    print(f"Long list: {length} items")

# With walrus — compute and use in one line
if (length := len(data)) > 3:
    print(f"Long list: {length} items")

# In while loops — very natural
import re

text = "name: Alice\nage: 25\ncity: NYC"

# Without walrus:
lines = text.split("\n")
for line in lines:
    match = re.match(r"(\w+): (.+)", line)
    if match:
        print(f"Field: {match.group(1)}, Value: {match.group(2)}")

# With walrus — cleaner
for line in text.split("\n"):
    if match := re.match(r"(\w+): (.+)", line):
        print(f"Field: {match.group(1)}, Value: {match.group(2)}")

# In list comprehensions
data = range(-5, 6)
processed = [y for x in data if (y := x * 2) > 10]
print(processed)  # [16, 25, 16, 25] — computed once, used twice

# ─────────────────────────────────────────
# MATCH-CASE (Python 3.10+)
# Structural pattern matching
# Like switch/case but MUCH more powerful
# ─────────────────────────────────────────

def handle_command(command: dict) -> str:
    match command:
        # Match by value
        case {"action": "quit"}:
            return "Goodbye!"

        # Match with variable capture
        case {"action": "greet", "name": name}:
            return f"Hello, {name}!"

        # Match with condition (guard)
        case {"action": "deposit", "amount": amount} if amount > 0:
            return f"Deposited ${amount}"

        case {"action": "deposit", "amount": amount}:
            return f"Invalid amount: {amount}"

        # Match a list pattern
        case {"action": "bulk_create", "users": [first, *rest]}:
            return f"Creating {1 + len(rest)} users, first: {first}"

        # Match any (wildcard)
        case _:
            return f"Unknown command: {command}"

print(handle_command({"action": "quit"}))
# Goodbye!
print(handle_command({"action": "greet", "name": "Alice"}))
# Hello, Alice!
print(handle_command({"action": "deposit", "amount": 100}))
# Deposited $100
print(handle_command({"action": "deposit", "amount": -50}))
# Invalid amount: -50
print(handle_command({"action": "bulk_create", "users": ["Alice", "Bob", "Charlie"]}))
# Creating 3 users, first: Alice
print(handle_command({"action": "unknown"}))
# Unknown command: {'action': 'unknown'}

# Match with classes
class HTTPRequest:
    def __init__(self, method: str, path: str, body: dict = None):
        self.method = method
        self.path = path
        self.body = body or {}

def route_request(request: HTTPRequest) -> str:
    match request:
        case HTTPRequest(method="GET", path="/users"):
            return "List all users"
        case HTTPRequest(method="POST", path="/users", body=body):
            return f"Create user: {body}"
        case HTTPRequest(method="GET", path=path) if path.startswith("/users/"):
            user_id = path.split("/")[-1]
            return f"Get user {user_id}"
        case HTTPRequest(method="DELETE", path=path):
            return f"Delete resource at {path}"
        case _:
            return "404 Not Found"

req1 = HTTPRequest("GET", "/users")
req2 = HTTPRequest("POST", "/users", {"name": "Alice"})
req3 = HTTPRequest("GET", "/users/42")

print(route_request(req1))  # List all users
print(route_request(req2))  # Create user: {'name': 'Alice'}
print(route_request(req3))  # Get user 42
```

---

## Visual Summary

```
┌────────────────────────────────────────────────────────────────┐
│                   ADVANCED PYTHON OVERVIEW                     │
│                                                                │
│  GENERATORS                    DECORATORS                      │
│  ──────────                    ──────────                      │
│  yield → pause/resume          @functools.wraps                │
│  lazy evaluation               wrap behavior around functions  │
│  memory efficient              logging, auth, retry, cache     │
│  (x for x in y)                stack with multiple @           │
│                                                                │
│  CONTEXT MANAGERS              EXCEPTIONS                      │
│  ────────────────              ──────────                      │
│  with statement                try/except/else/finally         │
│  __enter__/__exit__            custom exception hierarchy      │
│  @contextmanager               raise X from Y (chaining)       │
│  guaranteed cleanup            specific before general         │
│                                                                │
│  COLLECTIONS                   FUNCTOOLS                       │
│  ───────────                   ─────────                       │
│  Counter → count               lru_cache → memoize            │
│  defaultdict → default         partial → freeze args          │
│  namedtuple → named data       reduce → collapse to one       │
│  deque → fast both ends        wraps → preserve metadata      │
│                                                                │
│  TYPING                        ENUMS                           │
│  ──────                        ─────                           │
│  Optional, Union               Enum, IntEnum, Flag            │
│  Literal, Final                auto() values                  │
│  Protocol → duck type          methods on enums               │
│  TypeVar → generics            value/name access              │
│                                                                │
│  NEW FEATURES:                                                 │
│  := walrus operator → assign in expression                    │
│  match/case → structural pattern matching                     │
└────────────────────────────────────────────────────────────────┘
```

---

## Knowledge Check

1. What is the difference between a list and a generator?
2. What does `yield` do? How is it different from `return`?
3. What does `@functools.wraps` do and why is it important?
4. What are the three levels of a decorator with arguments?
5. What is `__enter__` and `__exit__` used for?
6. What is the difference between `except Exception` and `except:`?
7. When would you use `defaultdict` over a regular dict?
8. What is `lru_cache` and when would you use it?
9. What is the difference between `Enum` and `IntEnum`?
10. What does the walrus operator `:=` do?
11. What is `Protocol` and how does it differ from ABC?
12. What does `partial()` do?

---

## Practice Exercises

```python
# Exercise 1: Generator Pipeline
# Write a generator pipeline that:
# 1. Generates numbers 1 to 1,000,000
# 2. Filters only even numbers (generator)
# 3. Squares them (generator)
# 4. Takes only the first 10 (itertools.islice)
# All steps should be lazy (generators, not lists)
# Print the total memory used vs a list approach

# Exercise 2: Decorator Stack
# Build these two decorators:
#   @validate_input → checks function args are not None
#   @log_execution → logs function name, args, result, duration
# Apply BOTH to a function: create_order(user_id, product_id, quantity)
# Test with valid and invalid inputs

# Exercise 3: Context Manager
# Build a context manager called retry_context(max_attempts, delay)
# That automatically retries the code block on exception
# Usage:
#   with retry_context(max_attempts=3, delay=0.1) as ctx:
#       response = requests.get("https://flaky-api.com")

# Exercise 4: Custom Exception Hierarchy
# Build a complete exception hierarchy for a payment system:
#   PaymentError (base)
#     ├── InsufficientFundsError(amount, balance)
#     ├── InvalidCardError(card_number)
#     ├── PaymentDeclinedError(reason)
#     └── TransactionLimitError(limit, attempted)
# Each should have a to_dict() method
# Write a process_payment() function that raises appropriate exceptions

# Exercise 5: Enum State Machine
# Create an OrderStatus enum with states:
#   CREATED → PAID → PROCESSING → SHIPPED → DELIVERED
#         → CANCELLED (from CREATED or PAID only)
# Add methods:
#   can_transition_to(new_status) → bool
#   transition_to(new_status) → OrderStatus (or raise InvalidTransitionError)
#   available_transitions() → List[OrderStatus]
```

---

## Phase 1.4 Complete!
**You now know:**
```
Iterators and generators (yield, send, yield from)
Generator expressions
Decorators (function, class-based, with arguments, stacking)
Context managers (__enter__/__exit__, @contextmanager)
Full exception handling (custom exceptions, chaining)
File handling (text, JSON, CSV, pathlib)
Regular expressions (patterns, groups, compile)
Collections (Counter, defaultdict, namedtuple, deque)
Functools (lru_cache, partial, reduce, cached_property)
Itertools (chain, groupby, combinations, product)
Typing module (Literal, Protocol, TypeVar, Callable)
Enums (Enum, IntEnum, Flag, methods)
Walrus operator and match-case
```

---