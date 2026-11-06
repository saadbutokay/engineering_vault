**C++ is a high-performance, general-purpose programming language** created by Danish computer scientist Bjarne Stroustrup in 1985. Developed at Bell Labs as an extension of the traditional C language (originally called "C with Classes"), it was designed to add object-oriented programming (OOP) functionality while keeping the efficiency and low-level resource control of C.

**Target version:** C++20 as the required baseline, C++23 features used where compilers support them, and C++26 covered as an overview only.

---
## Phase 0: Environment, Tooling, and Foundations

- **0.1 How C++ programs are built and run:** Compiled vs interpreted languages, the four build stages (preprocess, compile, assemble, link), object files, translation units, static vs dynamic linking, executables and libraries per platform, what the abstract machine and the as-if rule mean, and a terminal refresher for PowerShell, zsh, and bash (navigation, paths, standard streams, exit codes).
- **0.2 Installing a toolchain:** Windows (Visual Studio Build Tools with MSVC, Developer PowerShell, MSYS2 with MinGW-w64, `clang-cl`, WSL2), macOS (Xcode Command Line Tools with Apple Clang, Homebrew LLVM and GCC for newer versions), Linux (`build-essential`, `dnf` groups, `clang`), installing CMake and Ninja, `g++ --version` and `cl`, `PATH`, multiple compilers side by side, why the compiler choice matters on each platform.
- **0.3 Compiling and running code:** `g++`, `clang++`, and `cl`, selecting the standard with `-std=c++20`, `-o`, `-c`, `-E`, and `-S`, warning flags (`-Wall`, `-Wextra`, `-Wpedantic`, `-Werror`), optimization levels `-O0` to `-O3`, `-g` for debug symbols, compiling and linking several files by hand, Compiler Explorer, command-line arguments, exit codes.
- **0.4 Code editor setup:** Visual Studio Code with `clangd` (and why it is preferred over the default IntelliSense engine), CLion and Visual Studio as alternatives, `compile_commands.json`, format on save with `clang-format`, debugging with `gdb`, `lldb`, and the MSVC debugger, running and debugging from the editor.
- **0.5 Build systems and CMake:** Why `make` and hand-written commands do not scale, `CMakeLists.txt`, `project`, `add_executable`, `add_library`, `target_link_libraries`, `target_include_directories`, `target_compile_features`, out-of-source builds, the configure, build, and test steps, build types (`Debug`, `Release`, `RelWithDebInfo`), generators, Ninja, `CMakePresets.json`, `CTest` basics, CMake 3.28 or newer.
- **0.6 Dependencies and package managers:** Header-only vs compiled libraries, system packages and `pkg-config`, `find_package`, `FetchContent`, vcpkg in manifest mode, Conan overview, pinning versions and baselines, vendoring, ABI compatibility between your code and prebuilt libraries, supply-chain hygiene.
- **0.7 Project layout and configuration:** `include/`, `src/`, `tests/`, `apps/`, and `cmake/` folders, public vs private headers, build directories kept out of source control, `.gitignore`, `.env` files and why secrets never go into code, compile definitions, `configure_file`.
- **0.8 Code style:** `clang-format` and `.clang-format` files, naming conventions, `#pragma once` vs include guards, include ordering, comments, Doxygen-style documentation comments, an overview of the C++ Core Guidelines and the Google C++ Style Guide and why teams adopt one.
- **0.9 Git essentials:** `git init`, `status`, `add`, `commit`, `log`, `diff`, branches, merging, remotes, pushing to GitHub, what to commit (`CMakeLists.txt`, `vcpkg.json`, lock files) and what to ignore (build directories, binaries, object files).
- **0.10 Quality tools:** Strict compiler warnings, `clang-tidy` and `.clang-tidy`, `cppcheck`, AddressSanitizer and UndefinedBehaviorSanitizer enabled from CMake, `pre-commit` hooks, running all of these in a minimal CI pipeline on all three operating systems.

---

## Phase 1: Core Fundamentals

- **1.1 Program structure and execution model:** `int main`, `#include`, namespaces and `std::`, statements and expressions, comments, declarations vs definitions, how `main` starts and ends, the One Definition Rule introduced, and the three kinds of non-portable behavior: implementation-defined, unspecified, and undefined.
- **1.2 Variables, constants, and initialization:** Declarations, the many forms of initialization and why brace initialization is preferred, narrowing conversions, uninitialized variables and why reading them is undefined, `auto`, `const`, `constexpr`, `constinit`, scope and lifetime, `using` aliases, `decltype`.
- **1.3 Fundamental types and operators:** Integer types and `<cstdint>` fixed-width types, `signed` vs `unsigned` pitfalls, integer promotion, signed overflow as undefined behavior, `char`, `bool`, `float` and `double` and floating-point pitfalls, `size_t`, implicit conversions, `static_cast`, arithmetic, comparison, logical, and bitwise operators, `<cmath>`, `<limits>`, `<numeric>`, `<bit>`, safe integer comparison with `std::cmp_less`, the three-way operator `<=>`, `std::byte`.
- **1.4 Strings:** `std::string`, `std::string_view` and its lifetime hazards, C-style strings and `char` arrays (so students can recognize them), common methods, literal types, raw string literals, small string optimization, UTF-8 and why `std::string` is a byte container, `std::format`, `std::to_string`, `std::from_chars` and `std::to_chars`.
- **1.5 Input and output:** `<iostream>`, `std::cin`, `std::cout`, `std::cerr`, `std::getline`, stream failure states and recovery, `'\n'` vs `std::endl`, formatting manipulators, `std::format` and `std::print` (C++23), `std::ios::sync_with_stdio`, parsing user input safely, `<fstream>` previewed.
- **1.6 Control flow:** `if` and `switch` with init statements, `else if` chains, `switch` and `[[fallthrough]]`, `for`, range-based `for` with init, `while`, `do-while`, `break` and `continue`, the conditional operator, `goto` and why to avoid it, short-circuit evaluation, `if constexpr` previewed.
- **1.7 Reading errors and debugging basics:** Reading compiler errors from the first line, the difference between warnings and errors, linker errors (`undefined reference`, `unresolved external symbol`, multiple definition), runtime failures (segmentation fault, abort, failed assertion), a first look at a sanitizer report, `assert`, `gdb` and `lldb` basics, a systematic way to debug a failing program.

---

## Phase 2: Standard Containers and Value Types

- **2.1 Arrays and `std::array`:** Built-in arrays and array-to-pointer decay, `std::array`, `at()` vs `operator[]`, bounds and undefined behavior, multi-dimensional arrays, `std::span` as a non-owning view, why plain arrays are avoided in new code.
- **2.2 `std::vector`:** Size vs capacity, `push_back` vs `emplace_back`, `reserve` and `shrink_to_fit`, insertion and erasure, `std::erase` and `std::erase_if`, contiguous storage, iterator and reference invalidation rules, `std::vector<bool>` as a trap, vectors of vectors, performance characteristics.
- **2.3 Associative and hash containers:** `std::map`, `std::set`, `std::multimap`, `std::unordered_map`, `std::unordered_set`, `insert`, `emplace`, `try_emplace`, `insert_or_assign`, `operator[]` inserting by surprise, custom keys and hash functions, heterogeneous lookup, node handles, `std::flat_map` and `std::flat_set` (C++23, where supported), choosing between ordered and hashed.
- **2.4 Other containers and adapters:** `std::deque`, `std::list`, `std::forward_list`, `std::stack`, `std::queue`, `std::priority_queue`, `std::bitset`, `std::span` and `std::mdspan` (C++23), why `std::vector` is the default, a decision guide for choosing a container.
- **2.5 Pairs, tuples, and structured bindings:** `std::pair`, `std::tuple`, `std::tie`, structured bindings with arrays, structs, and maps, returning multiple values, when a named struct is better than a tuple.
- **2.6 Structs and aggregates:** `struct` declaration, aggregate initialization, designated initializers (C++20), defaulted comparison, memory layout, `sizeof`, `alignof`, padding, nested structs, bit-fields, a first look at unions, plain data types as the base of good design.
- **2.7 Complexity and algorithm basics:** Big-O notation, complexity guarantees of each container operation, `std::sort`, `std::find`, `std::lower_bound`, `std::accumulate`, why `std::list` is rarely faster than `std::vector`, cache locality introduced.

---

## Phase 3: Functions

- **3.1 Function basics:** Declarations, definitions, and prototypes, return types and trailing return types, `void`, `auto` return, default arguments, overloading and overload resolution, `inline`, `[[nodiscard]]`, `[[maybe_unused]]`, `noexcept` introduced, `constexpr` functions introduced.
- **3.2 Parameters and argument passing:** Pass by value, by lvalue reference, by `const` reference, and by pointer, the standard guidance for choosing between them, output parameters vs returning values, returning structs, `std::string_view` and `std::span` parameters, `std::initializer_list`, C-style variadic functions (recognize and avoid). References are introduced here at user level, and their lifetime rules come in Module 4.2.
- **3.3 Scope and storage duration:** Block, function, namespace, and class scope, name hiding, automatic, static, thread, and dynamic storage durations, `static` local variables, `thread_local`, global state and why to avoid it, the static initialization order problem.
- **3.4 Function pointers and callable objects:** Function pointers and their syntax, function objects (functors), `std::function`, `std::move_only_function` (C++23), member function pointers, `std::invoke`, `std::bind_front`, the cost of type erasure.
- **3.5 Lambdas and closures:** Lambda syntax, capture by value and by reference, init-captures, capturing `this` and `*this`, generic lambdas, templated lambdas (C++20), `mutable`, `constexpr` lambdas, immediately invoked lambdas for initialization, closure types, dangling captures, passing lambdas to algorithms.
- **3.6 Functional patterns:** Higher-order functions, composition, partial application, memoization, strategy as a callable, callback-style APIs, pure functions and testability, `std::invoke_result`.
- **3.7 Recursion:** Base case and recursive case, stack overflow, why tail-call optimization is not guaranteed, memoization, `constexpr` recursion, converting recursion to iteration with an explicit stack.

---

## Phase 4: Pointers, Memory, and Ownership

- **4.1 Pointers:** Address-of and dereference, `nullptr`, pointer arithmetic and arrays, `const` pointers and pointers to `const`, pointers to pointers, `void*`, function pointers revisited, dangling pointers, why raw pointers should be non-owning in modern code.
- **4.2 References and object lifetime:** Lvalue references and `const` references, binding to temporaries and lifetime extension, dangling references, returning references safely, `std::reference_wrapper`, the lifetime of temporaries in a full expression, iterator and view invalidation.
- **4.3 The memory model:** Stack, heap, static storage, thread storage, `new` and `delete`, `new[]` and `delete[]`, `malloc` and `free` and why not to mix them, memory leaks, double free, use after free, buffer overruns, alignment, placement `new` overview.
- **4.4 RAII and ownership:** The RAII principle, owning vs non-owning, scope-bound resources, writing a small RAII class, the Rule of Zero, why destructors run during stack unwinding, ownership as a design question answered in types.
- **4.5 Smart pointers:** `std::unique_ptr`, `std::shared_ptr`, `std::weak_ptr`, `make_unique` and `make_shared`, custom deleters, transferring ownership, reference-count costs and thread-safety, ownership cycles, `enable_shared_from_this`, which smart pointer to use in a function signature, non-owning observers.
- **4.6 Move semantics:** Value categories (lvalue, prvalue, xvalue), rvalue references, `std::move` as a cast, moved-from state, guaranteed copy elision and return value optimization, moving into containers, move-only types such as `std::unique_ptr`, why `return std::move(x)` is usually wrong.
- **4.7 Undefined behavior and memory safety:** A catalogue of common undefined behavior (out-of-bounds access, null dereference, signed overflow, use after free, data races, strict aliasing violations, uninitialized reads), why the optimizer makes it dangerous, sanitizers in practice, hardened standard library modes, bounds-checked access policies, a safe-by-default coding checklist.

---

## Phase 5: Headers, Modules, and the Build Model

- **5.1 Headers and source files:** Declaration vs definition, what belongs in a header and what belongs in a source file, include guards and `#pragma once`, forward declarations, include-what-you-use, the One Definition Rule in detail, `inline` functions and variables, header-only libraries and their trade-offs.
- **5.2 Namespaces and linkage:** Namespace declaration and nesting, `using` declarations and why `using namespace` is banned in headers, anonymous namespaces, `static` at namespace scope, internal vs external linkage, `extern`, inline namespaces, argument-dependent lookup.
- **5.3 The preprocessor:** `#include` mechanics, `#define` and macro dangers, conditional compilation, `__has_include`, predefined macros, feature-test macros, platform and compiler detection, replacing macros with `constexpr`, `inline`, and templates, the cases where macros are still justified.
- **5.4 Compilation and linking mechanics:** Translation units and object files, static and dynamic libraries (`.a` and `.lib`, `.so`, `.dylib`, and `.dll`), symbol visibility and export macros, name mangling, `extern "C"`, link order, linker errors in depth, runtime library search paths, compile-time costs, precompiled headers, unity builds.
- **5.5 C++20 modules:** `export module`, `import`, module interface and implementation units, partitions, header units, module support in CMake 3.28 or newer with Ninja, current tool support and when modules are appropriate for production, migration strategy.
- **5.6 Library and API boundaries:** Public API vs implementation, the PIMPL idiom, API vs ABI stability, versioning with inline namespaces, exposing a C interface, designing headers that are cheap to include, `install` and export rules in CMake.

---

## Phase 6: Errors, Files, and Resources

- **6.1 Exceptions:** `throw`, `try`, `catch`, catching by `const` reference, `std::exception` and the standard hierarchy, stack unwinding, `noexcept`, `std::terminate`, `std::exception_ptr` and nested exceptions, exception safety guarantees (basic, strong, no-throw), the cost model, when not to use exceptions.
- **6.2 Optional, variant, and expected:** `std::optional` for absence, `std::variant` and `std::visit`, the overloaded-lambdas idiom, `std::any` and why it is rarely right, `std::expected` (C++23) for recoverable errors, monadic operations (`and_then`, `transform`, `or_else`), representing closed sets of alternatives.
- **6.3 Error-handling strategy:** Exceptions vs error codes vs `std::expected`, `std::error_code` and error categories, `assert` and `static_assert`, preconditions and contracts overview, `std::source_location`, `std::stacktrace` (C++23), designing application error types, handling errors once at the boundary, errors across a C API boundary.
- **6.4 Exception safety and resource wrappers:** Writing RAII wrappers for file handles, sockets, locks, and C library resources, scope guards, destructors that must not throw, `std::lock_guard` as a model, `std::unique_ptr` with custom deleters for C handles, cleanup order.
- **6.5 File handling:** `std::ifstream`, `std::ofstream`, open modes, text vs binary, reading line by line, reading a whole file, checking stream state, buffering, processing large files in chunks, binary data and endianness, memory-mapped files overview.
- **6.6 `std::filesystem` and the environment:** `std::filesystem::path`, directory iteration, recursive traversal, creating, copying, and removing files and directories, temporary files, permissions, `std::getenv`, error-code overloads vs throwing overloads, cross-platform path and UTF-8 issues on Windows.
- **6.7 Data formats:** JSON with `nlohmann/json` (and `simdjson` for speed), CSV parsing by hand, TOML with `toml++`, YAML with `yaml-cpp`, Protocol Buffers overview, binary serialization, parsing with `std::from_chars`, validating untrusted input, never deserializing untrusted data into raw memory.
- **6.8 Logging:** Why the standard library has no logger, `spdlog` (levels, sinks, formatting, asynchronous logging), structured logging, correlation identifiers, never logging secrets, avoiding log injection, why `std::cout` is not production logging.

---

## Phase 7: Classes and Object-Oriented Design

- **7.1 Classes and objects:** `class` vs `struct`, data members, member functions, access specifiers, `this`, `const` member functions, `static` members, in-class initializers, splitting a class between header and source.
- **7.2 Constructors and destructors:** Default, parameterized, and delegating constructors, member initializer lists and initialization order, `explicit`, converting constructors, destructors and object lifetime, `= default` and `= delete`.
- **7.3 Special member functions:** Copy constructor and copy assignment, move constructor and move assignment, when the compiler generates each, the Rule of Zero, Three, and Five, shallow vs deep copy, copy-and-swap, `noexcept` moves and why `std::vector` cares, writing a resource-owning class correctly.
- **7.4 Operator overloading and customization points:** Arithmetic and comparison operators, defaulted `operator<=>` and `operator==`, stream insertion, `operator[]`, `operator()` and functors, conversion operators, `swap`, `std::hash` specializations, `std::formatter` specializations, overloading guidelines.
- **7.5 Inheritance and polymorphism:** Public inheritance, `virtual`, `override`, and `final`, pure virtual functions, virtual destructors, the vtable at a conceptual level, object slicing, `dynamic_cast` and RTTI, protected members, multiple inheritance and the diamond problem, the non-virtual interface idiom.
- **7.6 Composition vs inheritance:** "Has-a" vs "is-a", the Liskov substitution principle, delegation, mixins, preferring composition, when inheritance is correct, a preview of CRTP and type erasure.
- **7.7 Abstract classes and interfaces:** Interface classes with pure virtual functions, dependency inversion, writing test fakes against an interface, runtime polymorphism vs compile-time polymorphism with templates.
- **7.8 Enums and unions:** Unscoped `enum` vs `enum class`, underlying types, `std::to_underlying` (C++23), `using enum` (C++20), bit flags with operator overloads, converting enums to and from strings, `union` and why `std::variant` replaces it.
- **7.9 Encapsulation and invariants:** Access control, `friend`, nested classes, const-correctness, `mutable`, class invariants and establishing them in constructors, value types vs entity types, `[[nodiscard]]` on types, designing types that cannot be put in an invalid state.

---

## Phase 8: Templates and Generic Programming

- **8.1 Function templates:** Template parameters, argument deduction rules, explicit template arguments, overloading with templates, specialization and why overloading is preferred, abbreviated function templates with `auto`, trailing return types and `decltype`.
- **8.2 Class templates:** Defining members, class template argument deduction (CTAD), full and partial specialization, default template arguments, non-type template parameters, alias templates, variable templates, dependent names and the `typename` and `template` keywords, why template definitions live in headers, explicit instantiation.
- **8.3 Variadic templates and perfect forwarding:** Parameter packs, `sizeof...`, fold expressions, forwarding references, `std::forward`, reference collapsing, implementing `emplace`-style functions, `std::apply`, `std::index_sequence`, recursion over packs.
- **8.4 Concepts and constraints (C++20):** `requires` clauses and `requires` expressions, the standard concepts in `<concepts>`, constrained `auto`, writing your own concepts, subsumption and overload selection, replacing SFINAE, reading concept-related error messages.
- **8.5 Compile-time programming:** `constexpr` variables and functions, `consteval`, `constinit`, `if constexpr`, `static_assert`, `<type_traits>`, legacy SFINAE and `enable_if` (recognize only), `constexpr` containers and algorithms, compile-time lookup tables, user-defined literals.
- **8.6 Generic design idioms:** CRTP, type erasure, policy-based design, tag dispatch, traits classes, deducing `this` (C++23), expression templates overview, building a small type-erased wrapper.
- **8.7 Template pitfalls and judgment:** Code bloat, compile-time cost, unreadable error messages, ODR issues with templates, `extern template`, when plain functions or inheritance are better, readability rules for template code.

---

## Phase 9: Iterators, Algorithms, Ranges, and Generators

- **9.1 Iterators:** Iterator categories from input to contiguous, `begin` and `end`, reverse iterators, iterator invalidation across containers, `std::iterator_traits`, sentinels, writing a custom iterator for a custom container.
- **9.2 Algorithms:** `<algorithm>` and `<numeric>`, searching, sorting, partitioning, transforming, accumulating, `std::remove_if` and the erase-remove idiom, `std::unique`, `std::copy_if`, `std::min` and `std::max`, projections in `std::ranges` algorithms, execution policies introduced, algorithm complexity guarantees.
- **9.3 Ranges and views (C++20 and C++23):** The range concept, views (`filter`, `transform`, `take`, `drop`, `iota`, `join`, `split`, and in C++23 `zip`, `enumerate`, `chunk`, `slide`), the pipe operator, lazy evaluation, `std::ranges::to` (C++23), borrowed ranges, dangling iterators, view lifetime pitfalls.
- **9.4 Coroutines and generators:** `co_yield`, `co_await`, and `co_return`, the promise type and coroutine frame at a conceptual level, `std::generator` (C++23), lazy sequences, lifetime hazards of references held across suspension points.
- **9.5 Pipeline readability:** When to use a loop, an algorithm, or a range pipeline, naming intermediate steps, nested pipelines and their limits, debugging a pipeline, performance trade-offs.

---

## Phase 10: Concurrency and Parallelism

- **10.1 Concepts:** Concurrency vs parallelism, CPU-bound vs I/O-bound work, processes vs threads, `hardware_concurrency`, shared memory and visibility, cache lines and false sharing, choosing a model.
- **10.2 Threads:** `std::thread` and `std::jthread` (C++20), `join` and `detach`, passing arguments, `std::stop_token` and cooperative cancellation, exceptions in threads, `thread_local`, thread-safe output with `std::osyncstream`, lifetime bugs with captured references.
- **10.3 Mutexes and synchronization:** `std::mutex`, `std::lock_guard`, `std::unique_lock`, `std::scoped_lock`, `std::shared_mutex`, `std::call_once`, `std::condition_variable` and spurious wakeups, `std::latch`, `std::barrier`, `std::counting_semaphore`, data races defined, deadlock and lock ordering.
- **10.4 Atomics and the memory model:** `std::atomic`, `compare_exchange`, memory orders (`relaxed`, `acquire`, `release`, `seq_cst`), happens-before, atomic `wait` and `notify`, lock-free overview, the ABA problem, why `seq_cst` is the right default.
- **10.5 Futures and asynchronous tasks:** `std::async` and launch policies, `std::promise`, `std::future`, `std::shared_future`, `std::packaged_task`, limitations of `std::future`, building a thread pool by hand.
- **10.6 Parallel algorithms:** Execution policies (`std::execution::par` and `par_unseq`), requirements for element functions, partitioning work, reductions, false sharing in practice, OpenMP and Intel TBB overview, the `std::execution` senders and receivers model adopted for C++26 (overview).
- **10.7 Coroutines for asynchronous code:** Awaitable types, writing a minimal `task` type, schedulers and executors, `co_await` with I/O using standalone Asio, cancellation, lifetime of coroutine frames and captured state, structured concurrency overview.
- **10.8 Concurrency patterns:** Thread pools, thread-safe queues, producer and consumer, pipelines, reader and writer locks, bounded concurrency, the active object pattern, graceful shutdown with `std::stop_token` and signals.
- **10.9 Race conditions, deadlocks, and debugging:** Data races vs race conditions, ThreadSanitizer, Helgrind, reproducing and diagnosing deadlocks, lock-free pitfalls, testing concurrent code, a checklist of common bugs.

---

## Phase 11: Advanced Language Features

- **11.1 The object model and layout:** Object lifetime rules, `sizeof` and `alignof` in depth, standard-layout and trivially copyable types, vtable layout, empty base optimization, `[[no_unique_address]]`, strict aliasing, `std::bit_cast`, `std::launder`, `std::start_lifetime_as` (C++23).
- **11.2 Custom memory management:** Overloading `operator new` and `operator delete`, allocators and allocator-aware containers, `std::pmr` and polymorphic memory resources, arena and pool allocators, small buffer optimization, cache-friendly and data-oriented design overview.
- **11.3 Deduction and forwarding in depth:** `auto`, `decltype`, `decltype(auto)`, forwarding references and reference collapsing, perfect forwarding pitfalls, value categories revisited, copy elision rules, return value optimization limits.
- **11.4 Introspection and reflection:** `typeid`, `std::type_info` and `std::type_index`, RTTI costs and flags, feature-test macros, type traits as compile-time introspection, why C++ has no run-time reflection, static reflection in C++26 (overview, with compiler status).
- **11.5 C++26 overview:** Static reflection, contracts, `std::execution`, `std::inplace_vector`, pack indexing, `#embed`, `std::simd`, hardened standard library, what is safe to use in production today and what is not.
- **11.6 Interoperability:** The C ABI and `extern "C"`, opaque handles, calling C libraries safely, exposing C++ to Python with `pybind11` or `nanobind`, loading libraries at run time with `dlopen` and `LoadLibrary`, platform APIs (POSIX and Win32) overview.
- **11.7 Compiler and optimizer behavior:** Inlining, loop unrolling, vectorization, devirtualization, link-time optimization, profile-guided optimization, reading assembly in Compiler Explorer, how undefined behavior enables optimization, intrinsics and SIMD overview.

---

## Phase 12: Testing, Debugging, and Performance

- **12.1 Unit testing:** GoogleTest and Catch2 compared, test structure, assertions, fixtures, parameterized and typed tests, death tests, `CTest` and test discovery, test naming and organization.
- **12.2 Test doubles and testable design:** Interfaces as seams, hand-written fakes, GoogleMock, constructor injection and template injection, controlling time with an injectable clock, hermetic tests, testing code that uses the filesystem and network.
- **12.3 Coverage and fuzzing:** `gcov`, `llvm-cov`, `gcovr`, line vs branch coverage, meaningful coverage vs vanity metrics, libFuzzer and AFL++, fuzzing with sanitizers, property-based testing overview.
- **12.4 Debugging:** `gdb`, `lldb`, and the Visual Studio debugger, breakpoints, watchpoints, backtraces, core dumps, debug symbols and release builds, why bugs differ between debug and release, reverse debugging with `rr` overview.
- **12.5 Sanitizers and runtime analysis:** AddressSanitizer, LeakSanitizer, UndefinedBehaviorSanitizer, ThreadSanitizer, MemorySanitizer, Valgrind, reading and acting on reports, sanitizer limitations per platform (including MSVC), running them in CI.
- **12.6 Benchmarking:** Google Benchmark, microbenchmark pitfalls (dead-code elimination, cache warm-up, noisy machines), `benchmark::DoNotOptimize`, comparing runs statistically, benchmarking in Release mode only.
- **12.7 Profiling:** `perf`, Instruments, the Visual Studio profiler, flame graphs, `callgrind`, heap profilers such as `heaptrack`, cache misses and branch mispredictions, finding hot spots and memory growth.
- **12.8 Optimization basics:** Measure first, choosing containers and layouts, `reserve`, avoiding copies with moves and views, small buffer optimization, cache locality, avoiding premature optimization, allocation reduction, link-time optimization in practice.

---

## Phase 13: Standard Library and Libraries for Production Work

- **13.1 Time:** `<chrono>` durations, time points, `system_clock` vs `steady_clock`, calendar and time zone support (C++20, with platform status), formatting and parsing with `std::format`, ISO 8601, UTC discipline, injectable clocks.
- **13.2 Text processing:** `<regex>` and why it is slow and risky for production, RE2, PCRE2, and CTRE as alternatives, `std::format` format specifications and custom formatters, `std::print`, `<charconv>`, locale pitfalls, Unicode and ICU overview, `std::text_encoding` (C++26) overview.
- **13.3 Command-line tools:** `argc` and `argv`, CLI11 and `cxxopts`, exit codes, returning from `main` vs `std::exit`, `std::quick_exit` and `std::atexit`, static destruction order, standard streams, detecting terminals, `std::signal`.
- **13.4 System interaction:** Launching processes (`std::system` risks, `posix_spawn`, `CreateProcess`, Boost.Process), signals and graceful shutdown, environment variables, platform detection, wrapping OS APIs behind a portable interface.
- **13.5 Security and cryptography:** OpenSSL, libsodium, and Botan, SHA-2 hashing, HMAC, `std::random_device` and `<random>` vs a cryptographically secure generator, base64, password hashing with Argon2, constant-time comparison, wiping secrets from memory, buffer overflow and integer overflow classes, compiler hardening flags (`-fstack-protector-strong`, `_FORTIFY_SOURCE`, PIE, control-flow integrity).
- **13.6 Databases:** SQLite C API wrapped in RAII, SQLiteCpp, prepared statements and parameter binding, transactions, `libpqxx` for PostgreSQL, connection pooling, SQL injection prevention, repository interfaces with fakes.
- **13.7 Networking and HTTP:** BSD sockets and Winsock, TCP basics, byte order, blocking vs non-blocking I/O, `epoll`, `kqueue`, and IOCP overview, standalone Asio, HTTP clients with `libcurl` or `cpr`, TLS overview, timeouts, why networking is not in the standard library, writing a small HTTP server on Asio.
- **13.8 Numeric and utility libraries:** `<random>` engines and distributions, `<numeric>`, `<numbers>`, `<complex>`, `<ratio>`, `<bit>`, `std::bitset`, money as integer minor units, Boost.Multiprecision overview, Eigen overview, `{fmt}`, Abseil, and Boost overview and when to depend on them.

---

## Phase 14: Professional Practices

- **14.1 Packaging and distribution:** CMake `install` and export targets, package configuration files for `find_package`, CPack, publishing to vcpkg and Conan, semantic versioning, shared vs static libraries, symbol versioning and `SOVERSION`, ABI stability promises.
- **14.2 Configuration:** Compile-time configuration with CMake options and definitions, run-time configuration from environment, files, and command line, typed configuration structs with validation, failing fast on missing settings, secrets handling, twelve-factor basics.
- **14.3 Design principles:** SOLID applied to C++, dependency injection through constructors and templates, the composition root, value semantics, the zero-overhead principle, avoiding global state and static initialization order problems, separation of concerns.
- **14.4 Design patterns:** Factory, Strategy (virtual functions, templates, and callables), Observer, Adapter, Decorator, Command, Visitor with `std::variant`, PIMPL, CRTP, Singleton (the Meyers singleton and its problems), Repository, type erasure, patterns that language features make unnecessary.
- **14.5 Clean code and the Core Guidelines:** Naming, function size, const-correctness, no owning raw pointers, no raw `new` and `delete`, avoiding C-style casts and macros, enforcing guidelines with `clang-tidy`, refactoring safely, code review habits.
- **14.6 Documentation and collaboration:** Doxygen, writing useful comments, README and CONTRIBUTING structure, pull requests, branching workflows, commit message conventions, changelogs, architecture decision records.
- **14.7 Building and shipping:** Release builds and optimization flags, link-time optimization, stripping symbols and keeping separate debug symbols, static linking, cross-compilation with CMake toolchain files, multi-stage Docker builds with minimal runtime images, reproducible builds, a CI matrix across Windows, macOS, and Linux and across GCC, Clang, and MSVC, `ccache`, sanitizer jobs, artifact signing, software bill of materials.

---

## Phase 15: Capstone Projects (Standard Library Plus a Fixed Set of Libraries, No Frameworks)

Allowed libraries: GoogleTest, Google Benchmark, CLI11, spdlog, nlohmann/json, SQLite, and standalone Asio. Application frameworks and HTTP server frameworks (Drogon, Crow, Oat++) are not allowed.

1. **Command-line task manager:** CLI11 subcommands, JSON or SQLite persistence, value types and classes, `std::expected` for errors, `spdlog`, GoogleTest suite, CMake presets, clean under AddressSanitizer and UndefinedBehaviorSanitizer.
2. **Concurrent URL health checker:** Asio with coroutines or a thread pool, timeouts and cancellation, retries with backoff, graceful shutdown on signals, structured logging, clean under ThreadSanitizer.
3. **Mini job queue library:** `std::jthread` and `std::stop_token`, templates and concepts for typed jobs, custom error types, interfaces for storage backends, Google Benchmark results, a full test suite, installable as a CMake package that other projects load with `find_package`.