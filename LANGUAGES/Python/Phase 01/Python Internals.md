## Why learn internals as a beginner?
You don't need to know HOW a car engine works to drive. But if you want to be a MECHANIC - you do.

As a backend engineer you will face:
  - "Why is my code so slow?"
  - "Why is my server running out of memory?"
  - "Why did this variable change when I didn't touch it?"
  - "Why is this code thread-safe but that one isn't?"
  - "What's the difference between is and == really?"

Internals answer ALL of these questions.
They separate engineers who guess from engineers who KNOW.

---
## Setup
```bash
cd ~/projects
mkdir python_internals
cd python_internals
python3 -m venv venv
source venv/bin/activate
touch internals.py
code .
```

---
## 1. How Python Executes Code
Python code is executed using a two-step hybrid process where the Python interpreter first compiles human-readable source code into intermediate [bytecode](https://opensource.com/article/18/4/introduction-python-bytecode), which is then dynamically executed by the Python Virtual Machine (PVM). While often labeled an interpreted language, Python does not run raw text directly; it automates a seamless translation pipeline every time a script is triggered
### The Journey from `.py` to Running Program
```
Your file: main.py
    ↓
Step 1: SOURCE CODE (text you write)
    "x = 10 + 5"
    ↓
Step 2: LEXER (tokenizer)
    Breaks code into tokens
    [NAME 'x'] [OP '='] [NUMBER '10'] [OP '+'] [NUMBER '5']
    ↓
Step 3: PARSER
    Builds Abstract Syntax Tree (AST)
    Checks syntax rules
    Assign(
        target=Name(id='x'),
        value=BinOp(
            left=Constant(value=10),
            op=Add(),
            right=Constant(value=5)
        )
    )
    ↓
Step 4: COMPILER
    Converts AST to BYTECODE (.pyc files in __pycache__)
    ↓
Step 5: PVM (Python Virtual Machine)
    Executes bytecode instruction by instruction
    RESULT: x = 15
```

### Looking at Bytecode
```python
# internals.py

import dis  # disassembler module

def add_numbers(a, b):
    result = a + b
    return result

# See the bytecode instructions
print("Bytecode for add_numbers")
dis.dis(add_numbers)

# Output:
# 2           0 RESUME              0
# 
# 3           2 LOAD_FAST            0 (a)
#             4 LOAD_FAST            1 (b)
#             6 BINARY_OP            0 (+)
#            10 STORE_FAST           2 (result)
# 
# 4          12 LOAD_FAST            2 (result)
#            14 RETURN_VALUE

# Reading bytecode:
# LOAD_FAST   → push local variable onto stack
# BINARY_OP   → pop two items, apply operation, push result
# STORE_FAST  → pop from stack, store in local variable
# RETURN_VALUE → pop from stack and return it

# Python uses a STACK machine
# Values are pushed onto a stack
# Operations pop values, do something, push result

print("\n=== Bytecode for list comprehension ===")
dis.dis(compile("[x*2 for x in range(10)]", "<string>", "eval"))
```

### The Abstract Syntax Tree
```python
import ast

code = """
def greet(name):
    message = f'Hello, {name}!'
    return message
"""

# Parse into AST
tree = ast.parse(code)

# See the tree structure
print(ast.dump(tree, indent=2))

# AST lets you analyze or transform code programmatically
# Used by: linters, formatters, type checkers, IDEs

# Example: find all function names in code
class FunctionFinder(ast.NodeVisitor):
    def visit_FunctionDef(self, node):
        print(f"Found function: {node.name}")
        self.generic_visit(node)  # visit children

finder = FunctionFinder()
finder.visit(tree)
# Found function: greet
```

### `.pyc` Files and `__pycache__`
```python
# When you run a .py file:
# Python compiles it to bytecode and saves it
# So next run is FASTER (skips compilation if unchanged)

# __pycache__/
#     main.cpython-312.pyc  ← bytecode cache

# The .pyc file contains:
#   1. Magic number (Python version identifier)
#   2. Timestamp/hash of source file
#   3. Bytecode

# Python checks: is source newer than .pyc?
# If yes → recompile
# If no  → use cached .pyc directly

# You can compile manually:
import py_compile
import compileall

py_compile.compile("internals.py")       # compile single file
compileall.compile_dir(".")              # compile entire directory

# This is useful when deploying — pre-compile for faster startup
```

---

## 2. The GIL
The Global Interpreter Lock (GIL) is a mutex (lock) that allows only ONE thread to execute Python bytecode at a time.
Even if you have 8 CPU cores and 8 threads, only ONE thread executes Python code at any moment.

**Why does it exist?**
CPython (the standard Python) uses reference counting for memory management. Reference counting is NOT thread-safe without protection. The GIL protects Python's internal state.

### Visualizing the GIL
```
WITHOUT GIL (theoretical):
  Thread 1: ████████████████ (runs on CPU 1)
  Thread 2: ████████████████ (runs on CPU 2)
  Thread 3: ████████████████ (runs on CPU 3)
  → True parallelism

WITH GIL (actual CPython):
  Thread 1: ████░░░░████░░░░ (runs, waits, runs)
  Thread 2: ░░░░████░░░░████ (waits, runs, waits)
  Thread 3: always waiting...
  → Concurrency, NOT parallelism
  → Only one thread runs Python code at a time
```

```python
import threading
import time

# DEMONSTRATING THE GIL EFFECT

def count_up(n):
    """CPU-bound task."""
    count = 0
    while count < n:
        count += 1
    return count

N = 50_000_000

# ─────────────────────────────────────────
# Single thread — baseline
# ─────────────────────────────────────────
start = time.perf_counter()
count_up(N)
count_up(N)
single_time = time.perf_counter() - start
print(f"Single thread: {single_time:.2f}s")

# ─────────────────────────────────────────
# Two threads — expect speedup? NOPE!
# ─────────────────────────────────────────
start = time.perf_counter()
t1 = threading.Thread(target=count_up, args=(N,))
t2 = threading.Thread(target=count_up, args=(N,))
t1.start()
t2.start()
t1.join()
t2.join()
thread_time = time.perf_counter() - start
print(f"Two threads: {thread_time:.2f}s")

# Result: two threads is NOT faster than single thread
# Because of the GIL — only one runs at a time
# Sometimes even SLOWER due to context switching overhead

# ─────────────────────────────────────────
# BUT: GIL is RELEASED for I/O operations
# ─────────────────────────────────────────
# When a thread does I/O (network, disk):
#   1. Thread releases the GIL
#   2. Other threads can run
#   3. I/O completes, thread reacquires GIL
# This is why threads ARE useful for I/O-bound tasks!

import urllib.request

def fetch_url(url):
    try:
        with urllib.request.urlopen(url, timeout=5) as response:
            return len(response.read())
    except:
        return 0

urls = [
    "https://python.org",
    "https://github.com",
    "https://stackoverflow.com",
]

# Sequential
start = time.perf_counter()
for url in urls:
    fetch_url(url)
seq_time = time.perf_counter() - start
print(f"Sequential I/O: {seq_time:.2f}s")

# Threaded (faster! because GIL released during I/O)
start = time.perf_counter()
threads = [threading.Thread(target=fetch_url, args=(url,)) for url in urls]
for t in threads: t.start()
for t in threads: t.join()
thread_io_time = time.perf_counter() - start
print(f"Threaded I/O: {thread_io_time:.2f}s")
# Threaded is noticeably faster!
```

### GIL Implications for Backend Engineering

```
CPU-BOUND tasks (heavy computation):
  ❌ Threads don't help (GIL prevents parallelism)
  ✅ Use multiprocessing (each process has its OWN GIL)
  ✅ Use async/await (if operations can be non-blocking)

I/O-BOUND tasks (network, database, disk):
  ✅ Threads help (GIL released during I/O)
  ✅ Async/await is even better (no thread overhead)

Backend web servers:
  Most work is I/O-bound (waiting for DB, network)
  → Threading and async work great
  → GIL is rarely a bottleneck in web backends
  → This is why FastAPI/Django work well despite GIL

GIL is being worked on:
  Python 3.12: "no-GIL" experimental mode
  Python 3.13: optional free-threading (PEP 703)
  Future: Python may become truly multi-threaded
```

---

## 3. Memory Management & Garbage Collection

### How Python Stores Objects

```python
# Everything in Python is an OBJECT in memory
# Even integers, booleans, functions

x = 42
print(type(x))  # <class 'int'>
print(id(x))    # memory address of the object

# id() returns the memory address of an object
# Two variables with same id() point to SAME object

a = "hello"
b = "hello"
print(id(a))   # some address
print(id(b))   # SAME address (Python interns short strings)
print(a is b)  # True (same object!)

# Different for larger strings
c = "hello world this is a long string"
d = "hello world this is a long string"
print(id(c))   # some address
print(id(d))   # might be DIFFERENT address
# (but Python sometimes interns these too — implementation detail)
```

### Reference Counting

```python
import sys

# Python tracks how many variables point to each object
# This count is called the REFERENCE COUNT

x = [1, 2, 3]
print(sys.getrefcount(x))  # 2 (x + temporary getrefcount arg)

y = x  # another reference to same object
print(sys.getrefcount(x))  # 3

z = x  # another reference
print(sys.getrefcount(x))  # 4

del y  # remove one reference
print(sys.getrefcount(x))  # 3

del z  # remove another
print(sys.getrefcount(x))  # 2

# When reference count reaches 0:
# Object is IMMEDIATELY deallocated (memory freed)
# No waiting for garbage collector
del x  # reference count → 0 → object freed
# Now [1, 2, 3] is gone from memory
```

### The Garbage Collector

```python
import gc

# Reference counting handles most cases
# BUT fails with CIRCULAR REFERENCES:

class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

# Create circular reference
a = Node(1)
b = Node(2)
a.next = b  # a → b
b.next = a  # b → a (circular!)

# Now even if we delete a and b:
del a
del b
# The objects STILL exist! They reference each other
# Reference count never reaches 0

# Reference counting can't handle this
# Python's CYCLIC GARBAGE COLLECTOR solves this
# It periodically looks for circular reference cycles
# and frees them

# The gc module lets you control this
print(gc.isenabled())      # True (usually)
gc.collect()               # force collection right now

# gc has three "generations"
# Objects that survive collection move to next generation
# Older generations collected less frequently (optimization)
print(gc.get_threshold())  # (700, 10, 10)

# Generation 0: collect after 700 new objects
# Generation 1: collect after gen0 collected 10 times
# Generation 2: collect after gen1 collected 10 times

# For most backend code: you don't touch gc
# Just AVOID circular references where possible
# Or use weakref module for weak references
```

### Small Integer Caching

```python
# Python CACHES small integers (-5 to 256)
# They're created at startup and reused

a = 100
b = 100
print(a is b)    # True (same cached object)
print(id(a))
print(id(b))     # same id!

c = 1000
d = 1000
print(c is d)    # False (too large, separate objects)
print(id(c))
print(id(d))     # different ids

# This is why:
x = 256
y = 256
print(x is y)    # True (cached)

x = 257
y = 257
print(x is y)    # False (not cached)

# KEY LESSON:
# Never use `is` to compare integers or strings
# Always use == for VALUE comparison
# Use `is` ONLY for None, True, False checks

# ✅ Correct:
value = None
if value is None:     # correct way to check None
    pass

flag = True
if flag is True:      # works but `if flag:` is more Pythonic
    pass

# ❌ Wrong:
count = get_count()
if count is 42:       # WRONG! might be False for count==42
    pass
```

### String Interning

```python
import sys

# Python "interns" (caches) some strings
# Strings that look like identifiers (letters, digits, underscores)
# are usually interned

a = "hello"
b = "hello"
print(a is b)  # True (interned)

c = "hello world"  # has a space — may or may not be interned
d = "hello world"
print(c is d)  # implementation-dependent (don't rely on this!)

# Explicit interning
x = sys.intern("my_custom_string")
y = sys.intern("my_custom_string")
print(x is y)  # True (guaranteed)

# When is manual interning useful?
# When you have MILLIONS of repeated strings (e.g., field names)
# Interning saves memory and speeds up comparison

# Real backend use:
# Large datasets with repeated string values
# intern them to save memory

# DON'T rely on string interning behavior for correctness
# It's an implementation detail that can change
```

---

## 4. Mutable vs Immutable Objects

### The Core Concept

```
IMMUTABLE = cannot be changed after creation
  int, float, bool, str, tuple, frozenset, bytes

MUTABLE = can be changed after creation
  list, dict, set, bytearray, custom objects

This distinction affects:
  - How variables behave when assigned
  - What can be used as dict keys
  - Thread safety
  - Function argument behavior
  - Memory usage
```

```python
# ─────────────────────────────────────────
# IMMUTABLE — creating new objects
# ─────────────────────────────────────────
x = "hello"
print(id(x))    # memory address A
x += " world"   # creates a NEW string object
print(id(x))    # memory address B (different!)

# Original "hello" still exists in memory
# (until its reference count reaches 0)

y = x           # y points to same object as x
x = x.upper()   # x now points to NEW object
print(x)        # HELLO WORLD
print(y)        # hello world (y unchanged!)

# ─────────────────────────────────────────
# MUTABLE — modifying in place
# ─────────────────────────────────────────
a = [1, 2, 3]
print(id(a))    # memory address C
a.append(4)     # SAME object modified
print(id(a))    # memory address C (same!)
print(a)        # [1, 2, 3, 4]

b = a           # b points to SAME object as a
b.append(5)     # modifying through b
print(a)        # [1, 2, 3, 4, 5] ← a also changed!
print(b)        # [1, 2, 3, 4, 5]
print(a is b)   # True (same object)
```

### Mutability in Functions

```python
# This is THE most common source of bugs for new Python devs

# ─────────────────────────────────────────
# IMMUTABLE — function can't modify caller's variable
# ─────────────────────────────────────────
def try_to_modify(x):
    x = 100  # creates NEW local binding
    print(f"Inside: {x}")  # 100

value = 42
try_to_modify(value)
print(f"Outside: {value}")  # 42 — UNCHANGED

# Why? int is immutable. x = 100 creates a new int object
# and rebinds the LOCAL x. Original value untouched.

# ─────────────────────────────────────────
# MUTABLE — function CAN modify caller's variable
# ─────────────────────────────────────────
def add_item(my_list, item):
    my_list.append(item)  # modifies the SAME list object

shopping_cart = ["apple", "banana"]
add_item(shopping_cart, "cherry")
print(shopping_cart)  # ['apple', 'banana', 'cherry'] ← modified!

# Why? list is mutable. my_list and shopping_cart
# point to the SAME object. Appending changes that object.

# ─────────────────────────────────────────
# Reassignment vs Mutation
# ─────────────────────────────────────────
def reassign(my_list):
    my_list = [99, 100]  # NEW list, rebinds LOCAL my_list
    print(f"Inside: {my_list}")  # [99, 100]

def mutate(my_list):
    my_list.clear()           # mutates the SAME object
    my_list.extend([99, 100])

original = [1, 2, 3]
reassign(original)
print(f"After reassign: {original}")  # [1, 2, 3] — unchanged
mutate(original)
print(f"After mutate: {original}")    # [99, 100] — changed!
```

### Practical Implications

```python
# ─────────────────────────────────────────
# Dict keys must be HASHABLE (immutable)
# ─────────────────────────────────────────
d = {}

# Immutable → can be dict keys
d[(1, 2)] = "tuple key"      # ✅ tuple is hashable
d["string"] = "string key"   # ✅ string is hashable
d[42] = "int key"            # ✅ int is hashable
d[True] = "bool key"         # ✅ bool is hashable

# Mutable → CANNOT be dict keys
try:
    d[[1, 2, 3]] = "list key"  # ❌ TypeError: unhashable type: 'list'
except TypeError as e:
    print(e)

try:
    d[{"a": 1}] = "dict key"   # ❌ TypeError: unhashable type: 'dict'
except TypeError as e:
    print(e)

# ─────────────────────────────────────────
# Set elements must be HASHABLE
# ─────────────────────────────────────────
s = {1, 2, "hello", (1, 2)}  # ✅ all hashable

try:
    s = {[1, 2], [3, 4]}     # ❌ unhashable type: 'list'
except TypeError as e:
    print(e)

# ─────────────────────────────────────────
# Making a custom class hashable
# ─────────────────────────────────────────
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

    def __hash__(self):
        return hash((self.x, self.y))  # hash the tuple of values

p1 = Point(1, 2)
p2 = Point(3, 4)
p3 = Point(1, 2)

point_set = {p1, p2, p3}
print(len(point_set))  # 2 (p1 and p3 are equal, so set deduplicates)

cache = {p1: "result1", p2: "result2"}
print(cache[Point(1, 2)])  # result1 (p3 equals p1)
```

---

## 5. Shallow vs Deep Copy

```python
import copy

# ─────────────────────────────────────────
# ASSIGNMENT — not a copy at all!
# ─────────────────────────────────────────
original = [1, 2, [3, 4, 5]]
alias = original  # same object, NOT a copy
alias.append(99)
print(original)   # [1, 2, [3, 4, 5], 99] ← changed!

# ─────────────────────────────────────────
# SHALLOW COPY — new container, same elements
# ─────────────────────────────────────────
original = [1, 2, [3, 4, 5]]
shallow1 = original.copy()    # list method
shallow2 = list(original)     # using constructor
shallow3 = original[:]        # slice
shallow4 = copy.copy(original) # copy module

# Outer list is NEW
shallow1.append(99)
print(original)     # [1, 2, [3, 4, 5]] ← outer unchanged ✅

# But inner objects are SHARED
shallow1[2].append(99)  # modifying the inner list
print(original)     # [1, 2, [3, 4, 5, 99]] ← inner changed! ❌
print(shallow1)     # [1, 2, [3, 4, 5, 99], 99]

# ─────────────────────────────────────────
# DEEP COPY — completely independent copy
# ─────────────────────────────────────────
original = [1, 2, [3, 4, 5]]
deep = copy.deepcopy(original)

# Everything is independent
deep[2].append(99)
print(original)  # [1, 2, [3, 4, 5]] ← completely unchanged ✅
print(deep)      # [1, 2, [3, 4, 5, 99]]
```

### Visual Representation

```
ASSIGNMENT:
  original ──► [1, 2, ──► [3,4,5]]
  alias ────► (same object)

SHALLOW COPY:
  original ──► [1, 2, ──► [3,4,5]]
  shallow ───► [1, 2, ──► (same inner list)]
                                ↑ SHARED!

DEEP COPY:
  original ──► [1, 2, ──► [3,4,5]]
  deep ──────► [1, 2, ──► [3,4,5]] (completely new)
```

### Deep Copy with Custom Objects

```python
class User:
    def __init__(self, name, permissions):
        self.name = name
        self.permissions = permissions  # mutable list

    def __repr__(self):
        return f"User(name={self.name!r}, permissions={self.permissions})"

admin = User("Alice", ["read", "write", "delete"])

# Shallow copy — permissions list is shared
shallow_user = copy.copy(admin)
shallow_user.name = "Bob"  # ok, name is immutable
shallow_user.permissions.append("admin")  # modifies SHARED list!
print(admin)  # User(name='Alice', permissions=['read', 'write', 'delete', 'admin'])
# Alice's permissions changed — BUG!

# Deep copy — completely independent
admin2 = User("Alice", ["read", "write", "delete"])
deep_user = copy.deepcopy(admin2)
deep_user.name = "Bob"
deep_user.permissions.append("admin")  # only deep_user's list
print(admin2)  # User(name='Alice', permissions=['read', 'write', 'delete']) ✅
```

### When to Use Which

```
Assignment    → when you want the same object (aliasing)
Shallow copy  → flat data structures (no nested mutables)
Deep copy     → nested mutable structures
                but: deep copy is SLOWER (recursively copies everything)
```

```python
# Real backend use: protecting default config

DEFAULT_CONFIG = {
    "host": "localhost",
    "port": 8000,
    "database": {
        "pool_size": 5,
        "timeout": 30
    }
}

def get_config(overrides=None):
    # Always deep copy to protect defaults
    config = copy.deepcopy(DEFAULT_CONFIG)
    if overrides:
        config.update(overrides)
    return config

config1 = get_config({"port": 9000})
config1["database"]["pool_size"] = 20   # only affects config1

config2 = get_config()
print(config2["database"]["pool_size"])        # still 5 ✅
print(DEFAULT_CONFIG["database"]["pool_size"]) # still 5 ✅
```

---

## 6. Name Binding & References

### How Names Work

```python
# In Python, names (variables) are LABELS
# Objects exist independently in memory
# Names are just bindings that point to objects

# ─────────────────────────────────────────
# MULTIPLE NAMES, ONE OBJECT
# ─────────────────────────────────────────
x = [1, 2, 3]      # create list, bind name x to it
y = x              # bind name y to SAME object
z = x              # bind name z to SAME object

print(id(x) == id(y) == id(z))  # True — all same object

x.append(4)
print(y)  # [1, 2, 3, 4] — y sees the change
print(z)  # [1, 2, 3, 4] — z sees the change

# ─────────────────────────────────────────
# REBINDING — changing what a name points to
# ─────────────────────────────────────────
x = [1, 2, 3]
y = x           # y points to same object

x = [99, 100]   # x now points to a NEW object
# original [1, 2, 3] still exists!

print(x)  # [99, 100] — new object
print(y)  # [1, 2, 3] — still points to original

# The difference:
# x.append(4)     → MUTATION (modifies the object)
# x = [99, 100]   → REBINDING (changes what x points to)

# ─────────────────────────────────────────
# del — removes NAME, not object
# ─────────────────────────────────────────
x = [1, 2, 3]
y = x

del x  # removes name 'x', but object still exists!
# y still points to it
print(y)  # [1, 2, 3] — object still alive
# print(x)  # NameError — name 'x' doesn't exist anymore

# Object is freed only when ALL names pointing to it are deleted
del y  # now reference count = 0 → object freed
```

### Namespace and Scope Revisited

```python
# Python uses NAMESPACES to store name bindings
# A namespace is essentially a dictionary of {name: object}

# ─────────────────────────────────────────
# Types of namespaces
# ─────────────────────────────────────────

# Built-in namespace
import builtins
print(type(builtins.__dict__))  # <class 'dict'>
print("print" in dir(builtins))  # True
print("len" in dir(builtins))    # True

# Global namespace (current module)
MY_CONSTANT = 42
def my_function():
    pass

print(globals()["MY_CONSTANT"])  # 42
print("my_function" in globals())  # True
print("MY_CONSTANT" in globals())  # True

# Local namespace (inside a function)
def show_locals():
    local_var = "I'm local"
    another = 123
    print(locals())  # {'local_var': "I'm local", 'another': 123}

show_locals()

# Class namespace
class MyClass:
    class_var = "shared"
    def method(self):
        pass

print(MyClass.__dict__)  # {'class_var': 'shared', 'method': <function ...>, ...}

# Instance namespace
obj = MyClass()
obj.instance_var = "mine"
print(obj.__dict__)  # {'instance_var': 'mine'}

# Lookup order for attributes: instance → class → base classes → ...
```

---

## 7. `id()` and `is` vs `==`

```python
# ─────────────────────────────────────────
# id() — unique identifier of an object
# In CPython: this is the MEMORY ADDRESS
# ─────────────────────────────────────────
x = "hello"
print(id(x))  # e.g., 140234567890

# Two variables at same id → same object
a = [1, 2, 3]
b = a
print(id(a) == id(b))  # True — same object

# ─────────────────────────────────────────
# is — checks IDENTITY (same object in memory)
#      compares id()s
# == — checks EQUALITY (same value)
#      calls __eq__ method
# ─────────────────────────────────────────

# Lists
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)  # True — same values
print(a is b)  # False — different objects
print(a is c)  # True — same object (c = a)

# Strings (interning)
s1 = "hello"
s2 = "hello"
print(s1 == s2)  # True
print(s1 is s2)  # True (usually — due to string interning)

# Integers (small int caching)
x = 100
y = 100
print(x == y)  # True
print(x is y)  # True (cached)

x = 1000
y = 1000
print(x == y)  # True
print(x is y)  # False (not cached)
```

### Rules for When to Use `is` vs `==`

```python
# USE is FOR:
value = None
if value is None:    # ✅ checking None
    pass

flag = True
if result is True:   # ✅ checking exact True (not just truthy)
    pass

if result is False:  # ✅ checking exact False
    pass

# USE == FOR:
count = 100
if count == 100:     # ✅ value comparison
    pass

name = "Alice"
if name == "Alice":  # ✅ string value comparison
    pass

items = []
if items == []:      # ✅ but better: if not items:
    pass

# NEVER DO:
x = 1000
if x is 1000:        # ❌ unreliable! (SyntaxWarning)
    pass

name = "hello world"
if name is "hello world":  # ❌ unreliable! (SyntaxWarning)
    pass
```

### Custom `__eq__`

```python
# Custom __eq__ — what == calls

class User:
    def __init__(self, id, name):
        self.id = id
        self.name = name

    def __eq__(self, other):
        if not isinstance(other, User):
            return NotImplemented
        return self.id == other.id  # equality based on ID only

    def __ne__(self, other):
        result = self.__eq__(other)
        if result is NotImplemented:
            return result
        return not result

u1 = User(1, "Alice")
u2 = User(1, "Alice Smith")  # same ID, different name
u3 = User(2, "Bob")

print(u1 == u2)  # True (same ID)
print(u1 == u3)  # False (different ID)
print(u1 is u2)  # False (different objects in memory)
```

---

## 8. Putting It All Together - Real Implications

### Performance: Understanding Object Creation

```python
import timeit

# String concatenation in loop — SLOW
# Creates new string object every iteration
def slow_concat(n):
    result = ""
    for i in range(n):
        result += str(i)  # creates new string each time
    return result

# Join — FAST
# Builds list first, then creates ONE string
def fast_join(n):
    parts = []
    for i in range(n):
        parts.append(str(i))
    return "".join(parts)  # single string creation

# Even faster with generator comprehension
def fastest_join(n):
    return "".join(str(i) for i in range(n))

n = 10000
slow_time = timeit.timeit(lambda: slow_concat(n), number=10)
fast_time = timeit.timeit(lambda: fast_join(n), number=10)
fastest_time = timeit.timeit(lambda: fastest_join(n), number=10)

print(f"Slow concat: {slow_time:.3f}s")
print(f"Join: {fast_time:.3f}s")
print(f"Generator join: {fastest_time:.3f}s")
# String join is usually 5-10x faster!
```

### Memory: Understanding Large Data Structures

```python
import sys

# Lists store references (pointers) to objects
small_list = [1, 2, 3, 4, 5]
big_list = list(range(1000000))

print(f"Small list size: {sys.getsizeof(small_list)} bytes")
print(f"Big list size: {sys.getsizeof(big_list):,} bytes")

# Tuples are more memory efficient than lists
small_tuple = (1, 2, 3, 4, 5)
print(f"Same as tuple: {sys.getsizeof(small_tuple)} bytes")
# Tuple is slightly smaller (no extra capacity buffer)

# Generators — almost zero memory
gen = (x ** 2 for x in range(1000000))
print(f"Generator size: {sys.getsizeof(gen)} bytes")  # ~112 bytes!
```

### Thread Safety: The GIL and Shared State

```python
import threading

# The GIL protects PYTHON objects from corruption
# but your logic can still have race conditions

counter = 0

def increment_unsafe():
    global counter
    for _ in range(100000):
        counter += 1  # NOT atomic in Python!
        # Read counter, add 1, write back
        # GIL can switch threads between these steps

threads = [threading.Thread(target=increment_unsafe) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

print(f"Expected: 500000, Got: {counter}")
# Usually less than 500000 due to race condition!

# Fix with a lock
counter = 0
lock = threading.Lock()

def increment_safe():
    global counter
    for _ in range(100000):
        with lock:  # acquire lock → only one thread at a time
            counter += 1  # now atomic

threads = [threading.Thread(target=increment_safe) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

print(f"Expected: 500000, Got: {counter}")  # always 500000 ✅
```

### Everything in Context: A Backend Debug Session

```python
# A realistic debugging scenario using internals knowledge

# BUG REPORT: "Our user cache is corrupting data"

DEFAULT_USER = {
    "id": None,
    "name": "Anonymous",
    "permissions": ["read"],  # mutable list!
    "settings": {
        "theme": "light",
        "notifications": True
    }
}

# ─────────────────────────────────────────
# BUGGY VERSION
# ─────────────────────────────────────────
def get_user_buggy(user_id):
    # Shallow copy — settings dict is SHARED
    user = DEFAULT_USER.copy()
    user["id"] = user_id
    return user

user1 = get_user_buggy(1)
user2 = get_user_buggy(2)

# User 1 modifies their settings
user1["settings"]["theme"] = "dark"
user1["permissions"].append("write")

# BUG: User 2's data is corrupted!
print(user2["settings"]["theme"])   # "dark" ← WRONG! should be "light"
print(user2["permissions"])         # ["read", "write"] ← WRONG!

# WHY? user1["settings"] and user2["settings"]
# point to THE SAME dict object (shallow copy)
# Modifying through user1 also changes user2

# ─────────────────────────────────────────
# FIXED VERSION
# ─────────────────────────────────────────
import copy

def get_user_fixed(user_id):
    # Deep copy — completely independent
    user = copy.deepcopy(DEFAULT_USER)
    user["id"] = user_id
    return user

user3 = get_user_fixed(3)
user4 = get_user_fixed(4)

user3["settings"]["theme"] = "dark"
user3["permissions"].append("write")

print(user4["settings"]["theme"])   # "light" ✅
print(user4["permissions"])         # ["read"] ✅
print(DEFAULT_USER["settings"]["theme"])  # "light" ✅

# LESSON: Mutable defaults + shallow copy = subtle bugs
# Always deep copy mutable data structures you intend to modify
```

---

## Visual Summary - Python Memory Model

```
┌────────────────────────────────────────────────────────────┐
│                    PYTHON MEMORY MODEL                     │
│                                                            │
│  NAMES (variables)        OBJECTS (in memory)              │
│  ──────────────────       ────────────────────             │
│                                                            │
│  x ──────────────────────► [int: 42]                       │
│  y ──────────────────────► [str: "hello"]                  │
│                                                            │
│  a ──┐                                                     │
│      └───────────────► [list: [1, 2, 3]]                   │
│  b ──┘  ↑ same object                                      │
│                                                            │
│  REFERENCE COUNTING:                                       │
│    Each object has a count of how many names point to it  │
│    count = 0 → object freed immediately                   │
│                                                            │
│  GARBAGE COLLECTOR:                                        │
│    Handles circular references that ref counting can't     │
│    Runs periodically (3 generations)                       │
│                                                            │
│  GIL:                                                      │
│    Only 1 thread executes Python bytecode at a time       │
│    Released during I/O                                    │
│    Problem for CPU-bound → use multiprocessing            │
│    Not a problem for I/O-bound → threads/async work fine  │
│                                                            │
│  is vs ==:                                                  │
│    is  → same object? (id comparison)                     │
│    ==  → same value? (__eq__ call)                        │
│    Use is ONLY for: None, True, False                     │
│                                                            │
│  COPY TYPES:                                               │
│    Assignment → alias (same object)                        │
│    Shallow   → new container, shared elements             │
│    Deep      → completely independent copy                │
└────────────────────────────────────────────────────────────┘
```

---

## Knowledge Check

1. What are the steps Python takes to run your .py file?
2. What is bytecode and where is it stored?
3. What is the GIL and what problem does it solve?
4. Does the GIL prevent all race conditions? Why or why not?
5. When is the GIL released?
6. How does reference counting work?
7. What problem does the garbage collector solve that reference counting cannot?
8. Why are small integers (-5 to 256) all the same object?
9. What is the difference between mutation and rebinding?
10. Why can't lists be used as dictionary keys?
11. When should you use `is` vs `==`?
12. What is the difference between shallow and deep copy?
13. What happens when you do: `del x`?
14. What is string interning?

---

## Practice Exercises

```python
# Exercise 1: Reference Tracing
# Without running the code, predict the output:
a = [1, 2, 3]
b = a
c = a[:]  # shallow copy

a.append(4)
b.append(5)
c.append(99)

print(a)       # predict this
print(b)       # predict this
print(c)       # predict this
print(a is b)  # predict this
print(a is c)  # predict this
print(a == b)  # predict this

# Exercise 2: GIL Impact
# Write a function that:
# a) Runs a CPU-bound task with 4 threads and measures time
# b) Runs same task with 4 processes and measures time
# c) Explains the difference in results
# Task: count from 0 to 10_000_000

# Exercise 3: Memory Investigation
# Use sys.getsizeof to compare:
# - An empty list vs list with 1000 integers
# - An empty dict vs dict with 1000 entries
# - A generator expression vs list comprehension
#   (both producing 1000 squares)
# Print memory usage for each
# Which is most memory efficient?

# Exercise 4: Copy Bug Fix
# The following code has a subtle bug.
# Identify it, explain why it happens, and fix it:
class Config:
    defaults = {
        "debug": False,
        "database": {
            "host": "localhost",
            "port": 5432
        },
        "allowed_hosts": ["127.0.0.1"]
    }

    def get_config(self, overrides=None):
        config = self.defaults.copy()
        if overrides:
            config.update(overrides)
        return config

config1 = Config().get_config()
config1["database"]["port"] = 9999
config1["allowed_hosts"].append("0.0.0.0")

config2 = Config().get_config()
print(config2["database"]["port"])     # what does this print?
print(config2["allowed_hosts"])        # what does this print?
# Why is this wrong? How do you fix it?

# Exercise 5: Bytecode Reading
# Use dis.dis() to inspect these functions
# and explain what each bytecode instruction does:
def mystery1(items):
    return [x * 2 for x in items if x > 0]

def mystery2(a, b, c):
    return a + b * c

import dis
dis.dis(mystery1)
dis.dis(mystery2)
```

---

## Phase 1.5 - Complete!

**You now understand:**

```
✅ How Python executes code (lexer → parser → compiler → PVM)
✅ Bytecode and .pyc files
✅ The GIL — what it is, when it matters, when it doesn't
✅ Reference counting and garbage collection
✅ Small integer caching and string interning
✅ Mutable vs immutable objects and their implications
✅ Shallow vs deep copy and when to use each
✅ Name binding vs mutation
✅ id() and is vs == — when to use which
✅ Namespaces and how Python finds names
✅ Thread safety and the GIL
```

---

