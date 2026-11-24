**Open VS Code right now. Create this:**
```bash
# In your terminal
mkdir ~/projects/python_basics
cd ~/projects/python_basics
python3 -m venv venv
source venv/bin/activate  # Mac/Linux
# venv\Scripts\activate   # Windows
touch basics.py
code . # install it from your IDE first
```

Or, with Poetry:
```bash
cd ~/projects
poetry new project_folder_name
cd project_folder_name
poetry config virtualenvs.in-project true
poetry install
source <(poetry env activate)
touch basics.py
```
**Write every example yourself. Don't copy-paste. Type it.**

---
## 1. Variables & Data Types
In Python, **variables are reserved memory locations used to store data values**, created automatically the moment you assign a value to them using the `=` operator. Python is **dynamically typed**, meaning you do not need to explicitly declare a variable's data type, and the same variable can change types during execution.

A variable is a named container that holds a value.
```
Think of it like a labeled box:

name = "Alice"
  ↑        ↑
label   what's inside the box
```

```python
# basics.py

# Creating variables
name = "Alice"
age = 25
height = 5.7
is_active = True

# Print them
print(name)      # Alice
print(age)       # 25
print(height)    # 5.7
print(is_active) # True
```

Run it:
```bash
python3 basics.py
```

### 1.1 Variable Names
A variable can have a short name (like `x` and `y`) or a more descriptive name (`age`, `carname`, `total_volume`).

**Rules for Python variables:**
- A variable name must start with a letter or the underscore character
- A variable name cannot start with a number
- A variable name can only contain alpha-numeric characters and underscores (`A-z`, `0-9`, and `_` )
- Variable names are case-sensitive (`age`, `Age` and `AGE` are three different variables)
- A variable name cannot be any of the [Python keywords](https://www.w3schools.com/python/python_ref_keywords.asp).

### 1.2 Python's Built-in Data Types
Python has **built-in data types** categorized into numeric, text, sequence, mapping, set, boolean, and binary types. Because Python is **dynamically typed**, you do not need to explicitly declare a data type when creating a variable; the interpreter infers it automatically at runtime.
```python
# ─────────────────────────────────────────
# TEXT
# ─────────────────────────────────────────
name = "Alice"          # str (string)
message = 'Hello World' # str (single quotes work too)
long_text = """
This spans
multiple lines
"""                     # str (triple quotes for multiline)

# ─────────────────────────────────────────
# NUMBERS
# ─────────────────────────────────────────
age = 25                # int (integer - whole number)
price = 19.99           # float (decimal number)
big_number = 1_000_000  # int (underscores for readability)
complex_num = 3 + 4j    # complex (you'll rarely use this)

# ─────────────────────────────────────────
# BOOLEAN
# ─────────────────────────────────────────
is_active = True        # bool
is_deleted = False      # bool
# Note: True and False are CAPITALIZED in Python

# ─────────────────────────────────────────
# NOTHING
# ─────────────────────────────────────────
middle_name = None      # NoneType — means "no value"
# Like null in other languages
# You'll use this ALL the time in backend dev

# ─────────────────────────────────────────
# CHECKING THE TYPE
# ─────────────────────────────────────────
print(type(name))        # <class 'str'>
print(type(age))         # <class 'int'>
print(type(price))       # <class 'float'>
print(type(is_active))   # <class 'bool'>
print(type(middle_name)) # <class 'NoneType'>
```

### 1.3 How Variables Work Internally
Python variables are LABELS, not boxes. They point to objects in memory.
```python
x = 10    # x points to object 10 in memory
y = x     # y points to the SAME object

print(id(x))  # id() shows memory address
print(id(y))  # same address as x!

x = 20        # now x points to a NEW object 20
print(id(x))  # different address
print(id(y))  # y still points to 10 — unchanged
print(y)      # 10

# This matters when we get to mutable vs immutable
# For now, just know: variables are labels, not boxes
```

```
Memory visualization:

x = 10, y = x:
  x ──► [10] ◄── y

x = 20:
  x ──► [20]
  [10] ◄── y
```

### 1.4 Type Conversion
**Type conversion in Python** is the process of changing a value from one data type to another. Python supports two categories of type conversion: **Implicit Type Conversion**, which happens automatically, and **Explicit Type Conversion (Type Casting)**, which you must trigger manually using built-in functions.
```python
# Converting between types (casting)

# String to int
age_text = "25"
age_number = int(age_text)
print(age_number + 1)      # 26
print(type(age_number))    # <class 'int'>

# Int to string
user_id = 42
user_id_text = str(user_id)
print("User ID: " + user_id_text)  # User ID: 42

# String to float
price_text = "19.99"
price = float(price_text)
print(price * 2)  # 39.98

# Float to int (cuts off decimal — doesn't round)
score = 98.7
print(int(score))  # 98 ← not 99!

# Int to bool
print(bool(0))   # False
print(bool(1))   # True
print(bool(42))  # True
print(bool(-1))  # True (anything non-zero is True)

# String to bool
print(bool(""))       # False (empty string)
print(bool("hello"))  # True (non-empty string)

# None to bool
print(bool(None))  # False
```

**Why this matters in backend:**
```python
# User sends data from a form (everything arrives as string)
user_input_age = "25"      # came from HTTP request body
user_input_price = "19.99"

# You must convert before doing math
age = int(user_input_age)
price = float(user_input_price)

# This is why input validation matters so much in APIs
# What if user sends: age = "twenty-five" ?
# int("twenty-five") → ValueError: invalid literal
```

### 1.5 What Happens When Conversion Fails
```python
# Safe conversion (you'll write this pattern often)
user_input = "abc"
try:
    age = int(user_input)
except ValueError:
    print("That's not a valid number!")
    age = 0
```
Don't worry about try/except fully yet. We cover it in Advanced Python.
Just know: invalid conversions raise errors.

---
## 2. Strings
Strings are text. In backend dev, you work with text constantly: API responses, user data, error messages, SQL queries, JSON etc.

### 2.1 Creating Strings
Four ways to create strings:
```python
single = 'Hello'
double = "Hello"
triple_single = '''Hello
World'''
triple_double = """Hello
World"""

# They're all the same type
print(type(single))  # <class 'str'>

# When to use which:
text1 = "It's a beautiful day"       # use double when text has apostrophe
text2 = 'He said "Hello"'            # use single when text has double quotes
text3 = "He said \"Hello\""          # OR escape with backslash
```

### 2.2 String Methods
```python
name = "  Alice Smith  "

# Case operations
print(name.upper())          # "  ALICE SMITH  "
print(name.lower())          # "  alice smith  "
print(name.title())          # "  Alice Smith  "
print(name.capitalize())     # "  alice smith  "

# Whitespace
print(name.strip())          # "Alice Smith" (removes both ends)
print(name.lstrip())         # "Alice Smith  " (removes left)
print(name.rstrip())         # "  Alice Smith" (removes right)

# Searching
email = "alice@example.com"
print(email.find("@"))       # 5 (index position of @)
print(email.find("xyz"))     # -1 (not found)
print(email.index("@"))      # 5 (like find but raises error if not found)
print(email.count("e"))      # 2 (how many times 'e' appears)

print("alice" in email)      # True
print("@" in email)          # True
print(email.startswith("alice"))  # True
print(email.endswith(".com"))     # True

# Replacing
print(email.replace("example", "gmail"))  # "alice@gmail.com"

# Splitting (very useful for parsing)
sentence = "Python,FastAPI,PostgreSQL,Redis"
parts = sentence.split(",")
print(parts)        # ['Python', 'FastAPI', 'PostgreSQL', 'Redis']
print(type(parts))  # <class 'list'>

# Joining (opposite of split)
technologies = ["Python", "FastAPI", "PostgreSQL"]
joined = ", ".join(technologies)
print(joined)       # "Python, FastAPI, PostgreSQL"

# Checking content
print("123".isdigit())      # True
print("abc".isalpha())      # True
print("abc123".isalnum())   # True
print("   ".isspace())      # True
```

### 2.3 String Slicing
Strings are like a sequence of characters. Each character has an index (position)
```python
text = "Python"
#       012345 ← positive indexes (left to right)
#      -654321 ← negative indexes (right to left)

# Access single character
print(text[0])    # P (first character)
print(text[1])    # y
print(text[-1])   # n (last character)
print(text[-2])   # o (second to last)

# Slicing: text[start:stop:step]
print(text[0:3])   # Pyt (index 0,1,2 — stop is excluded)
print(text[2:5])   # tho
print(text[:3])    # Pyt (from beginning to index 3)
print(text[3:])    # hon (from index 3 to end)
print(text[:])     # Python (entire string)
print(text[::2])   # Pto (every 2nd character)
print(text[::-1])  # nohtyP (reversed!)

# Real backend example:
token = "Bearer eyJhbGciOiJIUzI1NiJ9.abc123"
# Remove "Bearer " prefix (first 7 characters)
jwt_token = token[7:]
print(jwt_token)  # eyJhbGciOiJIUzI1NiJ9.abc123
```

### 2.4 f-strings
```python
# Old way (don't do this)
name = "Alice"
age = 25
message = "Hello, " + name + ". You are " + str(age) + " years old."

# f-string (modern, clean, use this always)
message = f"Hello, {name}. You are {age} years old."
print(message)  # Hello, Alice. You are 25 years old.

# You can put ANY expression inside {}
price = 19.99
quantity = 3
print(f"Total: ${price * quantity}")  # Total: $59.97

# Formatting numbers
pi = 3.14159265
print(f"Pi is {pi:.2f}")  # Pi is 3.14 (2 decimal places)
print(f"Pi is {pi:.4f}")  # Pi is 3.1416

big = 1000000
print(f"Big number: {big:,}")  # Big number: 1,000,000

# Padding and alignment
name = "Alice"
print(f"|{name:<10}|")  # |Alice     | (left aligned, 10 wide)
print(f"|{name:>10}|")  # |     Alice| (right aligned)
print(f"|{name:^10}|")  # |  Alice   | (centered)

# Real backend example:
user_id = 42
username = "alice"
endpoint = "/users"
status = 200
print(f"[{status}] {endpoint} - User {user_id} ({username}) accessed")
# [200] /users - User 42 (alice) accessed
```

### 2.5 String Immutability
```python
# Strings CANNOT be changed after creation
text = "Hello"
# text[0] = "J" ← This CRASHES with TypeError

# You must create a NEW string
text = "J" + text[1:]
print(text)  # Jello

# Why this matters:
name = "alice"
upper_name = name.upper()
print(name)       # alice ← original unchanged
print(upper_name) # ALICE ← new string

# String methods always RETURN new strings
# They never modify the original
```

---
## 3. Numbers
```python
# ─────────────────────────────────────────
# INTEGERS
# ─────────────────────────────────────────
x = 10
y = 3

print(x + y)   # 13         addition
print(x - y)   # 7          subtraction
print(x * y)   # 30         multiplication
print(x / y)   # 3.3333...  division (always returns float)
print(x // y)  # 3          floor division (integer result, rounds down)
print(x % y)   # 1          modulo (remainder after division)
print(x ** y)  # 1000       exponentiation (10 to the power of 3)

# Modulo is surprisingly useful:
print(10 % 2)  # 0   (even number — remainder is 0)
print(11 % 2)  # 1   (odd number — remainder is 1)
print(15 % 5)  # 0   (divisible by 5)

# Real use: check if number is even
number = 42
if number % 2 == 0:
    print("Even")
else:
    print("Odd")

# ─────────────────────────────────────────
# FLOATS
# ─────────────────────────────────────────
price = 19.99
tax_rate = 0.08
tax = price * tax_rate
print(f"Tax: {tax}")  # Tax: 1.5992

# Float precision issue (important to know!)
print(0.1 + 0.2)            # 0.30000000000000004
print(0.1 + 0.2 == 0.3)     # False ← !!!

# Why? Computers store floats in binary
# Some decimals can't be represented exactly

# Solution for money: use Decimal module
from decimal import Decimal
price = Decimal("19.99")
tax = Decimal("0.08")
print(price * tax)  # 1.5992 (exact)

# ─────────────────────────────────────────
# USEFUL NUMBER FUNCTIONS
# ─────────────────────────────────────────
print(abs(-42))              # 42   (absolute value)
print(round(3.7))            # 4    (rounds to nearest int)
print(round(3.14159, 2))     # 3.14 (round to 2 decimal places)
print(max(1, 5, 3))          # 5
print(min(1, 5, 3))          # 1
print(sum([1, 2, 3, 4]))     # 10

import math
print(math.floor(3.9))  # 3   (always rounds down)
print(math.ceil(3.1))   # 4   (always rounds up)
print(math.sqrt(16))    # 4.0
print(math.pi)          # 3.141592653589793
```

---
## 4. Booleans & Truthiness
```python
# ─────────────────────────────────────────
# BOOLEANS
# ─────────────────────────────────────────
is_active = True
is_deleted = False

print(type(True))    # <class 'bool'>
print(int(True))     # 1 (True is literally 1)
print(int(False))    # 0 (False is literally 0)

# Boolean operations
print(True and True)     # True
print(True and False)    # False
print(False and True)    # False
print(False and False)   # False

print(True or True)      # True
print(True or False)     # True
print(False or True)     # True
print(False or False)    # False

print(not True)   # False
print(not False)  # True

# ─────────────────────────────────────────
# TRUTHINESS — extremely important
# ─────────────────────────────────────────
# In Python, ANY value can be used as a boolean
# Some values are "falsy" (act like False)
# Everything else is "truthy" (acts like True)

# FALSY VALUES (memorize these):
print(bool(False))    # False
print(bool(None))     # False
print(bool(0))        # False
print(bool(0.0))      # False
print(bool(""))       # False ← empty string
print(bool([]))       # False ← empty list
print(bool({}))       # False ← empty dict
print(bool(set()))    # False ← empty set
print(bool(()))       # False ← empty tuple

# TRUTHY VALUES (everything else):
print(bool(True))     # True
print(bool(1))        # True
print(bool(-1))       # True
print(bool("hello"))  # True ← non-empty string
print(bool([1, 2]))   # True ← non-empty list
print(bool({"a": 1})) # True ← non-empty dict

# WHY THIS MATTERS in backend code:
users = []  # empty list from database query
if users:   # same as: if len(users) > 0:
    print("Found users")
else:
    print("No users found")  # this runs

username = ""  # empty string from form input
if not username:  # same as: if username == "":
    print("Username is required")  # this runs

# Real pattern you'll write constantly:
def get_user(user_id):
    user = find_in_database(user_id)  # returns None if not found

    if not user:
        return {"error": "User not found"}

    return user
```

---
## 5. Input & Output
In Python, the two primary built-in functions used for handling standard input and output are **`input()`** and **`print()`**.
### 5.1 Output
The primary way to output data in Python is by ==using the **`print()` function**==, which displays text, variables, or expressions directly to the console.
```python
# OUTPUT
print("Hello World")          # basic print

# Multiple values
print("Name:", "Alice", "Age:", 25)  # Name: Alice Age: 25

# Custom separator
print("Python", "FastAPI", "Redis", sep=" | ")
# Python | FastAPI | Redis

# Custom end (default is newline)
print("Loading", end="")
print("...")       # Loading...

# Print nothing (just a blank line)
print()
```

### 5.2 Input
In Python, you accept user input by using the built-in **`input()` function**. This function pauses program execution, displays an optional message to the console, and waits for the user to type something and press Enter.
```python
# INPUT (terminal input - for CLI apps)
# Note: In web backends, input comes from HTTP requests
# But input() is useful for CLI tools and learnit
name = input("Enter your name: ")
print(f"Hello, {name}!")

# input() ALWAYS returns a string
age_text = input("Enter your age: ")
age = int(age_text)  # must convert
print(f"Next year you'll be {age + 1}")

# Safe input handling:
try:
    age = int(input("Enter your age: "))
    print(f"Your age is {age}")
except ValueError:
    print("Please enter a valid number")
```

---
## 6. Operators

### 6.1 Comparison Operators
**Comparison operators in Python** evaluate the relationship between two values and always return a Boolean result: either `True` or `False`. They are fundamental for decision-making structures like `if` statements and loops.
```python
x = 10
y = 20

print(x == y)   # False (equal to)
print(x != y)   # True  (not equal to) (! & =)
print(x < y)    # True  (less than)
print(x > y)    # False (greater than)
print(x <= y)   # True  (less than or equal)
print(x >= y)   # False (greater than or equal)

# Comparing strings
print("alice" == "alice")     # True
print("alice" == "Alice")     # False (case sensitive!)
print("alice" < "bob")        # True (alphabetical comparison)

# Comparing with None
user = None
print(user == None)    # True (works but not recommended)
print(user is None)    # True (PREFERRED way to check None)
print(user is not None) # False

# IMPORTANT: == vs is
# == checks VALUE equality
# is checks IDENTITY (same object in memory)

a = [1, 2, 3]
b = [1, 2, 3]

print(a == b)  # True (same values)
print(a is b)  # False (different objects in memory)

# Use: is None / is not None (for None checks)
# Use: == for everything else
```

### 6.2 Logical Operators
Python uses three logical operators to combine or modify conditional statements: **`and`**, **`or`**, and **`not`**. Unlike many programming languages that use symbols like `&&`, `||`, or `!`, Python relies entirely on these **lowercase English keywords**.
```python
age = 25
has_subscription = True
is_banned = False

# and — BOTH must be True
can_access = age >= 18 and has_subscription
print(can_access)  # True

# or — AT LEAST ONE must be True
can_see_content = has_subscription or age >= 18
print(can_see_content)  # True

# not — flips the boolean
print(not is_banned)        # True
print(not has_subscription) # False

# Real backend example:
def can_delete_post(user_role, is_author):
    return user_role == "admin" or is_author

def can_view_dashboard(is_logged_in, has_permission):
    return is_logged_in and has_permission and not is_banned

# Short-circuit evaluation (important for performance)
# and: if first is False, second is NOT evaluated
# or:  if first is True, second is NOT evaluated

def expensive_check():
    print("Checking database...")
    return True

is_valid = False and expensive_check()  # "Checking database..." NOT printed
is_valid = True or expensive_check()    # "Checking database..." NOT printed
```

### 6.3 Assignment Operators
**Assignment operators** in Python are used to **store or update values** in variables. The most common and foundational assignment operator is the single equal sign (`=`), but Python also includes **augmented (compound) assignment operators** that combine math or bitwise operations with assignment to make your code more concise.
```python
count = 0

count = count + 1  # basic
count += 1         # shorthand (same thing)
count -= 1         # subtract
count *= 2         # multiply
count /= 2         # divide
count //= 2        # floor divide
count **= 2        # power
count %= 3         # modulo

print(count)

# Real use:
total_price = 0
total_price += 19.99  # add item
total_price += 5.99   # add another
print(f"Total: ${total_price}")  # Total: $25.98
```

### 6.4 Bitwise Operators (Know They Exist)
These operate on binary representations, you'll rarely use these in web backend. But they appear in systems programming and flags.
```python
a = 0b1010  # 10 in binary
b = 0b1100  # 12 in binary

print(a & b)   # AND: 0b1000 = 8
print(a | b)   # OR:  0b1110 = 14
print(a ^ b)   # XOR: 0b0110 = 6
print(~a)      # NOT: -11
print(a << 1)  # left shift: 0b10100 = 20
print(a >> 1)  # right shift: 0b0101 = 5

# Common real use: permission flags
READ = 0b001    # 1
WRITE = 0b010   # 2
EXECUTE = 0b100 # 4

user_perms = READ | WRITE  # 0b011 = 3
print(user_perms & READ)   # non-zero = has READ permission
print(user_perms & EXECUTE)  # 0 = does NOT have EXECUTE permission
```

---
## 7. Conditional Statements
**Conditional statements in Python** allow you to make decisions and execute specific blocks of code based on whether a condition is `True` or `False`. Python relies on **mandatory indentation** (usually 4 spaces) instead of curly braces to define the scope of these code blocks.

### 7.1 The `if` Statement
The most basic form executes code **only if** a specific condition evaluates to `True`.
```python
age = 20
if age >= 18:
    print("You are eligible to vote.")  # Runs because condition is True
```

### 7.2 The `if-else` Statement
Provides an alternative block of code that runs **only if** the `if` condition evaluates to `False`.
```python
score = 45
if score >= 50:
    print("Passed")
else:
    print("Failed")  # Runs because score is less than 50
```

### 7.3 The `if-elif-else` Ladder
The `elif` keyword stands for **"else if"**. It allows you to chain multiple sequential checks. Python exits the chain as soon as it finds the _first_ matching truth condition.

```python
traffic_light = "Yellow"

if traffic_light == "Red":
    print("Stop")
elif traffic_light == "Yellow":
    print("Slow down")  # Runs and skips the rest
else:
    print("Go")
```

### 7.4 Nested Conditions
You can place conditional blocks **inside** other conditional blocks to handle complex multi-layered logic.
```python
has_ticket = True
is_vip = False

if has_ticket:
    if is_vip:
        print("Welcome to the lounge.")
    else:
        print("Welcome to the standard seating.")  # Runs
else:
    print("Access denied.")
```

### 7.5 Ternary Operator
In Python, the **ternary operator** (also called a conditional expression) allows you to write an `if/else` statement in a single line.
```python
age = 20
status = "adult" if age >= 18 else "minor"
print(status)  # adult

# Nested ternary (use sparingly - gets confusing)
score = 85
grade = "A" if score >= 90 else "B" if score >= 80 else "C"
print(grade)  # B

# Real backend example:
user = {"name": "Alice", "is_premium": True}
plan = "Premium" if user["is_premium"] else "Free"
print(f"{user['name']} is on the {plan} plan")
```

---
## 8. Loops
Python has **two primitive loop commands**: the `for` loop and the `while` loop. These structures allow you to automate repetitive tasks efficiently.
### 8.1 `for` Loops
A `for` loop is used to **iterate over a sequence** (like a list, tuple, dictionary, set, or string) or an iterable object. It executes a block of code a predetermined number of times.
```python
# Loop over a list
users = ["Alice", "Bob", "Charlie"]
for user in users:
    print(f"Hello, {user}!")
# Hello, Alice!
# Hello, Bob!
# Hello, Charlie!

# Loop over a string
for char in "Python":
    print(char)
# P  y  t  h  o  n
```

#### Using the `range()` Function
To repeat code a specific number of times, pair the loop with the python `range()` [function](https://www.w3schools.com/python/ref_func_range.asp). The endpoint is exclusive.
```python
# Prints 0, 1, 2, 3, 4
for i in range(5):
    print(i)
```

```python
# Loop over a range
for i in range(5):
    print(i)
# 0 1 2 3 4

# range(start, stop, step)
for i in range(1, 11):  # 1 to 10
    print(i)

for i in range(0, 10, 2):  # even numbers
    print(i)  # 0 2 4 6 8

for i in range(10, 0, -1):  # countdown
    print(i)  # 10 9 8 7 6 5 4 3 2 1
```

#### Getting the Index with `enumerate()`
If you need both the item and its index position, use `enumerate()`.
```python
fruits = ["apple", "banana", "cherry"]

# Without enumerate (old way):
for i in range(len(fruits)):
    print(f"{i}: {fruits[i]}")

# With enumerate (pythonic way):
for index, fruit in enumerate(fruits):
    print(f"{index}: {fruit}")
# 0: apple
# 1: banana
# 2: cherry

# Start index from 1
for index, fruit in enumerate(fruits, start=1):
    print(f"{index}. {fruit}")
# 1. apple
# 2. banana
# 3. cherry

# Real backend use:
errors = ["Email invalid", "Password too short", "Username taken"]
for i, error in enumerate(errors, start=1):
    print(f"Error {i}: {error}")
```

### 8.2 `while` Loops
A `while` loop repeatedly executes a block of code **as long as a specific condition remains true**. Use it when you do not know in advance how many iterations are needed.
```python
count = 0
while count < 5:
    print(count)
    count += 1
# 0 1 2 3 4

# Real backend use: retry logic
import time

max_retries = 3
attempt = 0
success = False

while attempt < max_retries p not success:
    attempt += 1
    print(f"Attempt {attempt}...")
    # try to connect to database
    # if connected: success = True
    # if not: wait and retry

    # Simulating failure:
    if attempt == 3:
        success = True

if success:
    print("Connected!")
else:
    print("Failed after all retries")
```

### 8.3 Loop Control Statements
You can alter the behavior of a loop using specific control keywords:

- `break`: **Exits the loop entirely**, even if the loop condition is still met.
- `continue`: **Skips the rest of the current iteration** and jumps directly to the next one.

**`break` - exit the loop immediately:**
```python
users = ["Alice", "Bob", "Charlie", "Dave"]
target = "Charlie"

for user in users:
    if user == target:
        print(f"Found: {user}")
        break  # stop searching, we found it
    print(f"Checking {user}...")
# Checking Alice...
# Checking Bob...
# Found: Charlie
```

**`CONTINUE` - skip to next iteration:**
```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
for n in numbers:
    if n % 2 == 0:
        continue  # skip even numbers
    print(n)
# 1 3 5 7 9

# Real backend use: skip invalid records
records = [
    {"id": 1, "email": "alice@test.com"},
    {"id": 2, "email": None},       # invalid
    {"id": 3, "email": "charlie@test.com"},
    {"id": 4, "email": ""},          # invalid
]

for record in records:
    if not record["email"]:  # skip None or empty email
        continue
    print(f"Processing: {record['email']}")
# Processing: alice@test.com
# Processing: charlie@test.com
```

**`else` on loops (unique to Python):**
```python
# The else block runs if loop completed WITHOUT break

users = ["Alice", "Bob", "Charlie"]
target = "Dave"

for user in users:
    if user == target:
        print(f"Found {target}")
        break
else:
    print(f"{target} not found")  # runs because no break
# Dave not found

# Real backend use:
def find_user_by_email(email, users):
    for user in users:
        if user["email"] == email:
            return user
    else:
        return None  # not found
```

### 8.4 Nested Loops
You can place loops inside other loops. The inner loop executes completely for every single iteration of the outer loop.

```python
for i in range(2):        # Outer loop
    for j in range(3):    # Inner loop
        print(f"({i}, {j})")
```

---
## 9. Collections - Lists, Tuples, Sets, Dicts
These are the most important data structures in Python. You will use these EVERY SINGLE DAY.
```python
nums = [1, 2, 3, 4, 5] # lists (Ordered, mutable, duplicatable)  
nums2 = {1, 2, 3, 4, 5} # sets (Unordered, mutable, not duplicatable)  
nums3 = (1, 2, 3, 4, 4) # tuples (Ordered, immutable, duplicatable)
```
Collection = single "variable" used to store multiple values.
- `LISTS` - `[]` **ordered** & **changeable**. Duplicates OK.
- `SET` - `{}` **Unordered** & **Unique**. but Add/Remove OK. No Duplicates.
- `TUPLES` - `()` **ordered** & **Unchangeable**. Duplicate OK. Faster.

Watch [this](https://youtu.be/gOMW_n2-2Mw) for better understanding. Also [this](https://youtu.be/11WrzU81q68).
### 9.1 Lists - Ordered, Changeable
In programming, an **ordered** and **changeable** (mutable) list is a data structure, most commonly associated with [Python Lists](https://www.w3schools.com/python/python_lists.asp) that allows you to store multiple items, modify them after creation, and maintain a fixed sequence.
```python
# LISTS
# Ordered: items have positions (indexes)
# Mutable: can add, remove, change items
# Allows duplicates

# Creating
empty_list = []
numbers = [1, 2, 3, 4, 5]
names = ["Alice", "Bob", "Charlie"]
mixed = [1, "hello", True, None, 3.14]  # can mix types
nested = [[1, 2], [3, 4], [5, 6]]       # list of lists

# Accessing
print(numbers[0])      # 1 (first)
print(numbers[-1])     # 5 (last)
print(numbers[1:3])    # [2, 3] (slicing)

# CRUD Operations:
# CREATE (adding items)
numbers.append(6)            # add to end: [1,2,3,4,5,6]
numbers.insert(0, 0)         # insert at index: [0,1,2,3,4,5,6]
numbers.extend([7, 8, 9])    # add multiple: [0,1,2,3,4,5,6,7,8,9]

# READ (searching)
print(3 in numbers)          # True
print(numbers.index(3))      # 3 (position of value 3)
print(numbers.count(3))      # 1 (how many times 3 appears)

# UPDATE (changing)
numbers[0] = 100  # change by index
print(numbers)    # [100, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# DELETE (removing)
numbers.remove(100)       # remove by VALUE
popped = numbers.pop()    # remove & return last item
popped_at = numbers.pop(0)  # remove & return at index
del numbers[0]            # delete by index
numbers.clear()           # remove ALL items

# Useful list operations
nums = [3, 1, 4, 1, 5, 9, 2, 6]

nums.sort()                 # sort in-place: [1,1,2,3,4,5,6,9]
nums.sort(reverse=True)     # sort descending: [9,6,5,4,3,2,1,1]
nums.reverse()              # reverse in-place

sorted_nums = sorted(nums)           # returns NEW sorted list
reversed_nums = list(reversed(nums)) # returns NEW reversed list

print(len(nums))    # length
print(min(nums))    # minimum value
print(max(nums))    # maximum value
print(sum(nums))    # sum of all values

# Real backend example:
user_ids = [1, 5, 3, 2, 4]
user_ids.sort()
print(user_ids)  # [1, 2, 3, 4, 5]

# Getting page of results (pagination)
all_posts = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
page = 1
page_size = 3
start = (page - 1) * page_size  # 0
end = start + page_size          # 3
page_posts = all_posts[start:end]
print(page_posts)  # [1, 2, 3]
```

### 9.2 Sets - Unordered, Unique
In Python, a **set** is a built-in data structure that represents an **unordered collection of unique items**. This means a set automatically filters out duplicate entries and does not keep track of element positions or insertion history.
```python
# SETS
# Unordered: no guaranteed order, no indexes
# Unique: NO duplicates allowed
# Mutable: can add/remove items

# Creating
empty_set = set()   # NOT {} — that's a dict!
numbers = {1, 2, 3, 4, 5}
with_dups = {1, 2, 2, 3, 3, 3}
print(with_dups)  # {1, 2, 3} ← duplicates removed!

# Adding / Removing
numbers.add(6)          # add one item
numbers.update([7, 8])  # add multiple
numbers.remove(1)       # remove (raises error if not found)
numbers.discard(99)     # remove (no error if not found)
popped = numbers.pop()  # remove and return arbitrary item

# Checking membership (VERY FAST — O(1))
print(3 in numbers)   # True
print(99 in numbers)  # False

# Set Operations (like math sets)
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

print(a | b)  # Union:        {1,2,3,4,5,6,7,8} (all items)
print(a & b)  # Intersection: {4,5} (items in BOTH)
print(a - b)  # Difference:   {1,2,3} (in a but not b)
print(a ^ b)  # Symmetric diff: {1,2,3,6,7,8} (in one but not both)

# Real backend examples:

# Remove duplicates from a list
user_ids_with_dups = [1, 2, 3, 2, 1, 4, 3, 5]
unique_ids = list(set(user_ids_with_dups))
print(unique_ids)  # [1, 2, 3, 4, 5]

# Check permissions
user_permissions = {"read", "write", "comment"}
required_permissions = {"read", "write"}
has_access = required_permissions.issubset(user_permissions)
print(has_access)  # True

# Find users who haven't completed onboarding
all_users = {1, 2, 3, 4, 5}
completed_onboarding = {1, 3, 5}
pending = all_users - completed_onboarding
print(pending)  # {2, 4}
```

### 9.3 Tuples - Ordered, Unchangeable
Tuples in Python are **ordered**, **immutable** collections of items. They allow duplicate values and can hold mixed data types (strings, numbers, etc.). Once created, you cannot change, add, or remove their elements.

Examples of tuples are mixed with [[#11. Unpacking & Packing|packing & unpacking]]. Try to understand both at the same time and learn Tuples [here](https://youtu.be/w6hL_dszMxk).
```python
# TUPLES
# Like lists BUT cannot be changed after creation
# Use when data should NOT change

# Creating
empty_tuple = ()
coordinates = (40.7128, -74.0060)  # NYC lat/long
rgb_color = (255, 128, 0)
single_item = (42,)       # ← comma required for single item!
single_wrong = (42)       # this is just int 42, NOT a tuple

# Accessing (same as list)
print(coordinates[0])  # 40.7128
print(coordinates[-1]) # -74.0060
print(coordinates[0:2])# (40.7128, -74.0060)

# Unpacking (very Pythonic)
lat, lng = coordinates
print(lat)  # 40.7128
print(lng)  # -74.0060

# Swapping variables (Python magic)
a, b = 1, 2
a, b = b, a
print(a, b)  # 2 1

# Extended unpacking
first, *rest = [1, 2, 3, 4, 5]
print(first)  # 1
print(rest)   # [2, 3, 4, 5]

*start, last = [1, 2, 3, 4, 5]
print(start)  # [1, 2, 3, 4]
print(last)   # 5

# Tuples are IMMUTABLE
point = (10, 20)
# point[0] = 99 ← TypeError: cannot modify tuple

# When to use tuple vs list:
# tuple → fixed data (coordinates, RGB, DB record columns)
# list  → data that changes (shopping cart, user list)

# Real backend example:
# Database returns rows as tuples
db_row = (1, "alice@email.com", "Alice Smith", True)
user_id, email, name, is_active = db_row
print(f"User {user_id}: {name}")
```

### 9.4 Dictionaries - Key-Value Pairs
A **Python dictionary stores data in key-value pairs** where each unique key maps directly to a specific value. Dictionaries are written using curly braces `{}` with colons `:` separating the keys and values. Dicts are ordered & changeable, No Duplicates.

Watch [this](https://youtu.be/MZZSMaEAC2g) for better understanding.
```python
# ─────────────────────────────────────────
# DICTIONARIES
# ─────────────────────────────────────────
# Key-value pairs (like JSON)
# Keys must be unique
# Ordered (since Python 3.7)
# Mutable

# Creating
empty_dict = {}
user = {
    "id": 1,
    "name": "Alice Smith",
    "email": "alice@example.com",
    "age": 25,
    "is_active": True,
    "role": "admin"
}

# Accessing values
print(user["name"])        # Alice Smith
print(user.get("name"))    # Alice Smith
print(user.get("phone"))   # None (key doesn't exist)
print(user.get("phone", "N/A"))  # N/A (default value)

# user["phone"]      ← raises KeyError if not found
# user.get("phone")  ← returns None if not found
# ALWAYS use .get() when key might not exist

# CRUD Operations:

# CREATE (add new key-value)
user["phone"] = "+1-555-0123"
user.update({"country": "USA", "city": "New York"})

# READ
print("email" in user)      # True (check key exists)
print("password" in user)   # False

print(user.keys())     # dict_keys(['id', 'name', ...])
print(user.values())   # dict_values([1, 'Alice Smith', ...])
print(user.items())    # dict_items([('id', 1), ('name', 'Alice'), ...])

# UPDATE
user["name"] = "Alice Johnson"
user.update({"age": 26, "city": "Boston"})

# DELETE
del user["phone"]
removed = user.pop("city")        # removes & returns value
user.pop("nonexistent", None)     # safe pop with default

# Iterating
for key in user:
    print(key)

for value in user.values():
    print(value)

for key, value in user.items():  # most common
    print(f"{key}: {value}")

# Nested dictionaries
api_response = {
    "status": "success",
    "data": {
        "user": {
            "id": 1,
            "name": "Alice",
            "address": {
                "city": "New York",
                "country": "USA"
            }
        }
    },
    "errors": []
}

# Accessing nested data
print(api_response["data"]["user"]["name"])   # Alice
print(api_response["data"]["user"]["address"]["city"])  # New York

# Safe nested access
city = api_response.get("data", {}).get("user", {}).get("address", {}).get("city")
print(city)  # New York

# Real backend: this IS your API request/response format
# JSON in Python = dictionary
```

---
## 10. Comprehensions - Python's Superpower
**Comprehensions** in Python are concise, single-line syntactic patterns used to create new collections from existing iterables. They replace verbose `for` loops, making your code cleaner, more readable, and often faster.

For better understanding, watch [this](https://youtu.be/YlY2g2xrl6Q).
```python
# LIST COMPREHENSION
# Short, readable way to create lists

# Old way:
squares = []
for n in range(1, 6):
    squares.append(n ** 2)
print(squares)  # [1, 4, 9, 16, 25]

# List comprehension (one line):
squares = [n ** 2 for n in range(1, 6)]
print(squares)  # [1, 4, 9, 16, 25]

# With condition:
even_squares = [n ** 2 for n in range(1, 11) if n % 2 == 0]
print(even_squares)  # [4, 16, 36, 64, 100]

# Real backend examples:
users = [
    {"name": "Alice", "is_active": True},
    {"name": "Bob", "is_active": False},
    {"name": "Charlie", "is_active": True},
]

# Get all active user names
active_names = [u["name"] for u in users if u["is_active"]]
print(active_names)  # ['Alice', 'Charlie']

# ─────────────────────────────────────────
# DICT COMPREHENSION
# ─────────────────────────────────────────
# Create dictionaries in one line
names = ["alice", "bob", "charlie"]

# Create dict of name → length
name_lengths = {name: len(name) for name in names}
print(name_lengths)  # {'alice': 5, 'bob': 3, 'charlie': 7}

# Real backend: transform a list into a lookup dict
users = [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"},
    {"id": 3, "name": "Charlie"},
]
users_by_id = {user["id"]: user for user in users}
print(users_by_id[2])  # {"id": 2, "name": "Bob"}
# Now you can look up any user by ID instantly

# ─────────────────────────────────────────
# SET COMPREHENSION
# ─────────────────────────────────────────
emails = ["alice@test.com", "bob@test.com", "alice@test.com"]
unique_emails = {email.lower() for email in emails}
print(unique_emails)  # {'alice@test.com', 'bob@test.com'}
```

---
## 11. Unpacking & Packing
**Packing and unpacking** in Python refer to grouping multiple individual values into a single collection (packing) and extracting values from a collection back into separate variables (unpacking). These operations make code highly readable, concise, and dynamic.
Watch [this](https://youtu.be/_kNL5NcOqkg) for better understanding. Also [this](https://youtu.be/mSUBfY-Geuc).
```python
# ─────────────────────────────────────────
# UNPACKING
# ─────────────────────────────────────────
# Extract values from collections into variables

# List/Tuple unpacking
coordinates = [40.7128, -74.0060]
lat, lng = coordinates
print(lat)  # 40.7128
print(lng)  # -74.0060

# Skip values with _
data = (1, "Alice", "admin", True)
user_id, name, _, is_active = data  # skip role
print(user_id, name, is_active)

# Extended unpacking with *
first, *middle, last = [1, 2, 3, 4, 5]
print(first)   # 1
print(middle)  # [2, 3, 4]
print(last)    # 5

# ─────────────────────────────────────────
# *args — packing positional arguments
# ─────────────────────────────────────────
def add_all(*numbers):
    print(type(numbers))  # <class 'tuple'>
    return sum(numbers)

print(add_all(1, 2, 3))          # 6
print(add_all(1, 2, 3, 4, 5))   # 15

# ─────────────────────────────────────────
# **kwargs — packing keyword arguments
# ─────────────────────────────────────────
def create_user(**fields):
    print(type(fields))  # <class 'dict'>
    print(fields)

create_user(name="Alice", email="alice@test.com", age=25)
# {'name': 'Alice', 'email': 'alice@test.com', 'age': 25}

# Using * and ** to UNPACK when calling functions
def greet(name, greeting):
    print(f"{greeting}, {name}!")

args = ["Alice", "Hello"]
greet(*args)  # unpacks list into positional args

kwargs = {"name": "Alice", "greeting": "Hello"}
greet(**kwargs)  # unpacks dict into keyword args

# Real backend use:
def create_db_record(table, **data):
    # data is a dict of column: value pairs
    columns = ", ".join(data.keys())
    values = ", ".join([str(v) for v in data.values()])
    query = f"INSERT INTO {table} ({columns}) VALUES ({values})"
    print(query)

create_db_record("users", name="Alice", email="alice@test.com", age=25)
# INSERT INTO users (name, email, age) VALUES (Alice, alice@test.com, 25)
```

---
## Visual Summary - Data Types Cheat Sheet
```
┌──────────┬───────────────┬──────────┬────────────┬──────────────┐
│ Type     │ Example       │ Ordered  │ Mutable    │ Duplicates   │
├──────────┼───────────────┼──────────┼────────────┼──────────────┤
│ list     │ [1, 2, 3]     │    Yes   │    Yes     │  Allowed     │
│ set      │ {1, 2, 3}     │    No    │    Yes     │  Not allowed │
│ tuple    │ (1, 2, 3)     │    Yes   │    No      │  Allowed     │
│ dict     │ {"a": 1}      │    Yes   │    Yes     │  Keys unique │
│ str      │ "hello"       │    Yes   │    No      │  Allowed     │
└──────────┴───────────────┴──────────┴────────────┴──────────────┘
```
**When to use what:**
  `list`  → collection that changes (cart items, query results)
  `tuple` → fixed data (coordinates, db columns, function returns)
  `set`   → unique items, fast lookup (permissions, tags)
  `dict`  → key-value data (user object, API response, config)

---
## Knowledge Check

1. What are the falsy values in Python?
2. What is the difference between `==` and `is`?
3. What does `str.split()` return?
4. How do you safely access a dict key that might not exist?
5. What is the difference between list and tuple?
6. When would you use a set instead of a list?
7. What does `enumerate()` give you?
8. What does the `*` operator do in function arguments?
9. What is a list comprehension?
10. What's wrong with: `0.1 + 0.2 == 0.3`?
11. What does `pop()` do on a list? On a dict?
12. How do you remove duplicates from a list?

---
## Practice Exercises - Do These Now

### Exercise 1:
```python
# Given this list of users, write code to:
# a) Get only active users
# b) Get list of just their emails (lowercase)
# c) Create a dict of id -> user
users = [
    {"id": 1, "name": "Alice", "email": "Alice@Test.com", "is_active": True},
    {"id": 2, "name": "Bob", "email": "Bob@Test.com", "is_active": False},
    {"id": 3, "name": "Charlie", "email": "Charlie@Test.com", "is_active": True},
    {"id": 4, "name": "Dave", "email": "Dave@Test.com", "is_active": True},
]
```

### Exercise 2:
```python
# Given a string of comma-separated tags:
# "python,backend,fastapi,python,api,backend"
# Get a sorted list of UNIQUE tags
tags_string = "python,backend,fastapi,python,api,backend"
```

### Exercise 3:
```python
# Given a list of HTTP status codes,
# print what each one means using if/elif/else
status_codes = [200, 404, 500, 201, 403, 301]
```
### Exercise 4:
```python
# Build a simple grade calculator
# scores = [85, 92, 78, 95, 88]
# Calculate: average, highest, lowest, pass/fail
# Pass = average >= 80
scores = [85, 92, 78, 95, 88]
```

---