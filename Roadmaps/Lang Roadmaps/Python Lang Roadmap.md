## Phase 0: Environment, Tooling, and Foundations

1. [[0.1 How Programs Run and the Terminal]] - What a program is, interpreter vs compiler, what the terminal and shell are, essential commands on PowerShell (Windows), zsh (macOS), and bash (Linux): print working directory, change directory, list files, make and remove folders, file paths (absolute vs relative).

2. [[0.2 Installing Python]] - Windows (python.org installer, `py` launcher, PATH), macOS (python.org installer, Homebrew), Linux (`apt`, `dnf`, `python3` command), verifying the install, installing `uv`, managing multiple Python versions.

3. [[0.3 Running Python Code]] - The REPL, running a `.py` file, `python -m`, command-line arguments, exiting, difference between `python` and `python3`.

4. [[0.4 Code Editor Setup]] - Visual Studio Code, the Python extension, selecting an interpreter, running and debugging from the editor, integrated terminal.
5. [[0.5 Virtual Environments and Packages]] - Why isolation matters, `venv`, activation on each OS, `pip install`, `pip freeze`, `requirements.txt`, `uv venv` and `uv pip`.

6. [[0.6 Project Layout and Configuration]] - `src/` layout, `tests/` folder, `pyproject.toml` basics, `.gitignore`, `.env` files and why secrets never go into code.

7. [[0.7 Code Style]] - PEP 8, PEP 20 (Zen of Python), naming conventions, line length, docstring styles (Google and NumPy).

8. [[0.8 Git Essentials]] - `git init`, `status`, `add`, `commit`, `log`, `diff`, branches, merging, remotes, pushing to GitHub, what to commit and what to ignore.

9. [[0.9 Quality Tools]] - `ruff` (lint and format), `mypy` basics, `pre-commit` hooks, configuring all three in `pyproject.toml`.

---
## Phase 1: Core Fundamentals

1. [[1.1 Syntax and Execution Model]] - Indentation, comments, statements vs expressions, line continuation, how Python runs code (source, bytecode, interpreter).

2. [[1.2 Variables and Objects]] - Names vs objects, assignment, dynamic typing, `id()`, `type()`, mutability vs immutability, `is` vs `==`, `None`, naming rules.

3. [[1.3 Numbers and Booleans]] - `int`, `float`, `bool`, `complex`, `Decimal`, `Fraction`, floating-point pitfalls, arithmetic operators, comparison, logical operators, bitwise operators, truthiness and falsiness, `math` module basics.

4. [[1.4 Strings]] - Indexing, slicing, immutability, common methods, f-strings, `format()`, escape sequences, raw strings, multiline strings, Unicode, `bytes` vs `str`, encoding and decoding.

5. [[1.5 Input and Output]] - `print()` parameters, `input()`, type conversion, formatted output, reading from standard input.

6. [[1.6 Control Flow]] - `if/elif/else`, conditional expressions, `match/case`, `while`, `for`, `range`, `break`, `continue`, loop `else`, walrus operator `:=`.

7. [[1.7 Reading Errors]] - Anatomy of a traceback, `SyntaxError`, `NameError`, `TypeError`, `ValueError`, `IndexError`, `KeyError`, a systematic way to debug a failing script.

---

## Phase 2: Built-in Data Structures

1. [[2.1 Lists]] Creation, indexing, slicing, methods, `sort` vs `sorted`, `key=`, list comprehensions, nested lists.

2. [[2.2 Tuples]] Immutability, packing and unpacking, star unpacking, tuples as records, when to prefer tuples over lists.

3. [[2.3 Dictionaries]] Creation, methods, iteration, `get`, `setdefault`, merging with `|`, dict comprehensions, insertion order, hashing basics.

4. [[2.4 Sets]] Set operations, `frozenset`, set comprehensions, membership-test performance.

5. [[2.5 Copying and Aliasing]] References, shallow copy, deep copy, the common bugs they cause.

6. [[2.6 The collections Module]] `defaultdict`, `Counter`, `deque`, `namedtuple`, `ChainMap`.

7. [[2.7 Algorithmic Helpers]] `heapq`, `bisect`, Big-O notation, time complexity of built-in operations.

---
## Phase 3: Functions

- **3.1 Function basics:** `def`, parameters, return values, returning multiple values, docstrings, `None` as implicit return.
- **3.2 Arguments:** Positional, keyword, defaults, the mutable default argument pitfall, `*args`, `**kwargs`, positional-only (`/`), keyword-only (`*`), argument unpacking at call sites.
- **3.3 Scope:** LEGB rule, `global`, `nonlocal`, why to avoid global state.
- **3.4 First-class functions:** Functions as objects, passing and returning functions, `lambda`, `map`, `filter`, `sorted(key=)`.
- **3.5 Closures:** Free variables, function factories, state capture, late-binding pitfall.
- **3.6 Decorators:** Basic decorators, `functools.wraps`, decorators with arguments, stacking decorators.
- **3.7 The `functools` module:** `partial`, `lru_cache`, `cache`, `reduce`, `singledispatch`, `cached_property`.
- **3.8 Recursion:** Base case, recursive case, recursion limit, memoization, when to use iteration instead.

---

## Phase 4: Modules, Packages, and the Import System

- **4.1 Modules:** `import`, `from ... import`, aliasing, `__name__ == "__main__"`, module docstrings.
- **4.2 Packages:** `__init__.py`, absolute vs relative imports, namespace packages, controlling public API with `__all__`.
- **4.3 Import mechanics:** `sys.path`, `sys.modules`, import caching, circular imports and how to fix them.
- **4.4 Third-party packages:** Finding packages on PyPI, installing, pinning versions, lock files, reading package documentation, evaluating package quality.

---

## Phase 5: Errors, Files, and Resources

- **5.1 Exceptions:** `try/except/else/finally`, built-in exception hierarchy, `raise`, `raise ... from`, exception chaining, EAFP vs LBYL.
- **5.2 Custom exceptions:** Designing exception hierarchies for applications, `ExceptionGroup`, `except*`, adding notes to exceptions.
- **5.3 Context managers:** `with` statement, `__enter__` and `__exit__`, `contextlib.contextmanager`, `ExitStack`, `suppress`.
- **5.4 File handling:** `open()`, modes, encodings, reading and writing text, binary files, buffering, iterating large files.
- **5.5 `pathlib`, `os`, and the filesystem:** `Path` operations, directory traversal, `os.environ`, `shutil`, `tempfile`.
- **5.6 Data formats:** `json`, `csv`, `tomllib`, `pickle` and its security risks.
- **5.7 Logging:** `logging` module, levels, loggers, handlers, formatters, structured logging, why `print` is not used in production.

---

## Phase 6: Object-Oriented Programming

- **6.1 Classes and objects:** `class`, `__init__`, instance vs class attributes, `self`, methods.
- **6.2 Method types:** Instance methods, `@classmethod`, `@staticmethod`, `@property`, setters and deleters.
- **6.3 Dunder methods:** `__repr__`, `__str__`, `__eq__`, `__hash__`, `__lt__`, `__len__`, `__getitem__`, `__iter__`, `__call__`, `__bool__`, operator overloading.
- **6.4 Inheritance:** Single and multiple inheritance, MRO, `super()`, mixins, method overriding.
- **6.5 Composition vs inheritance:** Design guidelines, "has-a" vs "is-a", when to use each.
- **6.6 Abstract classes:** `abc.ABC`, `@abstractmethod`, defining interfaces.
- **6.7 Dataclasses:** `@dataclass`, `field`, `frozen`, `slots`, `__post_init__`, `kw_only`, ordering.
- **6.8 Enums:** `Enum`, `IntEnum`, `StrEnum`, `auto`, `Flag`.
- **6.9 Encapsulation conventions:** `_protected`, `__private` and name mangling, `__slots__`.

---

## Phase 7: Iteration and Generators

- **7.1 Iteration protocol:** Iterables vs iterators, `__iter__`, `__next__`, `StopIteration`, how `for` works internally.
- **7.2 Generators:** `yield`, generator expressions, lazy evaluation, memory benefits, `yield from`, `send()`.
- **7.3 The `itertools` module:** `chain`, `islice`, `groupby`, `product`, `permutations`, `combinations`, `batched`, `accumulate`, `tee`.
- **7.4 Comprehension mastery:** Nested comprehensions, conditional logic in comprehensions, readability limits.

---

## Phase 8: Type Hints and Static Typing

- **8.1 Type hint basics:** Annotating variables, parameters, and returns, `list[int]`, `dict[str, int]`, `X | None`.
- **8.2 The `typing` essentials:** `Any`, `Union`, `Optional`, `Callable`, `Literal`, `Final`, `ClassVar`, `Annotated`.
- **8.3 Generics:** `TypeVar`, `Generic`, PEP 695 syntax (`def f[T](x: T)`), bounded and constrained types.
- **8.4 Structural typing:** `Protocol`, duck typing formalized, runtime-checkable protocols.
- **8.5 Structured types:** `TypedDict`, `NamedTuple`, `TypeAlias`, `NewType`.
- **8.6 Static checking:** Running `mypy`, strict mode, fixing common errors, runtime vs static typing.

---

## Phase 9: Advanced Language Features

- **9.1 The Python data model:** How `len()`, `for`, `with`, `+`, and attribute access map to dunder methods.
- **9.2 Attribute access:** `__getattr__`, `__getattribute__`, `__setattr__`, descriptors.
- **9.3 Advanced decorators:** Class-based decorators, decorating classes, typed decorators with `ParamSpec`.
- **9.4 Metaclasses:** `type`, `__new__` vs `__init__`, `__init_subclass__`, when metaclasses are justified.
- **9.5 Memory model:** Reference counting, garbage collector, `weakref`, `sys.getsizeof`, interning.
- **9.6 CPython internals (overview):** Bytecode with `dis`, the GIL, free-threading overview.
- **9.7 Introspection:** `inspect`, `dir`, `vars`, `getattr`, `setattr`, `hasattr`.

---

## Phase 10: Concurrency and Parallelism

- **10.1 Concepts:** CPU-bound vs I/O-bound, concurrency vs parallelism, the GIL, choosing a model.
- **10.2 Threading:** `threading.Thread`, locks, queues, race conditions, deadlocks.
- **10.3 Multiprocessing:** `Process`, `Pool`, shared state, inter-process communication.
- **10.4 `concurrent.futures`:** `ThreadPoolExecutor`, `ProcessPoolExecutor`, futures, error handling.
- **10.5 `asyncio`:** Event loop, `async`/`await`, tasks, `gather`, `TaskGroup`, timeouts, cancellation, semaphores, queues.
- **10.6 Async patterns:** Async context managers, async iterators, async generators, mixing sync and async code.

---

## Phase 11: Testing, Debugging, and Performance

- **11.1 `pytest`:** Test structure, assertions, fixtures, `parametrize`, markers, `conftest.py`.
- **11.2 Mocking:** `unittest.mock`, `patch`, `MagicMock`, dependency injection as an alternative.
- **11.3 Coverage:** `pytest-cov`, branch coverage, meaningful coverage vs vanity metrics.
- **11.4 Debugging:** `pdb`, `breakpoint()`, reading tracebacks, the editor debugger.
- **11.5 Profiling:** `timeit`, `cProfile`, `tracemalloc`, finding bottlenecks.
- **11.6 Optimization basics:** Choosing data structures, avoiding premature optimization, caching strategies.

---

## Phase 12: Standard Library for Production Work

- **12.1 Date and time:** `datetime`, `timezone`, `zoneinfo`, `timedelta`, UTC discipline, ISO 8601.
- **12.2 Text processing:** `re` (regular expressions), `string`, `textwrap`.
- **12.3 Command-line tools:** `argparse`, `sys.argv`, exit codes, standard streams.
- **12.4 System interaction:** `subprocess`, `signal`, `platform`, graceful shutdown.
- **12.5 Security and identity:** `hashlib`, `hmac`, `secrets`, `uuid`, `base64`.
- **12.6 Databases:** `sqlite3`, parameterized queries, transactions, SQL injection prevention.
- **12.7 Networking basics:** `urllib`, `http.client`, `socket` (overview).
- **12.8 Numeric and utility modules:** `decimal` for money, `random`, `statistics`, `operator`, `copy`, `pprint`.

---

## Phase 13: Professional Practices

- **13.1 Packaging:** `pyproject.toml` in depth, build backends, wheels, publishing basics, semantic versioning.
- **13.2 Configuration:** Environment-based config, settings objects, secrets handling.
- **13.3 Design principles:** SOLID in Python, DRY, KISS, separation of concerns, dependency injection.
- **13.4 Design patterns:** Factory, Strategy, Observer, Adapter, Singleton (and the module alternative), Repository.
- **13.5 Clean code:** Naming, function size, readable comprehensions, refactoring, code review habits.
- **13.6 Documentation and collaboration:** Docstrings, type hints as documentation, README structure, pull requests, branching workflows, commit message conventions.

---

## Phase 14: Capstone Projects (Pure Python, No Frameworks)

1. **Command-line task manager:** `argparse`, JSON or SQLite persistence, dataclasses, logging, tests.
2. **Async URL health checker:** `asyncio`, timeouts, retries with decorators, structured logging, fully typed.
3. **Mini job queue library:** Threads or processes, context managers, custom exceptions, full type hints, `pytest` suite, packaged with `pyproject.toml`.