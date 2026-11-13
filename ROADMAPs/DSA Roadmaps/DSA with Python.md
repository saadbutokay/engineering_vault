**Data Structures and Algorithms (DSA) in Python** combines problem-solving logic with Python's clean and highly readable syntax. While low-level languages like C++ or Java are often favored for competitive programming due to execution speed, **Python is widely considered the best language for coding interviews** because its short, expressive syntax allows you to focus purely on the algorithm rather than boilerplate code.

**Target version:** Python 3.12 or newer.

---
## Phase 0: Foundations of Problem Solving and Analysis

1. [[How to Study DSA and Solve Problems]] - Why DSA matters in industry, the problem-solving loop (restate, examples, constraints, brute force, optimize, code, test, analyze), tracing by hand, a reusable solution template, a weekly practice routine, how interview problems differ from production problems, how the notes in this roadmap are organized.

2. [[Complexity Analysis]] - Input size, Big-O, Big-Omega, Big-Theta, best, average, and worst case, common growth rates (constant, logarithmic, linear, `n log n`, quadratic, exponential, factorial), analyzing loops and nested loops, analyzing recursion with recurrences, space complexity including the call stack, amortized analysis, when constants matter, measuring growth empirically with `timeit` and doubling tests.

3. [[The Python Cost Model]] - How CPython builds lists (over-allocated dynamic arrays), dicts (compact insertion-ordered hash tables), sets, strings, and `deque`, the complexity of every common built-in operation, hidden costs (slicing copies, `x in list`, `insert(0, x)`, `pop(0)`, `+=` on strings, `list.index`), arbitrary-precision integers and their cost, object overhead and `sys.getsizeof`, the recursion limit, why C-implemented built-ins beat hand-written loops.

4. [[Testing and Verifying Algorithms]] - `pytest` for algorithms, an edge-case checklist (empty, single element, duplicates, negatives, maximum size, sorted, reverse sorted), brute-force oracle comparison, randomized stress tests, property-based testing with Hypothesis, loop invariants and `assert`, debugging by tracing, a small benchmark harness.

5. [[The Python Toolbox for DSA]] - `collections` (`deque`, `Counter`, `defaultdict`, `OrderedDict`), `heapq`, `bisect`, `itertools` (`accumulate`, `pairwise`, `batched`, `combinations`, `permutations`, `product`), `functools` (`cache`, `lru_cache`, `cmp_to_key`), `graphlib`, `math` (`gcd`, `lcm`, `comb`, `perm`, `isqrt`, `inf`), `array`, `dataclasses`, generic classes with PEP 695 syntax, `sortedcontainers` as the main third-party addition, fast input with `sys.stdin`.

6. [[Mathematical Foundations]] - Summations and arithmetic and geometric series, logarithms and exponents, modular arithmetic, `gcd` and Euclid's algorithm, primes and the Sieve of Eratosthenes, fast exponentiation, counting with permutations and combinations, probability basics for randomized algorithms, proof by induction, loop invariants, proof by contradiction.

---
## Phase 1: Arrays, Strings, and Hashing

1. [[Arrays and Dynamic Arrays]] - Contiguous memory and constant-time indexing, how Python lists store references, implementing a dynamic array with a growth factor, amortized append, insertion and deletion costs, in-place vs extra-space algorithms, reversing and rotating, removing duplicates in place, the `array` module and `bytearray`.

2. [[Strings as Sequences]] - Immutability and the cost of repeated concatenation, `str.join` and building with lists, character frequency, comparison, palindromes, anagrams, `ord` and `chr`, ASCII vs Unicode concerns, parsing numbers by hand, run-length compression, validating formatted input.

3. [[Hash Tables]] - Hash functions, collisions, chaining vs open addressing, load factor and resizing, implementing a hash map from scratch, how CPython's `dict` and `set` work, the `__hash__` and `__eq__` contract, why mutable keys break, hash randomization, worst-case behavior and hash-flooding attacks.

4. [[Counting and Grouping Patterns]] - Frequency maps with `Counter`, the Two Sum family, grouping with `defaultdict`, anagram groups, first unique element, majority element with Boyer-Moore voting, duplicate detection, canonical-form keys, pigeonhole reasoning.

5. [[Prefix Sums and Difference Arrays]] - One-dimensional prefix sums, range-sum queries, subarray sum equals K with a hash map, two-dimensional prefix sums, difference arrays for range updates, prefix XOR and product, running totals in analytics code.

6. [[Two Pointers]] - Opposite-end pointers, same-direction (fast and slow) pointers, partitioning arrays, merging sorted arrays, Three Sum, container with most water, in-place filtering, when the technique applies (sorted or monotonic data).

7. [[Sliding Window]] - Fixed-size windows, variable-size windows, windows with a hash map, longest substring without repeats, minimum window substring, at most K distinct elements, shrink conditions, sliding window vs prefix sums.

8. [[Matrices and Grids]] - Building 2D lists correctly (the `[[0] * n] * m` bug), row-major traversal, direction arrays, spiral traversal, rotating and transposing, searching a sorted matrix, in-bounds checks, grids as implicit graphs previewed.

---
## Phase 2: Recursion, Sorting, and Searching

1. [[Recursion and the Call Stack]] - Thinking recursively, base case and recursive case, the call stack and stack frames, recursion trees, deriving complexity from recurrences, the Master theorem, the recursion limit in Python, converting recursion to iteration with an explicit stack, memoization introduced.

2. [[Elementary Sorting]] - Bubble, selection, and insertion sort, stability, in-place sorting, adaptive behavior, why insertion sort still matters for small inputs, counting comparisons and swaps.

3. [[Efficient Sorting]] - Merge sort, quicksort (Lomuto and Hoare partitioning, pivot strategies, randomized pivots, worst case), heap sort previewed, the `n log n` lower bound for comparison sorts, how Timsort works and why Python's `sorted` is stable and adaptive, sorting with `key=` and `cmp_to_key`.

4. [[Non-Comparison Sorting]] - Counting sort, radix sort, bucket sort, when they beat `n log n`, memory trade-offs, sorting integers, strings, and fixed-width keys.

5. [[Binary Search]] - The classic form and its off-by-one traps, lower and upper bound, searching rotated and bitonic arrays, binary search on the answer space, floating-point binary search, the `bisect` module and its `key` parameter, searching sorted logs by timestamp.

6. [[Selection and Order Statistics]] - Partitioning, quickselect and its expected complexity, median of medians overview, kth largest with a heap vs quickselect, `heapq.nlargest` and `nsmallest`, finding medians and percentiles.

7. [[Intervals and Sweep Line]] - Merging intervals, inserting an interval, overlap detection, meeting rooms and minimum rooms, event-based sweep line, calendar booking, interval scheduling.

---
## Phase 3: Linear Data Structures

1. [[Linked Lists]] - Singly and doubly linked lists, nodes with classes and `__slots__`, sentinel nodes, insertion, deletion, and reversal, fast and slow pointers, Floyd's cycle detection, finding the middle, merging sorted lists, why linked lists are rarely the right choice in Python, when they still are.

2. [[Stacks]] - Last-in first-out behavior, using `list` as a stack, balanced brackets, expression evaluation, the shunting-yard algorithm, undo stacks, min stack, monotonic stack, next greater element, largest rectangle in a histogram, replacing recursion with a stack.

3. [[Queues and Deques]] - First-in first-out behavior, `collections.deque` and why `list.pop(0)` is wrong, circular buffers, a queue from two stacks, monotonic deque and sliding window maximum, `queue.Queue` and `queue.PriorityQueue` for threads, bounded queues and back-pressure.

4. [[Combining Structures]] - A hash map plus a doubly linked list for an LRU cache, `OrderedDict` and `functools.lru_cache` internals, LFU cache, ring-buffer log storage, browser history, insert-delete-random-in-constant-time, designing data structures from requirements.

---
## Phase 4: Trees

- **4.1 Binary trees:** Terminology (root, leaf, height, depth, level), node representations, array representation, recursive and iterative traversals (preorder, inorder, postorder, level order), height and size, tree equality and symmetry, tree construction from traversals.
- **4.2 Binary search trees:** The BST property, search, insert, delete (three cases), successor and predecessor, validating a BST, kth smallest, degeneration to a linked list, inorder iterators, why Python has no built-in BST.
- **4.3 Balanced trees and ordered containers:** Why balance matters, AVL trees and rotations, red-black trees overview, B-trees and why databases and filesystems use them, skip lists, `sortedcontainers` internals and when to use it, ordered maps in practice.
- **4.4 Heaps and priority queues:** The binary heap and array layout, sift up and sift down, `heapify` in linear time, `heapq` (min-heap, negation for max-heap, tuples with tie-breakers), top-K, merging K sorted lists, two heaps for a running median, lazy deletion, task and event schedulers.
- **4.5 Tries:** Prefix trees, insert, search, and prefix queries, dictionary-based vs array-based nodes, autocomplete, word search on a grid, longest common prefix, compressed tries and radix trees, memory costs in Python.
- **4.6 Tree algorithms:** Lowest common ancestor, diameter, maximum path sum, path sum variants, serialization and deserialization, N-ary trees, tree dynamic programming, tree-shaped data in real systems (filesystems, org charts, DOM, ASTs).

---

## Phase 5: Graphs

- **5.1 Graph fundamentals and representations:** Directed vs undirected, weighted, cyclic and acyclic, degree, adjacency list, adjacency matrix, edge list, choosing a representation, building a graph from input, implicit graphs, grids and state spaces as graphs.
- **5.2 Breadth-first and depth-first search:** BFS with `deque`, recursive and iterative DFS, visited sets, connected components, cycle detection in directed and undirected graphs, flood fill, shortest path in unweighted graphs, bipartite checking, multi-source BFS, 0-1 BFS, word ladder, path reconstruction.
- **5.3 Topological sort and DAGs:** Kahn's algorithm, DFS-based ordering, detecting cycles, course scheduling, build and dependency ordering, parallel scheduling levels, `graphlib.TopologicalSorter`.
- **5.4 Shortest paths:** Dijkstra with `heapq`, why negative weights break it, Bellman-Ford and negative cycles, Floyd-Warshall, A* with heuristics, choosing the right algorithm, reconstructing paths, shortest paths on implicit graphs.
- **5.5 Disjoint set union:** Union-find, path compression, union by rank or size, near-constant amortized cost, counting components, detecting cycles, accounts merging, offline connectivity problems.
- **5.6 Minimum spanning trees:** The cut property, Kruskal's algorithm with union-find, Prim's algorithm with a heap, network design, clustering, comparing the two.
- **5.7 Advanced graph algorithms:** Strongly connected components (Tarjan and Kosaraju), bridges and articulation points, Eulerian paths, bipartite matching overview, maximum flow overview with Edmonds-Karp, PageRank introduced.

---

## Phase 6: Algorithmic Paradigms

- **6.1 Brute force and backtracking:** The decision-tree model, subsets, permutations, combinations, N-Queens, Sudoku, word search, generating parentheses, pruning, handling duplicates, backtracking vs dynamic programming.
- **6.2 Divide and conquer:** The split, solve, and combine pattern, maximum subarray, counting inversions, closest pair of points overview, Karatsuba multiplication overview, analyzing with recurrences, parallelizable structure.
- **6.3 Greedy algorithms:** The greedy-choice property, exchange arguments, interval scheduling, Huffman coding, jump game, gas station, task scheduling, why greedy fails (the coin change counterexample), proving or disproving a greedy approach.
- **6.4 Dynamic programming foundations:** Overlapping subproblems and optimal substructure, defining state and transition, top-down memoization with `functools.cache`, bottom-up tabulation, space optimization, Fibonacci, climbing stairs, house robber, recursion limit issues in memoized code.
- **6.5 Classic dynamic programming:** 0/1 knapsack and unbounded knapsack, coin change (minimum coins and number of ways), longest increasing subsequence in quadratic and `n log n` time, longest common subsequence, edit distance, grid path counting, partition problems, reconstructing the solution.
- **6.6 Advanced dynamic programming:** Bitmask DP, interval DP, state-machine DP (stock-trading problems), DP on trees, DP on DAGs, digit DP overview, DP with hash-map states, how to recognize a DP problem.
- **6.7 Bit manipulation:** Bitwise operators on Python's arbitrary-precision integers, masks, setting, clearing, and toggling bits, XOR tricks, counting bits with `int.bit_count`, power-of-two checks, enumerating subsets with bitmasks, bit sets as compact state.

---

## Phase 7: String Algorithms

- **7.1 Pattern matching:** The naive method, KMP and the failure function, the Z-algorithm, Rabin-Karp and rolling hashes, comparing with `str.find` and `re`, what CPython's `find` actually does.
- **7.2 Palindromes and subsequences:** Expand-around-center, Manacher's algorithm, longest palindromic substring, subsequence checks, longest common substring, string DP revisited.
- **7.3 Multi-pattern matching and suffix structures:** Aho-Corasick built on tries, keyword filtering and content scanning, suffix arrays and the LCP array, suffix automaton overview, when each is used in practice.
- **7.4 Similarity and diff algorithms:** Edit distance and fuzzy matching, LCS-based diff, the Myers diff algorithm overview, `difflib`, Jaccard similarity, shingling and MinHash overview, spell checking, deduplicating near-identical records.

---

## Phase 8: Advanced Data Structures

- **8.1 Segment trees:** Range queries and point updates, array-based construction, lazy propagation for range updates, range minimum, sum, and maximum, iterative bottom-up variants, performance in Python.
- **8.2 Fenwick trees:** The binary indexed tree, prefix queries and updates in `log n`, range updates with two trees, counting inversions, choosing between Fenwick and segment trees.
- **8.3 Sparse tables and sqrt decomposition:** Static range minimum queries in constant time, idempotent operations, square-root decomposition, offline queries and Mo's algorithm overview.
- **8.4 Probabilistic data structures:** Bloom filters and their false-positive math, counting Bloom filters, count-min sketch, HyperLogLog overview, reservoir sampling, when approximate answers are the right answer.
- **8.5 Spatial and interval structures:** Interval trees, k-d trees, quadtrees, R-tree overview, geohash, nearest-neighbor queries, location-based services.
- **8.6 Memory-efficient structures in Python:** `__slots__`, the `array` module, `bytearray` and `memoryview`, `struct`, bit arrays by hand, string interning, compact node representations, `tracemalloc` to measure memory.

---

## Phase 9: Algorithms in Industry Systems

- **9.1 Rate limiting algorithms:** Fixed window, sliding window log, sliding window counter, token bucket, leaky bucket, per-client state with expiry, thread-safe implementations, complexity and memory per client.
- **9.2 Hashing in distributed systems:** Sharding by modulo and why it breaks, consistent hashing with virtual nodes, rendezvous hashing, load-balancing algorithms (round robin, weighted, least connections), replica placement.
- **9.3 Scheduling and top-K at scale:** Priority and deadline schedulers, delayed job queues, K-way merge, streaming top-K, heavy hitters, reservoir sampling, approximate counting for metrics.
- **9.4 Search and indexing:** Inverted indexes, tokenization, boolean queries, TF-IDF and BM25 overview, autocomplete with tries, B-tree indexes in databases, LSM trees overview, why indexes speed reads and slow writes.
- **9.5 Data processing algorithms:** External merge sort for data larger than memory, deduplication, hash join, sort-merge join, nested-loop join, group-by aggregation, window aggregates, streaming vs batch processing with generators.
- **9.6 Compression and encoding:** Run-length encoding, Huffman coding, LZ77 overview, varint encoding, base64 by hand, checksums and CRC, hashing for integrity, when to use `zlib` instead.
- **9.7 Graph algorithms in production:** Dependency and version resolution, build systems, routing and shortest paths, recommendation by graph traversal, social graph queries, PageRank, cycle detection in service dependencies.
- **9.8 Concurrency-aware data structures:** Thread-safe queues and caches, lock-based designs, lock-free ideas overview, producer and consumer, the effect of the GIL on data-structure design, `queue`, `threading.Lock`, and `asyncio.Queue`.

---

## Phase 10: Performance and Python-Specific Optimization

- **10.1 Measuring algorithm performance:** `timeit`, `cProfile`, empirical complexity by plotting runtime against input size, constant factors vs growth rate, benchmarking pitfalls, worst-case input generation.
- **10.2 Writing fast Python:** Prefer built-ins and comprehensions, local variable binding, avoiding repeated concatenation, `heapq`, `bisect`, and `itertools` over hand-written loops, iterative DFS instead of recursion, caching, avoiding unnecessary copies.
- **10.3 When Python is not enough:** NumPy vectorization overview, PyPy, `multiprocessing` for parallel algorithms, Cython and `ctypes` overview, choosing between optimizing the algorithm, the code, or the language.
- **10.4 Space optimization and streaming:** Generators and lazy pipelines, one-pass and streaming algorithms, memory profiling with `tracemalloc`, rolling arrays in DP, reducing object overhead, processing data larger than memory.

---

## Phase 11: Problem-Solving Practice

- **11.1 Pattern recognition and constraint analysis:** Mapping input size to the feasible complexity in Python (for example, `n` up to about 20 permits exponential, up to a few thousand permits quadratic, up to about a million requires linear or `n log n`), signals in problem statements, a decision framework for choosing among the patterns in Phases 1 to 6.
- **11.2 Communicating while solving:** Asking clarifying questions, stating brute force first, narrating trade-offs, writing code in a plain editor, testing with your own examples, handling being stuck, time management in timed settings.
- **11.3 Curated problem sets:** Graded sets of 10 to 15 problems per topic area, a mixed review set per phase, spaced repetition schedule, tracking weak patterns, revisiting problems after a week and a month.
- **11.4 Choosing data structures in real systems:** A decision guide from requirements to structure (lookup, ordering, uniqueness, priority, range queries, prefix queries, relationships), reading the complexity of library and database operations, avoiding premature cleverness, reviewing code for accidental quadratic behavior.

---

## Phase 12: Capstone Projects (Standard Library, `pytest`, and Hypothesis Only)

Allowed packages: `pytest` and `hypothesis`. Third-party data-structure libraries such as `sortedcontainers` are not allowed in the capstones.

1. **Collections library from scratch:** Dynamic array, linked list, hash map, heap, trie, LRU cache, and union-find, fully typed with PEP 695 generics, tested with `pytest` and Hypothesis against the built-in equivalents, and benchmarked to show where each one wins and loses.
2. **Search engine in miniature:** Tokenizer, inverted index, trie-based autocomplete, ranked results with a top-K heap, index persisted to disk, benchmarked on a text corpus of at least 100,000 documents.
3. **Route and dependency planner:** A graph library with BFS, Dijkstra, A*, topological sort, strongly connected components, and cycle reporting, a command-line interface, a real road or package-dependency dataset, and tests including large generated graphs.