## Overview

A stack is a collection where the last item added is the first item removed, a rule called last in, first out (LIFO). Picture a stack of plates: you put a new plate on top, and when you need one, you take it from the top. You can only touch the top.

That restriction sounds limiting, but it describes a surprising amount of real computing. Your program's own function calls form a stack. A browser's back button and an editor's undo history are stacks. Checking that brackets match, evaluating arithmetic, and parsing nested structures all rely on one. Even an algorithm that looks like it has nothing to do with stacks, such as finding the next larger value for every item in a list, becomes simple and fast once you keep a stack of candidates.

In Python, a stack needs no special class: a plain `list` with `append` and `pop` does the job in `O(1)` per operation. This note covers the LIFO idea, using a `list` as a stack, balanced brackets, expression evaluation and the shunting-yard algorithm, undo stacks, the min stack, monotonic stacks with next greater element and the largest rectangle in a histogram, and replacing recursion with an explicit stack.

## Core concepts

### The three operations

- Push: add an item on top. `stack.append(item)`. `O(1)` amortized.
- Pop: remove and return the top item. `stack.pop()`. `O(1)`.
- Peek: look at the top without removing it. `stack[-1]`. `O(1)`.

Popping or peeking an empty stack is an error, so check `if stack:` first or handle `IndexError`.

### Using a list as a stack

A Python list is a dynamic array, and `append` and `pop()` at the end are both constant time. Never push and pop at the front of a list (`insert(0, x)`, `pop(0)`): those shift every item and cost `O(n)`. `collections.deque` also works as a stack, and `queue.LifoQueue` provides a thread-safe version.

### Balanced brackets

A string of brackets is balanced if every opener has a matching closer of the same type in the right order. Scan left to right. Push each opener. When you see a closer, the top of the stack must be its matching opener, which you pop. At the end the stack must be empty. The stack remembers the most recent unmatched opener, which is exactly the one the next closer has to match.

### Expression evaluation and reverse Polish notation

Postfix notation, also called reverse Polish notation (RPN), writes the operator after its operands: `3 4 +` means `3 + 4`. It needs no parentheses and no precedence rules. Evaluating it is a stack algorithm: push numbers, and when you see an operator, pop two numbers, apply the operator, and push the result. At the end one number remains.

### The shunting-yard algorithm

Humans write infix notation: `3 + 4 * 2`. The shunting-yard algorithm converts infix to postfix with an operator stack. It reads tokens left to right. Numbers go straight to the output. An operator first pops any operators on the stack that bind at least as tightly (higher precedence, or equal precedence for left-associative operators) onto the output, then is pushed. A left parenthesis is pushed; a right parenthesis pops operators to the output until the matching left parenthesis. At the end, the remaining operators are popped. The result can then be evaluated with the stack method above.

### Undo stacks

An undo feature records each action on a stack. Undo pops the latest action and reverses it. A second stack, the redo stack, holds undone actions: undo moves an action from one stack to the other, and redo moves it back. Making a new action clears the redo stack, because the old future no longer applies.

### The min stack

A min stack supports push, pop, and "what is the smallest item?" all in `O(1)`. Alongside each item it stores the minimum of everything at or below it, so the current minimum is always on top.

### Monotonic stacks

A monotonic stack keeps its items in sorted order, either increasing or decreasing, by popping items that break the order before pushing a new one. Each item is pushed once and popped at most once, so the whole scan is `O(n)`. Monotonic stacks solve "find the next larger or smaller item" problems.

- Next greater element: for each item, the first larger item to its right.
- Largest rectangle in a histogram: for each bar, how far it can extend left and right.

### Replacing recursion with a stack

Recursion uses the call stack to remember pending work. An explicit stack does the same thing under your control: push the work still to do, pop the next piece, process it, and push whatever it spawns. This avoids Python's recursion limit and is how depth-first search is written iteratively.

## Basic syntax examples

### A list as a stack

```python
stack: list[int] = []
stack.append(10)  # push
stack.append(20)
stack.append(30)
print(stack[-1])  # 30, peek at the top without removing it
print(stack.pop())  # 30, pop
print(stack)  # [10, 20]
print(len(stack), bool(stack))  # 2 True

stack.clear()
try:
    stack.pop()
except IndexError as error:
    print(error)  # pop from empty list
```

### A small Stack class

```python
class Stack[T]:
    """A LIFO stack with clear error messages."""

    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        if not self._items:
            raise IndexError("pop from empty stack")
        return self._items.pop()

    def peek(self) -> T:
        if not self._items:
            raise IndexError("peek at empty stack")
        return self._items[-1]

    def __len__(self) -> int:
        return len(self._items)

    def __bool__(self) -> bool:
        return bool(self._items)


numbers = Stack[int]()
numbers.push(1)
numbers.push(2)
print(numbers.pop(), numbers.peek(), len(numbers))  # 2 1 1
```

### Balanced brackets

```python
PAIRS = {")": "(", "]": "[", "}": "{"}
OPENERS = frozenset(PAIRS.values())


def is_balanced(text: str) -> bool:
    """O(n) time, O(n) space for the stack of unmatched openers."""
    stack: list[str] = []
    for char in text:
        if char in OPENERS:
            stack.append(char)
        elif char in PAIRS:
            if not stack or stack.pop() != PAIRS[char]:
                return False  # a closer with no opener, or the wrong kind
    return not stack  # leftover openers mean the string is not balanced


print(is_balanced("{[()]}"))  # True
print(is_balanced("([)]"))  # False
print(is_balanced("(()"))  # False
print(is_balanced("a(b)c"))  # True, other characters are ignored
```

Trace of `([)]`:

```text
"("  push           stack: (
"["  push           stack: ( [
")"  top is "[", but ")" needs "(": mismatch, return False
```

### Evaluating reverse Polish notation

```python
def evaluate_rpn(tokens: list[str]) -> int:
    """Evaluate postfix tokens such as ['2', '1', '+', '3', '*']. Integer division."""
    stack: list[int] = []
    for token in tokens:
        if token in {"+", "-", "*", "/"}:
            right = stack.pop()  # the second operand is on top
            left = stack.pop()
            if token == "+":
                stack.append(left + right)
            elif token == "-":
                stack.append(left - right)
            elif token == "*":
                stack.append(left * right)
            else:
                stack.append(int(left / right))  # truncates toward zero
        else:
            stack.append(int(token))
    return stack[0]


print(evaluate_rpn(["2", "1", "+", "3", "*"]))  # 9, which is (2 + 1) * 3
print(evaluate_rpn(["4", "13", "5", "/", "+"]))  # 6, which is 4 + 13 / 5
```

### Infix to postfix: the shunting-yard algorithm

```python
PRECEDENCE = {"+": 1, "-": 1, "*": 2, "/": 2}


def to_postfix(tokens: list[str]) -> list[str]:
    """Convert infix tokens with + - * / and parentheses to postfix."""
    output: list[str] = []
    operators: list[str] = []
    for token in tokens:
        if token in PRECEDENCE:
            # Pop operators that bind at least as tightly (all are left associative).
            while (
                operators
                and operators[-1] != "("
                and PRECEDENCE[operators[-1]] >= PRECEDENCE[token]
            ):
                output.append(operators.pop())
            operators.append(token)
        elif token == "(":
            operators.append(token)
        elif token == ")":
            while operators and operators[-1] != "(":
                output.append(operators.pop())
            if not operators:
                raise ValueError("unbalanced ')'")
            operators.pop()  # discard the "("
        else:
            output.append(token)  # a number goes straight to the output
    while operators:
        operator = operators.pop()
        if operator == "(":
            raise ValueError("unbalanced '('")
        output.append(operator)
    return output


postfix = to_postfix(["3", "+", "4", "*", "2"])
print(postfix)  # ['3', '4', '2', '*', '+']
print(evaluate_rpn(postfix))  # 11
print(evaluate_rpn(to_postfix(["(", "3", "+", "4", ")", "*", "2"])))  # 14
```

Trace of `3 + 4 * 2`:

```text
token  action                                 output        operators
3      number to output                       3
+      push                                   3             +
4      number to output                       3 4           +
*      higher than +, so push                 3 4           + *
2      number to output                       3 4 2         + *
end    pop all operators                      3 4 2 * +
```

### Undo and redo with two stacks

```python
class TextEditor:
    """Each edit is a (before, after) pair of the whole text, for simplicity."""

    def __init__(self) -> None:
        self.text = ""
        self._undo: list[str] = []
        self._redo: list[str] = []

    def type(self, addition: str) -> None:
        self._undo.append(self.text)  # remember the state before this edit
        self.text += addition
        self._redo.clear()  # a new edit discards the undone future

    def undo(self) -> bool:
        if not self._undo:
            return False
        self._redo.append(self.text)
        self.text = self._undo.pop()
        return True

    def redo(self) -> bool:
        if not self._redo:
            return False
        self._undo.append(self.text)
        self.text = self._redo.pop()
        return True


editor = TextEditor()
editor.type("Hello")
editor.type(", world")
editor.undo()
print(editor.text)  # Hello
editor.redo()
print(editor.text)  # Hello, world
```

### A min stack

```python
class MinStack:
    """push, pop, top, and minimum are all O(1)."""

    def __init__(self) -> None:
        self._items: list[tuple[int, int]] = []  # (value, minimum so far)

    def push(self, value: int) -> None:
        current_min = min(value, self._items[-1][1]) if self._items else value
        self._items.append((value, current_min))

    def pop(self) -> int:
        return self._items.pop()[0]

    def minimum(self) -> int:
        return self._items[-1][1]


stack_with_min = MinStack()
for value in (5, 3, 7, 3, 8):
    stack_with_min.push(value)
print(stack_with_min.minimum())  # 3
stack_with_min.pop()
stack_with_min.pop()
stack_with_min.pop()
print(stack_with_min.minimum())  # 3, the first 3 is still there
stack_with_min.pop()
print(stack_with_min.minimum())  # 5
```

### Next greater element with a monotonic stack

```python
def next_greater(values: list[int]) -> list[int]:
    """For each item, the next larger value to its right, or -1. O(n)."""
    result = [-1] * len(values)
    waiting: list[int] = []  # indexes whose answer is not known yet; values decrease
    for index, value in enumerate(values):
        while waiting and values[waiting[-1]] < value:
            result[waiting.pop()] = value  # this value is the answer for those items
        waiting.append(index)
    return result


print(next_greater([2, 1, 2, 4, 3]))  # [4, 2, 4, -1, -1]
```

### Largest rectangle in a histogram

```python
def largest_rectangle(heights: list[int]) -> int:
    """Area of the largest rectangle under a histogram. O(n)."""
    stack: list[int] = []  # indexes of bars with increasing heights
    best = 0
    extended = [*heights, 0]  # a final zero-height bar flushes the stack
    for index, height in enumerate(extended):
        while stack and extended[stack[-1]] >= height:
            bar_height = extended[stack.pop()]
            left = stack[-1] if stack else -1  # the nearest shorter bar on the left
            width = index - left - 1  # the bar can extend from left + 1 to index - 1
            best = max(best, bar_height * width)
        stack.append(index)
    return best


print(largest_rectangle([2, 1, 5, 6, 2, 3]))  # 10, the bars 5 and 6 with height 5
```

### Depth-first search without recursion

```python
def dfs_order(graph: dict[str, list[str]], start: str) -> list[str]:
    """Visit nodes depth first with an explicit stack instead of recursion."""
    visited: set[str] = set()
    order: list[str] = []
    stack = [start]
    while stack:
        node = stack.pop()
        if node in visited:
            continue
        visited.add(node)
        order.append(node)
        stack.extend(reversed(graph.get(node, [])))  # reversed keeps the listed order
    return order


graph = {"a": ["b", "c"], "b": ["d"], "c": ["d"], "d": []}
print(dfs_order(graph, "a"))  # ['a', 'b', 'd', 'c']
```

## Deeper behavior

### Why LIFO fits nested structure

Nesting is the hallmark of problems that use stacks. In `{[()]}`, the innermost bracket closes first, so the most recently opened bracket must be the first to close. The stack holds the open scopes in the order they were entered, and closing always involves the innermost one. Function calls, nested JSON, XML tags, and parenthesized expressions all have the same shape.

### Amortized cost of the monotonic stack

The `while` loop inside the `for` loop looks quadratic. It is not: each index is pushed once and popped at most once, so the total number of pops across the whole run is at most `n`. The total work is `O(n)`. The same counting argument makes the largest-rectangle algorithm linear.

### Why the monotonic stack finds next greater

The stack holds indexes whose next greater element has not appeared yet, and their values are in decreasing order from bottom to top. When a new value arrives, every stacked value smaller than it has just found its answer: this new value is the first larger one to its right, because anything larger and earlier would already have resolved them. Pop those, record the answer, and push the new index. Items still on the stack at the end have no greater element, so they keep `-1`.

### Largest rectangle: what each pop means

When a bar is popped because the current bar is shorter or equal, two facts are known. The bar below it on the stack is the nearest shorter bar on the left. The current bar is the first shorter bar on the right. So the popped bar's height can extend over the whole range between them, and the rectangle's width is `index - left - 1`. Every bar is popped exactly once, so every candidate rectangle is considered.

### Operator precedence and associativity

Shunting-yard needs two properties of each operator.

- Precedence: `*` binds tighter than `+`.
- Associativity: for a left-associative operator such as `-`, `8 - 3 - 2` means `(8 - 3) - 2`, so an equal-precedence operator already on the stack must be popped first. For a right-associative operator such as exponentiation, `2 ^ 3 ^ 2` means `2 ^ (3 ^ 2)`, so you do not pop an equal-precedence operator, and the `>=` becomes `>`.

Unary minus (`-3`, or `2 * -3`) needs extra care: a minus is unary when it appears at the start, after another operator, or after `(`. Treat it as a separate high-precedence, right-associative operator.

### Min stack: why it works

Each entry stores the minimum of everything beneath it, including itself. Pushing computes the new minimum from the previous top in `O(1)`. Popping restores the earlier state automatically, because the previous entry still carries its own minimum. No search is ever needed. The cost is one extra number per item.

### Undo stacks in real systems

Real editors usually store commands rather than whole snapshots: each command object knows how to apply and how to reverse itself. Snapshots are simple but take memory proportional to the text size per edit. Commands take memory proportional to the edit size. Bounded history, where the oldest entries are dropped after a limit such as 100, keeps memory under control and is typically done with a `deque(maxlen=...)`.

### Stack depth and Python's call stack

Each Python function call occupies a frame on the interpreter's call stack, limited to about 1,000 nested calls by default. An explicit stack lives on the heap and can hold millions of items, so converting deep recursion to an explicit stack removes that limit. The conversion is mechanical: push the starting work item, loop while the stack is not empty, pop an item, process it, and push the follow-up items. The order in which you push children decides the traversal order, because the last pushed child is popped first.

### Choosing the container

- `list`: fastest and simplest for a single-threaded stack.
- `collections.deque`: also `O(1)` at both ends, with `maxlen` for bounded stacks.
- `queue.LifoQueue`: a stack safe to share between threads, with blocking `get` and an optional size limit. It adds lock overhead, so use it only across threads.

### Unbounded stacks

An input that keeps pushing without popping, such as a long run of opening brackets in untrusted text, grows the stack in proportion to the input. Cap the depth when the input comes from users, and fail with a clear error instead of consuming all memory.

## Industry-level snippet

A shop lets staff define pricing rules in a small formula language, for example `base * (1 - discount) + shipping`. The rule is stored in the database and evaluated for each order. Python's `eval` would run arbitrary code and must never be used on stored text, so the service parses the formula itself with the stack algorithms from this note.

The pipeline is a tokenizer, the shunting-yard conversion to postfix with a stack of operators, and a postfix evaluator with a stack of numbers. It supports `+`, `-`, `*`, `/`, parentheses, unary minus, decimal numbers, and named variables, and it uses `Decimal` so that money is never computed with binary floats. Every failure is a specific error: a syntax mistake, an unknown variable, or division by zero. When a rule is saved, it is validated first, including a check that it uses only the allowed variable names, so broken rules never reach the database. Saving uses a transaction.

```python
# pricing_rules.py
import logging
import re
import sqlite3
from collections.abc import Collection, Mapping, Sequence
from dataclasses import dataclass
from decimal import Decimal

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS pricing_rule (
    name TEXT PRIMARY KEY,
    expression TEXT NOT NULL
)
"""

TOKEN_PATTERN = re.compile(r"\s*(?:(\d+(?:\.\d+)?)|([A-Za-z_][A-Za-z0-9_]*)|(.))")

# symbol -> (precedence, right associative)
OPERATORS: dict[str, tuple[int, bool]] = {
    "+": (1, False),
    "-": (1, False),
    "*": (2, False),
    "/": (2, False),
    "neg": (3, True),  # unary minus
}


class RuleSyntaxError(ValueError):
    """Raised when a formula is malformed."""


class UnknownVariableError(ValueError):
    """Raised when a formula uses a variable that is not defined."""


class RuleMathError(ValueError):
    """Raised for arithmetic problems such as division by zero."""


class RuleNotFoundError(LookupError):
    """Raised when no rule with the given name is stored."""


@dataclass(frozen=True, slots=True)
class Token:
    kind: str  # "number", "name", "op", "lparen", or "rparen"
    text: str


def tokenize(expression: str) -> list[Token]:
    expression = expression.strip()
    tokens: list[Token] = []
    position = 0
    while position < len(expression):
        match = TOKEN_PATTERN.match(expression, position)
        if match is None:
            raise RuleSyntaxError(f"cannot read the formula at position {position}")
        number, name, other = match.groups()
        position = match.end()
        if number:
            tokens.append(Token("number", number))
        elif name:
            tokens.append(Token("name", name))
        elif other in ("+", "-", "*", "/"):
            tokens.append(Token("op", other))
        elif other == "(":
            tokens.append(Token("lparen", other))
        elif other == ")":
            tokens.append(Token("rparen", other))
        else:
            raise RuleSyntaxError(f"unexpected character {other!r}")
    return tokens


def to_rpn(tokens: Sequence[Token]) -> list[Token]:
    """Shunting-yard: infix tokens to postfix, with unary minus support."""
    output: list[Token] = []
    stack: list[Token] = []
    expect_operand = True  # True at the start, after an operator, and after "("
    for token in tokens:
        if token.kind in ("number", "name"):
            if not expect_operand:
                raise RuleSyntaxError("missing operator before a value")
            output.append(token)
            expect_operand = False
        elif token.kind == "op":
            symbol = token.text
            if expect_operand:
                if symbol != "-":
                    raise RuleSyntaxError(f"unexpected operator {symbol!r}")
                symbol = "neg"
            precedence, right_associative = OPERATORS[symbol]
            while stack and stack[-1].kind == "op":
                top_precedence, _ = OPERATORS[stack[-1].text]
                if top_precedence > precedence or (
                    top_precedence == precedence and not right_associative
                ):
                    output.append(stack.pop())
                else:
                    break
            stack.append(Token("op", symbol))
            expect_operand = True
        elif token.kind == "lparen":
            if not expect_operand:
                raise RuleSyntaxError("missing operator before '('")
            stack.append(token)
        else:  # rparen
            if expect_operand:
                raise RuleSyntaxError("missing value before ')'")
            while stack and stack[-1].kind != "lparen":
                output.append(stack.pop())
            if not stack:
                raise RuleSyntaxError("unbalanced ')'")
            stack.pop()
    if expect_operand:
        raise RuleSyntaxError("the formula is empty or ends with an operator")
    while stack:
        top = stack.pop()
        if top.kind == "lparen":
            raise RuleSyntaxError("unbalanced '('")
        output.append(top)
    return output


def evaluate_rpn(rpn: Sequence[Token], variables: Mapping[str, Decimal]) -> Decimal:
    stack: list[Decimal] = []
    try:
        for token in rpn:
            if token.kind == "number":
                stack.append(Decimal(token.text))
            elif token.kind == "name":
                if token.text not in variables:
                    raise UnknownVariableError(f"unknown variable {token.text!r}")
                stack.append(variables[token.text])
            elif token.text == "neg":
                stack.append(-stack.pop())
            else:
                right = stack.pop()
                left = stack.pop()
                if token.text == "+":
                    stack.append(left + right)
                elif token.text == "-":
                    stack.append(left - right)
                elif token.text == "*":
                    stack.append(left * right)
                else:
                    if right == 0:
                        raise RuleMathError("division by zero")
                    stack.append(left / right)
    except ArithmeticError as error:
        raise RuleMathError(str(error)) from error
    return stack[0]


def evaluate(expression: str, variables: Mapping[str, Decimal]) -> Decimal:
    return evaluate_rpn(to_rpn(tokenize(expression)), variables)


def validate_rule(expression: str, allowed_variables: Collection[str]) -> None:
    """Parse the formula and check every variable name against the allowed set."""
    for token in to_rpn(tokenize(expression)):
        if token.kind == "name" and token.text not in allowed_variables:
            raise UnknownVariableError(f"unknown variable {token.text!r}")


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def save_rule(
    connection: sqlite3.Connection,
    name: str,
    expression: str,
    allowed_variables: Collection[str],
) -> None:
    """Validate first, then store. A broken formula never reaches the database."""
    if not name:
        raise ValueError("rule name must not be empty")
    validate_rule(expression, allowed_variables)
    try:
        with connection:
            connection.execute(
                "INSERT INTO pricing_rule (name, expression) VALUES (?, ?) "
                "ON CONFLICT (name) DO UPDATE SET expression = excluded.expression",
                (name, expression),
            )
    except sqlite3.Error:
        logger.exception("could not save rule %r", name)
        raise
    logger.info("saved pricing rule %r", name)


def price(
    connection: sqlite3.Connection, rule_name: str, variables: Mapping[str, Decimal]
) -> Decimal:
    row = connection.execute(
        "SELECT expression FROM pricing_rule WHERE name = ?", (rule_name,)
    ).fetchone()
    if row is None:
        raise RuleNotFoundError(f"no pricing rule named {rule_name!r}")
    return evaluate(row[0], variables)
```

```python
# test_pricing_rules.py
import sqlite3
from collections.abc import Iterator
from decimal import Decimal

import pytest

from pricing_rules import (
    RuleMathError,
    RuleNotFoundError,
    RuleSyntaxError,
    UnknownVariableError,
    create_schema,
    evaluate,
    price,
    save_rule,
)

NO_VARIABLES: dict[str, Decimal] = {}


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


@pytest.mark.parametrize(
    ("expression", "expected"),
    [
        ("1 + 2 * 3", "7"),
        ("(1 + 2) * 3", "9"),
        ("10 / 4", "2.5"),
        ("2 - 3 - 4", "-5"),
        ("-2 * 3", "-6"),
        ("2 * -3", "-6"),
        ("--2", "2"),
        ("0.1 + 0.2", "0.3"),
        ("  7  ", "7"),
        ("((2))", "2"),
        ("8 / 2 / 2", "2"),
    ],
)
def test_arithmetic(expression: str, expected: str) -> None:
    assert evaluate(expression, NO_VARIABLES) == Decimal(expected)


def test_variables_are_substituted() -> None:
    variables = {
        "base": Decimal("100"),
        "discount": Decimal("0.15"),
        "shipping": Decimal("5"),
    }
    assert evaluate("base * (1 - discount) + shipping", variables) == Decimal("90")


@pytest.mark.parametrize(
    "expression",
    ["(1 + 2", "1 + 2)", "1 +", "", "()", "1 2", "2 $ 3", "*3", "1 + * 2", "(1)(2)"],
)
def test_syntax_errors(expression: str) -> None:
    with pytest.raises(RuleSyntaxError):
        evaluate(expression, NO_VARIABLES)


def test_unknown_variable() -> None:
    with pytest.raises(UnknownVariableError):
        evaluate("price * 2", NO_VARIABLES)


def test_division_by_zero() -> None:
    with pytest.raises(RuleMathError):
        evaluate("1 / (2 - 2)", NO_VARIABLES)


def test_saved_rules_are_validated_and_evaluated(
    connection: sqlite3.Connection,
) -> None:
    allowed = {"base", "discount"}
    save_rule(connection, "sale", "base * (1 - discount)", allowed)
    variables = {"base": Decimal("200"), "discount": Decimal("0.25")}
    result = price(connection, "sale", variables)
    assert result == Decimal("150")


def test_saving_again_replaces_the_formula(connection: sqlite3.Connection) -> None:
    save_rule(connection, "r", "base * 2", {"base"})
    save_rule(connection, "r", "base * 3", {"base"})
    assert price(connection, "r", {"base": Decimal("1")}) == Decimal("3")


def test_invalid_rules_are_not_stored(connection: sqlite3.Connection) -> None:
    with pytest.raises(RuleSyntaxError):
        save_rule(connection, "bad", "base *", {"base"})
    with pytest.raises(UnknownVariableError):
        save_rule(connection, "worse", "secret * 2", {"base"})
    with pytest.raises(RuleNotFoundError):
        price(connection, "bad", {"base": Decimal("1")})


def test_empty_rule_name_is_rejected(connection: sqlite3.Connection) -> None:
    with pytest.raises(ValueError):
        save_rule(connection, "", "1", set())
```

The tokenizer accepts only numbers, identifiers, the four operators, and parentheses, so there is no path from stored text to executable code. The parser is also strict about structure, so `"(1)(2)"` and `"1 2"` are rejected instead of being guessed at.

## Common mistakes

- Popping or peeking an empty stack without checking, which raises `IndexError`.
- Using `list.insert(0, x)` or `pop(0)` for a stack. Push and pop at the end.
- Forgetting to check that the stack is empty at the end of bracket matching, so `"(("` passes.
- Popping before checking whether the stack is empty, so a leading closer such as `")("` crashes.
- Comparing the closer with the wrong opener, or checking only the count of brackets and not their types and order.
- Getting the operand order wrong when evaluating postfix. For `-` and `/` the first popped item is the right operand.
- Using `>` instead of `>=` for left-associative operators in shunting-yard, which gives `8 - 3 - 2 = 7` instead of `3`.
- Treating a unary minus like binary subtraction.
- Using `eval` on formula strings. It executes arbitrary code.
- Computing money with binary floats instead of `Decimal` or integer cents.
- In a monotonic stack, storing values when you need indexes (or the reverse), and losing the positions you need for widths and answers.
- Forgetting to flush the stack at the end of a monotonic-stack algorithm. A sentinel value or a final loop handles the leftovers.
- Recursing deeply on untrusted input instead of using an explicit stack with a limit.

## Practice exercises

1. Write `reverse_string(text)` with a stack, then explain why slicing is better in real code.
2. Write `longest_valid_parentheses(text)` that returns the length of the longest balanced substring of `(` and `)`, using a stack of indexes.
3. Write `remove_adjacent_duplicates(text)` that repeatedly removes pairs of equal adjacent characters, for example `"abbaca"` becomes `"ca"`, with one stack pass.
4. Write `decode_string(text)` that expands patterns such as `"3[a2[c]]"` into `"accaccacc"`, using a stack of counts and partial strings.
5. Extend `to_postfix` to support the right-associative `^` operator, and test that `2 ^ 3 ^ 2` equals 512.
6. Write `simplify_path(path)` that turns a Unix path such as `"/a/./b/../../c/"` into the canonical form `"/c"` using a stack of directory names.
7. Write `daily_temperatures(temps)` that returns, for each day, how many days until a warmer one, using a monotonic stack.
8. Write `next_smaller(values)` and `previous_greater(values)` by adapting the monotonic stack, and test them against brute force on random lists.
9. Write `trap_rain_water(heights)` with a monotonic stack, and compare it with a two-pointer solution.
10. Write `stock_span(prices)` that returns, for each day, the number of consecutive previous days with a price at most today's price.
11. Implement `MinStack` with a second stack that records only when the minimum changes, and compare its memory use with the version above.
12. Implement a queue with two stacks (`enqueue`, `dequeue`) and argue why its operations are amortized `O(1)`.
13. Convert a recursive function that computes all subsets of a list into an iterative one with an explicit stack, and verify both give the same subsets.
14. Extend the pricing-rule language with a `min(a, b)` function and comma-separated arguments, and describe what the shunting-yard algorithm must do for function calls.

## Summary

A stack is last in, first out, and a Python `list` with `append` and `pop` implements it with `O(1)` operations. Stacks suit anything nested: bracket matching pushes openers and pops on closers, postfix evaluation pops two operands per operator, and shunting-yard uses an operator stack to turn infix into postfix while respecting precedence and associativity. Undo and redo are two stacks, a min stack stores the running minimum with each item, and a monotonic stack answers next-greater and histogram questions in `O(n)` because every item is pushed and popped once. An explicit stack replaces recursion, removing Python's depth limit. In production, parse untrusted formulas with your own tokenizer, parser, and stack evaluator instead of `eval`, use `Decimal` for money, and cap the depth of inputs from outside.
