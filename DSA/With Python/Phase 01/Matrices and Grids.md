## Overview

A matrix, or grid, is a table of items arranged in rows and columns. Spreadsheets, game boards, seating plans, maps, images, and warehouse shelves are all grids. In Python the standard way to store one is a list of lists, where the outer list holds the rows and each inner list holds the items of one row. You read an item with `grid[row][col]`.

Grids look simple, but they hide several classic traps: building one with the wrong syntax so that every row is secretly the same list, mixing up row and column order, stepping off the edge, and accidentally reading from the far side of the grid through a negative index. They also introduce direction arrays, a compact way to move up, down, left, and right, and they preview graph algorithms, because a grid is a graph whose vertices are cells and whose edges connect neighbors.

This note covers building 2D lists correctly, row-major traversal, direction arrays, spiral traversal, rotating and transposing, searching a sorted matrix, in-bounds checks, and grids as implicit graphs.

## Core concepts

### Rows, columns, and indexing

`grid[r][c]` means row `r`, column `c`, counted from zero. A grid with `rows` rows and `cols` columns is rectangular when every row has length `cols`. The number of cells is `rows * cols`, and most grid algorithms cost `O(rows * cols)` because they visit each cell a constant number of times.

### Building a grid correctly

Each row must be a separate list. The correct pattern is a list comprehension that builds a fresh row on every pass: `[[0] * cols for _ in range(rows)]`. The shortcut `[[0] * cols] * rows` repeats a reference to one single row `rows` times, so changing one cell appears to change a whole column.

### Row-major traversal

Visiting the cells row by row, left to right, is called row-major order. It matches how lists of lists are laid out in memory, so it is the natural and cache-friendly traversal. Column-major traversal visits column by column and jumps between different row lists.

### Direction arrays

Instead of writing four copies of the same code for up, down, left, and right, store the moves as pairs of row and column changes and loop over them:

```text
up:    (-1, 0)     down:  (1, 0)
left:  (0, -1)     right: (0, 1)
```

Add the diagonals for eight-way movement. Direction arrays make neighbor logic short and less error-prone.

### In-bounds checks

Before reading a neighbor, check `0 <= row < rows` and `0 <= col < cols`. Python will not warn you when you step off the top or left edge, because negative indexes wrap around to the other side. Only stepping past the bottom or right edge raises `IndexError`.

### Spiral traversal

Walk the border of the matrix clockwise, then shrink the boundaries and repeat. Four variables, `top`, `bottom`, `left`, and `right`, track the unvisited rectangle.

### Transposing and rotating

- Transpose: swap rows and columns, so the item at `(r, c)` moves to `(c, r)`.
- Rotate 90 degrees clockwise: transpose, then reverse every row. The item at `(r, c)` ends up at `(c, rows - 1 - r)`.

### Searching a sorted matrix

If every row is sorted left to right and every column is sorted top to bottom, you can find a value in `O(rows + cols)` by starting at the top-right corner and discarding a row or a column at each step.

### Grids as implicit graphs

Each cell is a vertex, and each pair of neighboring cells is an edge. You never build an adjacency list. The direction array generates the neighbors when you need them. That is why flood fill, counting islands, and shortest paths in a maze are graph algorithms on grids.

## Basic syntax examples

### Building a grid and the shared-row bug

```python
rows, cols = 3, 4

grid = [[0] * cols for _ in range(rows)]  # correct: three separate rows
grid[0][0] = 1
print(grid)  # [[1, 0, 0, 0], [0, 0, 0, 0], [0, 0, 0, 0]]

shared = [[0] * cols] * rows  # wrong: the same row three times
shared[0][0] = 1
print(shared)  # [[1, 0, 0, 0], [1, 0, 0, 0], [1, 0, 0, 0]]
print(shared[0] is shared[1])  # True
```

### Traversing in row-major order

```python
matrix = [
    [1, 2, 3],
    [4, 5, 6],
]

for row in matrix:
    for value in row:
        print(value, end=" ")  # 1 2 3 4 5 6
print()

for r in range(len(matrix)):
    for c in range(len(matrix[r])):
        print(f"({r},{c})={matrix[r][c]}", end=" ")
print()
```

### Direction arrays and bounds checks

```python
type Cell = tuple[int, int]

DIRECTIONS_4: tuple[Cell, ...] = ((-1, 0), (1, 0), (0, -1), (0, 1))


def in_bounds(grid: list[list[int]], row: int, col: int) -> bool:
    return 0 <= row < len(grid) and 0 <= col < len(grid[row])


def neighbors(grid: list[list[int]], row: int, col: int) -> list[Cell]:
    """The in-bounds orthogonal neighbors of a cell."""
    result: list[Cell] = []
    for d_row, d_col in DIRECTIONS_4:
        new_row, new_col = row + d_row, col + d_col
        if in_bounds(grid, new_row, new_col):
            result.append((new_row, new_col))
    return result


print(neighbors([[1, 2, 3], [4, 5, 6]], 0, 0))  # [(1, 0), (0, 1)]
```

The check `0 <= row < len(grid)` comes first and short-circuits, so `len(grid[row])` is evaluated only for a valid row.

### Transposing and rotating

```python
def transpose(matrix: list[list[int]]) -> list[list[int]]:
    """Rows become columns. Works for any rectangular matrix. O(rows * cols)."""
    return [list(column) for column in zip(*matrix)]


def rotate_clockwise(matrix: list[list[int]]) -> list[list[int]]:
    """Rotate 90 degrees clockwise by reading each column from bottom to top."""
    return [list(column) for column in zip(*matrix[::-1])]


matrix = [
    [1, 2, 3],
    [4, 5, 6],
]
print(transpose(matrix))  # [[1, 4], [2, 5], [3, 6]]
print(rotate_clockwise(matrix))  # [[4, 1], [5, 2], [6, 3]]
```

### Rotating a square matrix in place

```python
def rotate_in_place(matrix: list[list[int]]) -> None:
    """Rotate a square matrix 90 degrees clockwise. O(n^2) time, O(1) extra space."""
    size = len(matrix)
    for r in range(size):
        for c in range(r + 1, size):  # only the upper triangle: swap across the diagonal
            matrix[r][c], matrix[c][r] = matrix[c][r], matrix[r][c]
    for row in matrix:
        row.reverse()


square = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
]
rotate_in_place(square)
print(square)  # [[7, 4, 1], [8, 5, 2], [9, 6, 3]]
```

### Spiral traversal

```python
def spiral_order(matrix: list[list[int]]) -> list[int]:
    """Return the items in clockwise spiral order. O(rows * cols)."""
    if not matrix or not matrix[0]:
        return []
    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1
    result: list[int] = []
    while top <= bottom and left <= right:
        for col in range(left, right + 1):  # top row, left to right
            result.append(matrix[top][col])
        for row in range(top + 1, bottom + 1):  # right column, top to bottom
            result.append(matrix[row][right])
        if top < bottom:  # bottom row, right to left, if it is a different row
            for col in range(right - 1, left - 1, -1):
                result.append(matrix[bottom][col])
        if left < right:  # left column, bottom to top, if it is a different column
            for row in range(bottom - 1, top, -1):
                result.append(matrix[row][left])
        top += 1
        bottom -= 1
        left += 1
        right -= 1
    return result


print(spiral_order([[1, 2, 3], [4, 5, 6], [7, 8, 9]]))  # [1, 2, 3, 6, 9, 8, 7, 4, 5]
```

### Searching a sorted matrix

```python
def search_matrix(matrix: list[list[int]], target: int) -> tuple[int, int] | None:
    """Rows and columns are both sorted ascending. O(rows + cols) time."""
    if not matrix or not matrix[0]:
        return None
    row, col = 0, len(matrix[0]) - 1  # start at the top-right corner
    while row < len(matrix) and col >= 0:
        value = matrix[row][col]
        if value == target:
            return row, col
        if value > target:
            col -= 1  # everything below in this column is even larger
        else:
            row += 1  # everything to the left in this row is even smaller
    return None


sorted_matrix = [
    [1, 4, 7, 11],
    [2, 5, 8, 12],
    [3, 6, 9, 16],
]
print(search_matrix(sorted_matrix, 8))  # (1, 2)
print(search_matrix(sorted_matrix, 10))  # None
```

### A grid as a graph: counting islands

```python
from collections import deque


def count_islands(grid: list[list[int]]) -> int:
    """Count groups of connected 1 cells. Each cell is visited once: O(rows * cols)."""
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])
    seen = [[False] * cols for _ in range(rows)]
    islands = 0
    for start_row in range(rows):
        for start_col in range(cols):
            if grid[start_row][start_col] != 1 or seen[start_row][start_col]:
                continue
            islands += 1
            seen[start_row][start_col] = True
            queue: deque[Cell] = deque([(start_row, start_col)])
            while queue:  # breadth-first search from this cell
                row, col = queue.popleft()
                for d_row, d_col in DIRECTIONS_4:
                    new_row, new_col = row + d_row, col + d_col
                    if (
                        0 <= new_row < rows
                        and 0 <= new_col < cols
                        and grid[new_row][new_col] == 1
                        and not seen[new_row][new_col]
                    ):
                        seen[new_row][new_col] = True
                        queue.append((new_row, new_col))
    return islands


islands = [
    [1, 1, 0, 0],
    [1, 0, 0, 1],
    [0, 0, 1, 1],
]
print(count_islands(islands))  # 2
```

## Deeper behavior

### Lists of lists in memory

A grid built from lists is an outer array of pointers to separate row arrays. Reading `grid[r][c]` does two indexing steps. Rows can have different lengths (a jagged array), which is flexible but means that `len(grid[0])` does not guarantee the other rows match. Validate that every row has the same length before treating the data as a rectangle.

For very large grids, or when rows are accessed in tight loops, a single flat list with the index `r * cols + c` saves memory and one level of indirection:

```python
rows, cols = 3, 4
flat = [0] * (rows * cols)
flat[1 * cols + 2] = 7  # the cell at row 1, column 2
print(flat[1 * cols + 2])  # 7
```

### Why the negative index bug is silent

`grid[-1][0]` is the last row, not an error. When a neighbor calculation produces `-1` for the row or the column, Python happily reads the opposite edge of the grid and the algorithm continues with wrong data. Bounds checks must test the lower bound, `0 <= row`, not only the upper bound. A common defensive technique is to pad the grid with a border of sentinel values so that no check is needed, but then every coordinate shifts by one.

### Copying a grid

`grid.copy()`, `grid[:]`, and `list(grid)` copy only the outer list, so both copies share the same row lists. Changing a cell in one changes it in the other. Use a comprehension to copy the rows, or `copy.deepcopy`:

```python
original = [[1, 2], [3, 4]]
shallow = original[:]
deep = [row[:] for row in original]
original[0][0] = 99
print(shallow[0][0], deep[0][0])  # 99 1
```

### Order of the coordinates

Using `(row, col)` ties to how you write the code: `grid[row][col]`. Using `(x, y)` ties to geometry, where `x` is the column and `y` is the row, which reverses the order. Pick one convention for the whole program and name variables `row` and `col` to avoid mistakes. Mixing the two is a very common source of bugs in grid problems.

### Transpose with zip

`zip(*matrix)` unpacks the rows as separate arguments and groups the first items together, then the second items, and so on. It returns tuples, so wrap each in `list`. If rows have different lengths, `zip` stops at the shortest and silently drops items, which is another reason to validate that the grid is rectangular.

### Why the in-place rotation works

Transposing swaps `(r, c)` with `(c, r)` for pairs above the diagonal, so it touches each pair once and uses no extra space. Reversing each row afterwards maps the item at `(r, c)` to `(r, size - 1 - c)`. The composition sends `(r, c)` to `(c, size - 1 - r)`, which is a clockwise quarter turn. Rotating counterclockwise is the same idea with the order of the steps changed: reverse each row first, then transpose.

### Why the sorted-matrix search is correct

At the top-right corner, the value is the largest in its row and the smallest in its column. If it is greater than the target, nothing below it in the column can be the target (they are larger still), so the column is discarded. If it is smaller than the target, nothing to its left in the row can be the target, so the row is discarded. Each step removes a row or a column, so at most `rows + cols` steps are needed. For a matrix where each row begins after the previous row ends, you can instead treat it as one sorted flat list and use binary search in `O(log(rows * cols))`.

### The spiral boundary checks

After the top row and right column are walked, two boundary checks guard the bottom row and left column: `top < bottom` and `left < right`. Without them, a matrix with a single remaining row or column would be walked twice. Those two checks are the usual source of spiral bugs. Test with one row, one column, and a single cell.

### Eight directions and diagonal movement

Add `(-1, -1)`, `(-1, 1)`, `(1, -1)`, and `(1, 1)` to the direction array for diagonal neighbors. Chess-like moves and knight jumps use their own arrays of offsets. The bounds check does not change.

### Grids as graphs: what carries over

A cell is a vertex, and its in-bounds, non-blocked neighbors are its edges. Breadth-first search on a grid finds shortest paths in steps, depth-first search finds connected regions, and flood fill is a traversal from one cell. Each cell is visited at most once if you keep a `seen` structure, so the cost is `O(rows * cols)`. Keep the `seen` grid separate from the input if you must not modify the input, or reuse the input by marking cells (for example setting a visited `1` to `0`) when modification is acceptable.

## Industry-level snippet

A cinema booking service must find a block of adjacent seats for a group and reserve them. The seat map is a grid: rows are rows of seats and columns are seat numbers. Finding a run of `count` free seats in one row is a scan with a running counter, which is `O(rows * cols)`. Reserving is the dangerous part. Two customers may try to book overlapping seats at the same moment, so the reservation must be atomic and must fail if any seat has already been taken.

The code loads the free seats into a grid, finds a candidate block, then reserves the seats with a conditional `UPDATE ... WHERE status = 'free'` inside a transaction. If any seat is no longer free, the update affects zero rows, an error is raised, and the transaction rolls back every seat held so far. The in-memory grid is only a suggestion. The database statement is the authority.

```python
# seat_map.py
import logging
import sqlite3
from collections.abc import Sequence

logger = logging.getLogger(__name__)

type Seat = tuple[int, int]

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS seat (
    row_number INTEGER NOT NULL,
    seat_number INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'free' CHECK (status IN ('free', 'held')),
    PRIMARY KEY (row_number, seat_number)
)
"""

HOLD_SQL = (
    "UPDATE seat SET status = 'held' "
    "WHERE row_number = ? AND seat_number = ? AND status = 'free'"
)


class SeatUnavailableError(Exception):
    """Raised when a requested seat is missing or already held."""


class NoAdjacentSeatsError(Exception):
    """Raised when no row has enough adjacent free seats."""


def create_seats(connection: sqlite3.Connection, rows: int, seats_per_row: int) -> None:
    if rows < 1 or seats_per_row < 1:
        raise ValueError("rows and seats_per_row must be at least 1")
    cells = [(r, s) for r in range(rows) for s in range(seats_per_row)]
    with connection:
        connection.executemany(
            "INSERT OR IGNORE INTO seat (row_number, seat_number) VALUES (?, ?)", cells
        )


def load_free_grid(
    connection: sqlite3.Connection, rows: int, seats_per_row: int
) -> list[list[bool]]:
    """Build a rows by seats_per_row grid where True means the seat is free."""
    grid = [[False] * seats_per_row for _ in range(rows)]
    cursor = connection.execute(
        "SELECT row_number, seat_number FROM seat WHERE status = 'free'"
    )
    for row_number, seat_number in cursor:
        if 0 <= row_number < rows and 0 <= seat_number < seats_per_row:
            grid[row_number][seat_number] = True
    return grid


def find_adjacent_seats(grid: Sequence[Sequence[bool]], count: int) -> list[Seat] | None:
    """First block of `count` adjacent free seats in one row. O(rows * cols)."""
    if count < 1:
        raise ValueError("count must be at least 1")
    for row_number, row in enumerate(grid):
        run = 0
        for seat_number, is_free in enumerate(row):
            run = run + 1 if is_free else 0
            if run == count:
                first = seat_number - count + 1
                return [(row_number, seat) for seat in range(first, seat_number + 1)]
    return None


def hold_seats(connection: sqlite3.Connection, seats: Sequence[Seat]) -> None:
    """Hold every seat or none. Raises SeatUnavailableError on the first conflict."""
    try:
        with connection:  # an exception inside this block rolls everything back
            for row_number, seat_number in seats:
                cursor = connection.execute(HOLD_SQL, (row_number, seat_number))
                if cursor.rowcount != 1:
                    raise SeatUnavailableError(
                        f"seat row {row_number}, number {seat_number} is not available"
                    )
    except sqlite3.Error:
        logger.exception("could not hold %d seats", len(seats))
        raise


def book_adjacent(
    connection: sqlite3.Connection, rows: int, seats_per_row: int, count: int
) -> list[Seat]:
    """Find a block of adjacent seats and hold it."""
    grid = load_free_grid(connection, rows, seats_per_row)
    seats = find_adjacent_seats(grid, count)
    if seats is None:
        raise NoAdjacentSeatsError(f"no row has {count} adjacent free seats")
    hold_seats(connection, seats)
    logger.info("held seats %s", seats)
    return seats
```

```python
# test_seat_map.py
import sqlite3
from collections.abc import Iterator

import pytest

from seat_map import (
    NoAdjacentSeatsError,
    SeatUnavailableError,
    book_adjacent,
    create_seats,
    find_adjacent_seats,
    hold_seats,
    load_free_grid,
)

SCHEMA = """
CREATE TABLE seat (
    row_number INTEGER NOT NULL,
    seat_number INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'free' CHECK (status IN ('free', 'held')),
    PRIMARY KEY (row_number, seat_number)
)
"""


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    conn.execute(SCHEMA)
    create_seats(conn, rows=2, seats_per_row=5)
    yield conn
    conn.close()


def status_of(connection: sqlite3.Connection, row: int, seat: int) -> str:
    cursor = connection.execute(
        "SELECT status FROM seat WHERE row_number = ? AND seat_number = ?", (row, seat)
    )
    return str(cursor.fetchone()[0])


def test_find_adjacent_seats_skips_broken_runs() -> None:
    grid = [
        [True, False, True, True, False],
        [True, True, True, False, True],
    ]
    assert find_adjacent_seats(grid, 2) == [(0, 2), (0, 3)]
    assert find_adjacent_seats(grid, 3) == [(1, 0), (1, 1), (1, 2)]
    assert find_adjacent_seats(grid, 4) is None


def test_find_adjacent_seats_rejects_bad_counts() -> None:
    with pytest.raises(ValueError):
        find_adjacent_seats([[True]], 0)


def test_load_free_grid_reflects_held_seats(connection: sqlite3.Connection) -> None:
    hold_seats(connection, [(0, 1)])
    grid = load_free_grid(connection, rows=2, seats_per_row=5)
    assert grid[0] == [True, False, True, True, True]
    assert grid[1] == [True] * 5


def test_book_adjacent_holds_the_block(connection: sqlite3.Connection) -> None:
    seats = book_adjacent(connection, rows=2, seats_per_row=5, count=3)
    assert seats == [(0, 0), (0, 1), (0, 2)]
    assert status_of(connection, 0, 0) == "held"
    assert status_of(connection, 0, 3) == "free"


def test_booking_moves_to_the_next_row_when_needed(
    connection: sqlite3.Connection,
) -> None:
    book_adjacent(connection, rows=2, seats_per_row=5, count=3)  # row 0, seats 0-2
    second = book_adjacent(connection, rows=2, seats_per_row=5, count=3)
    assert second == [(1, 0), (1, 1), (1, 2)]


def test_conflict_rolls_back_every_hold(connection: sqlite3.Connection) -> None:
    hold_seats(connection, [(0, 1)])
    with pytest.raises(SeatUnavailableError):
        hold_seats(connection, [(0, 2), (0, 1)])  # (0, 2) is taken before the conflict
    assert status_of(connection, 0, 2) == "free"


def test_unknown_seat_is_unavailable(connection: sqlite3.Connection) -> None:
    with pytest.raises(SeatUnavailableError):
        hold_seats(connection, [(9, 9)])


def test_no_block_large_enough(connection: sqlite3.Connection) -> None:
    with pytest.raises(NoAdjacentSeatsError):
        book_adjacent(connection, rows=2, seats_per_row=5, count=6)
```

Two concurrent customers can both choose the same block from their own copy of the grid. The second `UPDATE` then sees a seat that is no longer free, raises `SeatUnavailableError`, and rolls back, so the caller can reload the grid and try again.

## Common mistakes

- Building a grid with `[[0] * cols] * rows`, which makes every row the same list.
- Swapping row and column order, so a rectangular grid raises `IndexError` or silently reads the wrong cell.
- Checking only the upper bound of an index. Negative indexes wrap around and hide the bug.
- Using `len(grid[0])` for every row of a jagged grid.
- Copying a grid with `grid[:]` or `grid.copy()` and then changing cells through the supposed copy.
- Forgetting the `top < bottom` and `left < right` guards in spiral traversal, so single rows or columns are visited twice.
- Rotating by the wrong formula. Test with a non-square matrix and a small square matrix and compare against a hand-drawn result.
- Using `zip(*matrix)` on rows of different lengths and losing data silently.
- Marking a cell as visited only when it is removed from the queue instead of when it is added, so the same cell is queued many times.
- Using `x` and `y` in some places and `row` and `col` in others.
- Treating the in-memory grid as the source of truth in a system where other users can change the data. Let the database enforce the rule.

## Practice exercises

1. Write `make_grid(rows, cols, fill)` that returns a correct grid, and a test that proves rows are independent objects.
2. Write `transpose(matrix)` without `zip`, using index loops, and compare its output with the `zip` version on non-square matrices.
3. Write `rotate_counterclockwise(matrix)` in place for a square matrix and test it against a hand-computed 3 by 3 example.
4. Write `diagonal_traversal(matrix)` that returns items in diagonal order starting from the top-left, for rectangular matrices.
5. Write `set_zeroes(matrix)` that sets the entire row and column of every zero to zero, in place, using the first row and column as markers to keep the extra space `O(1)`.
6. Write `search_matrix_flat(matrix, target)` for a matrix where each row begins after the previous row ends, using binary search on the flat index `mid // cols` and `mid % cols`.
7. Write `flood_fill(image, row, col, new_color)` that recolors a connected region of equal-colored cells, with a stack or queue instead of recursion.
8. Write `shortest_path(grid, start, end)` that returns the fewest steps through a grid of open and blocked cells with breadth-first search, or `-1` if there is no path.
9. Write `max_island_area(grid)` that returns the size of the largest connected group of ones.
10. Write `game_of_life_step(board)` that computes the next generation of Conway's Game of Life from a grid of 0 and 1 values, using eight neighbors and no in-place mistakes. Return a new grid first, then do it in place.
11. Write `is_valid_sudoku(board)` that checks rows, columns, and 3 by 3 boxes of a partially filled 9 by 9 board using sets.
12. Write `rotting_oranges(grid)` that returns the number of minutes until every fresh orange rots, using multi-source breadth-first search from all rotten cells at once.
13. Write `spiral_matrix(n)` that generates an `n` by `n` matrix filled with `1` to `n * n` in spiral order.
14. Extend the cinema booking code so that `book_adjacent` retries up to three times when `hold_seats` raises `SeatUnavailableError`, reloading the grid each time, and test it by simulating a competing booking between the load and the hold.

## Summary

A grid is a list of lists indexed as `grid[row][col]`. Build it with a comprehension so each row is a separate list, keep one coordinate convention, and traverse in row-major order for locality. Direction arrays replace repetitive neighbor code, and bounds checks must test both the lower and the upper limits because negative indexes wrap around silently. Transposing swaps rows and columns, a clockwise rotation is a transpose followed by reversing each row, spiral traversal shrinks four boundaries with guards for single rows and columns, and a matrix sorted by rows and columns can be searched from a corner in `O(rows + cols)`. Every cell is also a graph vertex, so flood fill, islands, and shortest paths on grids are graph algorithms that never build an adjacency list. In real systems the grid is a view, and conditional updates inside a transaction decide what is actually true.
