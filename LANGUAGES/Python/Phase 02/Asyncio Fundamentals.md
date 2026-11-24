```
The problem with synchronous code in a backend:

Request 1: "Give me user data"
  → Server asks database (waits 50ms)
  → Server is BLOCKED, doing nothing
  → Request 2 has to WAIT in line

Request 1: ████████████████████░░░░░░░░░░░░████████  (blocked waiting for DB)
Request 2:                                          ████████  (only starts after 1 finishes)
Request 3:                                                    ████████  (waits even longer)

With async:
Request 1: ████░░░░░░░░░░░░████  (starts, waits for DB, resumes)
Request 2: ████░░░░░░████        (starts while 1 is waiting!)
Request 3: ████░░████             (starts while 1 and 2 are waiting!)

All three finish in ~same time as one synchronous request.

This is how FastAPI handles thousands of concurrent requests
with a single thread.
```

---

## Setup

Bash

```
cd ~/projects
mkdir async_learning
cd async_learning
python3 -m venv venv
source venv/bin/activate
pip install httpx aiofiles asyncpg fastapi uvicorn
touch async_demo.py
code .
```

---

## 1. 🧠 Sync vs Async — The Core Difference

Python

```
# async_demo.py
import time
import asyncio

# ─────────────────────────────────────────
# SYNCHRONOUS — blocks and waits
# ─────────────────────────────────────────
def sync_fetch_user(user_id: int) -> dict:
    """Simulate fetching user from database."""
    print(f"  [sync] Fetching user {user_id}...")
    time.sleep(1)    # BLOCKED — doing absolutely nothing for 1 second
    print(f"  [sync] Got user {user_id}")
    return {"id": user_id, "name": f"User {user_id}"}


def sync_main():
    """Fetch 3 users synchronously."""
    start = time.perf_counter()

    user1 = sync_fetch_user(1)    # waits 1 second
    user2 = sync_fetch_user(2)    # waits 1 second
    user3 = sync_fetch_user(3)    # waits 1 second

    total = time.perf_counter() - start
    print(f"Sync total: {total:.2f}s")  # ~3 seconds


# ─────────────────────────────────────────
# ASYNCHRONOUS — suspends and resumes
# ─────────────────────────────────────────
async def async_fetch_user(user_id: int) -> dict:
    """Async version — suspends while waiting, lets others run."""
    print(f"  [async] Starting fetch for user {user_id}...")
    await asyncio.sleep(1)   # SUSPENDS here, other tasks run
    print(f"  [async] Got user {user_id}")
    return {"id": user_id, "name": f"User {user_id}"}


async def async_main():
    """Fetch 3 users asynchronously — all at the same time!"""
    start = time.perf_counter()

    # Run all three CONCURRENTLY
    user1, user2, user3 = await asyncio.gather(
        async_fetch_user(1),   # all three start almost simultaneously
        async_fetch_user(2),
        async_fetch_user(3),
    )

    total = time.perf_counter() - start
    print(f"Async total: {total:.2f}s")  # ~1 second!


print("=== Synchronous (sequential) ===")
sync_main()

print("\n=== Asynchronous (concurrent) ===")
asyncio.run(async_main())

# Output:
# === Synchronous (sequential) ===
#   [sync] Fetching user 1...
#   [sync] Got user 1
#   [sync] Fetching user 2...
#   [sync] Got user 2
#   [sync] Fetching user 3...
#   [sync] Got user 3
# Sync total: 3.00s
#
# === Asynchronous (concurrent) ===
#   [async] Starting fetch for user 1...
#   [async] Starting fetch for user 2...
#   [async] Starting fetch for user 3...
#   [async] Got user 1
#   [async] Got user 2
#   [async] Got user 3
# Async total: 1.00s
```

---

## 2. 🔄 The Event Loop

Python

```
"""
THE EVENT LOOP — the heart of async Python

The event loop is a single loop that:
  1. Runs coroutines
  2. When a coroutine hits 'await', suspends it
  3. Runs other ready coroutines
  4. When the awaited thing completes, resumes the coroutine
  5. Repeat forever

Think of it like a chef in a restaurant:
  - Start cooking dish 1 (put it in oven)
  - While dish 1 is cooking (IO wait), start dish 2
  - While dish 2 is marinating, go back to dish 1
  - Never idle when there's work to do

The chef doesn't cook multiple dishes simultaneously —
they're just smart about not waiting idle.
"""

import asyncio


async def demonstrate_event_loop():
    """Show how the event loop manages tasks."""

    async def task(name: str, delay: float):
        print(f"Task '{name}' starting")
        await asyncio.sleep(delay)    # give up control here
        print(f"Task '{name}' finished after {delay}s")
        return name

    print("Event loop starts...")

    # Sequential — one at a time
    print("\n--- Sequential ---")
    start = time.perf_counter()
    await task("A", 0.3)
    await task("B", 0.3)
    await task("C", 0.3)
    print(f"Sequential: {time.perf_counter()-start:.2f}s")

    # Concurrent — all at once
    print("\n--- Concurrent (gather) ---")
    start = time.perf_counter()
    results = await asyncio.gather(
        task("A", 0.3),
        task("B", 0.3),
        task("C", 0.3)
    )
    print(f"Concurrent: {time.perf_counter()-start:.2f}s")
    print(f"Results: {results}")


asyncio.run(demonstrate_event_loop())
```

### The Event Loop — Visualized

Python

```
"""
WHAT HAPPENS INSIDE THE EVENT LOOP:

asyncio.gather(task_A, task_B, task_C)

Time 0.0s:
  Event loop creates 3 tasks: A, B, C
  
Time 0.0s+:
  Starts task A → runs until await asyncio.sleep(0.3) → SUSPENDS task A
  Starts task B → runs until await asyncio.sleep(0.3) → SUSPENDS task B
  Starts task C → runs until await asyncio.sleep(0.3) → SUSPENDS task C

Time 0.3s:
  Sleep timer fires for all three
  Event loop wakes up A → runs to completion
  Event loop wakes up B → runs to completion
  Event loop wakes up C → runs to completion

Total: 0.3s (not 0.9s)

The key: while A is "sleeping" (waiting for IO),
         the event loop runs B and C.
         Nobody is blocked — everyone is waiting efficiently.
"""


async def show_execution_order():
    """Demonstrate exact execution order."""

    events = []

    async def step_one():
        events.append("1: before await")
        await asyncio.sleep(0.1)       # YIELDS control here
        events.append("1: after await")
        return "one"

    async def step_two():
        events.append("2: before await")
        await asyncio.sleep(0.1)       # YIELDS control here
        events.append("2: after await")
        return "two"

    await asyncio.gather(step_one(), step_two())

    print("Execution order:")
    for event in events:
        print(f"  {event}")

    # Output:
    # 1: before await   ← task 1 starts
    # 2: before await   ← task 2 starts (while task 1 is sleeping!)
    # 1: after await    ← task 1 resumes
    # 2: after await    ← task 2 resumes


asyncio.run(show_execution_order())
```

---

## 3. ⚡ Coroutines — `async def` and `await`

Python

```
"""
COROUTINE = a function that can be paused and resumed.

Regular function:
  - Starts, runs to completion, returns
  - Cannot be paused

Coroutine (async def):
  - Starts, can pause at await points
  - Another coroutine runs while this one is paused
  - Resumes when awaited thing is done
  - Returns when done

KEY RULES:
  1. async def → creates a coroutine function
  2. await     → pause here, wait for this coroutine/future
  3. await can ONLY be used inside async def
  4. Calling async_func() returns a coroutine object (not the result!)
  5. Must await it to actually run it
"""

import asyncio


# ─────────────────────────────────────────
# BASIC COROUTINE
# ─────────────────────────────────────────
async def greet(name: str) -> str:
    """A simple coroutine."""
    await asyncio.sleep(0)   # yield control (0 = no actual wait)
    return f"Hello, {name}!"


async def basic_coroutine_demo():
    # WRONG — just creates coroutine object, doesn't run it
    result_obj = greet("Alice")
    print(type(result_obj))         # <class 'coroutine'>
    print(result_obj)               # <coroutine object greet at 0x...>

    # Must close unused coroutines to avoid ResourceWarning
    result_obj.close()

    # RIGHT — await to actually execute
    result = await greet("Alice")
    print(result)                   # Hello, Alice!


asyncio.run(basic_coroutine_demo())


# ─────────────────────────────────────────
# COROUTINE CHAIN
# ─────────────────────────────────────────
async def fetch_user_id(email: str) -> int:
    """Step 1: Get user ID by email."""
    await asyncio.sleep(0.1)  # simulate DB query
    return 42


async def fetch_user_data(user_id: int) -> dict:
    """Step 2: Get user data by ID."""
    await asyncio.sleep(0.1)  # simulate DB query
    return {"id": user_id, "name": "Alice", "email": "alice@test.com"}


async def fetch_user_posts(user_id: int) -> list:
    """Step 3: Get user's posts."""
    await asyncio.sleep(0.1)  # simulate DB query
    return [{"id": 1, "title": "Post 1"}, {"id": 2, "title": "Post 2"}]


async def get_user_profile(email: str) -> dict:
    """Chain multiple coroutines together."""
    # These must be sequential (each depends on previous)
    user_id = await fetch_user_id(email)           # wait for ID first
    user_data = await fetch_user_data(user_id)     # then get data

    # But posts can be fetched simultaneously with user data
    # if we restructure:
    user_data, posts = await asyncio.gather(
        fetch_user_data(user_id),
        fetch_user_posts(user_id)   # fetch in parallel!
    )

    return {**user_data, "posts": posts}


import time
start = time.perf_counter()
profile = asyncio.run(get_user_profile("alice@test.com"))
print(f"Profile fetched in {(time.perf_counter()-start)*1000:.0f}ms")
print(f"Posts: {len(profile['posts'])}")
```

---

## 4. 📦 Tasks — Concurrent Execution

Python

```
"""
Task = a coroutine scheduled to run in the event loop.

asyncio.create_task() starts a coroutine as a task immediately.
The task runs concurrently with the current coroutine.

Difference:
  await coroutine()  → runs it NOW, wait for completion
  task = asyncio.create_task(coroutine())  → schedule it, runs when you yield
  await task         → wait for the task you already started
"""

import asyncio
import time


async def background_work(name: str, duration: float) -> str:
    """Simulates background work."""
    print(f"  [{name}] started")
    await asyncio.sleep(duration)
    print(f"  [{name}] completed ({duration}s)")
    return f"{name} result"


async def task_demo():
    """Demonstrate tasks vs coroutines."""

    # ─────────────────────────────────────────
    # gather() — run coroutines concurrently
    # ─────────────────────────────────────────
    print("\n1. gather() - run and wait for all:")
    start = time.perf_counter()

    results = await asyncio.gather(
        background_work("DB query", 0.3),
        background_work("Cache lookup", 0.1),
        background_work("External API", 0.5),
    )
    print(f"   All done in {(time.perf_counter()-start):.2f}s")
    print(f"   Results: {results}")

    # ─────────────────────────────────────────
    # create_task() — fire and forget (mostly)
    # ─────────────────────────────────────────
    print("\n2. create_task() - start immediately:")
    start = time.perf_counter()

    # Create tasks — they START running immediately
    task1 = asyncio.create_task(
        background_work("Task 1", 0.3),
        name="task_one"           # optional name for debugging
    )
    task2 = asyncio.create_task(
        background_work("Task 2", 0.2),
        name="task_two"
    )

    print("   Tasks created and running!")
    print("   Doing other work while tasks run...")
    await asyncio.sleep(0.1)     # other work here
    print("   Still doing other work...")

    # Now wait for tasks to complete
    result1 = await task1
    result2 = await task2
    print(f"   Both tasks done in {(time.perf_counter()-start):.2f}s")

    # ─────────────────────────────────────────
    # wait() — more control than gather()
    # ─────────────────────────────────────────
    print("\n3. wait() - with timeout:")
    tasks = {
        asyncio.create_task(background_work("Fast", 0.1)),
        asyncio.create_task(background_work("Slow", 1.0)),
    }

    # Wait with timeout
    done, pending = await asyncio.wait(
        tasks,
        timeout=0.5   # cancel after 0.5s
    )

    print(f"   Completed: {len(done)}, Still running: {len(pending)}")

    # Cancel pending tasks
    for task in pending:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            print(f"   Cancelled pending task")


asyncio.run(task_demo())


# ─────────────────────────────────────────
# Task Groups (Python 3.11+) — preferred way
# ─────────────────────────────────────────
async def task_group_demo():
    """
    TaskGroup is the modern way (Python 3.11+).
    Better error handling than gather().
    """
    print("\nTaskGroup (Python 3.11+):")
    start = time.perf_counter()

    async with asyncio.TaskGroup() as tg:
        # All tasks start immediately
        task1 = tg.create_task(background_work("API call", 0.3))
        task2 = tg.create_task(background_work("DB query", 0.2))
        task3 = tg.create_task(background_work("Cache", 0.1))
    # TaskGroup waits for ALL tasks to complete

    print(f"All done: {task1.result()}, {task2.result()}, {task3.result()}")
    print(f"Total: {(time.perf_counter()-start):.2f}s")


asyncio.run(task_group_demo())
```

---

## 5. ⚠️ Error Handling in Async Code

Python

```
import asyncio


async def might_fail(name: str, should_fail: bool = False) -> str:
    await asyncio.sleep(0.1)
    if should_fail:
        raise ValueError(f"{name} failed!")
    return f"{name} succeeded"


async def error_handling_demo():
    """Different ways to handle errors in async code."""

    # ─────────────────────────────────────────
    # 1. gather() — default: one fails = all fail
    # ─────────────────────────────────────────
    print("1. gather() - one fails = exception raised:")
    try:
        results = await asyncio.gather(
            might_fail("Task A"),
            might_fail("Task B", should_fail=True),   # this fails
            might_fail("Task C"),
        )
    except ValueError as e:
        print(f"   Exception: {e}")
        # Note: Task A and C may have completed but results are lost

    # ─────────────────────────────────────────
    # 2. gather(return_exceptions=True) — collect errors
    # ─────────────────────────────────────────
    print("\n2. gather(return_exceptions=True):")
    results = await asyncio.gather(
        might_fail("Task A"),
        might_fail("Task B", should_fail=True),
        might_fail("Task C"),
        return_exceptions=True    # return exceptions as values
    )
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"   Task {i}: FAILED — {result}")
        else:
            print(f"   Task {i}: OK — {result}")

    # ─────────────────────────────────────────
    # 3. Individual try/except in each task
    # ─────────────────────────────────────────
    print("\n3. Per-task error handling:")

    async def safe_task(name: str, should_fail: bool = False):
        try:
            result = await might_fail(name, should_fail)
            return {"status": "ok", "result": result}
        except Exception as e:
            return {"status": "error", "error": str(e)}

    results = await asyncio.gather(
        safe_task("Task A"),
        safe_task("Task B", should_fail=True),
        safe_task("Task C"),
    )
    for r in results:
        print(f"   {r}")

    # ─────────────────────────────────────────
    # 4. TaskGroup — any failure cancels all
    # ─────────────────────────────────────────
    print("\n4. TaskGroup — strict error handling:")
    try:
        async with asyncio.TaskGroup() as tg:
            task_a = tg.create_task(might_fail("Task A"))
            task_b = tg.create_task(might_fail("Task B", should_fail=True))
            task_c = tg.create_task(might_fail("Task C"))
        # If any task fails, TaskGroup cancels the others
        # and raises ExceptionGroup
    except* ValueError as eg:
        print(f"   TaskGroup caught: {eg.exceptions}")

    # ─────────────────────────────────────────
    # 5. Cancellation
    # ─────────────────────────────────────────
    print("\n5. Cancellation:")

    async def long_running():
        try:
            print("   Long task: starting...")
            await asyncio.sleep(10)     # simulating long work
            print("   Long task: completed!")
        except asyncio.CancelledError:
            print("   Long task: was cancelled! Cleaning up...")
            # Always re-raise CancelledError after cleanup!
            raise

    task = asyncio.create_task(long_running())

    await asyncio.sleep(0.2)   # let it start
    task.cancel()              # cancel it

    try:
        await task
    except asyncio.CancelledError:
        print("   Main: confirmed task was cancelled")


asyncio.run(error_handling_demo())
```

---

## 6. ⏰ Timeouts

Python

```
import asyncio
import time


async def slow_operation(name: str, duration: float) -> str:
    """Simulates a slow external call."""
    await asyncio.sleep(duration)
    return f"{name} completed"


async def timeout_demo():
    """Different ways to handle timeouts."""

    # ─────────────────────────────────────────
    # asyncio.wait_for() — timeout a single operation
    # ─────────────────────────────────────────
    print("1. wait_for() timeout:")
    try:
        result = await asyncio.wait_for(
            slow_operation("DB query", duration=2.0),
            timeout=0.5    # cancel if takes more than 0.5s
        )
        print(f"   Result: {result}")
    except asyncio.TimeoutError:
        print("   Timed out after 0.5s!")

    # ─────────────────────────────────────────
    # asyncio.timeout() context manager (Python 3.11+)
    # ─────────────────────────────────────────
    print("\n2. timeout() context manager:")
    try:
        async with asyncio.timeout(0.5):
            result = await slow_operation("API call", 2.0)
    except asyncio.TimeoutError:
        print("   Timed out!")

    # ─────────────────────────────────────────
    # Retry with timeout — real pattern
    # ─────────────────────────────────────────
    print("\n3. Retry with exponential backoff:")

    async def fetch_with_retry(
        url: str,
        max_attempts: int = 3,
        base_delay: float = 0.1,
        timeout: float = 1.0
    ) -> str:
        """Fetch URL with retry and exponential backoff."""
        last_error = None

        for attempt in range(1, max_attempts + 1):
            try:
                print(f"   Attempt {attempt}/{max_attempts}")
                async with asyncio.timeout(timeout):
                    # Simulate sometimes-failing request
                    if attempt < 3:
                        raise ConnectionError(f"Connection failed (attempt {attempt})")
                    await asyncio.sleep(0.1)  # successful request
                    return f"Response from {url}"

            except asyncio.TimeoutError:
                last_error = f"Timeout after {timeout}s"
                print(f"   Attempt {attempt} timed out")

            except ConnectionError as e:
                last_error = str(e)
                print(f"   Attempt {attempt} failed: {e}")

            if attempt < max_attempts:
                delay = base_delay * (2 ** (attempt - 1))  # exponential
                print(f"   Waiting {delay}s before retry...")
                await asyncio.sleep(delay)

        raise ConnectionError(
            f"All {max_attempts} attempts failed. Last error: {last_error}"
        )

    try:
        result = await fetch_with_retry("https://api.example.com/data")
        print(f"   Success: {result}")
    except ConnectionError as e:
        print(f"   Final failure: {e}")


asyncio.run(timeout_demo())
```

---

## 7. 🔒 Synchronization Primitives

Python

```
"""
Even with async (single-threaded), you can have race conditions
when multiple coroutines share mutable state.

asyncio provides synchronization primitives:
  Lock      → mutual exclusion (only one at a time)
  Event     → signal between coroutines
  Semaphore → limit concurrent access to N
  Queue     → pass data between coroutines
"""

import asyncio
import time
from typing import List


# ─────────────────────────────────────────
# LOCK — only one coroutine at a time
# ─────────────────────────────────────────
async def lock_demo():
    """Demonstrate asyncio.Lock for mutual exclusion."""
    shared_counter = 0
    lock = asyncio.Lock()

    async def increment_without_lock(name: str):
        nonlocal shared_counter
        # Race condition: read, sleep, write — another coroutine
        # might change the value between read and write
        current = shared_counter
        await asyncio.sleep(0.01)  # yield — another coroutine runs here!
        shared_counter = current + 1
        print(f"  {name}: counter = {shared_counter}")

    async def increment_with_lock(name: str):
        nonlocal shared_counter
        async with lock:           # only ONE coroutine in here at a time
            current = shared_counter
            await asyncio.sleep(0.01)
            shared_counter = current + 1
            print(f"  {name}: counter = {shared_counter}")

    # Without lock — race condition
    shared_counter = 0
    print("Without lock:")
    await asyncio.gather(*[
        increment_without_lock(f"Task-{i}")
        for i in range(5)
    ])
    print(f"Expected: 5, Got: {shared_counter}")

    # With lock — safe
    shared_counter = 0
    print("\nWith lock:")
    await asyncio.gather(*[
        increment_with_lock(f"Task-{i}")
        for i in range(5)
    ])
    print(f"Expected: 5, Got: {shared_counter}")


# ─────────────────────────────────────────
# SEMAPHORE — limit concurrent access
# ─────────────────────────────────────────
async def semaphore_demo():
    """
    Semaphore: allow at most N coroutines to run a section simultaneously.

    Real use case:
      - Limit concurrent database connections
      - Limit concurrent HTTP requests to external API
      - Rate limiting your own code
    """
    semaphore = asyncio.Semaphore(3)   # max 3 concurrent

    async def api_call(call_id: int) -> str:
        async with semaphore:           # blocks if 3 already running
            print(f"  API call {call_id} started")
            await asyncio.sleep(0.3)   # simulate API call
            print(f"  API call {call_id} done")
            return f"response_{call_id}"

    # 10 requests but only 3 at a time
    print("Semaphore (max 3 concurrent):")
    start = time.perf_counter()
    results = await asyncio.gather(*[api_call(i) for i in range(10)])
    print(f"10 API calls done in {time.perf_counter()-start:.2f}s")
    # ~1.0s (10/3 = ~3.33 rounds × 0.3s)


# ─────────────────────────────────────────
# EVENT — signal between coroutines
# ─────────────────────────────────────────
async def event_demo():
    """
    Event: one coroutine signals others that something happened.
    """
    event = asyncio.Event()
    results = []

    async def wait_for_signal(name: str):
        print(f"  {name}: waiting for signal...")
        await event.wait()              # blocks until event is set
        print(f"  {name}: got the signal!")
        results.append(name)

    async def send_signal():
        print("  Preparing data...")
        await asyncio.sleep(0.5)        # do some work
        print("  Data ready! Setting event.")
        event.set()                     # wake up all waiters

    await asyncio.gather(
        wait_for_signal("Worker 1"),
        wait_for_signal("Worker 2"),
        wait_for_signal("Worker 3"),
        send_signal(),
    )
    print(f"  Workers notified: {results}")


# ─────────────────────────────────────────
# QUEUE — producer-consumer pattern
# ─────────────────────────────────────────
async def queue_demo():
    """
    Queue: safe data passing between coroutines.
    Classic producer-consumer pattern.

    Real use: async job queue, data pipeline, batch processing
    """
    queue = asyncio.Queue(maxsize=5)  # buffer of 5 items
    results = []

    async def producer(items: List[str]):
        """Puts items into the queue."""
        for item in items:
            await queue.put(item)          # blocks if queue is full
            print(f"  Produced: {item}")
            await asyncio.sleep(0.1)       # simulates creation time

        # Signal that production is done
        await queue.put(None)              # sentinel value

    async def consumer(name: str):
        """Takes items from the queue."""
        while True:
            item = await queue.get()       # blocks until item available
            if item is None:
                queue.task_done()
                break                      # sentinel received, stop

            print(f"  {name} consuming: {item}")
            await asyncio.sleep(0.2)       # simulates processing time
            results.append(f"{name}:{item}")
            queue.task_done()

    items = [f"job_{i}" for i in range(5)]

    await asyncio.gather(
        producer(items),
        consumer("Worker-1"),
    )

    print(f"  Processed: {results}")


print("=== Lock Demo ===")
asyncio.run(lock_demo())

print("\n=== Semaphore Demo ===")
asyncio.run(semaphore_demo())

print("\n=== Event Demo ===")
asyncio.run(event_demo())

print("\n=== Queue Demo ===")
asyncio.run(queue_demo())
```

---

## 8. 🌊 Async Context Managers & Iterators

Python

```
import asyncio
from typing import AsyncGenerator, AsyncIterator


# ─────────────────────────────────────────
# ASYNC CONTEXT MANAGER
# ─────────────────────────────────────────
class AsyncDatabaseConnection:
    """
    Async context manager for database connections.
    Used as: async with AsyncDatabaseConnection() as conn:
    """

    def __init__(self, url: str):
        self.url = url
        self.connection = None

    async def __aenter__(self):
        """Setup: async connection establishment."""
        print(f"Connecting to {self.url}...")
        await asyncio.sleep(0.1)   # simulate async connect
        self.connection = {"url": self.url, "connected": True}
        print("Connected!")
        return self.connection

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Teardown: always runs, even on exception."""
        print("Closing connection...")
        await asyncio.sleep(0.05)  # simulate async disconnect
        self.connection = None
        print("Connection closed.")

        if exc_type:
            print(f"Exception occurred: {exc_val}")
        return False  # don't suppress exceptions


# Using @asynccontextmanager (easier to write)
from contextlib import asynccontextmanager


@asynccontextmanager
async def managed_http_client(base_url: str):
    """Create and manage an HTTP client."""
    import httpx
    client = httpx.AsyncClient(base_url=base_url, timeout=10.0)
    print(f"HTTP client created for {base_url}")
    try:
        yield client
    finally:
        await client.aclose()
        print("HTTP client closed")


async def context_manager_demo():
    # Class-based
    async with AsyncDatabaseConnection("postgresql://localhost/mydb") as conn:
        print(f"Using connection: {conn}")
        await asyncio.sleep(0.1)   # do work
        print("Work done!")

    # Decorator-based
    async with managed_http_client("https://api.github.com") as client:
        # In real code: response = await client.get("/users/octocat")
        print("HTTP client ready to use")


# ─────────────────────────────────────────
# ASYNC ITERATORS
# ─────────────────────────────────────────
class AsyncDatabaseCursor:
    """
    Async iterator that fetches database rows one at a time.
    Memory efficient for large result sets.
    """

    def __init__(self, query: str, total_rows: int = 10):
        self.query = query
        self.total_rows = total_rows
        self._current = 0

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self._current >= self.total_rows:
            raise StopAsyncIteration   # done iterating

        await asyncio.sleep(0.01)   # simulate fetching each row
        row = {
            "id": self._current + 1,
            "name": f"User {self._current + 1}"
        }
        self._current += 1
        return row


# Async generator (easier to write)
async def stream_large_dataset(
    batch_size: int = 100,
    total: int = 1000
) -> AsyncGenerator[dict, None]:
    """
    Stream large dataset in batches.
    Never loads everything into memory.
    """
    for offset in range(0, total, batch_size):
        # Simulate fetching a batch from database
        await asyncio.sleep(0.05)
        batch = [
            {"id": i, "data": f"record_{i}"}
            for i in range(offset, min(offset + batch_size, total))
        ]
        for record in batch:
            yield record   # yield one record at a time


async def iterator_demo():
    print("=== Async Iterator ===")
    cursor = AsyncDatabaseCursor("SELECT * FROM users", total_rows=5)
    async for row in cursor:
        print(f"  Row: {row}")

    print("\n=== Async Generator (streaming) ===")
    count = 0
    async for record in stream_large_dataset(batch_size=50, total=200):
        count += 1
        # Process one record at a time — constant memory usage!

    print(f"  Processed {count} records with constant memory")


asyncio.run(context_manager_demo())
asyncio.run(iterator_demo())
```

---

## 9. 🚀 Real-World Async Patterns

Python

```
import asyncio
import time
from typing import List, Dict, Any, Optional


# ─────────────────────────────────────────
# PATTERN 1: Async Database Operations
# ─────────────────────────────────────────
class AsyncRepository:
    """
    Demonstrates async database access pattern.
    In real code: use asyncpg or SQLAlchemy async.
    """

    def __init__(self, db_pool):
        self.pool = db_pool

    async def get_user(self, user_id: int) -> Optional[Dict]:
        async with self.pool.acquire() as conn:       # get connection from pool
            row = await conn.fetchrow(                 # async query
                "SELECT * FROM users WHERE id = $1",
                user_id
            )
            return dict(row) if row else None

    async def get_users_batch(self, user_ids: List[int]) -> List[Dict]:
        """Fetch multiple users concurrently."""
        async with self.pool.acquire() as conn:
            rows = await conn.fetch(
                "SELECT * FROM users WHERE id = ANY($1)",
                user_ids
            )
            return [dict(row) for row in rows]


# ─────────────────────────────────────────
# PATTERN 2: Concurrent Data Fetching
# ─────────────────────────────────────────
async def get_dashboard_data(user_id: int) -> Dict[str, Any]:
    """
    Fetch all dashboard data concurrently.
    Without async: ~0.5s + 0.3s + 0.4s + 0.2s = 1.4s
    With async: max(0.5, 0.3, 0.4, 0.2) = 0.5s
    """

    async def get_user_profile(uid: int) -> dict:
        await asyncio.sleep(0.5)    # DB query
        return {"id": uid, "name": "Alice", "email": "alice@test.com"}

    async def get_user_posts(uid: int) -> list:
        await asyncio.sleep(0.3)    # DB query
        return [{"id": i, "title": f"Post {i}"} for i in range(3)]

    async def get_user_stats(uid: int) -> dict:
        await asyncio.sleep(0.4)    # Computed query
        return {"followers": 120, "following": 85, "post_count": 15}

    async def get_notifications(uid: int) -> list:
        await asyncio.sleep(0.2)    # DB query
        return [{"id": 1, "message": "New follower"}]

    # Fetch ALL concurrently — takes time of slowest, not sum
    start = time.perf_counter()
    profile, posts, stats, notifications = await asyncio.gather(
        get_user_profile(user_id),
        get_user_posts(user_id),
        get_user_stats(user_id),
        get_notifications(user_id),
    )
    elapsed = (time.perf_counter() - start) * 1000
    print(f"Dashboard fetched in {elapsed:.0f}ms (would be ~1400ms sync)")

    return {
        "profile": profile,
        "posts": posts,
        "stats": stats,
        "notifications": notifications,
    }


# ─────────────────────────────────────────
# PATTERN 3: Rate-Limited Async Requests
# ─────────────────────────────────────────
async def fetch_with_rate_limit(
    urls: List[str],
    max_concurrent: int = 10,
    delay_between: float = 0.0
) -> List[Dict]:
    """
    Fetch multiple URLs with rate limiting.
    Uses Semaphore to limit concurrent requests.
    """
    semaphore = asyncio.Semaphore(max_concurrent)
    results = []

    async def fetch_one(url: str) -> Dict:
        async with semaphore:
            try:
                # Simulate HTTP request
                await asyncio.sleep(0.1)
                if delay_between:
                    await asyncio.sleep(delay_between)
                return {"url": url, "status": 200, "data": f"content of {url}"}
            except Exception as e:
                return {"url": url, "status": 500, "error": str(e)}

    return await asyncio.gather(*[fetch_one(url) for url in urls])


# ─────────────────────────────────────────
# PATTERN 4: Background Task Queue
# ─────────────────────────────────────────
class AsyncTaskQueue:
    """
    Simple async background task queue.
    For production: use Celery or arq.
    """

    def __init__(self, num_workers: int = 3):
        self.queue = asyncio.Queue()
        self.num_workers = num_workers
        self.running = False
        self._workers: List[asyncio.Task] = []

    async def submit(self, coro) -> asyncio.Task:
        """Submit a coroutine for background execution."""
        task = asyncio.create_task(coro)
        await self.queue.put(task)
        return task

    async def worker(self, worker_id: int):
        """Process tasks from the queue."""
        print(f"Worker {worker_id} started")
        while self.running:
            try:
                task = await asyncio.wait_for(
                    self.queue.get(),
                    timeout=1.0
                )
                print(f"Worker {worker_id} processing task")
                await task
                self.queue.task_done()
            except asyncio.TimeoutError:
                continue   # no task, keep waiting

    async def start(self):
        """Start worker coroutines."""
        self.running = True
        self._workers = [
            asyncio.create_task(self.worker(i))
            for i in range(self.num_workers)
        ]

    async def stop(self):
        """Gracefully stop the queue."""
        await self.queue.join()   # wait for all tasks to complete
        self.running = False
        for worker in self._workers:
            worker.cancel()


# ─────────────────────────────────────────
# PATTERN 5: Async Cache with Lock
# ─────────────────────────────────────────
class AsyncCache:
    """
    Thread-safe async cache.
    Prevents cache stampede: multiple coroutines
    computing the same thing simultaneously.
    """

    def __init__(self):
        self._cache: Dict[str, Any] = {}
        self._locks: Dict[str, asyncio.Lock] = {}
        self._meta_lock = asyncio.Lock()

    async def get_lock(self, key: str) -> asyncio.Lock:
        """Get or create a lock for a specific key."""
        async with self._meta_lock:
            if key not in self._locks:
                self._locks[key] = asyncio.Lock()
            return self._locks[key]

    async def get_or_set(
        self,
        key: str,
        factory,        # async callable
        ttl: float = 300
    ) -> Any:
        """
        Get from cache or compute and store.
        Uses per-key locking to prevent cache stampede.
        """
        if key in self._cache:
            return self._cache[key]   # cache hit

        lock = await self.get_lock(key)

        async with lock:
            # Double-check after acquiring lock
            if key in self._cache:
                return self._cache[key]  # another coroutine computed it

            # Compute the value
            print(f"Cache miss for '{key}', computing...")
            value = await factory()
            self._cache[key] = value

            # Schedule cache expiry
            async def expire():
                await asyncio.sleep(ttl)
                self._cache.pop(key, None)
            asyncio.create_task(expire())

            return value


# Test all patterns
async def test_patterns():
    print("=== Concurrent Dashboard Fetch ===")
    data = await get_dashboard_data(42)
    print(f"  Got: profile, {len(data['posts'])} posts, stats, notifications")

    print("\n=== Rate Limited Fetching ===")
    urls = [f"https://api.example.com/item/{i}" for i in range(20)]
    start = time.perf_counter()
    results = await fetch_with_rate_limit(urls, max_concurrent=5)
    print(f"  Fetched {len(results)} URLs in {(time.perf_counter()-start):.2f}s")

    print("\n=== Async Cache ===")
    cache = AsyncCache()

    async def expensive_db_query():
        await asyncio.sleep(0.5)   # simulate slow query
        return {"users": 100, "posts": 500}

    # Simulate multiple coroutines requesting same data
    results = await asyncio.gather(*[
        cache.get_or_set("stats", expensive_db_query)
        for _ in range(5)   # 5 simultaneous requests
    ])
    print(f"  All got same value: {all(r == results[0] for r in results)}")
    print(f"  (but computed only ONCE)")


asyncio.run(test_patterns())
```

---

## 10. 🔬 Async with FastAPI (Preview)

Python

```
# Showing how asyncio connects to what you've already built

from fastapi import FastAPI
import asyncio
import time

app = FastAPI()


# ─────────────────────────────────────────
# SYNC vs ASYNC endpoints in FastAPI
# ─────────────────────────────────────────

# Sync endpoint — FastAPI runs it in a thread pool
# Use for: CPU-heavy work, code you can't make async
@app.get("/sync")
def sync_endpoint():
    time.sleep(0.1)    # blocks the thread (OK in thread pool)
    return {"type": "sync"}


# Async endpoint — FastAPI runs it in event loop
# Use for: I/O-heavy work (DB, HTTP calls, file operations)
@app.get("/async")
async def async_endpoint():
    await asyncio.sleep(0.1)   # yields control (other requests run!)
    return {"type": "async"}


# IMPORTANT: Never use time.sleep() in async endpoints!
# It BLOCKS the entire event loop — no other requests run.
@app.get("/wrong")
async def wrong_endpoint():
    time.sleep(1)    # ← WRONG! Blocks ALL requests for 1 second!
    return {"bad": "practice"}


@app.get("/right")
async def right_endpoint():
    await asyncio.sleep(1)    # ← RIGHT! Other requests can run
    return {"good": "practice"}


# ─────────────────────────────────────────
# CONCURRENT DATA FETCHING IN A ROUTE
# ─────────────────────────────────────────
@app.get("/users/{user_id}/dashboard")
async def get_dashboard(user_id: int):
    """
    Fetch all data concurrently — much faster than sequential.
    """

    async def get_profile():
        await asyncio.sleep(0.1)   # DB query
        return {"name": "Alice", "email": "alice@test.com"}

    async def get_posts():
        await asyncio.sleep(0.15)  # DB query
        return [{"title": "Post 1"}, {"title": "Post 2"}]

    async def get_notifications():
        await asyncio.sleep(0.05)  # DB query
        return [{"msg": "New follower"}]

    # Concurrent: takes max(0.1, 0.15, 0.05) = 0.15s
    # Sequential: would take 0.1 + 0.15 + 0.05 = 0.30s
    profile, posts, notifications = await asyncio.gather(
        get_profile(),
        get_posts(),
        get_notifications()
    )

    return {
        "user_id": user_id,
        "profile": profile,
        "posts": posts,
        "notifications": notifications
    }


# ─────────────────────────────────────────
# BACKGROUND TASKS in FastAPI
# ─────────────────────────────────────────
from fastapi import BackgroundTasks


@app.post("/users/register")
async def register(background_tasks: BackgroundTasks):
    """Register user and send welcome email in background."""

    # Process registration (fast)
    user = {"id": 42, "email": "alice@test.com"}

    # Schedule background task (runs AFTER response is sent)
    async def send_welcome_email(email: str):
        await asyncio.sleep(1)    # slow email sending
        print(f"Welcome email sent to {email}")

    background_tasks.add_task(send_welcome_email, user["email"])

    # Response sent immediately — email sends after
    return {"message": "Registered!", "user": user}
```

---

## 11. ⚡ Async vs Sync — When to Use Which

Python

```
"""
DECISION GUIDE: When to use async?

USE ASYNC WHEN:
  ✅ Making HTTP requests to external APIs
  ✅ Database queries (with async driver)
  ✅ File I/O operations
  ✅ Multiple I/O operations that can run concurrently
  ✅ WebSocket connections
  ✅ Server-sent events
  ✅ Long-polling endpoints
  ✅ Any operation where you're waiting for something external

USE SYNC WHEN:
  ✅ CPU-intensive computation (async doesn't help!)
  ✅ Working with sync-only libraries
  ✅ Simple sequential scripts
  ✅ Tests (mostly — pytest-asyncio for async tests)

THE GOLDEN RULES:

1. In async code: NEVER use blocking calls
   ❌ time.sleep()           → use await asyncio.sleep()
   ❌ requests.get()         → use await httpx.AsyncClient.get()
   ❌ open() + read()        → use aiofiles.open()
   ❌ psycopg2 queries       → use asyncpg or SQLAlchemy async

2. CPU-bound work blocks the event loop too
   ❌ for i in range(10**9): → use run_in_executor()
   
3. Use run_in_executor() for sync code in async context
"""

import asyncio
import time
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor


async def mixing_sync_and_async():
    """How to use sync code safely in async context."""

    # ─────────────────────────────────────────
    # Run sync I/O in thread pool
    # ─────────────────────────────────────────
    loop = asyncio.get_event_loop()

    def blocking_io():
        """A blocking operation we can't make async."""
        time.sleep(0.5)   # could be any blocking operation
        return "sync result"

    # Run in thread pool — doesn't block event loop!
    result = await loop.run_in_executor(None, blocking_io)
    print(f"Sync result (non-blocking): {result}")

    # With custom thread pool
    with ThreadPoolExecutor(max_workers=5) as pool:
        result = await loop.run_in_executor(pool, blocking_io)
        print(f"Custom pool result: {result}")

    # ─────────────────────────────────────────
    # Run CPU-bound work in process pool
    # ─────────────────────────────────────────
    def cpu_heavy(n: int) -> int:
        """CPU-bound computation."""
        return sum(i ** 2 for i in range(n))

    with ProcessPoolExecutor(max_workers=4) as pool:
        # Run in separate process — doesn't block event loop
        # AND truly parallel (no GIL!)
        result = await loop.run_in_executor(pool, cpu_heavy, 1_000_000)
        print(f"CPU result: {result}")

    # ─────────────────────────────────────────
    # asyncio.to_thread (Python 3.9+) — easier
    # ─────────────────────────────────────────
    def sync_database_operation() -> dict:
        """Using a sync database library."""
        time.sleep(0.1)  # simulate sync DB call
        return {"data": "from sync db"}

    # Runs sync function in thread without blocking event loop
    result = await asyncio.to_thread(sync_database_operation)
    print(f"Thread result: {result}")


asyncio.run(mixing_sync_and_async())
```

---

## 📊 Visual Summary

text

```
┌────────────────────────────────────────────────────────────────┐
│                    ASYNCIO FUNDAMENTALS                        │
│                                                                │
│  CORE CONCEPTS:                                                │
│  Event loop      → single thread managing many coroutines     │
│  Coroutine       → async def function, pauses at await        │
│  Task            → coroutine scheduled in event loop          │
│  Future          → placeholder for result not yet computed    │
│  await           → pause here, let other coroutines run       │
│                                                                │
│  RUNNING COROUTINES:                                           │
│  asyncio.run()          → run from sync code (entry point)   │
│  await coroutine()      → run sequentially                    │
│  await asyncio.gather() → run concurrently, wait for all     │
│  asyncio.create_task()  → schedule, don't wait immediately   │
│  async with TaskGroup   → modern, strict error handling       │
│                                                                │
│  SYNCHRONIZATION:                                              │
│  Lock      → one at a time                                    │
│  Semaphore → N at a time                                       │
│  Event     → signal between coroutines                        │
│  Queue     → pass data between coroutines                     │
│                                                                │
│  TIMEOUTS:                                                     │
│  asyncio.wait_for(coro, timeout=5.0)                          │
│  async with asyncio.timeout(5.0):                             │
│                                                                │
│  RULES:                                                        │
│  ❌ time.sleep() in async → ✅ await asyncio.sleep()          │
│  ❌ requests.get() → ✅ await httpx.AsyncClient.get()         │
│  ❌ blocking code → ✅ await asyncio.to_thread(sync_fn)       │
│  CPU work → run_in_executor(ProcessPoolExecutor)              │
│                                                                │
│  PATTERNS:                                                     │
│  Concurrent fetch    → asyncio.gather(q1, q2, q3)            │
│  Rate limiting       → asyncio.Semaphore(N)                   │
│  Background tasks    → asyncio.create_task()                  │
│  Retry with backoff  → loop + asyncio.wait_for               │
│  Producer-consumer   → asyncio.Queue                          │
└────────────────────────────────────────────────────────────────┘
```

---

## ✅ Knowledge Check

text

```
1.  What is the event loop and how does it work?
2.  What does 'await' actually do?
3.  What happens when you call async_func() without await?
4.  What is the difference between gather() and create_task()?
5.  When does gather() raise an exception?
    How do you handle errors for each task individually?
6.  What is asyncio.Semaphore and when would you use it?
7.  Why should you never use time.sleep() in an async function?
8.  What does run_in_executor() do?
9.  When would you use a ProcessPoolExecutor vs ThreadPoolExecutor?
10. What is the difference between async context manager
    and regular context manager?
11. What is an async generator? Give a use case.
12. What is the cache stampede problem and how does
    per-key locking solve it?
13. When does async NOT help performance?
14. What is asyncio.Queue used for?
```

---

## 🛠️ Practice Exercises

Python

```
# Exercise 1: Concurrent API Aggregator
# Build a function: aggregate_data(user_id: int) -> dict
# That fetches from 5 "APIs" (simulate with asyncio.sleep):
#   - user profile (200ms)
#   - user posts (300ms)
#   - user followers (150ms)
#   - user settings (100ms)
#   - recommendations (400ms)
# All concurrently. Total should be ~400ms not ~1150ms.
# Handle failures gracefully (if one fails, others still return)

# Exercise 2: Rate-Limited Scraper
# Build an async scraper:
# async def scrape_urls(urls: list, rate_limit: int = 5) -> list
# - Max 5 concurrent requests at any time
# - Add 100ms delay between starting each request
# - Retry failed requests up to 3 times
# - Timeout each request at 2 seconds
# - Return list of {url, status, content, error}

# Exercise 3: Async Producer-Consumer Pipeline
# Build a pipeline:
# producer → queue_1 → transformer → queue_2 → consumer
# producer: generate 50 items (simulated, 50ms each)
# transformer: process each item (simulated, 100ms each)
# consumer: save each item (simulated, 50ms each)
# Run 1 producer, 3 transformers, 2 consumers
# All concurrently using asyncio.Queue

# Exercise 4: Async Cache
# Build AsyncCache with:
# - get(key) → Optional[Any]
# - set(key, value, ttl_seconds) → None
# - delete(key) → bool
# - get_or_compute(key, factory, ttl) → Any
#   (prevents stampede with per-key locks)
# - clear() → None
# Test with 100 concurrent requests for same key
# Verify factory is called only ONCE

# Exercise 5: Graceful Shutdown
# Build an async server that:
# - Accepts "requests" from a queue
# - Processes each request (simulated work)
# - On SIGINT (Ctrl+C), finishes current requests
#   then shuts down cleanly
# - Logs: started, each request, shutdown
# Handle signal with asyncio.get_event_loop().add_signal_handler()
```

---

## ✅ Phase 5.1 Complete!

**You now understand:**

text

```
✅ Why async exists and what problem it solves
✅ The event loop — how it manages coroutines
✅ async def and await — writing coroutines
✅ asyncio.gather() — concurrent execution
✅ asyncio.create_task() — fire and don't wait
✅ TaskGroup — modern concurrent task management
✅ Error handling in async code (all 4 strategies)
✅ Timeouts (wait_for, asyncio.timeout)
✅ Lock, Semaphore, Event, Queue
✅ Async context managers and iterators
✅ Async generators for streaming
✅ Real patterns: concurrent fetching, rate limiting, caching
✅ Mixing sync and async (run_in_executor, to_thread)
✅ FastAPI's async model
✅ When async helps and when it doesn't
```