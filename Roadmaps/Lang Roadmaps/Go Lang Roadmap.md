**Go (often called Golang) is an open-source, statically typed, and compiled programming language developed by Google.** Created in 2007 by computer science pioneers Robert Griesemer, Rob Pike, and Ken Thompson, Go was officially released to the public in 2009. It was specifically designed to tackle modern software engineering challenges, combining the blazing performance of lower-level languages like C++ with the clean readability and simplicity of languages like Python.

**Target version:** Go 1.23 or newer.

---
## Phase 0: Environment, Tooling, and Foundations

- **0.1 How Go programs are built and run:** Compiled vs interpreted languages, what a compiler and linker do, static binaries, cross-compilation overview, the Go toolchain, and a terminal refresher for PowerShell, zsh, and bash (navigation, paths, standard streams, exit codes).
- **0.2 Installing Go:** Windows (MSI installer, `winget`), macOS (pkg installer, Homebrew), Linux (official tarball, distribution packages and why they are often outdated), `PATH`, `GOROOT`, `GOPATH`, `GOBIN`, `go version`, `go env`, toolchain switching with `GOTOOLCHAIN`.
- **0.3 Running Go code:** `go run`, `go build`, `go install`, build output locations, the Go Playground, `go fmt`, `go vet`, `go doc`, command-line arguments, exit codes, `-o` and `-race` flags, the build cache.
- **0.4 Code editor setup:** Visual Studio Code with the Go extension, `gopls` (the language server), GoLand as an alternative, format on save, organizing imports, the Delve debugger, running and debugging from the editor.
- **0.5 Modules and dependencies:** What a module is, `go mod init`, `go.mod`, `go.sum`, `go get`, `go mod tidy`, `go list -m all`, semantic versioning, the module proxy and checksum database, vendoring, `go work` workspaces.
- **0.6 Project layout and configuration:** `cmd/`, `internal/`, and package folders, why there is no official layout and what the community converges on, `.gitignore`, `.env` files and why secrets never go into code, `.editorconfig`.
- **0.7 Code style and idioms:** `gofmt`, Effective Go, Go Proverbs, naming conventions (short names, `MixedCaps`, no underscores), exported vs unexported names, package naming, doc comment format.
- **0.8 Git essentials:** `git init`, `status`, `add`, `commit`, `log`, `diff`, branches, merging, remotes, pushing to GitHub, what to commit (`go.mod`, `go.sum`) and what to ignore (binaries).
- **0.9 Quality tools:** `go vet`, `staticcheck`, `golangci-lint`, `govulncheck`, `pre-commit` hooks, a `Makefile` or `Taskfile` for common commands, a minimal CI pipeline.

---

## Phase 1: Core Fundamentals

- **1.1 Program structure and execution model:** `package main`, `func main`, imports, the `init` function, how the compiler turns source into a binary, statements and automatic semicolon insertion, comments, the rule that unused variables and imports are compile errors.
- **1.2 Variables and constants:** `var`, short declaration `:=`, zero values, multiple assignment, shadowing, the blank identifier `_`, `const`, typed vs untyped constants, `iota`, block scope.
- **1.3 Basic types and operators:** `int` and sized integers, `uint`, `float64`, `complex128`, `bool`, `byte` and `rune` as aliases, integer overflow behavior, explicit type conversion, arithmetic, comparison, logical, and bitwise operators, floating-point pitfalls, `math`, `math/big`, built-in `min`, `max`, and `clear`.
- **1.4 Strings, bytes, and runes:** Strings as immutable byte sequences, UTF-8, `len` vs rune count, indexing, slicing, `for range` over a string, raw string literals, the `strings` package, `strings.Builder`, `strconv`, `unicode` and `unicode/utf8`, `[]byte` and `[]rune` conversions.
- **1.5 Input and output:** `fmt` print functions, formatting verbs (`%v`, `%+v`, `%T`, `%q`, `%x`, `%w`), `fmt.Sscanf` and `fmt.Scan`, reading lines with `bufio.Scanner` and `bufio.Reader`, `os.Stdin`, `os.Stdout`, `os.Stderr`.
- **1.6 Control flow:** `if` with an init statement, `for` in all its forms (three-part, condition-only, infinite, `range`), range over integers (Go 1.22), `switch` (expression, no-condition, `fallthrough`), `break` and `continue` with labels, `goto` and why to avoid it.
- **1.7 Reading errors and debugging basics:** Reading compiler errors, common messages (declared and not used, imported and not used, mismatched types, missing return), runtime panics and stack traces (index out of range, nil pointer dereference, nil map write), `fmt.Printf` debugging, a systematic way to debug a failing program.

---

## Phase 2: Built-in Data Structures

- **2.1 Arrays:** Fixed length as part of the type, creation, indexing, value semantics, comparison, multi-dimensional arrays, when arrays are actually used.
- **2.2 Slices:** The slice header (pointer, length, capacity), `make`, `append`, growth behavior, slicing expressions, full slice expressions, `copy`, nil vs empty slices, slices of slices, aliasing bugs, the `slices` package (`Sort`, `SortFunc`, `BinarySearch`, `Contains`, `Index`, `Reverse`, `Clone`).
- **2.3 Maps:** Creation with `make` and literals, the comma-ok idiom, `delete`, randomized iteration order, nil maps, key requirements, maps as sets, nested maps, the `maps` package, why maps are unsafe for concurrent writes.
- **2.4 Structs:** Declaration, literals, field access, anonymous structs, comparison, nested structs, struct tags, memory layout and padding, empty struct `struct{}`, zero-value-ready design.
- **2.5 Pointers:** Address-of `&` and dereference `*`, `new`, nil pointers, pointers to structs and automatic dereferencing, value vs pointer semantics, no pointer arithmetic, escape analysis overview.
- **2.6 Copying and value semantics:** Everything is passed by value, what a copy of a slice, map, or pointer actually copies, shallow vs deep copy, `slices.Clone` and `maps.Clone`, common aliasing bugs.
- **2.7 Algorithmic helpers:** `sort`, `slices`, `container/heap`, `container/list`, `container/ring`, binary search, Big-O notation, time complexity of built-in operations.

---

## Phase 3: Functions

- **3.1 Function basics:** `func`, parameters, return values, multiple return values, named returns and their risks, the `(value, error)` convention, unused parameters, function documentation.
- **3.2 Parameters and arguments:** Parameters are copies, passing pointers, variadic functions (`...T`), spreading a slice into a variadic call, `init` vs regular functions, why Go has no default or keyword arguments.
- **3.3 Scope and blocks:** Block scope, package scope, shadowing pitfalls (especially with `:=` and `err`), package-level variables, why to avoid global state.
- **3.4 First-class functions:** Function types, functions as values, anonymous functions, passing and returning functions, method values and method expressions.
- **3.5 Closures:** Captured variables, function factories, state capture, the per-iteration loop variable change in Go 1.22 and the old pitfall behind it.
- **3.6 `defer`, `panic`, and `recover`:** Deferred calls, evaluation time of deferred arguments, LIFO order, `defer` in loops, modifying named returns in a deferred function, `panic`, `recover`, when panics are appropriate and when they are not.
- **3.7 Functional patterns:** Higher-order functions, the functional options pattern, middleware as function wrappers, `sync.OnceFunc`, writing small composable functions.
- **3.8 Recursion:** Base case and recursive case, stack growth in Go, no tail-call optimization, memoization, converting recursion to iteration.

---

## Phase 4: Packages and the Build System

- **4.1 Packages:** Package declaration, one package per directory, exported and unexported identifiers, import paths, aliasing imports, blank imports and side effects, dot imports and why to avoid them, `init` execution order.
- **4.2 Package design and `internal`:** The `internal/` directory and compiler-enforced visibility, package naming, avoiding `util` and `common`, package cohesion, keeping the public API small, `doc.go`.
- **4.3 Import mechanics:** Import cycles and how to fix them, how `go build` resolves packages, package initialization order, build constraints and `//go:build` tags, files named by OS and architecture (`_linux.go`, `_windows.go`), the `embed` package.
- **4.4 Third-party packages:** Finding packages on `pkg.go.dev`, `go get` with versions, minimal version selection, `replace` and `exclude` directives, upgrading and downgrading, `go mod why`, evaluating package quality, `govulncheck`, supply-chain hygiene.

---

## Phase 5: Errors, Files, and Resources

- **5.1 Errors as values:** The `error` interface, `errors.New`, `fmt.Errorf`, wrapping with `%w`, `errors.Is`, `errors.As`, `errors.Unwrap`, `errors.Join`, sentinel errors, the "if err != nil" pattern and handling vs returning, the nil-interface error trap.
- **5.2 Custom errors and error design:** Custom error types, error types with fields, designing error hierarchies for applications, when to expose sentinel errors or types in a public API, adding context without leaking internals, panic vs error, handling errors once at the boundary.
- **5.3 Resource management:** `defer` for cleanup, `io.Closer`, checking errors from `Close`, closing in loops, `defer` ordering with multiple resources, `sync.OnceValue`, cleanup in tests with `t.Cleanup`.
- **5.4 File handling:** `os.Open`, `os.Create`, `os.OpenFile`, flags and permissions, `io.Reader` and `io.Writer`, `bufio`, `os.ReadFile` and `os.WriteFile`, reading large files in chunks and by line, binary data with `encoding/binary`, `io.Copy`, `io.TeeReader`, `io.Pipe`.
- **5.5 `path/filepath`, `os`, and the filesystem:** Cross-platform paths, `filepath.Join`, `filepath.WalkDir`, `os.Getenv`, `os.LookupEnv`, `os.MkdirAll`, `os.CreateTemp`, `os.MkdirTemp`, the `io/fs` interface, `embed.FS`.
- **5.6 Data formats:** `encoding/json` (marshal, unmarshal, struct tags, `omitempty`, custom `MarshalJSON`, streaming with `Decoder` and `Encoder`, unknown fields, `json.RawMessage`), `encoding/csv`, `encoding/xml` overview, `encoding/gob`, third-party YAML and TOML libraries.
- **5.7 Logging:** The `log` package, `log/slog` (levels, text and JSON handlers, attributes, groups, `With`, custom handlers, context-aware logging), structured logging, why `fmt.Println` is not used in production.

---

## Phase 6: Types, Methods, and Interfaces

- **6.1 Methods:** Methods on types, value receivers vs pointer receivers, method sets, choosing a receiver type, methods on non-struct types, method values.
- **6.2 Embedding and composition:** Struct embedding, promoted fields and methods, shadowing, embedding interfaces, embedding vs inheritance, composition as Go's reuse model.
- **6.3 Interfaces:** Implicit satisfaction, interface values and their internals (type and value), the empty interface and `any`, type assertions, type switches, nil interfaces vs interfaces holding nil, comparing interfaces.
- **6.4 Standard library interfaces:** `fmt.Stringer`, `error`, `io.Reader`, `io.Writer`, `io.Closer`, `sort.Interface`, `json.Marshaler`, `http.Handler`, `context.Context`, composing small interfaces.
- **6.5 Interface design:** "Accept interfaces, return structs", small interfaces, defining interfaces at the point of use, interface segregation, avoiding premature abstraction, compile-time interface checks (`var _ I = (*T)(nil)`).
- **6.6 Constructors and configuration patterns:** `NewT` functions, making the zero value useful, validation in constructors, the functional options pattern, config structs, builder pattern and when it is overkill.
- **6.7 Defined types and enums:** Type definitions vs type aliases, underlying types, enums with `iota`, `String` methods, parsing and validating enum values, bit flags, the `stringer` tool.
- **6.8 Encapsulation:** Exported vs unexported fields and methods, package-level encapsulation, immutable-style types, returning copies vs references, `internal` packages as boundaries.
- **6.9 Equality and ordering:** The `comparable` constraint, struct and array comparison, structs as map keys, the `cmp` package (`Compare`, `Or`), custom sort orders with `slices.SortFunc`.

---

## Phase 7: Generics and Iterators

- **7.1 Generics basics:** Type parameters, generic functions, type inference, the `any` and `comparable` constraints, instantiation.
- **7.2 Constraints and generic types:** Constraint interfaces, union types, the tilde `~T`, `cmp.Ordered`, generic structs and methods, generic type aliases, limitations (no generic methods, no specialization).
- **7.3 Generic data structures and algorithms:** Stack, queue, set, linked list, `Map`, `Filter`, `Reduce`, ordered maps, generic caches, reading the generic code in `slices` and `maps`.
- **7.4 Iterators:** Range-over-function (Go 1.23), `iter.Seq` and `iter.Seq2`, writing iterators, `iter.Pull`, iterator helpers in `slices` and `maps` (`All`, `Values`, `Collect`, `Sorted`), lazy evaluation, stopping iteration early.
- **7.5 When to use generics:** Guidelines from the Go team, when interfaces are better, when plain code is better, readability and compile-time cost, generics and performance.

---

## Phase 8: Concurrency

- **8.1 Concepts:** Concurrency vs parallelism, CPU-bound vs I/O-bound work, goroutines vs threads, the Go scheduler (G, M, P), `GOMAXPROCS`, choosing a model.
- **8.2 Goroutines:** The `go` statement, `sync.WaitGroup` (including `wg.Go` in newer versions), goroutine lifetimes, passing data into goroutines, why `main` exiting kills everything, the loop variable fix in Go 1.22.
- **8.3 Channels:** Unbuffered and buffered channels, send and receive, `close`, `range` over a channel, directional channel types, nil channels, channel ownership rules, channels vs mutexes.
- **8.4 `select`:** Multiplexing channels, `default`, timeouts with `time.After`, ticker loops, random selection, disabling cases with nil channels.
- **8.5 The `sync` and `sync/atomic` packages:** `Mutex`, `RWMutex`, `Once`, `Cond`, `Pool`, `Map`, `sync/atomic` types, when atomics are enough, copying locks (a vet error).
- **8.6 `context`:** `context.Background`, `WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue` and its limits, `WithCancelCause`, passing context as the first parameter, propagation through call chains, checking `ctx.Done()`.
- **8.7 Concurrency patterns:** Worker pools, fan-out and fan-in, pipelines, semaphores with buffered channels, `errgroup`, rate limiting, bounded parallelism, graceful shutdown, producer and consumer.
- **8.8 Race conditions, deadlocks, and leaks:** Data races, the race detector (`-race`), deadlock detection, goroutine leaks and how to find them, forgotten cancellation, common concurrency bugs.
- **8.9 The Go memory model:** Happens-before, synchronization guarantees, why unsynchronized reads are bugs, safe publication.

---

## Phase 9: Advanced Language Features

- **9.1 Type system in depth:** Underlying types, type identity, assignability rules, conversions between named types, untyped constants in depth, method set rules in detail.
- **9.2 Reflection:** The `reflect` package, `reflect.Type` and `reflect.Value`, reading struct tags, setting values, how `encoding/json` works internally, costs and risks of reflection.
- **9.3 `unsafe` and cgo (overview):** `unsafe.Pointer`, `unsafe.Sizeof`, `unsafe.String` and `unsafe.Slice`, why `unsafe` breaks guarantees, what cgo is and why it is expensive, when neither is justified.
- **9.4 Memory model and garbage collection:** Stack vs heap, escape analysis and `-gcflags=-m`, the garbage collector (concurrent, tri-color), `GOGC` and `GOMEMLIMIT`, reducing allocations, `sync.Pool`, `runtime.ReadMemStats`.
- **9.5 Runtime internals (overview):** Slice, map, string, and interface internals, how goroutine stacks grow, the scheduler, `runtime.Gosched`, `runtime.NumGoroutine`, `runtime/debug`, `weak` pointers and `unique` (recent additions), finalizers and cleanups.
- **9.6 Build system features:** Build tags in depth, `//go:generate`, `//go:embed`, `ldflags` and injecting version information, cross-compiling with `GOOS` and `GOARCH`, linking statically, reproducible builds.
- **9.7 Tooling and code generation:** `go generate` workflows, `stringer`, `go/ast` and `go/parser` overview, writing a simple code generator, `go tool` and tool directives in `go.mod`.

---

## Phase 10: Testing, Debugging, and Performance

- **10.1 The `testing` package:** Test files and functions, `go test`, table-driven tests, subtests with `t.Run`, `t.Helper`, `t.Cleanup`, `t.Parallel`, `t.TempDir`, `TestMain`, example tests as documentation.
- **10.2 Test doubles and HTTP testing:** Interfaces as seams, hand-written fakes and stubs, `testify` (and the argument for using only the standard library), `httptest`, `iotest`, testing time-dependent code, golden files.
- **10.3 Coverage and fuzzing:** `go test -cover`, `-coverprofile`, HTML coverage reports, meaningful coverage vs vanity metrics, fuzz tests with `testing.F`, the fuzz corpus.
- **10.4 Benchmarks:** `testing.B`, `b.Loop` and `b.N`, `-benchmem`, reading allocation counts, comparing runs with `benchstat`, avoiding benchmark pitfalls (compiler optimizations, setup cost).
- **10.5 Debugging:** Reading panics and goroutine dumps, Delve (`dlv`), debugging from VS Code, `GOTRACEBACK`, `-race` in tests, `runtime.Stack`, logging for debuggability.
- **10.6 Profiling:** `pprof` for CPU, heap, goroutine, and block profiles, `net/http/pprof`, flame graphs, `go tool trace`, `runtime/trace`, finding bottlenecks and leaks, profile-guided optimization overview.
- **10.7 Optimization basics:** Measure first, choosing data structures, preallocating slices and maps, reducing allocations, `strings.Builder` vs concatenation, avoiding premature optimization, caching strategies.

---

## Phase 11: Standard Library for Production Work

- **11.1 `time`:** `time.Time`, `time.Duration`, monotonic clock, time zones and `time.LoadLocation`, the reference layout `2006-01-02T15:04:05Z07:00` and why it exists, parsing and formatting, RFC 3339, timers and tickers, UTC discipline, injecting a clock for tests.
- **11.2 Text processing:** `regexp` (RE2 syntax and its guarantees), `text/template`, `bufio.Scanner` and its token limit, `strings` and `bytes` utilities, `text/tabwriter`, Unicode normalization overview.
- **11.3 Command-line tools:** `os.Args`, the `flag` package, subcommands with `flag.FlagSet`, exit codes with `os.Exit`, why `os.Exit` skips deferred calls, structuring `main` to return errors, overview of Cobra and urfave/cli.
- **11.4 System interaction:** `os/exec`, capturing output, `os/signal`, `signal.NotifyContext`, graceful shutdown, `runtime` and `os` information, environment handling.
- **11.5 Security and cryptography:** `crypto/sha256`, `crypto/hmac`, `crypto/rand` vs `math/rand`, `encoding/base64` and `encoding/hex`, password hashing with `golang.org/x/crypto/bcrypt` or `argon2`, `crypto/tls` overview, UUID generation, constant-time comparison.
- **11.6 Databases:** `database/sql`, drivers and registration, `Query`, `QueryRow`, `Exec`, scanning rows, `sql.Null` types, prepared statements, transactions, connection pool settings, context-aware queries, SQL injection prevention, SQLite for local work.
- **11.7 Networking and HTTP:** `net`, `net/http` client with timeouts and connection reuse, `net/http` server, `http.Handler` and `HandlerFunc`, `ServeMux` method and wildcard patterns (Go 1.22), middleware, server timeouts, `httptest`, JSON APIs with the standard library only.
- **11.8 Utility packages:** `math/rand/v2`, `bytes`, `io` utilities, `sort`, `container/*`, `unique`, `hash/fnv`, `os/user`, `text/scanner`, `go/format`, `expvar`.

---

## Phase 12: Professional Practices

- **12.1 Modules and releasing:** Semantic versioning, major version suffixes (`/v2`), tagging releases, publishing a module, `go.mod` directives (`go`, `toolchain`, `retract`, `replace`), deprecation, backward compatibility rules.
- **12.2 Configuration:** Environment-based configuration, flags and environment variables together, typed config structs with validation, secrets handling, twelve-factor basics, loading `.env` in development only.
- **12.3 Design principles:** SOLID applied to Go, dependency injection without frameworks, package by feature vs package by layer, separation of concerns, ports and adapters, avoiding global state and `init` side effects.
- **12.4 Design patterns:** Factory, Strategy, Observer, Adapter, Decorator and middleware, Repository, Singleton with `sync.Once` (and the package-level alternative), Functional options, and patterns Go does not need.
- **12.5 Clean code and idioms:** Effective Go, Go Code Review Comments, naming, function size, early returns, error handling style, avoiding stutter (`user.UserService`), simplicity over cleverness, refactoring, code review habits.
- **12.6 Documentation and collaboration:** Doc comments and `go doc`, `pkg.go.dev` rendering, runnable examples as tests, README structure, `CONTRIBUTING`, pull requests, branching workflows, commit message conventions, changelogs.
- **12.7 Building and shipping binaries:** `go build` flags, stripping and embedding version info, cross-compilation matrix, static binaries with `CGO_ENABLED=0`, multi-stage Docker builds, minimal images, a release pipeline, `goreleaser` overview.

---

## Phase 13: Capstone Projects (Standard Library Only, No Frameworks)

1. **Command-line task manager:** `flag` subcommands, JSON or SQLite persistence, structs and interfaces, `log/slog`, table-driven tests.
2. **Concurrent URL health checker:** goroutines, `context` timeouts and cancellation, worker pool, retries with backoff, graceful shutdown on signals, structured logging, race-free and tested with `-race`.
3. **Mini job queue library:** Goroutines and channels, generics for typed jobs, custom errors, `context` support, interfaces for storage backends, benchmarks, a full test suite, published as a versioned module.