```
Python has THREE ways to do things concurrently:

┌─────────────────────────────────────────────────────────────┐
│  Threading      → multiple threads, one process             │
│  Multiprocessing→ multiple processes, bypasses GIL          │
│  Async (asyncio)→ single thread, cooperative multitasking   │
└─────────────────────────────────────────────────────────────┘

The WRONG choice costs you:
  ❌ Async for CPU work  → event loop blocked, app freezes
  ❌ Threading for CPU   → GIL prevents true parallelism
  ❌ Multiprocessing for IO → massive overhead for simple tasks

The RIGHT choice gives you:
  ✅ 10x-50x performance improvement
  ✅ No wasted resources
  ✅ Predictable behavior

The decision comes down to ONE question:
  Is my bottleneck CPU or I/O?
```

---

## Setup

Bash

```
cd ~/projects
mkdir concurrency_deep_dive
cd concurrency_deep_dive
python3 -m venv venv
source venv/bin/activate
pip install httpx psutil matplotlib
touch concurrency.py
code .
```

---

## 1. 🧠 Understanding the Problem Space

Python

```
# concurrency.py
import time
import sys

"""
TWO TYPES OF TASKS:

I/O-BOUND: waiting for something external
  - Database queries (waiting for PostgreSQL)
  - HTTP requests (waiting for remote server)
  - File reads (waiting for disk)
  - Cache lookups (waiting for Redis)
  
  CPU is IDLE during the wait.
  Concurrency helps: while waiting for one, start another.

CPU-BOUND: doing actual computation
  - Image/video processing
  - Data encryption/decryption
  - Machine learning inference
  - Large data aggregations
  - JSON serialization of huge datasets
  
  CPU is BUSY the entire time.
  Concurrency only helps if you have multiple CPUs.
  Multiple threads DON'T help due to the GIL.
  Multiple processes DO help.

MIXED:
  - Some operations combine both
  - Need to profile to understand the bottleneck
"""

def measure(label: str, func, *args):
    """Measure and print execution time."""
    start = time.perf_counter()
    result = func(*args)
    elapsed = (time.perf_counter() - start) * 1000
    print(f"  {label}: {elapsed:.0f}ms")
    return result


# Simulate I/O-bound work (database query, HTTP call)
def simulate_io(duration: float = 0.1) -> str:
    time.sleep(duration)   # blocked, doing nothing
    return "io_result"


# Simulate CPU-bound work (heavy computation)
def simulate_cpu(iterations: int = 5_000_000) -> int:
    total = 0
    for i in range(iterations):
        total += i * i
    return total


# Baseline: how long each takes
print("=== Task Baselines ===")
measure("Single I/O task (100ms)", simulate_io, 0.1)
measure("Single CPU task", simulate_cpu, 5_000_000)
print()
```

---

## 2. 🔒 The GIL — Deep Understanding

Python

```
"""
GIL = Global Interpreter Lock

CPython (standard Python) has one lock that protects
all Python objects from concurrent modification.

Only ONE thread can execute Python bytecode at a time.

Why it exists:
  CPython uses reference counting for memory management.
  Reference counting is NOT thread-safe without protection.
  The GIL is that protection.

What it means for you:
  Threads share memory but can't truly run Python code in parallel.
  They take turns: Thread A runs → releases GIL → Thread B runs.
  
  For I/O: GIL is released while waiting for I/O.
           Other threads can run while Thread A waits for DB.
           
  For CPU: GIL held the entire time.
           Other threads starve.
           No benefit from multiple cores.

Proof:
"""

import threading
import multiprocessing
import time


def count_to_n(n: int) -> int:
    """Pure CPU work."""
    count = 0
    for i in range(n):
        count += i
    return count


N = 50_000_000


def demo_gil_impact():
    """Show the GIL preventing CPU parallelism with threads."""
    print("=== GIL Impact on CPU-Bound Work ===\n")

    # Single thread baseline
    start = time.perf_counter()
    count_to_n(N)
    count_to_n(N)
    single_time = time.perf_counter() - start
    print(f"Sequential (1 thread): {single_time:.2f}s")

    # Two threads — you'd EXPECT this to be faster (2 cores)
    # But the GIL prevents true parallelism
    start = time.perf_counter()
    t1 = threading.Thread(target=count_to_n, args=(N,))
    t2 = threading.Thread(target=count_to_n, args=(N,))
    t1.start()
    t2.start()
    t1.join()
    t2.join()
    thread_time = time.perf_counter() - start
    print(f"Two threads (GIL):     {thread_time:.2f}s")

    # Two processes — each has its OWN GIL
    # True parallelism! Uses 2 CPU cores.
    start = time.perf_counter()
    p1 = multiprocessing.Process(target=count_to_n, args=(N,))
    p2 = multiprocessing.Process(target=count_to_n, args=(N,))
    p1.start()
    p2.start()
    p1.join()
    p2.join()
    process_time = time.perf_counter() - start
    print(f"Two processes:         {process_time:.2f}s")

    print(f"\nThreads are {thread_time/single_time:.1f}x vs sequential (overhead!)")
    print(f"Processes are {single_time/process_time:.1f}x faster than sequential")

    # Expected output (on 2+ core machine):
    # Sequential:    2.00s
    # Two threads:   2.10s  ← SLOWER (GIL + context switching overhead)
    # Two processes: 1.10s  ← ~2x faster (true parallelism)


demo_gil_impact()
```

---

## 3. 🧵 Threading — For I/O-Bound Work

Python

```
import threading
import time
import queue
from typing import List, Callable, Any
from concurrent.futures import ThreadPoolExecutor, as_completed
import httpx


# ─────────────────────────────────────────
# BASIC THREADING
# ─────────────────────────────────────────
def threading_basics():
    """Understand thread creation and management."""
    print("\n=== Threading Basics ===\n")

    results = []
    lock = threading.Lock()     # protect shared state

    def worker(task_id: int, duration: float):
        """Simulate I/O work."""
        thread_name = threading.current_thread().name
        print(f"  Thread {thread_name}: task {task_id} starting")
        time.sleep(duration)   # I/O wait (GIL released here!)
        with lock:             # protect shared list
            results.append(f"task_{task_id}_done")
        print(f"  Thread {thread_name}: task {task_id} done")

    # Create and start threads
    threads = []
    start = time.perf_counter()

    for i in range(5):
        t = threading.Thread(
            target=worker,
            args=(i, 0.3),
            name=f"Worker-{i}",
            daemon=True    # thread dies when main thread dies
        )
        threads.append(t)
        t.start()

    # Wait for all to complete
    for t in threads:
        t.join(timeout=5)   # wait max 5 seconds

    elapsed = time.perf_counter() - start
    print(f"\n  5 I/O tasks: {elapsed:.2f}s (would be 1.5s sync)")
    print(f"  Results: {results}")


# ─────────────────────────────────────────
# THREAD POOL — the right way to use threads
# ─────────────────────────────────────────
def thread_pool_demo():
    """ThreadPoolExecutor — preferred way to use threads."""
    print("\n=== ThreadPoolExecutor ===\n")

    def fetch_user(user_id: int) -> dict:
        """Simulate database query."""
        time.sleep(0.1)   # I/O wait
        return {"id": user_id, "name": f"User {user_id}"}

    def process_payment(amount: float) -> dict:
        """Simulate payment processing."""
        time.sleep(0.2)   # I/O wait
        return {"amount": amount, "status": "processed"}

    # ─── map(): apply function to many items ─────────────────
    print("map() - process many items with same function:")
    start = time.perf_counter()

    with ThreadPoolExecutor(max_workers=10) as executor:
        # map() keeps order, blocks until all done
        users = list(executor.map(fetch_user, range(1, 11)))

    elapsed = time.perf_counter() - start
    print(f"  10 DB queries: {elapsed:.2f}s (would be 1.0s sync)")

    # ─── submit(): more control per task ──────────────────────
    print("\nsubmit() - submit individual tasks:")
    start = time.perf_counter()

    futures = []
    with ThreadPoolExecutor(max_workers=5) as executor:
        # Submit different functions
        for i in range(1, 6):
            future = executor.submit(fetch_user, i)
            futures.append(future)
            future = executor.submit(process_payment, i * 10.0)
            futures.append(future)

    results = [f.result() for f in futures]
    elapsed = time.perf_counter() - start
    print(f"  10 mixed tasks: {elapsed:.2f}s")

    # ─── as_completed(): process results as they arrive ───────
    print("\nas_completed() - handle results as they finish:")
    with ThreadPoolExecutor(max_workers=5) as executor:
        future_to_id = {
            executor.submit(fetch_user, i): i
            for i in range(1, 6)
        }

        for future in as_completed(future_to_id):
            task_id = future_to_id[future]
            try:
                result = future.result()
                print(f"  Task {task_id} completed: {result}")
            except Exception as e:
                print(f"  Task {task_id} failed: {e}")


# ─────────────────────────────────────────
# THREAD SAFETY — shared state is dangerous
# ─────────────────────────────────────────
def thread_safety_demo():
    """Show why you need locks for shared mutable state."""
    print("\n=== Thread Safety ===\n")

    # ─── Unsafe counter ───────────────────────────────────────
    unsafe_counter = 0

    def increment_unsafe():
        nonlocal unsafe_counter
        for _ in range(10_000):
            # Read-modify-write is NOT atomic!
            unsafe_counter += 1

    threads = [threading.Thread(target=increment_unsafe) for _ in range(5)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    print(f"Unsafe counter: {unsafe_counter} (expected 50,000)")
    print(f"  Race condition: {unsafe_counter != 50_000}")

    # ─── Safe counter with Lock ────────────────────────────────
    safe_counter = 0
    lock = threading.Lock()

    def increment_safe():
        nonlocal safe_counter
        for _ in range(10_000):
            with lock:         # only one thread at a time
                safe_counter += 1

    threads = [threading.Thread(target=increment_safe) for _ in range(5)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    print(f"Safe counter:   {safe_counter} (expected 50,000)")
    print(f"  Correct: {safe_counter == 50_000}")

    # ─── Thread-safe data structures ──────────────────────────
    # queue.Queue is thread-safe by design
    task_queue = queue.Queue()

    def producer():
        for i in range(10):
            task_queue.put(f"task_{i}")
            time.sleep(0.01)
        task_queue.put(None)  # sentinel

    def consumer(name: str):
        while True:
            item = task_queue.get()
            if item is None:
                task_queue.put(None)   # pass sentinel to next consumer
                break
            print(f"  {name} processed: {item}")
            task_queue.task_done()

    p = threading.Thread(target=producer)
    c = threading.Thread(target=consumer, args=("Consumer-1",))
    p.start()
    c.start()
    p.join()
    c.join()


# ─────────────────────────────────────────
# REAL BACKEND USE: Parallel I/O with threads
# ─────────────────────────────────────────
def real_world_threading():
    """Practical threading examples for backend work."""
    print("\n=== Real Backend Threading ===\n")

    # Example 1: Parallel database queries (using sync DB driver)
    def get_user_data_parallel(user_ids: List[int]) -> List[dict]:
        """Fetch multiple users in parallel using threads."""
        def fetch_one(uid: int) -> dict:
            time.sleep(0.05)   # simulate sync DB query
            return {"id": uid, "name": f"User {uid}"}

        with ThreadPoolExecutor(max_workers=min(len(user_ids), 10)) as pool:
            return list(pool.map(fetch_one, user_ids))

    start = time.perf_counter()
    users = get_user_data_parallel(list(range(1, 21)))
    print(f"20 parallel DB queries: {(time.perf_counter()-start)*1000:.0f}ms")

    # Example 2: Call multiple external services simultaneously
    def call_external_services(user_id: int) -> dict:
        """
        Call multiple external services in parallel.
        Common pattern: one API needs data from multiple sources.
        """
        def get_profile():
            time.sleep(0.2)    # external API call
            return {"type": "profile", "data": "..."}

        def get_permissions():
            time.sleep(0.15)   # auth service call
            return {"type": "permissions", "data": "..."}

        def get_preferences():
            time.sleep(0.1)    # preferences service
            return {"type": "preferences", "data": "..."}

        with ThreadPoolExecutor(max_workers=3) as pool:
            futures = {
                "profile": pool.submit(get_profile),
                "permissions": pool.submit(get_permissions),
                "preferences": pool.submit(get_preferences),
            }

        return {key: future.result() for key, future in futures.items()}

    start = time.perf_counter()
    data = call_external_services(42)
    print(f"3 parallel service calls: {(time.perf_counter()-start)*1000:.0f}ms "
          f"(vs 450ms sequential)")


threading_basics()
thread_pool_demo()
thread_safety_demo()
real_world_threading()
```

---

## 4. 🔀 Multiprocessing — For CPU-Bound Work

Python

```
import multiprocessing
import multiprocessing.pool
import os
import time
import math
from concurrent.futures import ProcessPoolExecutor, as_completed
from typing import List, Tuple


# ─────────────────────────────────────────
# WHY MULTIPROCESSING FOR CPU WORK
# ─────────────────────────────────────────
def cpu_bound_task(n: int) -> int:
    """Simulates heavy computation — image processing, encryption, etc."""
    # Calculate many primes (CPU intensive)
    count = 0
    for num in range(2, n):
        is_prime = True
        for divisor in range(2, int(math.sqrt(num)) + 1):
            if num % divisor == 0:
                is_prime = False
                break
        if is_prime:
            count += 1
    return count


def multiprocessing_basics():
    """Basic multiprocessing concepts."""
    print("\n=== Multiprocessing Basics ===\n")

    cpu_count = multiprocessing.cpu_count()
    print(f"Available CPUs: {cpu_count}")

    # Single process baseline
    start = time.perf_counter()
    result = cpu_bound_task(50_000)
    single_time = time.perf_counter() - start
    print(f"Single process: {single_time:.2f}s (found {result} primes)")

    # Multiple processes — true parallelism!
    start = time.perf_counter()
    with multiprocessing.Pool(processes=cpu_count) as pool:
        results = pool.map(
            cpu_bound_task,
            [50_000] * cpu_count   # run same task on each CPU
        )
    multi_time = time.perf_counter() - start
    print(f"  {cpu_count} processes: {multi_time:.2f}s ({single_time/multi_time:.1f}x speedup)")


# ─────────────────────────────────────────
# PROCESS POOL EXECUTOR (preferred in modern Python)
# ─────────────────────────────────────────
def process_pool_demo():
    """ProcessPoolExecutor — preferred way to use multiprocessing."""
    print("\n=== ProcessPoolExecutor ===\n")

    def image_resize(image_id: int, width: int, height: int) -> dict:
        """Simulate CPU-intensive image resizing."""
        # Real: use Pillow to resize actual image
        time.sleep(0.1)   # simulate CPU work
        result = sum(i * i for i in range(100_000))  # actual CPU work
        return {
            "image_id": image_id,
            "width": width,
            "height": height,
            "pixels_processed": result
        }

    tasks = [(i, 1920, 1080) for i in range(8)]

    # Sequential
    print("Sequential:")
    start = time.perf_counter()
    results = [image_resize(*task) for task in tasks]
    seq_time = time.perf_counter() - start
    print(f"  8 images: {seq_time:.2f}s")

    # Parallel with processes
    print("Parallel (ProcessPoolExecutor):")
    start = time.perf_counter()
    with ProcessPoolExecutor(max_workers=4) as executor:
        futures = [executor.submit(image_resize, *task) for task in tasks]
        results = [f.result() for f in futures]
    parallel_time = time.perf_counter() - start
    print(f"  8 images: {parallel_time:.2f}s ({seq_time/parallel_time:.1f}x faster)")

    # map() with starmap pattern
    print("\nmap() with unpacked args:")
    start = time.perf_counter()
    with ProcessPoolExecutor(max_workers=4) as executor:
        # map only takes single iterable, use starmap for multiple args
        results = list(executor.map(
            lambda args: image_resize(*args),
            tasks
        ))
    print(f"  8 images: {(time.perf_counter()-start):.2f}s")

    # as_completed — process results as they finish
    print("\nas_completed - handle results in order of completion:")
    with ProcessPoolExecutor(max_workers=4) as executor:
        future_to_task = {
            executor.submit(image_resize, *task): task[0]
            for task in tasks
        }
        for future in as_completed(future_to_task):
            img_id = future_to_task[future]
            result = future.result()
            print(f"  Image {img_id} done: {result['pixels_processed']:,} pixels")


# ─────────────────────────────────────────
# INTER-PROCESS COMMUNICATION
# ─────────────────────────────────────────
def inter_process_communication():
    """How processes share data."""
    print("\n=== Inter-Process Communication ===\n")

    # ─── Queue: pass data between processes ───────────────────
    print("Queue: producer → consumer across processes")

    def producer(q: multiprocessing.Queue, n: int):
        """Generate data and put in queue."""
        for i in range(n):
            result = sum(j * j for j in range(10_000))  # CPU work
            q.put(result)
            print(f"  Produced: {result}")
        q.put(None)   # sentinel

    def consumer(q: multiprocessing.Queue):
        """Process data from queue."""
        while True:
            item = q.get()
            if item is None:
                break
            print(f"  Consumed: {item * 2}")

    q = multiprocessing.Queue()
    p1 = multiprocessing.Process(target=producer, args=(q, 3))
    p2 = multiprocessing.Process(target=consumer, args=(q,))

    p1.start()
    p2.start()
    p1.join()
    p2.join()

    # ─── Shared Memory (Python 3.8+) ──────────────────────────
    print("\nShared memory: fast shared arrays between processes")

    from multiprocessing import shared_memory
    import numpy as np

    # Create shared memory block
    shm = shared_memory.SharedMemory(create=True, size=4 * 1_000)
    shared_array = np.ndarray(1_000, dtype=np.int32, buffer=shm.buf)
    shared_array[:] = 0   # initialize

    def fill_shared_array(shm_name: str, start: int, end: int, value: int):
        """Fill portion of shared array in a different process."""
        existing_shm = shared_memory.SharedMemory(name=shm_name)
        arr = np.ndarray(1_000, dtype=np.int32, buffer=existing_shm.buf)
        arr[start:end] = value
        existing_shm.close()

    processes = []
    for i in range(4):
        p = multiprocessing.Process(
            target=fill_shared_array,
            args=(shm.name, i * 250, (i + 1) * 250, i + 1)
        )
        processes.append(p)
        p.start()

    for p in processes:
        p.join()

    print(f"  Shared array (first 20): {shared_array[:20].tolist()}")
    shm.close()
    shm.unlink()    # free the shared memory


# ─────────────────────────────────────────
# REAL BACKEND USE: CPU-intensive tasks
# ─────────────────────────────────────────
def real_world_multiprocessing():
    """Practical multiprocessing for backend work."""
    print("\n=== Real Backend Multiprocessing ===\n")

    # Example 1: Bulk image processing
    def process_image_batch(image_ids: List[int]) -> List[dict]:
        """Process multiple images using all CPU cores."""
        def process_one(img_id: int) -> dict:
            # Simulate PIL image resize + compression
            result = 0
            for i in range(500_000):   # CPU work
                result += i % 7
            return {"id": img_id, "processed": True, "checksum": result}

        with ProcessPoolExecutor(max_workers=os.cpu_count()) as executor:
            return list(executor.map(process_one, image_ids))

    start = time.perf_counter()
    images = process_image_batch(list(range(8)))
    print(f"Bulk image processing ({len(images)} images): "
          f"{(time.perf_counter()-start):.2f}s")

    # Example 2: Report generation (data aggregation)
    def generate_report(data_chunk: List[dict]) -> dict:
        """Compute statistics on a chunk of data."""
        values = [d["value"] for d in data_chunk]
        return {
            "count": len(values),
            "sum": sum(values),
            "min": min(values),
            "max": max(values),
            "mean": sum(values) / len(values)
        }

    # Split large dataset into chunks, process in parallel
    large_dataset = [{"value": i % 1000} for i in range(100_000)]
    cpu_count = os.cpu_count()
    chunk_size = len(large_dataset) // cpu_count
    chunks = [
        large_dataset[i:i + chunk_size]
        for i in range(0, len(large_dataset), chunk_size)
    ]

    start = time.perf_counter()
    with ProcessPoolExecutor(max_workers=cpu_count) as executor:
        chunk_results = list(executor.map(generate_report, chunks))

    # Merge results from all processes
    total_count = sum(r["count"] for r in chunk_results)
    total_sum = sum(r["sum"] for r in chunk_results)
    overall_mean = total_sum / total_count

    print(f"Report on {total_count:,} records: "
          f"{(time.perf_counter()-start)*1000:.0f}ms "
          f"(mean={overall_mean:.1f})")


multiprocessing_basics()
process_pool_demo()
inter_process_communication()
real_world_multiprocessing()
```

---

## 5. ⚡ Async — For I/O-Bound Work at Scale

Python

```
import asyncio
import time
from typing import List


def async_comparison():
    """Show async vs threads for I/O-bound work."""

    async def fetch_data(task_id: int, duration: float = 0.1) -> dict:
        await asyncio.sleep(duration)  # I/O wait (non-blocking!)
        return {"id": task_id, "data": f"result_{task_id}"}

    async def async_demo(n_tasks: int = 100):
        print(f"\n=== Async: {n_tasks} concurrent I/O tasks ===")
        start = time.perf_counter()

        # All tasks run concurrently — just one thread!
        results = await asyncio.gather(*[
            fetch_data(i, 0.1) for i in range(n_tasks)
        ])

        elapsed = time.perf_counter() - start
        print(f"  {n_tasks} tasks: {elapsed:.2f}s "
              f"({n_tasks/elapsed:.0f} tasks/s)")
        return results

    asyncio.run(async_demo(100))
    asyncio.run(async_demo(1000))
    asyncio.run(async_demo(10000))


async_comparison()
```

---

## 6. 📊 The Complete Comparison

Python

```
import time
import asyncio
import threading
import multiprocessing
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor


def run_comprehensive_benchmark():
    """
    Comprehensive comparison across all three approaches.
    Shows exactly when each wins.
    """
    print("\n" + "="*60)
    print("COMPREHENSIVE CONCURRENCY BENCHMARK")
    print("="*60)

    N = 20     # number of tasks
    IO_DURATION = 0.1    # 100ms per I/O task
    CPU_ITERATIONS = 2_000_000   # iterations per CPU task

    # ─────────────────────────────────────────
    # I/O-BOUND COMPARISON
    # ─────────────────────────────────────────
    print(f"\n--- I/O-Bound ({N} tasks, {IO_DURATION*1000:.0f}ms each) ---")
    print(f"Expected sequential time: {N * IO_DURATION:.1f}s")

    def io_task(task_id: int) -> str:
        time.sleep(IO_DURATION)
        return f"done_{task_id}"

    async def async_io_task(task_id: int) -> str:
        await asyncio.sleep(IO_DURATION)
        return f"done_{task_id}"

    # Sequential
    start = time.perf_counter()
    [io_task(i) for i in range(N)]
    print(f"Sequential:       {time.perf_counter()-start:.2f}s")

    # Threading
    start = time.perf_counter()
    with ThreadPoolExecutor(max_workers=N) as pool:
        list(pool.map(io_task, range(N)))
    print(f"Threading:        {time.perf_counter()-start:.2f}s ← winner for sync I/O")

    # Multiprocessing
    start = time.perf_counter()
    with ProcessPoolExecutor(max_workers=min(N, 8)) as pool:
        list(pool.map(io_task, range(N)))
    print(f"Multiprocessing:  {time.perf_counter()-start:.2f}s ← overkill for I/O")

    # Async
    async def run_async_io():
        await asyncio.gather(*[async_io_task(i) for i in range(N)])

    start = time.perf_counter()
    asyncio.run(run_async_io())
    print(f"Async:            {time.perf_counter()-start:.2f}s ← winner for async I/O")

    # ─────────────────────────────────────────
    # CPU-BOUND COMPARISON
    # ─────────────────────────────────────────
    print(f"\n--- CPU-Bound ({N} tasks, {CPU_ITERATIONS:,} iterations each) ---")

    def cpu_task(task_id: int) -> int:
        total = sum(i * i for i in range(CPU_ITERATIONS))
        return total

    async def async_cpu_task(task_id: int) -> int:
        # Running CPU work in async context — BAD but showing impact
        total = sum(i * i for i in range(CPU_ITERATIONS))
        return total

    # Sequential
    start = time.perf_counter()
    [cpu_task(i) for i in range(4)]   # just 4 CPU tasks
    seq_time = time.perf_counter() - start
    print(f"Sequential (4 tasks):      {seq_time:.2f}s")

    # Threading — no speedup due to GIL
    start = time.perf_counter()
    with ThreadPoolExecutor(max_workers=4) as pool:
        list(pool.map(cpu_task, range(4)))
    thread_time = time.perf_counter() - start
    print(f"Threading (4 threads):     {thread_time:.2f}s ← no speedup (GIL!)")

    # Multiprocessing — true parallelism
    start = time.perf_counter()
    with ProcessPoolExecutor(max_workers=4) as pool:
        list(pool.map(cpu_task, range(4)))
    proc_time = time.perf_counter() - start
    print(f"Multiprocessing (4 procs): {proc_time:.2f}s ← winner for CPU work!")
    print(f"  Speedup: {seq_time/proc_time:.1f}x")

    # Async — event loop blocked!
    async def run_async_cpu():
        # This BLOCKS the event loop for each task!
        await asyncio.gather(*[async_cpu_task(i) for i in range(4)])

    start = time.perf_counter()
    asyncio.run(run_async_cpu())
    async_time = time.perf_counter() - start
    print(f"Async (event loop blocked):{async_time:.2f}s ← worst (blocks everything!)")

    # ─────────────────────────────────────────
    # SUMMARY
    # ─────────────────────────────────────────
    print("\n" + "="*60)
    print("SUMMARY TABLE")
    print("="*60)
    print(f"""
┌──────────────────┬──────────────┬──────────────┬──────────────┐
│ Approach         │ I/O-Bound    │ CPU-Bound    │ Overhead     │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│ Sequential       │ ❌ Slow      │ ❌ Slow      │ None         │
│ Threading        │ ✅ Fast      │ ❌ GIL!      │ Low          │
│ Multiprocessing  │ 🟡 Overkill  │ ✅ Fastest   │ High         │
│ Async            │ ✅ Fastest   │ ❌ Blocks!   │ Very Low     │
└──────────────────┴──────────────┴──────────────┴──────────────┘
""")


run_comprehensive_benchmark()
```

---

## 7. 🔄 Mixing Approaches in FastAPI

Python

```
# The real-world pattern: async FastAPI + offload to thread/process pool

import asyncio
from fastapi import FastAPI, BackgroundTasks
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor
from typing import List
import time

app = FastAPI()

# Global pools — created once at startup
_process_pool: ProcessPoolExecutor = None
_thread_pool: ThreadPoolExecutor = None


# ─────────────────────────────────────────
# STARTUP / SHUTDOWN
# ─────────────────────────────────────────
@app.on_event("startup")
async def startup():
    global _process_pool, _thread_pool
    _process_pool = ProcessPoolExecutor(max_workers=4)
    _thread_pool = ThreadPoolExecutor(max_workers=10)
    print("Pools created")


@app.on_event("shutdown")
async def shutdown():
    _process_pool.shutdown(wait=True)
    _thread_pool.shutdown(wait=True)
    print("Pools shut down")


# ─────────────────────────────────────────
# PATTERN 1: CPU work in async route
# ─────────────────────────────────────────
def compress_data(data: bytes) -> bytes:
    """CPU-intensive compression."""
    import zlib
    time.sleep(0.1)   # simulate CPU work
    return zlib.compress(data, level=9)


@app.post("/compress")
async def compress_endpoint(data: str) -> dict:
    """
    CPU work in async route — use process pool.
    Without this: event loop blocked, no other requests handled.
    With this: offloaded to process, event loop free.
    """
    loop = asyncio.get_event_loop()

    # Run CPU work in process pool (non-blocking to event loop)
    compressed = await loop.run_in_executor(
        _process_pool,
        compress_data,
        data.encode()
    )

    return {
        "original_size": len(data),
        "compressed_size": len(compressed),
        "ratio": len(compressed) / len(data)
    }


# ─────────────────────────────────────────
# PATTERN 2: Sync I/O library in async route
# ─────────────────────────────────────────
def sync_database_query(user_id: int) -> dict:
    """
    Sync database call using psycopg2 (not asyncpg).
    Example: legacy code, or library that doesn't support async.
    """
    time.sleep(0.05)   # simulate sync DB query
    return {"id": user_id, "name": f"User {user_id}"}


@app.get("/users-sync/{user_id}")
async def get_user_sync_db(user_id: int) -> dict:
    """
    Use sync DB library in async route — thread pool.
    Doesn't block event loop.
    """
    loop = asyncio.get_event_loop()
    user = await loop.run_in_executor(
        _thread_pool,
        sync_database_query,
        user_id
    )
    return user


# ─────────────────────────────────────────
# PATTERN 3: asyncio.to_thread (Python 3.9+)
# Cleaner syntax for thread pool
# ─────────────────────────────────────────
@app.get("/users/{user_id}")
async def get_user(user_id: int) -> dict:
    """
    asyncio.to_thread — run sync function in thread pool.
    Simpler than run_in_executor.
    """
    user = await asyncio.to_thread(sync_database_query, user_id)
    return user


# ─────────────────────────────────────────
# PATTERN 4: Mixed - async + CPU + I/O
# ─────────────────────────────────────────
def cpu_heavy_report(data: List[dict]) -> dict:
    """CPU-intensive report generation."""
    total = sum(d["value"] for d in data)
    avg = total / len(data) if data else 0
    return {"total": total, "avg": avg, "count": len(data)}


async def fetch_data_async() -> List[dict]:
    """Async I/O operation."""
    await asyncio.sleep(0.1)   # async DB query
    return [{"value": i} for i in range(1000)]


@app.get("/report")
async def generate_report() -> dict:
    """
    Mix async I/O with CPU processing.
    Best of both worlds.
    """
    loop = asyncio.get_event_loop()

    # 1. Async I/O (doesn't block event loop)
    data = await fetch_data_async()

    # 2. CPU processing (offloaded to process pool)
    report = await loop.run_in_executor(
        _process_pool,
        cpu_heavy_report,
        data
    )

    return report


# ─────────────────────────────────────────
# PATTERN 5: Background heavy work
# ─────────────────────────────────────────
def send_bulk_emails(user_ids: List[int]) -> None:
    """Send emails to many users — slow, sync operation."""
    for uid in user_ids:
        time.sleep(0.01)   # simulate email sending
        print(f"Email sent to user {uid}")


@app.post("/notify-users")
async def notify_users(
    user_ids: List[int],
    background_tasks: BackgroundTasks
) -> dict:
    """
    Accept request immediately.
    Do heavy work in background — user doesn't wait.
    """
    # Schedule background work (runs after response sent)
    background_tasks.add_task(
        asyncio.get_event_loop().run_in_executor,
        _thread_pool,
        send_bulk_emails,
        user_ids
    )

    return {
        "message": f"Notifications queued for {len(user_ids)} users",
        "status": "accepted"
    }
```

---

## 8. 🎯 Decision Framework

Python

```
"""
THE COMPLETE DECISION FRAMEWORK

Step 1: Identify your bottleneck
  Is your code waiting? → I/O-bound
  Is your code computing? → CPU-bound
  Unsure? → Profile first!

Step 2: Choose the right tool

I/O-BOUND:
  Using async framework (FastAPI)?
    → Use async/await (asyncpg, httpx, aiofiles)
  Using sync framework (Django, Flask)?
    → Use ThreadPoolExecutor
  Need to call sync I/O from async code?
    → await asyncio.to_thread(sync_func)

CPU-BOUND:
  Need true parallelism?
    → ProcessPoolExecutor
  One-off heavy computation?
    → multiprocessing.Process
  Calling from async code?
    → await loop.run_in_executor(process_pool, func)
  Very quick computation (< 1ms)?
    → Run synchronously (overhead > benefit)

BOTH:
  Async for I/O parts
  ProcessPool for CPU parts
  Combine with run_in_executor

Step 3: Know the costs

THREADING:
  ✅ Shared memory (no serialization)
  ✅ Low overhead
  ✅ GIL released for I/O
  ❌ GIL prevents CPU parallelism
  ❌ Race conditions if not careful
  ❌ Hard to debug

MULTIPROCESSING:
  ✅ True CPU parallelism
  ✅ No GIL
  ✅ Process isolation (crash in one = others ok)
  ❌ High startup cost (~50-100ms per process)
  ❌ Serialization overhead (data must cross process boundary)
  ❌ No shared memory by default
  ❌ More complex IPC

ASYNC:
  ✅ Lowest overhead
  ✅ Handles thousands of concurrent I/O ops
  ✅ Single thread (no race conditions on state)
  ✅ Memory efficient
  ❌ Single thread (CPU work blocks everything)
  ❌ Requires async-compatible libraries
  ❌ "Async all the way down" requirement
  ❌ Harder to reason about execution order
"""

# Practical rules for backend engineers:
DECISION_RULES = {
    "Database queries (async driver)": "async/await",
    "Database queries (sync driver)": "ThreadPoolExecutor",
    "HTTP requests": "async/await with httpx",
    "File I/O": "async with aiofiles OR asyncio.to_thread",
    "Redis/Cache": "async/await with aioredis",
    "Image processing": "ProcessPoolExecutor",
    "Video encoding": "ProcessPoolExecutor + subprocess",
    "Data aggregation": "ProcessPoolExecutor",
    "PDF generation": "ProcessPoolExecutor OR subprocess",
    "Sending emails": "ThreadPoolExecutor OR Celery",
    "Background jobs": "Celery (separate process)",
    "WebSockets": "async/await",
    "Streaming": "async/await",
    "ML inference": "ProcessPoolExecutor OR dedicated service",
    "Encryption/hashing": "asyncio.to_thread (fast enough)",
}

print("\n=== Decision Guide ===")
for task, solution in DECISION_RULES.items():
    print(f"  {task:<35} → {solution}")
```

---

## 9. 🔬 Profiling to Find Your Bottleneck

Python

```
"""
Before choosing concurrency strategy, PROFILE.
Guessing wrong wastes time and creates complexity.
"""

import cProfile
import pstats
import io
import time
import asyncio
import tracemalloc


# ─────────────────────────────────────────
# CPU PROFILING
# ─────────────────────────────────────────
def profile_cpu(func, *args, **kwargs):
    """Profile CPU usage of a function."""
    profiler = cProfile.Profile()
    profiler.enable()

    result = func(*args, **kwargs)

    profiler.disable()

    # Print stats
    stream = io.StringIO()
    stats = pstats.Stats(profiler, stream=stream)
    stats.sort_stats("cumulative")
    stats.print_stats(10)   # top 10 functions

    print(stream.getvalue())
    return result


def example_function():
    """Function to profile."""
    total = 0
    for i in range(1_000_000):
        total += i * i
    return total


print("=== CPU Profile ===")
result = profile_cpu(example_function)


# ─────────────────────────────────────────
# MEMORY PROFILING
# ─────────────────────────────────────────
def profile_memory(func, *args, **kwargs):
    """Profile memory usage."""
    tracemalloc.start()

    result = func(*args, **kwargs)

    snapshot = tracemalloc.take_snapshot()
    tracemalloc.stop()

    top_stats = snapshot.statistics("lineno")
    print("Memory profile (top 3):")
    for stat in top_stats[:3]:
        print(f"  {stat}")

    return result


def memory_heavy():
    """Function that uses a lot of memory."""
    return [{"id": i, "data": "x" * 100} for i in range(10_000)]


print("\n=== Memory Profile ===")
profile_memory(memory_heavy)


# ─────────────────────────────────────────
# TIMING PROFILING
# ─────────────────────────────────────────
class Timer:
    """Context manager for timing code blocks."""

    def __init__(self, label: str):
        self.label = label

    def __enter__(self):
        self.start = time.perf_counter()
        return self

    def __exit__(self, *args):
        elapsed = (time.perf_counter() - self.start) * 1000
        print(f"  {self.label}: {elapsed:.2f}ms")


# Usage
print("\n=== Timing Profile ===")
with Timer("Database query simulation"):
    time.sleep(0.05)

with Timer("CPU computation"):
    sum(i * i for i in range(1_000_000))

with Timer("String building"):
    result = "".join(str(i) for i in range(100_000))


# ─────────────────────────────────────────
# IDENTIFY I/O vs CPU
# ─────────────────────────────────────────
def is_io_bound(func, *args, n_runs: int = 3) -> bool:
    """
    Simple heuristic: run in multiple threads.
    If threads speed it up → I/O bound (GIL released).
    If threads don't speed it up → CPU bound (GIL held).
    """
    from concurrent.futures import ThreadPoolExecutor

    # Single thread time
    start = time.perf_counter()
    for _ in range(n_runs):
        func(*args)
    single_time = time.perf_counter() - start

    # Multi-thread time
    start = time.perf_counter()
    with ThreadPoolExecutor(max_workers=n_runs) as pool:
        list(pool.map(func, [args] * n_runs))
    multi_time = time.perf_counter() - start

    speedup = single_time / multi_time
    print(f"  Speedup with {n_runs} threads: {speedup:.2f}x")

    if speedup > 1.5:
        print("  → I/O-bound (threading helps)")
        return True
    else:
        print("  → CPU-bound (threading doesn't help, use multiprocessing)")
        return False


print("\n=== I/O vs CPU Detection ===")
print("Testing I/O task:")
is_io_bound(time.sleep, 0.1)

print("Testing CPU task:")
is_io_bound(sum, range(1_000_000))
```

---

## 10. 🌟 Complete Real-World Example

Python

```
# A realistic FastAPI application using all three approaches
# appropriately based on task type

import asyncio
import time
from fastapi import FastAPI, BackgroundTasks, HTTPException
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor
from typing import List, Dict, Any
import os

app = FastAPI(title="Concurrency Patterns Demo")

# Initialize pools
process_pool = ProcessPoolExecutor(max_workers=os.cpu_count())
thread_pool = ThreadPoolExecutor(max_workers=20)


# ─── CPU-bound functions (run in process pool) ─────────────────
def generate_thumbnail(image_data: bytes, size: tuple) -> bytes:
    """CPU-intensive: resize image."""
    # Real: PIL.Image.open(io.BytesIO(image_data)).thumbnail(size)
    time.sleep(0.2)   # simulate CPU work
    return b"thumbnail_data"


def encrypt_file(data: bytes, key: bytes) -> bytes:
    """CPU-intensive: encrypt file."""
    # Real: use cryptography library
    time.sleep(0.1)
    return b"encrypted_data"


def generate_pdf_report(data: Dict) -> bytes:
    """CPU-intensive: generate PDF."""
    # Real: use reportlab or weasyprint
    time.sleep(0.3)
    return b"pdf_data"


# ─── Sync I/O functions (run in thread pool) ───────────────────
def send_email_sync(to: str, subject: str, body: str) -> bool:
    """Sync email sending (SMTP)."""
    time.sleep(0.5)   # simulate SMTP call
    print(f"Email sent to {to}: {subject}")
    return True


def call_legacy_api(endpoint: str, data: dict) -> dict:
    """Call a legacy sync-only API."""
    time.sleep(0.2)
    return {"status": "ok", "endpoint": endpoint}


# ─── Async functions (pure async I/O) ──────────────────────────
async def fetch_user_profile(user_id: int) -> dict:
    """Async DB query."""
    await asyncio.sleep(0.05)
    return {"id": user_id, "name": f"User {user_id}", "email": f"user{user_id}@test.com"}


async def fetch_user_permissions(user_id: int) -> List[str]:
    """Async permissions service call."""
    await asyncio.sleep(0.03)
    return ["read", "write", "comment"]


async def fetch_user_settings(user_id: int) -> dict:
    """Async settings from Redis."""
    await asyncio.sleep(0.02)
    return {"theme": "dark", "notifications": True}


# ─── Routes using the right approach for each task ─────────────

@app.get("/users/{user_id}/full-profile")
async def get_full_profile(user_id: int) -> dict:
    """
    Fetch all user data concurrently (async I/O).
    All three queries run simultaneously.
    """
    profile, permissions, settings = await asyncio.gather(
        fetch_user_profile(user_id),       # async DB
        fetch_user_permissions(user_id),    # async service
        fetch_user_settings(user_id),       # async Redis
    )

    return {
        "profile": profile,
        "permissions": permissions,
        "settings": settings
    }


@app.post("/images/process")
async def process_image(
    user_id: int,
    sizes: List[tuple] = [(200, 200), (800, 600), (1920, 1080)]
) -> dict:
    """
    Process image into multiple sizes (CPU-bound).
    Uses process pool to avoid blocking event loop.
    """
    loop = asyncio.get_event_loop()
    fake_image_data = b"fake_image_data" * 1000

    # Process all sizes in parallel using process pool
    tasks = [
        loop.run_in_executor(
            process_pool,
            generate_thumbnail,
            fake_image_data,
            size
        )
        for size in sizes
    ]

    thumbnails = await asyncio.gather(*tasks)

    return {
        "user_id": user_id,
        "thumbnails_created": len(thumbnails),
        "sizes": sizes
    }


@app.post("/reports/generate")
async def generate_report(
    user_id: int,
    background_tasks: BackgroundTasks
) -> dict:
    """
    Generate a complex report:
    1. Fetch data (async I/O)
    2. Process data (CPU — process pool)
    3. Send email (sync I/O — thread pool, background)
    """
    loop = asyncio.get_event_loop()

    # Step 1: Fetch data asynchronously
    user_data, permissions = await asyncio.gather(
        fetch_user_profile(user_id),
        fetch_user_permissions(user_id)
    )

    # Step 2: Generate PDF (CPU-bound, process pool)
    pdf_bytes = await loop.run_in_executor(
        process_pool,
        generate_pdf_report,
        {"user": user_data, "permissions": permissions}
    )

    # Step 3: Send email in background (sync, thread pool)
    async def send_report_email():
        await asyncio.to_thread(
            send_email_sync,
            user_data["email"],
            "Your Report",
            f"Report generated: {len(pdf_bytes)} bytes"
        )

    background_tasks.add_task(send_report_email)

    return {
        "status": "Report generated",
        "pdf_size": len(pdf_bytes),
        "email": "Sending in background"
    }


@app.get("/health/concurrency")
async def concurrency_health() -> dict:
    """
    Show current concurrency status.
    Useful for monitoring.
    """
    return {
        "process_pool_workers": process_pool._max_workers,
        "thread_pool_workers": thread_pool._max_workers,
        "event_loop": "running",
        "cpu_count": os.cpu_count()
    }


@app.on_event("shutdown")
async def cleanup():
    process_pool.shutdown(wait=False)
    thread_pool.shutdown(wait=False)
```

---

## 📊 Visual Summary

text

```
┌────────────────────────────────────────────────────────────────┐
│        THREADING vs MULTIPROCESSING vs ASYNC                   │
│                                                                │
│  THE GOLDEN RULE:                                              │
│  I/O-Bound → Async (best) or Threading (sync code)           │
│  CPU-Bound → Multiprocessing (only way to bypass GIL)         │
│                                                                │
│  THREADING:                                                    │
│  + Low overhead, shared memory, GIL released for I/O         │
│  - GIL prevents CPU parallelism, race conditions possible     │
│  Use: sync I/O, legacy sync code, parallel HTTP calls        │
│                                                                │
│  MULTIPROCESSING:                                              │
│  + True CPU parallelism, no GIL, process isolation           │
│  - High startup cost, serialization overhead, complex IPC     │
│  Use: image processing, PDF generation, ML inference          │
│       data aggregation, encryption, any CPU-intensive work    │
│                                                                │
│  ASYNC:                                                        │
│  + Lowest overhead, handles 10,000+ concurrent I/O ops       │
│  - Single thread: CPU work blocks EVERYTHING                  │
│  - Requires async libraries throughout                        │
│  Use: database queries, HTTP calls, WebSockets, streaming     │
│                                                                │
│  MIXING (Real-World Pattern):                                  │
│  Async endpoint                                                │
│    → await async_db_query()         ← pure async             │
│    → await asyncio.gather(q1, q2)   ← concurrent async       │
│    → await asyncio.to_thread(sync)  ← sync I/O in thread     │
│    → await loop.run_in_executor(    ← CPU in process         │
│           process_pool, cpu_func)                             │
│                                                                │
│  PROFILING FIRST:                                              │
│  Threads faster? → I/O-bound → use async or threading        │
│  Threads no help? → CPU-bound → use multiprocessing           │
└────────────────────────────────────────────────────────────────┘
```

---

## ✅ Knowledge Check

text

```
1.  What is the GIL? When does Python release it?
2.  Why doesn't threading help with CPU-bound work?
3.  When would you use threading over async?
4.  What is the startup cost of multiprocessing? Why?
5.  How do you pass data between processes?
6.  What is run_in_executor() and when do you use it?
7.  What is asyncio.to_thread() and when is it preferred?
8.  In FastAPI, when would you block the event loop?
    How do you avoid it?
9.  What is the difference between ThreadPoolExecutor
    and ProcessPoolExecutor?
10. How do you detect if your code is I/O or CPU bound?
11. What is shared memory and when would you use it
    with multiprocessing?
12. Why is multiprocessing worse than threading for I/O?
```

---

## 🛠️ Practice Exercises

Python

```
# Exercise 1: Benchmark Your Own Code
# Pick any function from your previous projects.
# Time it running:
#   a) Sequentially N times
#   b) With ThreadPoolExecutor N workers
#   c) With ProcessPoolExecutor N workers
#   d) With asyncio.gather (if async compatible)
# Determine: is it I/O or CPU bound?
# Choose the right concurrency approach for it.

# Exercise 2: Image Processing Service
# Build an endpoint: POST /process-images
# Accept: list of "image IDs" (simulate with integers)
# For each image:
#   a) Fetch metadata from "DB" (async, 50ms each)
#   b) Resize to 3 sizes (CPU, 100ms each)
#   c) Upload to "S3" (async, 30ms each)
# Optimize: fetch all metadata concurrently (async.gather)
#           resize all using ProcessPoolExecutor
#           upload all concurrently (async.gather)
# Measure total time vs sequential approach

# Exercise 3: Mixed Workload Server
# Build a FastAPI server with these endpoints:
# GET /hash/{text}        → bcrypt hash (CPU) → process pool
# GET /fetch/{url}        → HTTP fetch (I/O) → async httpx
# POST /send-email        → SMTP (sync I/O) → thread pool
# GET /aggregate/{n}      → sum 1..n (CPU) → process pool
# GET /multi-fetch        → fetch 5 URLs concurrently → async gather
# Each should be properly non-blocking.
# Add timing headers to each response.

# Exercise 4: Worker Pool Pattern
# Build a background worker system:
# - Task queue (asyncio.Queue)
# - 3 async workers consuming from the queue
# - Workers detect task type:
#     "io": handle with async sleep
#     "cpu": offload to process pool
# - Submit 20 tasks (mix of io and cpu)
# - Log: task submitted, started, completed with timing
# - Graceful shutdown: finish current tasks, then stop

# Exercise 5: Profiler Decorator
# Build a decorator @profile that:
# - Works on both sync and async functions
# - Measures: execution time, CPU time (time.process_time())
# - Detects if function is likely I/O or CPU bound
#   (if CPU time << wall time → I/O bound)
# - Logs results in structured format
# Apply it to 3 different functions from your codebase
```

---

## ✅ Phase 5.3 Complete!

**You now know:**

text

```
✅ The GIL — what it is, when it's released, its implications
✅ Threading — for I/O-bound work with sync code
✅ ThreadPoolExecutor — map, submit, as_completed
✅ Thread safety — locks, queue.Queue, race conditions
✅ Multiprocessing — for CPU-bound work, true parallelism
✅ ProcessPoolExecutor — the preferred multiprocessing API
✅ Inter-process communication — Queue, shared memory
✅ Async — single-threaded concurrent I/O
✅ Mixing approaches — run_in_executor, asyncio.to_thread
✅ FastAPI patterns — offloading CPU/sync to pools
✅ Profiling — finding your bottleneck before optimizing
✅ Decision framework — which tool for which job
```