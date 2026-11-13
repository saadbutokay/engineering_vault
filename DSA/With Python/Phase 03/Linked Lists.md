## Overview

A linked list is a sequence built from small objects called nodes. Each node holds a value and a reference to the next node. Unlike an array, the nodes do not need to sit side by side in memory: the references are the only thing that ties them together. To read the third item you start at the first node and follow the references, one hop at a time.

That makes a linked list the opposite of an array in cost. Reaching item `i` is slow, `O(n)`, but inserting or removing a node is fast once you hold a reference to the right place, because you only change a couple of references instead of shifting everything.

Linked lists are worth learning even though Python's built-in `list` and `collections.deque` are usually the better choice. They are the simplest example of a structure built from linked objects, which is the basis of trees and graphs. They train you to manipulate references carefully, and they power many interview problems. A few real systems still depend on them, for example caches that must remove an item from the middle in constant time. This note covers singly and doubly linked lists, nodes with classes and `__slots__`, sentinel nodes, insertion, deletion, and reversal, fast and slow pointers, Floyd's cycle detection, finding the middle, merging sorted lists, and why linked lists are rarely the right choice in Python and when they still are.

## Core concepts

### Nodes and links

A node stores a value and one or two references.

- Singly linked list: each node has `next`. You can walk only forward.
- Doubly linked list: each node has `next` and `prev`. You can walk both ways, and you can remove a node when you hold just that node.
- Head: the first node. Tail: the last node. The last node's `next` is `None`.
- A list is identified by its head. If you lose the reference to the head, you lose the whole list.

### Costs compared with a dynamic array

- Access the item at position `i`: `O(n)` for a linked list, `O(1)` for an array.
- Insert or delete at the front: `O(1)` for a linked list, `O(n)` for an array.
- Insert or delete at the back: `O(1)` for a linked list that keeps a tail pointer (deletion needs a doubly linked list), `O(1)` amortized for an array append.
- Insert or delete after a node you already hold: `O(1)` for a linked list, `O(n)` for an array.
- Search for a value: `O(n)` for both.
- Memory: each linked node stores a value reference plus one or two link references plus object overhead.

### Sentinel nodes

A sentinel (or dummy) node is a fake node that never holds real data. A dummy head placed before the first real node means that "insert at the front" and "delete the first node" use the same code as any other position, so the empty list and the one-element list stop being special cases. Doubly linked lists often use two sentinels, one at each end.

### Reversal

Reversing a singly linked list means turning every `next` reference around. The iterative method walks the list once with three references: the node behind (`previous`), the current node, and the node ahead (saved before you overwrite `next`). It takes `O(n)` time and `O(1)` space.

### Fast and slow pointers

Two references move through the list at different speeds: the slow one advances one node per step and the fast one advances two. When the fast pointer reaches the end, the slow pointer is at the middle. This idea also detects cycles.

### Floyd's cycle detection

If a list contains a cycle (a node's `next` points back to an earlier node), walking it never ends. Floyd's tortoise and hare algorithm detects this with `O(1)` extra memory: move a slow and a fast pointer; if they ever meet, there is a cycle, and if the fast pointer reaches `None`, there is not. A second phase finds where the cycle begins.

### Merging sorted lists

Given two sorted lists, build one sorted list by repeatedly taking the smaller front node. A dummy head makes the code short. Because you only relink nodes, no new nodes are needed: `O(n + m)` time and `O(1)` extra space.

### Why linked lists are rarely the right choice in Python

- A Python `list` stores references in a contiguous block with tiny per-item overhead (8 bytes per slot). A linked node is a full Python object, about 48 bytes even with `__slots__`, plus the value.
- Contiguous memory is friendly to the processor cache. Following references jumps around memory.
- For a queue or deque (adding and removing at both ends), `collections.deque` does everything a linked list does with less overhead.
- Most things people want from a linked list (fast insertion in the middle) require already holding a node, which usually means you also need a dictionary to find it.

### When linked lists still make sense

- You need to remove or move a node you already hold in `O(1)`, for example in an LRU cache where a dictionary finds the node and a doubly linked list orders them.
- Data is naturally stored as linked records (rows pointing to the next row in a database, a chain of workflow steps).
- You want a persistent (immutable) list that shares tails between versions.
- You are learning or interviewing, where manipulating references is the point.

## Basic syntax examples

### A node class with `__slots__`

`__slots__` tells Python the exact attributes a node has, so it does not create a per-object dictionary. That saves memory when you create many small objects.

```python
class Node[T]:
    """A singly linked list node."""

    __slots__ = ("value", "next")

    def __init__(self, value: T, next: "Node[T] | None" = None) -> None:
        self.value = value
        self.next = next


third = Node(30)
second = Node(20, third)
head = Node(10, second)  # head -> 20 -> 30


def to_list[T](head: Node[T] | None) -> list[T]:
    values: list[T] = []
    current = head
    while current is not None:
        values.append(current.value)
        current = current.next
    return values


def from_values[T](values: list[T]) -> Node[T] | None:
    """Build a list by walking backwards, so each node can point to the one after it."""
    head: Node[T] | None = None
    for value in reversed(values):
        head = Node(value, head)
    return head


print(to_list(head))  # [10, 20, 30]
print(to_list(from_values(["a", "b", "c"])))  # ['a', 'b', 'c']
```

### A singly linked list class

```python
from collections.abc import Iterator


class SinglyLinkedList[T]:
    """Head and tail pointers make push_front and push_back O(1)."""

    def __init__(self) -> None:
        self._head: Node[T] | None = None
        self._tail: Node[T] | None = None
        self._size = 0

    def __len__(self) -> int:
        return self._size

    def __iter__(self) -> Iterator[T]:
        current = self._head
        while current is not None:
            yield current.value
            current = current.next

    def push_front(self, value: T) -> None:
        node = Node(value, self._head)
        self._head = node
        if self._tail is None:  # the list was empty
            self._tail = node
        self._size += 1

    def push_back(self, value: T) -> None:
        node = Node(value)
        if self._tail is None:
            self._head = self._tail = node
        else:
            self._tail.next = node
            self._tail = node
        self._size += 1

    def pop_front(self) -> T:
        if self._head is None:
            raise IndexError("pop from empty list")
        node = self._head
        self._head = node.next
        if self._head is None:  # the list became empty
            self._tail = None
        self._size -= 1
        return node.value

    def remove(self, value: T) -> bool:
        """Remove the first node holding value. O(n) to find it."""
        previous: Node[T] | None = None
        current = self._head
        while current is not None:
            if current.value == value:
                if previous is None:
                    self._head = current.next
                else:
                    previous.next = current.next
                if current is self._tail:
                    self._tail = previous
                self._size -= 1
                return True
            previous, current = current, current.next
        return False


numbers = SinglyLinkedList[int]()
numbers.push_back(2)
numbers.push_back(3)
numbers.push_front(1)
print(list(numbers), len(numbers))  # [1, 2, 3] 3
numbers.remove(2)
print(list(numbers))  # [1, 3]
```

Each method updates `_head`, `_tail`, and `_size` together. A common bug is changing one and forgetting another, especially the tail when the last node is removed.

### Deleting with a dummy head

```python
def remove_all(head: Node[int] | None, target: int) -> Node[int] | None:
    """Remove every node holding target. The dummy node handles a removed head."""
    dummy = Node(0, head)
    previous = dummy
    while previous.next is not None:
        if previous.next.value == target:
            previous.next = previous.next.next  # skip over the node
        else:
            previous = previous.next
    return dummy.next


print(to_list(remove_all(from_values([1, 2, 6, 3, 6]), 6)))  # [1, 2, 3]
print(to_list(remove_all(from_values([6, 6]), 6)))  # []
```

### Reversing a list

```python
def reverse_list[T](head: Node[T] | None) -> Node[T] | None:
    """Iterative reversal. O(n) time, O(1) space."""
    previous: Node[T] | None = None
    current = head
    while current is not None:
        following = current.next  # save the rest before we overwrite next
        current.next = previous  # turn this link around
        previous = current
        current = following
    return previous  # the old tail is the new head


print(to_list(reverse_list(from_values([1, 2, 3, 4]))))  # [4, 3, 2, 1]
```

Trace for `1 -> 2 -> 3`:

```text
start:   previous=None, current=1
step 1:  following=2, 1.next=None,  previous=1, current=2     None <- 1    2 -> 3
step 2:  following=3, 2.next=1,     previous=2, current=3     None <- 1 <- 2    3
step 3:  following=None, 3.next=2,  previous=3, current=None  None <- 1 <- 2 <- 3
return previous, which is node 3
```

### Finding the middle with fast and slow pointers

```python
def middle_node[T](head: Node[T] | None) -> Node[T] | None:
    """For an even length this returns the second of the two middle nodes."""
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next if slow is not None else None
        fast = fast.next.next
    return slow


middle = middle_node(from_values([1, 2, 3, 4, 5]))
print(middle.value if middle else None)  # 3
middle = middle_node(from_values([1, 2, 3, 4]))
print(middle.value if middle else None)  # 3
```

### Floyd's cycle detection

```python
def has_cycle[T](head: Node[T] | None) -> bool:
    """Tortoise and hare. O(n) time, O(1) space."""
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next if slow is not None else None
        fast = fast.next.next
        if slow is fast:
            return True
    return False


a, b, c, d = Node(1), Node(2), Node(3), Node(4)
a.next, b.next, c.next = b, c, d
print(has_cycle(a))  # False
d.next = b  # create a cycle: 2 -> 3 -> 4 -> 2
print(has_cycle(a))  # True
```

### Merging two sorted lists

```python
def merge_sorted(first: Node[int] | None, second: Node[int] | None) -> Node[int] | None:
    """Relink existing nodes into one sorted list. O(n + m) time, O(1) space."""
    dummy = Node(0)
    tail = dummy
    while first is not None and second is not None:
        if first.value <= second.value:  # <= keeps the merge stable
            tail.next, first = first, first.next
        else:
            tail.next, second = second, second.next
        tail = tail.next
    tail.next = first if first is not None else second  # attach the leftover run
    return dummy.next


merged = merge_sorted(from_values([1, 4, 7]), from_values([2, 3, 9, 10]))
print(to_list(merged))  # [1, 2, 3, 4, 7, 9, 10]
```

### A doubly linked list with sentinels

```python
class DNode[T]:
    __slots__ = ("value", "prev", "next")

    def __init__(self, value: T) -> None:
        self.value = value
        self.prev: DNode[T] | None = None
        self.next: DNode[T] | None = None


class DoublyLinkedList[T]:
    """Two sentinel nodes, so no operation needs to check for an empty list."""

    def __init__(self, sentinel_value: T) -> None:
        self._head = DNode(sentinel_value)  # before the first real node
        self._tail = DNode(sentinel_value)  # after the last real node
        self._head.next = self._tail
        self._tail.prev = self._head
        self._size = 0

    def __len__(self) -> int:
        return self._size

    def __iter__(self) -> Iterator[T]:
        node = self._head.next
        while node is not None and node is not self._tail:
            yield node.value
            node = node.next

    def insert_after(self, node: DNode[T], value: T) -> DNode[T]:
        """O(1): four reference updates."""
        following = node.next
        assert following is not None  # only the tail sentinel has no next
        new = DNode(value)
        new.prev, new.next = node, following
        node.next = new
        following.prev = new
        self._size += 1
        return new

    def append(self, value: T) -> DNode[T]:
        before_tail = self._tail.prev
        assert before_tail is not None
        return self.insert_after(before_tail, value)

    def remove(self, node: DNode[T]) -> T:
        """O(1) removal of a node you already hold, from anywhere in the list."""
        before, after = node.prev, node.next
        assert before is not None and after is not None  # sentinels are never removed
        before.next = after
        after.prev = before
        node.prev = node.next = None  # help the garbage collector, prevent reuse bugs
        self._size -= 1
        return node.value


items = DoublyLinkedList[str]("")
first = items.append("a")
items.append("b")
items.append("c")
items.remove(first)
print(list(items), len(items))  # ['b', 'c'] 2
```

## Deeper behavior

### Why sentinels remove bugs

Without a dummy head, deleting the first node and deleting a middle node need different code, because the first node has no predecessor, and you must update the list's head pointer. With a dummy head, the first real node always has a predecessor, so one loop handles every position. In the doubly linked list, two sentinels guarantee that every real node has both a `prev` and a `next`, so `insert_after` and `remove` have no `if` statements at all. The cost is one or two extra nodes, which is nothing.

### Reversal, step by step

The order of the three assignments matters. You must save `following = current.next` before you overwrite `current.next`, or you lose the rest of the list. Doing `current.next = previous` first and then reading `current.next` reads the wrong node. A recursive reversal also exists, but it uses `O(n)` stack space and hits Python's recursion limit on long lists, so the iterative version is the right one.

### Why Floyd's algorithm works

If there is a cycle, both pointers eventually enter it. Inside the cycle the fast pointer gains one node on the slow pointer per step, so the gap between them shrinks by one each iteration until it is zero, which means they are on the same node. If there is no cycle, the fast pointer reaches `None` first.

To find the node where the cycle starts, let `a` be the number of nodes before the cycle, `b` the distance from the cycle start to the meeting point, and `c` the cycle length. When they meet, the slow pointer has moved `a + b` steps and the fast pointer `2(a + b)`, and the difference `a + b` is a multiple of the cycle length. So starting from the meeting point, walking `a` more steps lands exactly on the cycle start. Therefore, put one pointer back at the head, leave the other at the meeting point, advance both one step at a time, and they meet at the start of the cycle.

```python
def cycle_start[T](head: Node[T] | None) -> Node[T] | None:
    """Return the first node of the cycle, or None if there is no cycle."""
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next if slow is not None else None
        fast = fast.next.next
        if slow is fast:  # they met inside the cycle
            pointer = head
            while pointer is not slow:
                pointer = pointer.next if pointer is not None else None
                slow = slow.next if slow is not None else None
            return pointer
    return None
```

### Removing the nth node from the end

A classic two-pointer trick: advance a lead pointer `n` steps ahead, then move both pointers until the lead reaches the end. The trailing pointer now sits just before the node to delete, and a dummy head handles the case where the first node must go. It takes one pass instead of two.

```python
def remove_nth_from_end(head: Node[int] | None, n: int) -> Node[int] | None:
    """Remove the nth node counting from the end (n = 1 is the last node)."""
    if n < 1:
        raise ValueError("n must be at least 1")
    dummy = Node(0, head)
    lead = trail = dummy
    for _ in range(n):  # open a gap of n nodes between lead and trail
        if lead.next is None:
            raise ValueError("n is larger than the list")
        lead = lead.next
    while lead.next is not None:  # move both until lead is on the last node
        lead = lead.next
        assert trail.next is not None
        trail = trail.next
    assert trail.next is not None  # trail now sits just before the node to remove
    trail.next = trail.next.next
    return dummy.next


print(to_list(remove_nth_from_end(from_values([1, 2, 3, 4, 5]), 2)))  # [1, 2, 3, 5]
```

### Memory and speed in Python

A slotted node costs about 48 bytes before counting the value, while a list slot costs 8 bytes. A list of a million numbers holds a million 8-byte references, while a linked list holds a million 48-byte nodes. Without `__slots__`, each instance carries a dictionary for its attributes, which is much larger.

Traversal also costs more. Even when all nodes were created back to back, so that they sit close together in memory, summing 200,000 items through `next` references took about 35 percent longer in one test than looping over a list. In a long-running program where nodes are created and freed at different times, they end up scattered, and following references causes cache misses that make the gap larger. For random access the difference is complete: a list needs one index operation and a linked list needs `i` hops.

### Mutation and iteration

Changing links while walking a list is a common source of skipped or repeated nodes. If you delete the current node, save the next reference before you unlink it. If you insert a node while iterating, decide whether the loop should visit it.

### Garbage collection and cycles

Python frees an object when nothing refers to it. A doubly linked list has references in both directions, so every pair of neighboring nodes refer to each other, which forms reference cycles that only Python's cyclic garbage collector can clean up. If a long-lived structure builds many such nodes, setting `node.prev = node.next = None` when removing a node, as `remove` does above, lets reference counting free it immediately.

### Persistent lists

An immutable singly linked list lets different versions share the same tail. Prepending creates one new node that points to the old list, and nothing is copied. This is the basis of persistent data structures in functional languages. In Python it is a niche technique, but it shows a case where a linked list is genuinely better than an array: cheap, safe sharing.

## Industry-level snippet

An approval system stores each workflow as a chain of steps in a database table. Each row has its own id, a name, and the id of the next step. The chain is a linked list stored in rows. Administrators can insert a step after another one, remove a step, and ask for the rollback order, which is the chain reversed.

Because the links are plain integers in a table, bad data is possible: a typo can make a step point to a nonexistent id, or two edits can create a cycle. A loop that follows the links would hang forever on a cycle. The loader therefore validates before it builds anything. Floyd's algorithm checks for a cycle with `O(1)` extra memory, and the walk reports a missing target as a specific error.

Edits run in a transaction that takes the write lock first. Inserting after a step changes two rows (the new step and the predecessor), and removing a step reroutes the predecessor around it, so both must succeed or fail together.

```python
# workflow_chain.py
import logging
import sqlite3
from collections.abc import Mapping
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS workflow_step (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    next_id INTEGER REFERENCES workflow_step (id)
)
"""


class StepNotFoundError(LookupError):
    """Raised when a step id does not exist."""


class WorkflowCycleError(ValueError):
    """Raised when following next links never ends."""


class WorkflowBrokenError(ValueError):
    """Raised when a step points to a step that does not exist."""


class CannotRemoveFirstStepError(ValueError):
    """Raised when asked to remove a step that nothing points to."""


@dataclass(slots=True)
class StepNode:
    step_id: int
    name: str
    next: "StepNode | None" = None


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def load_links(
    connection: sqlite3.Connection,
) -> tuple[dict[int, str], dict[int, int | None]]:
    names: dict[int, str] = {}
    next_of: dict[int, int | None] = {}
    for step_id, name, next_id in connection.execute(
        "SELECT id, name, next_id FROM workflow_step"
    ):
        names[step_id] = name
        next_of[step_id] = next_id
    return names, next_of


def has_cycle(next_of: Mapping[int, int | None], start: int) -> bool:
    """Floyd's tortoise and hare over id links. O(n) time, O(1) extra space."""
    slow = fast = start
    while True:
        first_hop = next_of.get(fast)
        if first_hop is None:
            return False
        second_hop = next_of.get(first_hop)
        if second_hop is None:
            return False
        fast = second_hop
        slow_next = next_of.get(slow)
        if slow_next is None:
            return False
        slow = slow_next
        if slow == fast:
            return True


def load_chain(connection: sqlite3.Connection, first_id: int) -> StepNode:
    """Build the linked list, refusing cycles and dangling links."""
    names, next_of = load_links(connection)
    if first_id not in names:
        raise StepNotFoundError(f"step {first_id} does not exist")
    if has_cycle(next_of, first_id):
        raise WorkflowCycleError(f"the chain starting at {first_id} has a cycle")
    head = tail = StepNode(first_id, names[first_id])
    next_id = next_of[first_id]
    while next_id is not None:
        if next_id not in names:
            raise WorkflowBrokenError(
                f"step {tail.step_id} points to missing step {next_id}"
            )
        node = StepNode(next_id, names[next_id])
        tail.next = node
        tail = node
        next_id = next_of[next_id]
    return head


def step_names(head: StepNode | None) -> list[str]:
    names: list[str] = []
    current = head
    while current is not None:
        names.append(current.name)
        current = current.next
    return names


def reverse_chain(head: StepNode | None) -> StepNode | None:
    """Rollback order: iterative reversal of the in-memory list."""
    previous: StepNode | None = None
    current = head
    while current is not None:
        following = current.next
        current.next = previous
        previous = current
        current = following
    return previous


def middle_step(head: StepNode) -> StepNode:
    """The middle step (the second of two middles), by fast and slow pointers."""
    slow: StepNode = head
    fast: StepNode | None = head
    while fast is not None and fast.next is not None:
        assert slow.next is not None
        slow = slow.next
        fast = fast.next.next
    return slow


def insert_after(
    connection: sqlite3.Connection, after_id: int, new_id: int, name: str
) -> None:
    """Add a step after another one. Two row changes, one transaction."""
    if not name:
        raise ValueError("name must not be empty")
    try:
        with connection:
            connection.execute("BEGIN IMMEDIATE")
            row = connection.execute(
                "SELECT next_id FROM workflow_step WHERE id = ?", (after_id,)
            ).fetchone()
            if row is None:
                raise StepNotFoundError(f"step {after_id} does not exist")
            connection.execute(
                "INSERT INTO workflow_step (id, name, next_id) VALUES (?, ?, ?)",
                (new_id, name, row[0]),
            )
            connection.execute(
                "UPDATE workflow_step SET next_id = ? WHERE id = ?", (new_id, after_id)
            )
    except sqlite3.Error:
        logger.exception("could not insert step %d after %d", new_id, after_id)
        raise
    logger.info("inserted step %d after %d", new_id, after_id)


def remove_step(connection: sqlite3.Connection, step_id: int) -> None:
    """Unlink a step: its predecessor now points to its successor."""
    try:
        with connection:
            connection.execute("BEGIN IMMEDIATE")
            row = connection.execute(
                "SELECT next_id FROM workflow_step WHERE id = ?", (step_id,)
            ).fetchone()
            if row is None:
                raise StepNotFoundError(f"step {step_id} does not exist")
            predecessor = connection.execute(
                "SELECT id FROM workflow_step WHERE next_id = ?", (step_id,)
            ).fetchone()
            if predecessor is None:
                raise CannotRemoveFirstStepError(f"step {step_id} has no predecessor")
            connection.execute(
                "UPDATE workflow_step SET next_id = ? WHERE id = ?",
                (row[0], predecessor[0]),
            )
            connection.execute("DELETE FROM workflow_step WHERE id = ?", (step_id,))
    except sqlite3.Error:
        logger.exception("could not remove step %d", step_id)
        raise
    logger.info("removed step %d", step_id)
```

```python
# test_workflow_chain.py
import sqlite3
from collections.abc import Iterator

import pytest

from workflow_chain import (
    CannotRemoveFirstStepError,
    StepNotFoundError,
    WorkflowBrokenError,
    WorkflowCycleError,
    create_schema,
    has_cycle,
    insert_after,
    load_chain,
    middle_step,
    remove_step,
    reverse_chain,
    step_names,
)


type Row = tuple[int, str, int | None]


def seed(connection: sqlite3.Connection, rows: list[Row]) -> None:
    connection.executemany(
        "INSERT INTO workflow_step (id, name, next_id) VALUES (?, ?, ?)", rows
    )
    connection.commit()


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    seed(
        conn,
        [(1, "draft", 2), (2, "review", 3), (3, "approve", 4), (4, "publish", None)],
    )
    yield conn
    conn.close()


def test_chain_loads_in_order(connection: sqlite3.Connection) -> None:
    assert step_names(load_chain(connection, 1)) == [
        "draft",
        "review",
        "approve",
        "publish",
    ]


def test_reverse_gives_rollback_order(connection: sqlite3.Connection) -> None:
    reversed_head = reverse_chain(load_chain(connection, 1))
    assert step_names(reversed_head) == ["publish", "approve", "review", "draft"]
    assert reverse_chain(None) is None


def test_middle_step(connection: sqlite3.Connection) -> None:
    assert middle_step(load_chain(connection, 1)).name == "approve"
    assert middle_step(load_chain(connection, 2)).name == "approve"  # 3 steps: middle
    assert middle_step(load_chain(connection, 4)).name == "publish"


def test_cycle_is_detected_without_hanging(connection: sqlite3.Connection) -> None:
    connection.execute("UPDATE workflow_step SET next_id = 2 WHERE id = 4")
    connection.commit()
    with pytest.raises(WorkflowCycleError):
        load_chain(connection, 1)


def test_dangling_link_is_reported(connection: sqlite3.Connection) -> None:
    connection.execute("UPDATE workflow_step SET next_id = 99 WHERE id = 3")
    connection.commit()
    with pytest.raises(WorkflowBrokenError):
        load_chain(connection, 1)


def test_unknown_first_step(connection: sqlite3.Connection) -> None:
    with pytest.raises(StepNotFoundError):
        load_chain(connection, 42)


@pytest.mark.parametrize(
    ("links", "start", "expected"),
    [
        ({1: 2, 2: 3, 3: None}, 1, False),
        ({1: 2, 2: 1}, 1, True),
        ({1: 1}, 1, True),
        ({1: 2, 2: 3, 3: 4, 4: 2}, 1, True),
        ({1: None}, 1, False),
        ({1: 7}, 1, False),
    ],
)
def test_has_cycle(links: dict[int, int | None], start: int, expected: bool) -> None:
    assert has_cycle(links, start) is expected


def test_insert_after_updates_two_rows(connection: sqlite3.Connection) -> None:
    insert_after(connection, 2, 10, "legal check")
    assert step_names(load_chain(connection, 1)) == [
        "draft",
        "review",
        "legal check",
        "approve",
        "publish",
    ]


def test_insert_after_the_last_step(connection: sqlite3.Connection) -> None:
    insert_after(connection, 4, 11, "archive")
    assert step_names(load_chain(connection, 1))[-1] == "archive"


def test_failed_insert_changes_nothing(connection: sqlite3.Connection) -> None:
    with pytest.raises(sqlite3.IntegrityError):
        insert_after(connection, 2, 3, "duplicate id")  # id 3 already exists
    assert step_names(load_chain(connection, 1))[:3] == ["draft", "review", "approve"]
    with pytest.raises(StepNotFoundError):
        insert_after(connection, 77, 12, "orphan")


def test_remove_reroutes_the_predecessor(connection: sqlite3.Connection) -> None:
    remove_step(connection, 3)
    assert step_names(load_chain(connection, 1)) == ["draft", "review", "publish"]
    remove_step(connection, 4)
    assert step_names(load_chain(connection, 1)) == ["draft", "review"]


def test_first_step_cannot_be_removed(connection: sqlite3.Connection) -> None:
    with pytest.raises(CannotRemoveFirstStepError):
        remove_step(connection, 1)
    with pytest.raises(StepNotFoundError):
        remove_step(connection, 99)
```

The cycle test is the important one: without `has_cycle`, `load_chain` would loop forever on that data and take the service down with it. A bounded loop or a set of visited ids would also work, but Floyd's algorithm needs no memory proportional to the chain.

## Common mistakes

- Losing the head reference, which makes the rest of the list unreachable.
- Overwriting `current.next` before saving it, which cuts off the rest of the list during reversal or deletion.
- Forgetting to update the tail pointer when removing the last node, or when pushing into an empty list.
- Not handling the empty list and the one-node list. A dummy head fixes most of these cases.
- Skipping a node when deleting inside a loop, by advancing the pointer after a deletion that already moved it.
- Using `==` where identity is needed to compare nodes in cycle detection. Use `is`.
- Creating a cycle by accident when relinking, then looping forever. Print a few nodes or use `has_cycle` while debugging.
- Reaching for a linked list "because insertion is O(1)" when you do not already hold the node. Finding the position is `O(n)`, so a Python `list` or `deque` wins.
- Using `list.pop(0)` as a queue and then trying to fix it with a hand-written linked list. Use `collections.deque`.
- Forgetting `__slots__` on a class you create by the million.
- Recursing over a long list and hitting the recursion limit.

## Practice exercises

1. Write `length(head)` and `get_at(head, index)`, and state the time complexity of each.
2. Write `insert_at(head, index, value)` using a dummy head, and test inserting at the front, middle, and end.
3. Write `delete_node_given_only_that_node(node)` that deletes a node in the middle of a list when you are given only that node (not the head), by copying the next node's value into it. Explain why it fails for the tail.
4. Reverse a list recursively, then iteratively, and compare how large a list each can handle.
5. Write `reverse_between(head, left, right)` that reverses only the nodes from position `left` to position `right` in one pass.
6. Write `is_palindrome(head)` for a linked list in `O(n)` time and `O(1)` extra space, using the middle, a reversal, and a comparison.
7. Write `find_intersection(head_a, head_b)` that returns the node where two lists merge into one shared tail, using two pointers that switch lists at the end.
8. Write `remove_duplicates_sorted(head)` that deletes repeated values from a sorted list, and a version for an unsorted list that uses a set.
9. Write `partition(head, pivot)` that rearranges a list so that nodes with values less than the pivot come before the others, keeping the original order within each group.
10. Write `add_two_numbers(first, second)` where each list holds the digits of a number in reverse order (least significant digit first), returning the sum as a list.
11. Write `merge_k_sorted(lists)` for a list of sorted linked lists using a heap, and state the time complexity in terms of the total number of nodes and `k`.
12. Write `copy_list_with_random_pointer(head)` for nodes with an extra `random` reference, using a dictionary from old nodes to new ones.
13. Extend `DoublyLinkedList` with `move_to_front(node)` and `popleft()` and use them to sketch an LRU ordering.
14. Extend `insert_after` in the workflow code with a check that rejects inserting a step whose `new_id` is already referenced by another step, and add a test.

## Summary

A linked list is a chain of nodes connected by references. Access by position costs `O(n)`, but inserting or removing next to a node you already hold costs `O(1)`, and sentinel nodes remove the special cases at the ends. Reversal needs three references and one pass, fast and slow pointers find the middle and, with Floyd's algorithm, detect and locate cycles in `O(1)` space, and merging sorted lists only relinks nodes. In Python, a linked node is a heavyweight object, so `list` and `collections.deque` are usually better. Linked lists earn their place when you must unlink a known node in constant time, as in caches, when data is already stored as linked records, or when sharing immutable tails. Always update every pointer that refers to a node together, and validate linked data from outside your program for cycles and dangling links.
