**Right now you write code like this:**
```python
# Procedural — just functions and variables scattered around
user_name = "Alice"
user_email = "alice@test.com"
user_age = 25
user_is_active = True

def get_user_display(name, email):
    return f"{name} <{email}>"

def deactivate_user(user_is_active):
    return False
```

**Problems:**
- 100 users = 400 variables floating around
- Functions and data are SEPARATE
- No clear structure
- Impossible to scale

**With OOP:**
```python
class User:
    def __init__(self, name, email, age):
        self.name = name
        self.email = email
        self.age = age
        self.is_active = True

    def get_display(self):
        return f"{self.name} <{self.email}>"

    def deactivate(self):
        self.is_active = False

# Now you can have 1000 users cleanly
alice = User("Alice", "alice@test.com", 25)
bob = User("Bob", "bob@test.com", 30)

print(alice.get_display())   # Alice <alice@test.com>
alice.deactivate()
print(alice.is_active)       # False
```
**OOP bundles DATA and BEHAVIOR together into one unit called a CLASS.**
Watch [this](https://youtu.be/IbMDCwVm63M).

---
## Setup
```bash
cd ~/projects
mkdir oop_basics
cd oop_basics
python3 -m venv venv
source venv/bin/activate
touch oop.py
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
touch oop.py
code .
```

---
## 1. Classes & Objects
**CLASS** = blueprint / template.
**OBJECT** = actual thing built from the blueprint.
### The Blueprint Analogy
Blueprint for a house:
  - Has: rooms, doors, windows
  - Can: open doors, turn on lights

```
House 1 (object 1): 3 rooms, red door, built in NYC
House 2 (object 2): 5 rooms, blue door, built in LA
```
Both follow the SAME blueprint (class), but each has its OWN data (instance variables).

### Your First Class
```python
# oop.py
class User:
    pass  # empty class for now

# Creating objects (instances) from the class
alice = User()
bob = User()

print(type(alice))  # <class '__main__.User'>
print(type(bob))    # <class '__main__.User'>

# alice and bob are DIFFERENT objects
print(alice is bob)   # False
print(id(alice))      # different memory addresses
print(id(bob))
```

---
## 2. `__init__` and `self`
In Python, `__init__` is a **special method used to initialize new objects**, while `self` represents the **specific instance of the class** being created or manipulated. Together, they allow you to assign unique data to individual objects when using Object-Oriented Programming.

| Concept        | What It Is                | Purpose                                                 | How It is Passed                       |
| -------------- | ------------------------- | ------------------------------------------------------- | -------------------------------------- |
| **`__init__`** | Special built-in method   | Acts as an initializer/constructor to set up attributes | Called automatically by Python         |
| **`self`**     | Parameter name convention | References the specific object being created/modified   | Passed automatically behind the scenes |

### 2.1 What is `__init__`?
`__init__` = the initializer (constructor).
- It runs AUTOMATICALLY when you create an object.
- It sets up the initial state of the object.

`__init__` is a *"dunder"* method (double underscore on both sides). Also called "magic methods".

### 2.2 What is `self`?
`self` = the object itself.
```python
# When you write:
alice = User("Alice", "alice@test.com")
# Python calls:
User.__init__(alice, "Alice", "alice@test.com")
              # ↑
        # self IS alice
```
`self` lets the method know WHICH object it's working on. You always write it as the FIRST parameter. You never pass it manually - Python does that.

### 2.3 Example
```python
class User:
    def __init__(self, name: str, email: str, age: int):
        # ↑
        # self = the new object being created

        # Instance variables — stored ON the object
        self.name = name
        self.email = email
        self.age = age
        self.is_active = True  # default value
        self.posts = []        # starts empty

# Creating instances
alice = User("Alice", "alice@test.com", 25)
bob = User("Bob", "bob@test.com", 30)

# Accessing instance variables
print(alice.name)       # Alice
print(alice.email)      # alice@test.com
print(alice.is_active)  # True
print(bob.name)         # Bob
print(bob.age)          # 30

# Each object has its OWN copy of the data
alice.name = "Alice Smith"  # only changes alice's name
print(alice.name)           # Alice Smith
print(bob.name)             # Bob — unchanged
```

---
## 3. Instance Methods, Class Methods, Static Methods
In OOP languages in Python, **instance methods, class methods, and static methods** define how a function inside a class behaves and what data it can access.

| Method Type         | [[Advanced Python#2. Decorators\|Decorators]] | First Argument      | Can Modify Instance State? | Can Modify Class State? |
| ------------------- | --------------------------------------------- | ------------------- | -------------------------- | ----------------------- |
| **Instance Method** | None (Default)                                | `self` (the object) | **Yes**                    | **Yes**                 |
| **Class Method**    | `@classmethod`                                | `cls` (the class)   | No                         | **Yes**                 |
| **Static Method**   | `@staticmethod`                               | None                | No                         | No                      |

### Instance Methods
In Python, an **instance method** is a function defined inside a class that **operates directly on a specific instance (object)** of that class. They are the most common type of method used in Python Object-Oriented Programming (OOP).
```python
class User:
    def __init__(self, name: str, email: str, age: int):
        self.name = name
        self.email = email
        self.age = age
        self.is_active = True
        self.login_count = 0

    # Instance method — always has self as first parameter
    # Can access and modify instance variables
    def greet(self) -> str:
        return f"Hello, I'm {self.name}!"

    def login(self) -> None:
        if not self.is_active:
            raise ValueError("Account is deactivated")
        self.login_count += 1
        print(f"{self.name} logged in. Total logins: {self.login_count}")

    def deactivate(self) -> None:
        self.is_active = False
        print(f"Account {self.email} deactivated")

    def get_profile(self) -> dict:
        return {
            "name": self.name,
            "email": self.email,
            "age": self.age,
            "is_active": self.is_active,
            "login_count": self.login_count
        }

    # Methods can call OTHER methods using self
    def display_info(self) -> str:
        status = "active" if self.is_active else "inactive"
        return f"{self.greet()} | Status: {status}"

# Using instance methods
alice = User("Alice", "alice@test.com", 25)
print(alice.greet())          # Hello, I'm Alice!
alice.login()                 # Alice logged in. Total logins: 1
alice.login()                 # Alice logged in. Total logins: 2
print(alice.get_profile())
print(alice.display_info())
alice.deactivate()
print(alice.is_active)        # False
```

### Class Methods
In Python, a **class method** is a method that is bound to the class itself rather than its individual object instances. It is defined using the **`@classmethod` decorator** and automatically receives the class as its first argument, which is conventionally named `cls`. Watch [this](https://youtu.be/g-qRKZD3FgE). You can learn [[Advanced Python#2. Decorators|Decorators]] before studying this.
```python
class User:
    # Class variable — SHARED by ALL instances
    total_users = 0
    DEFAULT_ROLE = "user"

    def __init__(self, name: str, email: str, age: int, role: str = None):
        self.name = name
        self.email = email
        self.age = age
        self.role = role or User.DEFAULT_ROLE
        self.is_active = True

        # Increment class variable when new user created
        User.total_users += 1

    # Class method — works with the CLASS, not an instance
    # @classmethod decorator + cls as first parameter
    @classmethod
    def get_total_users(cls) -> int:
        """Return total number of users created."""
        return cls.total_users

    @classmethod
    def create_admin(cls, name: str, email: str, age: int) -> "User":
        """
        Alternative constructor — factory method.
        Creates a User with admin role.
        """
        user = cls(name, email, age, role="admin")
        return user

    @classmethod
    def from_dict(cls, data: dict) -> "User":
        """
        Alternative constructor.
        Create a User from a dictionary (like JSON from API).
        """
        return cls(
            name=data["name"],
            email=data["email"],
            age=data["age"],
            role=data.get("role", "user")
        )

    def __repr__(self):
        return f"User(name={self.name!r}, email={self.email!r})"

# Using class methods
alice = User("Alice", "alice@test.com", 25)
bob = User("Bob", "bob@test.com", 30)

print(User.get_total_users())  # 2
print(alice.get_total_users()) # 2 (can call on instance too)

# Factory method pattern — very common in backends
admin = User.create_admin("Admin", "admin@test.com", 35)
print(admin.role)  # admin

# from_dict — extremely common when reading API/JSON data
user_data = {
    "name": "Charlie",
    "email": "charlie@test.com",
    "age": 28,
    "role": "moderator"
}
charlie = User.from_dict(user_data)
print(charlie)  # User(name='Charlie', email='charlie@test.com')
```

### Static Methods
In Python, a **static method** is a function defined inside a class that **does not have access to the instance (`self`) or the class (`cls`) data**. It behaves exactly like an ordinary function but lives inside the class's namespace for logical grouping and organization. Watch [this](https://youtu.be/GZYWTm7JoWs).

```python
class User:
    def __init__(self, name: str, email: str, age: int):
        self.name = name
        self.email = email
        self.age = age

    # Static method — no self, no cls
    # Just a regular function that belongs to the class
    # Used for utility functions related to the class
    @staticmethod
    def validate_email(email: str) -> bool:
        """Validate email format."""
        if not email or not isinstance(email, str):
            return False
        parts = email.strip().split("@")
        if len(parts) != 2:
            return False
        local, domain = parts
        return bool(local) and "." in domain

    @staticmethod
    def hash_password(password: str) -> str:
        """Hash a password (simplified — use bcrypt in real code)."""
        if len(password) < 8:
            raise ValueError("Password too short")
        return f"hashed_{password}"  # simplified

    @staticmethod
    def generate_username(name: str) -> str:
        """Generate username from full name."""
        return name.lower().replace(" ", "_")

# Static methods — call on class OR instance
print(User.validate_email("alice@test.com"))   # True
print(User.validate_email("not-an-email"))     # False
print(User.hash_password("SecurePass1"))       # hashed_SecurePass1
print(User.generate_username("Alice Smith"))   # alice_smith

alice = User("Alice", "alice@test.com", 25)
print(alice.validate_email("test@test.com"))   # True (works on instance too)
```

### When to Use Which?
```
Instance method → needs access to instance data (self)
                  most common type
                  def method(self):

Class method   → needs access to CLASS itself (cls)
                  alternative constructors
                  working with class variables
                  @classmethod
                  def method(cls):

Static method  → doesn't need self OR cls utility/helper functions
                  logically belongs to the class
                  but doesn't USE the class
                  
                  @staticmethod
                  def method():
```

---
## 4. Encapsulation
**Encapsulation in Python** is the Object-Oriented Programming (OOP) concept of **bundling data (attributes) and methods (functions) into a single unit (a class)** while restricting direct access to some of the object's components. Instead of modifying data directly, you interact with it through an interface to prevent accidental changes and maintain data integrity.

Encapsulation = controlling access to data hiding internal implementation details exposing only what's necessary.

**Like a car:**
  - Public:   steering wheel, pedals, gear shift (you interact with these)
  - Private:  engine internals, fuel injection system (you don't touch these directly)

### Public, Protected, Private
In Python, **public, private, and protected members do not have strict enforcement** like they do in C++ or Java. Instead, Python relies on **naming conventions and a mechanism called name mangling** to control visibility and encapsulate data.

| Access Level  | Naming Convention | Enforced by Language?           | Intended Access Scope                    |
| ------------- | ----------------- | ------------------------------- | ---------------------------------------- |
| **Public**    | `member`          | No                              | Anywhere (Inside, subclass, and outside) |
| **Protected** | `_member`         | No (**Convention only**)        | Within the class and its subclasses      |
| **Private**   | `__member`        | Partially via **Name Mangling** | Only inside the defining class           |

#### 1. Public Members
All class attributes and methods in Python are **public by default**. They can be read or modified freely from inside the class, by derived subclasses, or from outside the class instance.

```python title:public.py
class Car:
def __init__(self, brand):
    self.brand = brand  # Public attribute

car = Car("Tesla")
print(car.brand)  # Works: accessible outside the class
```

#### 2. Protected Members
A member is marked protected by adding a **single leading underscore (`_`)** to its name.
- This is strictly a **gentleman's agreement**.
- Python will not throw an error if you access it from outside.
- It serves as a visual warning to other developers that the member is intended for internal use and subclassing only.

```python title:protected.py
class Vehicle:
    def __init__(self):
        self._engine_type = "Electric"  # Protected attribute

class Car(Vehicle):
    def display(self):
        print(self._engine_type)  # Intended usage: accessed in subclass

car = Car()
print(car._engine_type)  # Works, but highly discouraged by convention
```

#### 3. Private Members
A member is marked private by adding a **double leading underscore (`__`)** to its name.
- Python partially enforces this using **Name Mangling**.
- If you try to call `instance.__variable` from outside the class, Python raises an `AttributeError`.
- **How Name Mangling Works**: Internally, the interpreter renames the variable to `_ClassName__variable`. You can technically still access or break privacy by using this mangled name, though you should never do so in production code.
- **Note:** _Avoid adding a double trailing underscore (like `__init__`), as those are reserved "dunder" methods and are public._

```python title:private.py
class BankAccount:
    def __init__(self, pin):
        self.__pin = pin  # Private attribute

account = BankAccount(1234)

# print(account.__pin)  
# Error: AttributeError! Python hid it.

# How to bypass via Name Mangling (Avoid doing this!):
print(account._BankAccount__pin)  # Works: outputs 1234

```

#### All Together
```python
class BankAccount:
    def __init__(self, owner: str, initial_balance: float):
        # Public — anyone can access
        self.owner = owner

        # Protected — convention: single underscore
        # "please don't access directly, but Python won't stop you"
        # Used within the class and subclasses
        self._transaction_history = []

        # Private — double underscore
        # Python "mangles" the name to make it harder to access
        # Even subclasses can't easily access it
        self.__balance = initial_balance

    # Public interface — controlled access to private data
    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self.__balance += amount
        self._transaction_history.append(f"Deposit: +${amount}")
        print(f"Deposited ${amount}. New balance: ${self.__balance}")

    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if amount > self.__balance:
            raise ValueError("Insufficient funds")
        self.__balance -= amount
        self._transaction_history.append(f"Withdrawal: -${amount}")
        print(f"Withdrew ${amount}. New balance: ${self.__balance}")

    def get_balance(self) -> float:
        """Public method to safely read balance."""
        return self.__balance

    def get_history(self) -> list:
        """Public method to read transaction history."""
        return self._transaction_history.copy()  # return copy, not original

account = BankAccount("Alice", 1000.0)

# Public access — fine
print(account.owner)  # Alice

# Protected access — works but signals "don't do this"
print(account._transaction_history)  # []

# Private access — name mangling makes it hard
# print(account.__balance)           # AttributeError!

# Python mangles __balance to _BankAccount__balance
print(account._BankAccount__balance)  # 1000.0 (mangled name — don't do this)

# Use the public interface
account.deposit(500)
account.withdraw(200)
print(account.get_balance())  # 1300.0
print(account.get_history())  # ['Deposit: +$500', 'Withdrawal: -$200']
```

---
## 5. Access Modifiers
In Python, **access modifiers do not exist as strict keywords** like they do in languages like Java or C++. Instead, Python relies on a community-accepted **naming convention using underscores** to dictate the visibility and accessibility of class attributes and methods.

**Comparison of Python Access Modifiers:**

|Access Level|Syntax Pattern|Intended Scope|Enforced by Interpreter?|
|---|---|---|---|
|**Public**|`name`|Accessible from anywhere.|No (Default behavior).|
|**Protected**|`_name` (Single underscore)|Inside the class and its subclasses.|No (Convention only).|
|**Private**|`__name` (Double underscore)|Strictly inside the defining class.|**Yes, via Name Mangling**.|

### 1. Public Access Modifier
By default, **every attribute and method in a Python class is public**. They can be modified and accessed freely from both inside and outside the class.

```python
class Developer:
    def __init__(self, name):
        self.name = name  # Public attribute

dev = Developer("Alice")
print(dev.name)  # Output: Alice (Accessible directly)
```

### 2. Protected Access Modifier
To mark a member as protected, you prefix its name with a **single underscore (`_`)**. This acts as a warning to other developers that the variable is intended for internal use and within subclasses, but Python **will still allow you to access it externally** if you choose to ignore the convention.
```python
class Company:
    def __init__(self):
        self._project = "Internal Tool"  # Protected attribute

class Team(Company):
    def show_project(self):
        print(self._project)  # OK: Accessible in subclass

emp = Team()
print(emp._project)  # Output: Internal Tool (Works, but highly discouraged)
```

### 3. Private Access Modifier
To mark a member as private, you prefix its name with a **double underscore (`__`)**. If you attempt to access a private variable directly from an object outside the class, Python will raise an `AttributeError`.
```python
class Account:
    def __init__(self, pin):
        self.__pin = pin  # Private attribute

acc = Account(1234)
# print(acc.__pin)  # Raises AttributeError!
```

### How Python Enforces Privacy: Name Mangling
Python doesn't completely lock out private variables; instead, it uses a process called **Name Mangling**. The interpreter internally renames the attribute by adding a leading underscore and the class name to the front: `_ClassName__attributeName`.

You can technically still bypass the restriction using this mangled name:
```python
# Accessing private member externally via Name Mangling
print(acc._Account__pin)  # Output: 1234
```

**Note on Dunder Methods:** Attributes that both start and end with double underscores (like `__init__` or `__str__`) are called **dunder (double underscore) or special methods**. Name mangling **does not apply** to them, and they are not considered private modifiers.

---
## 6. Property Decorators
The **`@property` decorator** in Python is a built-in decorator that allows you to treat class methods like standard instance attributes. It provides an elegant way to implement **getters, setters, and deleters**, allowing you to execute background logic (such as data validation or logging) without breaking your class's public interface. Watch [this](https://youtu.be/HkbQ_NaH0Lc).

### The Problem
```python
class User:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age  # public — no validation

# Anyone can set invalid values
user = User("Alice", 25)
user.age = -5       # negative age — wrong but allowed
user.age = "old"    # string instead of int — wrong but allowed
```

### Solution: `@property`
The **`@property` decorator** in Python is a built-in tool that allows you to define a method but access it **like a regular data attribute** (without using parentheses `()`).

It is primarily used to implement **getters, setters, and deleters** in an elegant, Pythonic way. This lets you enforce validation, compute values on the fly, or make attributes read-only without breaking the public API of your class.
```python
class User:
    def __init__(self, name: str, age: int, email: str):
        self._name = name
        self._email = email
        self.age = age      # uses the setter below

    # ─────────────────────────────────────────
    # PROPERTY — getter
    # Accessed like an attribute: user.age
    # But runs code behind the scenes
    # ─────────────────────────────────────────
    @property
    def age(self) -> int:
        return self._age

    # SETTER — runs when you do: user.age = value
    @age.setter
    def age(self, value: int) -> None:
        if not isinstance(value, int):
            raise TypeError("Age must be an integer")
        if value < 0 or value > 150:
            raise ValueError(f"Age {value} is not realistic")
        self._age = value

    # DELETER — runs when you do: del user.age
    @age.deleter
    def age(self) -> None:
        del self._age
        print("Age deleted")

    # ─────────────────────────────────────────
    # Read-only property (no setter)
    # ─────────────────────────────────────────
    @property
    def email(self) -> str:
        return self._email

    # ─────────────────────────────────────────
    # Computed property — calculated on the fly
    # ─────────────────────────────────────────
    @property
    def name(self) -> str:
        return self._name

    @name.setter
    def name(self, value: str) -> None:
        if not value or not isinstance(value, str):
            raise ValueError("Name must be a non-empty string")
        self._name = value.strip().title()

    @property
    def display_name(self) -> str:
        """Computed from name — no setter needed."""
        return f"@{self._name.lower().replace(' ', '_')}"

    @property
    def is_adult(self) -> bool:
        """Computed property."""
        return self._age >= 18

    def __repr__(self) -> str:
        return f"User(name={self._name!r}, age={self._age})"

# Using properties — looks like attribute access
# but runs validation code
user = User("alice smith", 25, "alice@test.com")
print(user.name)          # Alice Smith (auto-formatted)
print(user.age)           # 25
print(user.display_name)  # @alice_smith
print(user.is_adult)      # True

user.age = 30             # uses setter — validates
print(user.age)           # 30

user.name = "  bob jones  "
print(user.name)          # Bob Jones (stripped and titled)

# Validation kicks in
try:
    user.age = -5         # ValueError
except ValueError as e:
    print(e)              # Age -5 is not realistic

try:
    user.age = "old"      # TypeError
except TypeError as e:
    print(e)              # Age must be an integer

# Read-only property — no setter
try:
    user.email = "new@test.com"  # AttributeError
except AttributeError as e:
    print(e)              # can't set attribute

# Real backend use:
class Product:
    def __init__(self, name: str, price: float, stock: int):
        self._name = name
        self.price = price    # triggers setter
        self.stock = stock    # triggers setter

    @property
    def price(self) -> float:
        return self._price

    @price.setter
    def price(self, value: float) -> None:
        if not isinstance(value, (int, float)):
            raise TypeError("Price must be a number")
        if value < 0:
            raise ValueError("Price cannot be negative")
        self._price = round(float(value), 2)

    @property
    def stock(self) -> int:
        return self._stock

    @stock.setter
    def stock(self, value: int) -> None:
        if not isinstance(value, int):
            raise TypeError("Stock must be an integer")
        if value < 0:
            raise ValueError("Stock cannot be negative")
        self._stock = value

    @property
    def is_available(self) -> bool:
        return self._stock > 0

    @property
    def formatted_price(self) -> str:
        return f"${self._price:.2f}"

product = Product("Laptop", 999.99, 50)
print(product.formatted_price)  # $999.99
print(product.is_available)     # True
product.stock = 0
print(product.is_available)     # False
```

---
## 7. Inheritance
**Inheritance** in Python is a core Object-Oriented Programming (OOP) feature that allows a child class to adopt the attributes and methods of a parent class. It maximizes code reusability and builds clear, logical hierarchies within a codebase

Inheritance = a class INHERITS attributes and methods from another class.
```
Parent class (Base class) → has common functionality
Child class (Subclass)    → inherits + adds specific stuff

Real world:
  Animal (parent) → has: name, eat(), sleep()
  Dog   (child)   → inherits Animal + adds: breed, bark()
  Cat   (child)   → inherits Animal + adds: indoor, meow()
```

### Single Inheritance
**Single inheritance in Python occurs when a child class (subclass) inherits attributes and methods from exactly one parent class (superclass)**. This represents a direct, one-to-one relationship that promotes code reusability and builds a clean logical hierarchy. Watch [this](https://youtu.be/an59YHkdK9A) for inheritance.
```python
# BASE CLASS (Parent)
class BaseModel:
    """Base class for all database models."""
    def __init__(self, id: int):
        self.id = id
        self.created_at = "2024-01-15"  # simplified
        self.updated_at = "2024-01-15"
        self.is_deleted = False

    def soft_delete(self) -> None:
        """Mark as deleted without removing from database."""
        self.is_deleted = True
        self.updated_at = "2024-01-16"

    def to_dict(self) -> dict:
        return {
            "id": self.id,
            "created_at": self.created_at,
            "updated_at": self.updated_at,
        }

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}(id={self.id})"
```

Watch [this](https://youtu.be/HzyhmZqiaE8) for `super()` function.
```python
# CHILD CLASS — inherits from BaseModel
class User(BaseModel):
    def __init__(self, id: int, name: str, email: str):
        super().__init__(id)   # ← call parent's __init__ first!
        self.name = name
        self.email = email
        self.role = "user"

    def to_dict(self) -> dict:
        # Get parent's dict and ADD to it
        base = super().to_dict()  # {'id': ..., 'created_at': ...}
        base.update({
            "name": self.name,
            "email": self.email,
            "role": self.role
        })
        return base

    def promote_to_admin(self) -> None:
        self.role = "admin"

class Post(BaseModel):
    def __init__(self, id: int, title: str, content: str, author_id: int):
        super().__init__(id)
        self.title = title
        self.content = content
        self.author_id = author_id
        self.published = False

    def publish(self) -> None:
        self.published = True
        self.updated_at = "2024-01-16"

    def to_dict(self) -> dict:
        base = super().to_dict()
        base.update({
            "title": self.title,
            "content": self.content,
            "author_id": self.author_id,
            "published": self.published
        })
        return base

# Using inheritance
user = User(1, "Alice", "alice@test.com")
print(user)                # User(id=1)
print(user.id)             # 1 (from BaseModel)
print(user.created_at)     # 2024-01-15 (from BaseModel)
print(user.name)           # Alice (from User)

user.soft_delete()         # inherited method from BaseModel
print(user.is_deleted)     # True
print(user.to_dict())

post = Post(1, "My First Post", "Hello World!", author_id=1)
print(post)                # Post(id=1)
post.publish()
print(post.published)      # True

# isinstance and issubclass
print(isinstance(user, User))       # True
print(isinstance(user, BaseModel))  # True (it IS a BaseModel)
print(isinstance(user, Post))       # False
print(issubclass(User, BaseModel))  # True
print(issubclass(Post, BaseModel))  # True
print(issubclass(User, Post))       # False
```

### `super()` - Calling Parent Methods
The **`super()` function in Python** gives you access to methods and properties of a parent (or sibling) class without naming it explicitly. It is primarily used to **extend or reuse parent class functionality** inside a subclass, ensuring clean initialization and proper method overriding.

```python
class Vehicle:
    def __init__(self, make: str, model: str, year: int):
        self.make = make
        self.model = model
        self.year = year
        self.is_running = False

    def start(self) -> str:
        self.is_running = True
        return f"{self.make} {self.model} started"

    def stop(self) -> str:
        self.is_running = False
        return f"{self.make} {self.model} stopped"

    def __str__(self) -> str:
        return f"{self.year} {self.make} {self.model}"

class ElectricVehicle(Vehicle):
    def __init__(self, make: str, model: str, year: int, battery_capacity: int):
        super().__init__(make, model, year)  # call Vehicle.__init__
        self.battery_capacity = battery_capacity
        self.charge_level = 100

    def start(self) -> str:
        if self.charge_level == 0:
            return "Cannot start: battery empty"
        # Call parent's start AND add extra behavior
        message = super().start()
        return f"{message} silently (electric)"

    def charge(self, amount: int) -> None:
        self.charge_level = min(100, self.charge_level + amount)
        print(f"Charging... Battery at {self.charge_level}%")

    def __str__(self) -> str:
        base = super().__str__()  # "2024 Tesla Model 3"
        return f"{base} (Electric, {self.battery_capacity}kWh)"

tesla = ElectricVehicle("Tesla", "Model 3", 2024, 75)
print(tesla)               # 2024 Tesla Model 3 (Electric, 75kWh)
print(tesla.start())       # 2024 Tesla Model 3 started silently (electric)
tesla.charge(50)           # can't charge beyond 100
print(tesla.charge_level)  # 100 (already full)
print(tesla.stop())        # 2024 Tesla Model 3 stopped
```

### Multiple Inheritance & MRO
**Multiple Inheritance** in Python occurs when a child class derives attributes and methods from **more than one parent class**. To handle conflicts when multiple parents share identical method names, Python relies on the **Method Resolution Order (MRO)**, an ordered lookup path computed via the **C3 Linearization algorithm**. Watch [this](https://youtu.be/Q8YlYHjksLo).
```python
# Python allows inheriting from multiple classes
# MRO = Method Resolution Order
# Determines which parent's method gets called

class Flyable:
    def move(self) -> str:
        return "Flying"
    def describe(self) -> str:
        return "I can fly"

class Swimmable:
    def move(self) -> str:
        return "Swimming"
    def describe(self) -> str:
        return "I can swim"

class Duck(Flyable, Swimmable):  # inherits from BOTH
    def __init__(self, name: str):
        self.name = name
    def quack(self) -> str:
        return f"{self.name} says: Quack!"

donald = Duck("Donald")
print(donald.quack())     # Donald says: Quack!
print(donald.move())      # Flying ← from Flyable (listed first)
print(donald.describe())  # I can fly ← from Flyable (listed first)

# MRO — the ORDER Python searches for methods
print(Duck.__mro__)
# (<class 'Duck'>, <class 'Flyable'>, <class 'Swimmable'>, <class 'object'>)
# Python searches: Duck → Flyable → Swimmable → object
```

### Real Backend - `Mixins`
```python
# Mixins — small reusable pieces

class TimestampMixin:
    """Add timestamp fields to any model."""
    def __init__(self):
        self.created_at = "2024-01-15"
        self.updated_at = "2024-01-15"

    def touch(self):
        """Update the updated_at timestamp."""
        self.updated_at = "2024-01-16"

class SoftDeleteMixin:
    """Add soft delete to any model."""
    def __init__(self):
        self.is_deleted = False
        self.deleted_at = None

    def delete(self):
        self.is_deleted = True
        self.deleted_at = "2024-01-16"

    def restore(self):
        self.is_deleted = False
        self.deleted_at = None

class Model:
    """Base database model."""
    def __init__(self, id: int):
        self.id = id

class Article(TimestampMixin, SoftDeleteMixin, Model):
    def __init__(self, id: int, title: str):
        Model.__init__(self, id)
        TimestampMixin.__init__(self)
        SoftDeleteMixin.__init__(self)
        self.title = title

article = Article(1, "Python OOP Guide")
print(article.id)            # 1
print(article.created_at)    # 2024-01-15
print(article.is_deleted)    # False

article.delete()
print(article.is_deleted)    # True
article.restore()
article.touch()
print(article.updated_at)    # 2024-01-16
```

---
## 8. Polymorphism
**Polymorphism** in Python is a core Object-Oriented Programming (OOP) concept that allows **different types of objects to respond to the same method or function call** in their own unique ways. Derived from the Greek words for _"many forms,"_ it shifts the programming focus away from an object's specific class to its actionable behaviors, making code significantly more flexible, modular, and reusable.

Polymorphism = "many forms"
Same method name → different behavior depending on the class
```
Like the word "speak":
  Human.speak() → "Hello"
  Dog.speak()   → "Woof"
  Cat.speak()   → "Meow"
```
Same method name, different implementation. You can call `.speak()` on ANY animal without knowing the type. Watch [this](https://youtu.be/tHN8I_4FIt8).

```python
class Notification:
    """Base notification class."""
    def __init__(self, recipient: str, message: str):
        self.recipient = recipient
        self.message = message

    def send(self) -> str:
        raise NotImplementedError("Subclasses must implement send()")

    def __str__(self) -> str:
        return f"{self.__class__.__name__} to {self.recipient}"

class EmailNotification(Notification):
    def __init__(self, recipient: str, message: str, subject: str):
        super().__init__(recipient, message)
        self.subject = subject

    def send(self) -> str:
        return f"Email sent to {self.recipient}: [{self.subject}] {self.message}"

class SMSNotification(Notification):
    def __init__(self, recipient: str, message: str, phone: str):
        super().__init__(recipient, message)
        self.phone = phone

    def send(self) -> str:
        return f"SMS sent to {self.phone}: {self.message[:160]}"

class PushNotification(Notification):
    def __init__(self, recipient: str, message: str, device_token: str):
        super().__init__(recipient, message)
        self.device_token = device_token

    def send(self) -> str:
        return f"Push sent to device {self.device_token[:8]}...: {self.message}"

class WebhookNotification(Notification):
    def __init__(self, recipient: str, message: str, url: str):
        super().__init__(recipient, message)
        self.url = url

    def send(self) -> str:
        return f"Webhook POST to {self.url}: {self.message}"

# POLYMORPHISM in action:
# We have a LIST of different notification types
# We call .send() on ALL of them — same interface
# Each does the right thing for its type

def send_all_notifications(notifications: list) -> None:
    """Send a list of notifications — works with any type."""
    for notification in notifications:
        result = notification.send()  # ← polymorphism here
        print(result)

# Create different types
notifications = [
    EmailNotification("alice@test.com", "Your order shipped!", "Order Update"),
    SMSNotification("alice@test.com", "Your order shipped!", "+1-555-0123"),
    PushNotification("alice@test.com", "Your order shipped!", "abc123token"),
    WebhookNotification("alice@test.com", "Order shipped", "https://api.site.com/hook"),
]

# Send all — we don't care what TYPE each is
send_all_notifications(notifications)

# The power: adding a new notification type
# just requires a new class with .send()
# No changes to send_all_notifications()!
```

### Duck Typing
**Duck typing** is a programming concept where an object's suitability for a task is determined by the presence of certain methods and attributes, rather than its actual type or explicit class inheritance. Watch [this](https://youtu.be/Qe03kCuTMoU).
```python
# Duck typing - another form of polymorphism
# "If it walks like a duck and quacks like a duck, it's a duck"
# Python doesn't require formal inheritance for polymorphism

class SlackNotification:
    # Doesn't inherit from Notification
    # But has a .send() method
    def send(self) -> str:
        return "Posted to Slack channel"

# Works perfectly in send_all_notifications!
notifications.append(SlackNotification())
send_all_notifications(notifications)
```

---
## 9. Abstraction
**Abstraction in Python** is the process of hiding complex internal implementation details and exposing only the essential features of an object. It reduces code complexity by forcing developers to focus on _what_ an object does rather than _how_ it does it.

Abstraction = define WHAT something does without defining HOW it does it.
```
Like a contract: "Any class that inherits from me MUST implement these methods"
If they don't implement them → Python raises an error
```
Watch [this](https://youtu.be/97V7ICVeTJc).
```python
from abc import ABC, abstractmethod
from typing import List, Optional

class BaseRepository(ABC):
    """
    Abstract base class for all repositories.
    Defines the CONTRACT — every repository MUST implement these.
    """

    @abstractmethod
    def get_by_id(self, id: int) -> Optional[dict]:
        """Get a single record by ID."""
        pass

    @abstractmethod
    def get_all(self) -> List[dict]:
        """Get all records."""
        pass

    @abstractmethod
    def create(self, data: dict) -> dict:
        """Create a new record."""
        pass

    @abstractmethod
    def update(self, id: int, data: dict) -> Optional[dict]:
        """Update an existing record."""
        pass

    @abstractmethod
    def delete(self, id: int) -> bool:
        """Delete a record. Returns True if deleted."""
        pass

    # Non-abstract method — shared by all repositories
    def exists(self, id: int) -> bool:
        """Check if a record exists."""
        return self.get_by_id(id) is not None

# Cannot instantiate abstract class directly
try:
    repo = BaseRepository()  # TypeError!
except TypeError as e:
    print(f"Error: {e}")

# Concrete implementation — in-memory storage
class InMemoryUserRepository(BaseRepository):
    def __init__(self):
        self._storage = {}
        self._next_id = 1

    def get_by_id(self, id: int) -> Optional[dict]:
        return self._storage.get(id)

    def get_all(self) -> List[dict]:
        return list(self._storage.values())

    def create(self, data: dict) -> dict:
        user = {"id": self._next_id, **data}
        self._storage[self._next_id] = user
        self._next_id += 1
        return user

    def update(self, id: int, data: dict) -> Optional[dict]:
        if id not in self._storage:
            return None
        self._storage[id].update(data)
        return self._storage[id]

    def delete(self, id: int) -> bool:
        if id not in self._storage:
            return False
        del self._storage[id]
        return True

# Another implementation — PostgreSQL (simulated)
class PostgreSQLUserRepository(BaseRepository):
    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        print(f"Connected to: {connection_string}")

    def get_by_id(self, id: int) -> Optional[dict]:
        print(f"SELECT * FROM users WHERE id = {id}")
        return {"id": id, "name": "Alice"}  # simulated

    def get_all(self) -> List[dict]:
        print("SELECT * FROM users")
        return []  # simulated

    def create(self, data: dict) -> dict:
        print(f"INSERT INTO users VALUES {data}")
        return {"id": 1, **data}  # simulated

    def update(self, id: int, data: dict) -> Optional[dict]:
        print(f"UPDATE users SET {data} WHERE id = {id}")
        return {"id": id, **data}  # simulated

    def delete(self, id: int) -> bool:
        print(f"DELETE FROM users WHERE id = {id}")
        return True

# POWER OF ABSTRACTION:
# The service layer doesn't care WHICH repository it uses
# It just uses the contract (BaseRepository interface)

class UserService:
    def __init__(self, repository: BaseRepository):
        self.repo = repository  # inject any implementation

    def register_user(self, name: str, email: str) -> dict:
        # Check if exists
        all_users = self.repo.get_all()
        for user in all_users:
            if user.get("email") == email:
                raise ValueError("Email already registered")
        return self.repo.create({"name": name, "email": email})

    def get_user(self, user_id: int) -> dict:
        user = self.repo.get_by_id(user_id)
        if not user:
            raise ValueError(f"User {user_id} not found")
        return user

# Swap implementations without changing UserService!
in_memory_repo = InMemoryUserRepository()
service = UserService(in_memory_repo)

alice = service.register_user("Alice", "alice@test.com")
print(alice)  # {'id': 1, 'name': 'Alice', 'email': 'alice@test.com'}
bob = service.register_user("Bob", "bob@test.com")
print(service.get_user(1))  # Alice

# Switch to PostgreSQL — service code unchanged!
# pg_repo = PostgreSQLUserRepository("postgresql://localhost/mydb")
# service = UserService(pg_repo)
# service.register_user("Alice", "alice@test.com")  # same code!
```

---
## 10. `Magic/Dunder` Methods
**Magic methods** (also called **dunder methods** because they start and end with **d**ouble **under**scores) are special built-in Python methods that let you define how custom objects interact with standard syntax, operators, and built-in functions. You rarely call them directly; instead, Python triggers them automatically behind the scenes when you perform specific operations like adding numbers (`+`), fetching container lengths (`len()`), or printing an object.
Watch [this](https://youtu.be/qqp6QN20CpE). Also [this](https://youtu.be/NwjSP1_WEfE) (BroCode).
```python
class Vector:
    """2D Vector — shows common dunder methods."""

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

    # String representation for DEVELOPERS (repr)
    def __repr__(self) -> str:
        return f"Vector({self.x}, {self.y})"

    # String representation for USERS (str)
    def __str__(self) -> str:
        return f"({self.x}, {self.y})"

    # Length
    def __len__(self) -> int:
        return 2  # a 2D vector always has 2 components

    # Equality check ==
    def __eq__(self, other) -> bool:
        if not isinstance(other, Vector):
            return False
        return self.x == other.x and self.y == other.y

    # Less than < (for sorting)
    def __lt__(self, other) -> bool:
        return self.magnitude() < other.magnitude()

    # Addition +
    def __add__(self, other: "Vector") -> "Vector":
        return Vector(self.x + other.x, self.y + other.y)

    # Subtraction -
    def __sub__(self, other: "Vector") -> "Vector":
        return Vector(self.x - other.x, self.y - other.y)

    # Multiplication *
    def __mul__(self, scalar: float) -> "Vector":
        return Vector(self.x * scalar, self.y * scalar)

    # in operator — membership test
    def __contains__(self, value: float) -> bool:
        return value in (self.x, self.y)

    # [] operator — indexing
    def __getitem__(self, index: int) -> float:
        if index == 0: return self.x
        if index == 1: return self.y
        raise IndexError("Vector index out of range")

    # bool() — truthiness
    def __bool__(self) -> bool:
        return self.x != 0 or self.y != 0

    def magnitude(self) -> float:
        return (self.x ** 2 + self.y ** 2) ** 0.5

v1 = Vector(3, 4)
v2 = Vector(1, 2)

print(repr(v1))               # Vector(3, 4)          ← __repr__
print(str(v1))                # (3, 4)               ← __str__
print(len(v1))                # 2                    ← __len__
print(v1 == v2)               # False                ← __eq__
print(v1 == Vector(3, 4))     # True
print(v1 + v2)                # (4, 6)               ← __add__
print(v1 - v2)                # (2, 2)               ← __sub__
print(v1 * 3)                 # (9, 12)              ← __mul__
print(3 in v1)                # True                 ← __contains__
print(v1[0])                  # 3.0                  ← __getitem__
print(bool(v1))               # True                 ← __bool__
print(bool(Vector(0, 0)))     # False

# Sorting uses __lt__
vectors = [Vector(5, 5), Vector(1, 1), Vector(3, 4)]
sorted_vectors = sorted(vectors)
print([str(v) for v in sorted_vectors])
# ['(1, 1)', '(3, 4)', '(5, 5)']
```

### The Most Important Dunders for Backend
```python
class APIResponse:
    """Common dunders you'll actually use in backend code."""

    def __init__(self, data: dict, status_code: int = 200):
        self.data = data
        self.status_code = status_code
        self._items = list(data.items())

    # ─── Always implement these two ───────────────────────────
    def __repr__(self) -> str:
        """For debugging — unambiguous."""
        return f"APIResponse(status={self.status_code}, data={self.data})"

    def __str__(self) -> str:
        """For display — readable."""
        return f"[{self.status_code}] {self.data}"

    # ─── Equality ─────────────────────────────────────────────
    def __eq__(self, other) -> bool:
        if not isinstance(other, APIResponse):
            return False
        return self.status_code == other.status_code and self.data == other.data

    # ─── Boolean — is this response truthy? ───────────────────
    def __bool__(self) -> bool:
        """True if successful (2xx status code)."""
        return 200 <= self.status_code < 300

    # ─── Context manager (with statement) ─────────────────────
    def __enter__(self):
        print(f"Processing response {self.status_code}")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Response processing complete")
        return False  # don't suppress exceptions

    # ─── Length ───────────────────────────────────────────────
    def __len__(self) -> int:
        return len(self.data)

response = APIResponse({"user": "Alice", "role": "admin"}, 200)
print(repr(response))   # APIResponse(status=200, data={...})
print(str(response))    # [200] {'user': 'Alice', 'role': 'admin'}
print(bool(response))   # True (200 is success)

error = APIResponse({"message": "Not found"}, 404)
print(bool(error))      # False (404 is failure)

# Context manager
with APIResponse({"id": 1}, 201) as resp:
    print(resp.data)    # {'id': 1}
# Processing response 201
# {'id': 1}
# Response processing complete
```

---
## 11. Dataclasses
A **Python dataclass** is a regular class decorated with `@dataclass` from the built-in `dataclasses` module that **automatically generates special boilerplate methods** like `__init__()`, `__repr__()`, and `__eq__()` based on type annotations. Introduced in Python 3.7, it streamlines the creation of classes meant primarily for storing data. Watch [this](https://youtu.be/Gsfsq2epdr8).

Dataclasses = a shortcut for creating classes that mainly HOLD data.
```
Without dataclass: write __init__, __repr__, __eq__ manually
With dataclass: decorator does it automatically
```

```python
from dataclasses import dataclass, field
from typing import List, Optional

# ─────────────────────────────────────────
# WITHOUT dataclass (old way)
# ─────────────────────────────────────────
class UserOld:
    def __init__(self, id: int, name: str, email: str):
        self.id = id
        self.name = name
        self.email = emai
    def __repr__(self):
        return f"User(id={self.id}, name={self.name!r})"
    def __eq__(self, other):
        if not isinstance(other, UserOld):
            return False
        return self.id == other.id

# ─────────────────────────────────────────
# WITH dataclass (new way)
# ─────────────────────────────────────────
@dataclass
class User:
    id: int
    name: str
    email: str
    role: str = "user"                    # default value
    is_active: bool = True                # default value
    tags: List[str] = field(default_factory=list)  # mutable default!

    # You can still add custom methods
    def deactivate(self):
        self.is_active = False

    def add_tag(self, tag: str):
        self.tags.append(tag)

user1 = User(1, "Alice", "alice@test.com")
user2 = User(2, "Bob", "bob@test.com", role="admin")
user3 = User(1, "Alice", "alice@test.com")

print(user1)                    # User(id=1, name='Alice', email='alice@test.com', ...)
print(user1 == user3)           # True (same values → equal)
print(user1 == user2)           # False

user1.add_tag("python")
user1.add_tag("fastapi")
print(user1.tags)               # ['python', 'fastapi']
print(user2.tags)               # [] ← separate list (default_factory!)

user1.deactivate()
print(user1.is_active)          # False

# ─────────────────────────────────────────
# FROZEN dataclass — immutable
# ─────────────────────────────────────────
@dataclass(frozen=True)
class Coordinates:
    latitude: float
    longitude: float

    def to_tuple(self):
        return (self.latitude, self.longitude)

nyc = Coordinates(40.7128, -74.0060)
print(nyc)                      # Coordinates(latitude=40.7128, longitude=-74.006)

try:
    nyc.latitude = 0            # FrozenInstanceError!
except Exception as e:
    print(e)                    # cannot assign to field 'latitude'

# Frozen dataclasses are hashable → can be used as dict keys
locations = {nyc: "New York City"}
print(locations[Coordinates(40.7128, -74.0060)])  # New York City
```

### Real Backend - API Models
```python
# Real backend — API request/response models
# (before you learn Pydantic, which does this even better)

@dataclass
class CreatePostRequest:
    title: str
    content: str
    author_id: int
    tags: List[str] = field(default_factory=list)
    published: bool = False

@dataclass
class PostResponse:
    id: int
    title: str
    content: str
    author_id: int
    tags: List[str]
    published: bool
    created_at: str

request = CreatePostRequest(
    title="My First Post",
    content="Hello World!",
    author_id=1,
    tags=["python", "tutorial"]
)
print(request)
```

---
## 12. `__slots__`
In Python, **`__slots__`** is a special class-level attribute that optimizes memory usage and speeds up attribute access by preventing the creation of the default instance dictionary (`__dict__`). Watch [this](https://youtu.be/Iwf17zsDAnY).

```python
# __slots__ restricts what attributes an object can have
# Saves memory — no __dict__ per instance

class UserWithSlots:
    __slots__ = ["id", "name", "email", "is_active"]

    def __init__(self, id: int, name: str, email: str):
        self.id = id
        self.name = name
        self.email = email
        self.is_active = True

user = UserWithSlots(1, "Alice", "alice@test.com")
print(user.name)  # Alice

# Can't add new attributes
try:
    user.age = 25  # AttributeError!
except AttributeError as e:
    print(e)  # 'UserWithSlots' object has no attribute 'age'

# No __dict__ on the instance
try:
    print(user.__dict__)  # AttributeError — no __dict__!
except AttributeError:
    print("No __dict__ with __slots__")

# When to use __slots__:
# - You create MILLIONS of objects (saves memory significantly)
# - You want to prevent accidental attribute creation
# - Performance-critical code

# Memory comparison:
import sys

class WithDict:
    def __init__(self): self.x = 1

class WithSlots:
    __slots__ = ["x"]
    def __init__(self): self.x = 1

print(sys.getsizeof(WithDict()))   # ~48 bytes + dict overhead
print(sys.getsizeof(WithSlots()))  # ~32 bytes (smaller)
```

---
## 13. Composition vs Inheritance
In Python object-oriented programming, **inheritance** models an **"is-a" relationship** where a child class derives directly from a parent class. **Composition** models a **"has-a" relationship** where a class builds complex behavior by referencing instances of other independent classes as attributes.

| Feature               | Inheritance                                     | Composition                                     |
| --------------------- | ----------------------------------------------- | ----------------------------------------------- |
| **Relationship Type** | **"Is-a"** (e.g., A `Car` _is a_ `Vehicle`)     | **"Has-a"** (e.g., A `Car` _has an_ `Engine`)   |
| **Coupling**          | **Tight**; modifications to parent cascade down | **Loose**; component classes remain independent |
| **Flexibility**       | **Static**; defined at design time              | **Dynamic**; swap behaviors freely at runtime   |
| **Code Reuse**        | Absorbs full implementation directly            | Delegates specific tasks via attributes         |

### The Classic Debate
```
Inheritance: "IS A" relationship
  A Dog IS A Animal
  An Admin IS A User

Composition: "HAS A" relationship
  A Car HAS A Engine
  A User HAS A Address

Rule of thumb: Prefer composition over inheritance
  Inheritance creates TIGHT coupling
  Composition is more flexible
```

```python
# ─────────────────────────────────────────
# INHERITANCE approach (can get messy)
# ─────────────────────────────────────────
class Animal:
    def breathe(self): return "breathing"

class FlyingAnimal(Animal):
    def fly(self): return "flying"

class SwimmingAnimal(Animal):
    def swim(self): return "swimming"

# What about a duck that BOTH flies AND swims?
# Multiple inheritance gets complicated fast...

# ─────────────────────────────────────────
# COMPOSITION approach (more flexible)
# ─────────────────────────────────────────

class Engine:
    def __init__(self, horsepower: int, fuel_type: str):
        self.horsepower = horsepower
        self.fuel_type = fuel_type
        self.is_running = False

    def start(self) -> str:
        self.is_running = True
        return f"{self.horsepower}hp {self.fuel_type} engine started"

    def stop(self) -> str:
        self.is_running = False
        return "Engine stopped"

class GPS:
    def __init__(self):
        self.current_location = (0.0, 0.0)

    def get_location(self) -> tuple:
        return self.current_location

    def navigate_to(self, destination: str) -> str:
        return f"Navigating to {destination}"

class MusicSystem:
    def __init__(self):
        self.volume = 50
        self.current_song = None

    def play(self, song: str) -> str:
        self.current_song = song
        return f"Playing: {song}"

class Car:
    """Car HAS AN engine, HAS A GPS, HAS A music system."""

    def __init__(self, make: str, model: str):
        self.make = make
        self.model = model

        # COMPOSITION — car contains these objects
        self.engine = Engine(300, "gasoline")
        self.gps = GPS()
        self.music = MusicSystem()

    def start(self) -> str:
        return self.engine.start()

    def navigate(self, destination: str) -> str:
        return self.gps.navigate_to(destination)

    def play_music(self, song: str) -> str:
        return self.music.play(song)

class ElectricCar:
    """Electric car — different engine, same GPS and music."""

    def __init__(self, make: str, model: str):
        self.make = make
        self.model = model

        # Different engine — easy swap!
        self.engine = Engine(400, "electric")
        self.gps = GPS()
        self.music = MusicSystem()

    def start(self) -> str:
        return self.engine.start()

car = Car("Toyota", "Camry")
print(car.start())                  # 300hp gasoline engine started
print(car.navigate("NYC"))         # Navigating to NYC
print(car.play_music("Blinding Lights"))  # Playing: Blinding Lights

tesla = ElectricCar("Tesla", "Model S")
print(tesla.start())               # 400hp electric engine started

# POWER: You can swap components without rewriting the class
# Want a car with a different engine? Just pass a different Engine object.
# This is the foundation of Dependency Injection.
```

### Real Backend - Composition in Services
```python
# Real backend example — composition in services

class EmailService:
    def send(self, to: str, subject: str, body: str) -> bool:
        print(f"Email to {to}: {subject}")
        return True

class SMSService:
    def send(self, phone: str, message: str) -> bool:
        print(f"SMS to {phone}: {message}")
        return True

class UserService:
    """UserService HAS AN email service and SMS service."""

    def __init__(self, email_service: EmailService, sms_service: SMSService):
        self.email = email_service     # composed
        self.sms = sms_service         # composed

    def register(self, name: str, email: str, phone: str) -> dict:
        user = {"name": name, "email": email, "phone": phone}

        # Use composed services
        self.email.send(email, "Welcome!", f"Welcome {name}!")
        self.sms.send(phone, f"Welcome {name}!")

        return user

service = UserService(EmailService(), SMSService())
service.register("Alice", "alice@test.com", "+1-555-0123")
# Email to alice@test.com: Welcome!
# SMS to +1-555-0123: Welcome Alice!
```

---
## Visual Summary - OOP Pillars
```
┌──────────────────────────────────────────────────────────────┐
│                        OOP PILLARS                           │
│                                                              │
│  ENCAPSULATION               INHERITANCE                     │
│  ─────────────               ───────────                     │
│  Bundle data +               Child class gets                │
│  behavior together           parent's methods                │
│  Control access              and attributes                  │
│  with public/                super() calls parent            │
│  protected/private           __init__                        │
│                                                              │
│  POLYMORPHISM                ABSTRACTION                     │
│  ─────────────               ───────────                     │
│  Same method                 Define WHAT                     │
│  different behavior          not HOW                         │
│  per class                   ABC + @abstractmethod           │
│  Duck typing                 Forces subclasses               │
│                              to implement methods            │
│                                                              │
│  CLASS ANATOMY:                                              │
│  class MyClass(Parent):                                      │
│      class_var = "shared"    ← class variable                │
│                                                              │
│      def __init__(self, x):  ← initializer                   │
│          self.x = x          ← instance variable             │
│                                                              │
│      def instance_method(self):  ← has self                  │
│          pass                                                │
│                                                              │
│      @classmethod                                            │
│      def class_method(cls):     ← has cls                    │
│          pass                                                │
│                                                              │
│      @staticmethod                                           │
│      def static_method():       ← no self/cls                │
│          pass                                                │
│                                                              │
│      @property                                               │
│      def computed(self):        ← getter                     │
│          return self._value                                  │
└──────────────────────────────────────────────────────────────┘
```

---
## Knowledge Check

1. What is the difference between a class and an object?
2. What does `super().__init__()` do?
3. What is the difference between instance, class, and static methods?
4. What does `@property` do? When would you use it?
5. What is encapsulation? What do single and double underscores mean?
6. What is the difference between inheritance and composition?
7. What is polymorphism? Give a real backend example.
8. What is an abstract class? Can you instantiate it?
9. What does `__repr__` do vs `__str__`?
10. What is the MRO and why does it matter?
11. What is a dataclass and when would you use one?
12. What is the mutable default in dataclasses and how do you fix it?

---
## Practice Exercises
Exercise 1:
```
# Build a Shape hierarchy using ABC
# - BaseShape: abstract methods → area(), perimeter()
# - Concrete classes: Circle, Rectangle, Triangle
# - Each implements area() and perimeter()
# - Add a method print_info() to BaseShape
#   that prints shape type, area, perimeter
```

Exercise 2:
```
# Build a simple Order management system using composition
# Classes needed:
#   - Customer (name, email)
#   - Product (name, price, stock)
#   - Order (customer, list of products)
# Order should have:
#   - add_product(product, quantity)
#   - total_price() → sum of all products
#   - can_fulfill() → True if all products have enough stock
#   - process() → deduct from stock, return receipt dict
```

Exercise 3:
```
# Create a dataclass for a BlogPost
# Fields: id, title, content, author, tags (list), published, created_at
# Add a method: word_count() → number of words in content
# Add a property: preview → first 100 chars of content + "..."
# Add a class method: from_dict(data: dict) -> BlogPost
```

---
## Phase 1.3 Complete!
**You now know:**
```
✅ Classes and objects
✅ __init__, self, instance variables
✅ Instance, class, and static methods
✅ Encapsulation (public, protected, private)
✅ Inheritance (single, multiple, MRO)
✅ super() and method overriding
✅ Polymorphism and duck typing
✅ Abstraction with ABC
✅ Magic/dunder methods
✅ Property decorators
✅ Dataclasses
✅ __slots__
✅ Composition vs Inheritance
```

---