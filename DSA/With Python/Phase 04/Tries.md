## Overview

A trie (pronounced "try", from re-trie-val), also called a prefix tree, stores a set of strings in a tree where each edge is labeled with one character and each path from the root spells a prefix. Words that share a beginning share the same path. The words `car`, `card`, and `care` share the path `c`, `a`, `r`, and then branch.

That sharing is what makes a trie good at one family of questions: questions about prefixes. Is there any word that starts with `ca`? Which words start with `car`? What is the longest prefix that all these words have in common? A hash set answers "is this exact word present?" in `O(1)` but cannot answer a prefix question without scanning every word. A trie answers it by walking one character at a time, so the cost depends on the length of the prefix and not on how many words are stored.

Tries power autocomplete, spell checkers, IP routing tables (using bits instead of letters), dictionary lookups in word games, and keyword filters. This note covers the structure, insert, search, and prefix queries, dictionary-based versus array-based nodes, autocomplete, word search on a grid, longest common prefix, compressed tries (radix trees), and the memory cost of a trie in Python, which is high and deserves an honest discussion.

## Core concepts

### Structure

Each node represents a prefix. It holds:

- a mapping from a character to a child node, and
- a flag saying whether the path from the root to this node is a complete word.

The root represents the empty prefix. The flag matters because a word can be a prefix of another word. In `car` and `card`, the node for `car` is a word and also has a child `d`.

```text
(root)
  c
  + a
    + r *          (* marks the end of a word)
      + d *
      + e *
```

### Insert

Walk from the root, following the child for each character of the word. When a child is missing, create it. After the last character, mark the node as a word. The cost is `O(L)` for a word of length `L`, regardless of how many words are stored.

### Search and prefix queries

- Search: walk the characters of the word. If any child is missing, the word is absent. If you reach the end, the word is present only if the final node is marked as a word.
- Prefix test (`starts_with`): the same walk, but you only need the nodes to exist, not to be marked.
- All words with a prefix: walk to the node for the prefix, then traverse the subtree below it, collecting every marked node.

All of these cost `O(L)` for the walk, plus the size of the output for the collection step.

### Dictionary-based versus array-based nodes

- Dictionary-based: each node has a `dict` from character to child. It handles any alphabet, including Unicode, and uses memory only for the children that exist. It is the natural choice in Python.
- Array-based: each node has a fixed-size list with one slot per letter of a small alphabet (26 for lowercase English). Lookup is an index calculation, `ord(char) - ord("a")`. It is fast and simple, but every node pays for all 26 slots even if it has one child.

### Autocomplete

Given a prefix typed by a user, return the best completions. The simple version walks to the prefix node and collects all words below it, then ranks them. That can be slow when the prefix is short and matches thousands of words. A faster design stores, at every node, the best few completions for that prefix, so a query is just a walk of the prefix length and a read of a short list.

### Word search on a grid

Given a grid of letters and a list of words, find which words can be formed by moving between adjacent cells without reusing a cell. Searching for each word separately repeats the same exploration. Building a trie of all the words and doing one depth-first search from each cell, following only the trie paths that exist, prunes the search: as soon as the letters on the path are not a prefix of any word, stop.

### Longest common prefix

The longest common prefix of a set of words is the path from the root that has exactly one child at every step and is not itself the end of a word. Walk down while those conditions hold.

### Compressed tries (radix trees)

Most of the nodes in a trie have exactly one child, forming long chains. A compressed trie, also called a radix tree or Patricia trie, merges each chain into a single edge labeled with a string instead of a character. It stores the same information in far fewer nodes. Routing tables, file systems, and many databases use them.

### Memory costs in Python

A Python trie node is an object with a dictionary, and a dictionary is large. A single-child node costs more than 200 bytes, many times the size of the character it represents. A set of strings, by contrast, costs a little over the length of each string. A trie therefore uses much more memory than a plain collection of the same words, and it wins only because of the queries it makes fast. That is a reason to compress, to use array-based nodes only for dense data, or to use a sorted list with binary search when the word list is fixed.

## Basic syntax examples

### A dictionary-based trie

```python
class TrieNode:
    __slots__ = ("children", "is_word")

    def __init__(self) -> None:
        self.children: dict[str, TrieNode] = {}
        self.is_word = False


class Trie:
    """A set of strings with fast prefix queries."""

    def __init__(self) -> None:
        self._root = TrieNode()
        self._size = 0

    @property
    def root(self) -> TrieNode:
        return self._root

    def __len__(self) -> int:
        return self._size

    def insert(self, word: str) -> bool:
        """Add a word in O(len(word)). Returns False if it was already present."""
        node = self._root
        for char in word:
            child = node.children.get(char)
            if child is None:
                child = TrieNode()
                node.children[char] = child
            node = child
        if node.is_word:
            return False
        node.is_word = True
        self._size += 1
        return True

    def _walk(self, text: str) -> TrieNode | None:
        node = self._root
        for char in text:
            child = node.children.get(char)
            if child is None:
                return None
            node = child
        return node

    def search(self, word: str) -> bool:
        node = self._walk(word)
        return node is not None and node.is_word

    def starts_with(self, prefix: str) -> bool:
        return self._walk(prefix) is not None

    def words_with_prefix(self, prefix: str, limit: int | None = None) -> list[str]:
        """Words below the prefix, in alphabetical order, with an explicit stack."""
        start = self._walk(prefix)
        if start is None:
            return []
        results: list[str] = []
        stack: list[tuple[TrieNode, str]] = [(start, prefix)]
        while stack:
            node, text = stack.pop()
            if node.is_word:
                results.append(text)
                if limit is not None and len(results) >= limit:
                    break
            for char in sorted(node.children, reverse=True):  # smallest pops first
                stack.append((node.children[char], text + char))
        return results

    def delete(self, word: str) -> bool:
        """Remove a word and prune the nodes that are no longer needed."""
        path: list[tuple[TrieNode, str]] = []  # (parent, character) along the word
        node = self._root
        for char in word:
            child = node.children.get(char)
            if child is None:
                return False
            path.append((node, char))
            node = child
        if not node.is_word:
            return False
        node.is_word = False
        self._size -= 1
        for parent, char in reversed(path):  # prune from the leaf upwards
            child = parent.children[char]
            if child.children or child.is_word:
                break  # this node is still needed by another word
            del parent.children[char]
        return True


trie = Trie()
for word in ("car", "card", "care", "cat", "dog"):
    trie.insert(word)

print(trie.search("car"), trie.search("ca"), trie.search("cards"))  # True False False
print(trie.starts_with("ca"), trie.starts_with("cb"))  # True False
print(trie.words_with_prefix("car"))  # ['car', 'card', 'care']
print(trie.words_with_prefix(""))  # ['car', 'card', 'care', 'cat', 'dog']
trie.delete("card")
print(trie.words_with_prefix("car"), len(trie))  # ['car', 'care'] 4
```

The words `car`, `card`, `care`, `cat`, and `dog`, as a tree:

```text
(root)
  + c
  | + a
  |   + r *
  |   | + d *
  |   | + e *
  |   + t *
  + d
    + o
      + g *
```

### An array-based node and the memory cost

```python
import sys


class ArrayTrieNode:
    """One slot per lowercase letter, so lookups are an index calculation."""

    __slots__ = ("children", "is_word")

    def __init__(self) -> None:
        self.children: list[ArrayTrieNode | None] = [None] * 26
        self.is_word = False


def insert_array(root: ArrayTrieNode, word: str) -> None:
    node = root
    for char in word:
        index = ord(char) - ord("a")  # only a-z is allowed
        child = node.children[index]
        if child is None:
            child = ArrayTrieNode()
            node.children[index] = child
        node = child
    node.is_word = True


dict_node = TrieNode()
dict_node.children["a"] = TrieNode()
print(sys.getsizeof(dict_node) + sys.getsizeof(dict_node.children))  # 232

array_node = ArrayTrieNode()
print(sys.getsizeof(array_node) + sys.getsizeof(array_node.children))  # 312
```

These sizes are for CPython 3.12 on a 64-bit machine. A node with one child costs about 230 bytes with a dictionary and about 310 bytes with an array of 26 slots, and each node represents a single character. The array version wins only when most of the 26 slots are in use.

### Longest common prefix

```python
def longest_common_prefix(words: list[str]) -> str:
    """Walk down while there is exactly one way to go and no word ends here."""
    trie = Trie()
    for word in words:
        trie.insert(word)
    node = trie.root
    prefix: list[str] = []
    while len(node.children) == 1 and not node.is_word:
        char, node = next(iter(node.children.items()))
        prefix.append(char)
    return "".join(prefix)


print(longest_common_prefix(["flower", "flow", "flight"]))  # fl
print(longest_common_prefix(["dog", "racecar", "car"]))  # (an empty string)
print(longest_common_prefix(["interview", "internet", "interval"]))  # inter
```

### Word search on a grid

```python
def find_words(board: list[list[str]], words: list[str]) -> list[str]:
    """Words that can be traced through adjacent cells, never reusing a cell."""
    trie = Trie()
    for word in words:
        trie.insert(word)
    rows = len(board)
    cols = len(board[0]) if board else 0
    found: set[str] = set()
    visited: set[tuple[int, int]] = set()

    def explore(row: int, col: int, node: TrieNode, path: str) -> None:
        child = node.children.get(board[row][col])
        if child is None:
            return  # no word starts with these letters: prune this whole branch
        path += board[row][col]
        if child.is_word:
            found.add(path)
        visited.add((row, col))
        for d_row, d_col in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            next_row, next_col = row + d_row, col + d_col
            if (
                0 <= next_row < rows
                and 0 <= next_col < cols
                and (next_row, next_col) not in visited
            ):
                explore(next_row, next_col, child, path)
        visited.remove((row, col))  # backtrack so other paths may use this cell

    for row in range(rows):
        for col in range(cols):
            explore(row, col, trie.root, "")
    return sorted(found)


board = [
    ["o", "a", "a", "n"],
    ["e", "t", "a", "e"],
    ["i", "h", "k", "r"],
    ["i", "f", "l", "v"],
]
print(find_words(board, ["oath", "pea", "eat", "rain"]))  # ['eat', 'oath']
```

### A compressed trie (radix tree)

```python
from dataclasses import dataclass, field


@dataclass(slots=True)
class RadixNode:
    label: str = ""  # the string on the edge leading into this node
    children: dict[str, "RadixNode"] = field(default_factory=dict)  # by first letter
    is_word: bool = False


def radix_insert(root: RadixNode, word: str) -> None:
    node = root
    while True:
        if not word:
            node.is_word = True
            return
        child = node.children.get(word[0])
        if child is None:
            node.children[word[0]] = RadixNode(word, is_word=True)
            return
        common = 0  # length of the shared prefix of the edge label and the word
        while (
            common < len(child.label)
            and common < len(word)
            and child.label[common] == word[common]
        ):
            common += 1
        if common < len(child.label):  # the word diverges inside the edge: split it
            rest = RadixNode(child.label[common:], child.children, child.is_word)
            child.label = child.label[:common]
            child.children = {rest.label[0]: rest}
            child.is_word = False
        node = child
        word = word[common:]


def radix_search(root: RadixNode, word: str) -> bool:
    node = root
    while word:
        child = node.children.get(word[0])
        if child is None or not word.startswith(child.label):
            return False
        word = word[len(child.label) :]
        node = child
    return node.is_word


def count_radix_nodes(node: RadixNode) -> int:
    return 1 + sum(count_radix_nodes(child) for child in node.children.values())


def count_trie_nodes(node: TrieNode) -> int:
    return 1 + sum(count_trie_nodes(child) for child in node.children.values())


names = ["romane", "romanus", "romulus", "rubens", "ruber", "rubicon", "rubicundus"]
radix_root = RadixNode()
plain_trie = Trie()
for name in names:
    radix_insert(radix_root, name)
    plain_trie.insert(name)

print(radix_search(radix_root, "ruber"), radix_search(radix_root, "rub"))  # True False
print(count_trie_nodes(plain_trie.root), count_radix_nodes(radix_root))  # 28 14
```

The radix tree for those seven words, where each edge shows a string:

```text
r
+ om
| + an
| | + e *
| | + us *
| + ulus *
+ ub
  + e
  | + ns *
  | + r *
  + ic
    + on *
    + undus *
```

The plain trie needs 28 nodes. The radix tree needs 14, because every chain of single-child nodes collapses into one edge.

### An alternative for a fixed dictionary: a sorted list

```python
from bisect import bisect_left


def words_with_prefix_sorted(words: list[str], prefix: str) -> list[str]:
    """Words must be sorted. O(log n) to find the start, then the matches."""
    result: list[str] = []
    for index in range(bisect_left(words, prefix), len(words)):
        if not words[index].startswith(prefix):
            break  # sorted order keeps all matches together
        result.append(words[index])
    return result


dictionary = sorted(["car", "card", "care", "cat", "dog"])
print(words_with_prefix_sorted(dictionary, "car"))  # ['car', 'card', 'care']
```

If the word list does not change, this answers the same prefix question with a fraction of the memory of a trie. A trie is the better choice when words are inserted and removed often, or when you attach data to every prefix.

## Deeper behavior

### Cost, precisely

- Insert, search, and prefix test: `O(L)`, where `L` is the length of the string. They do not depend on the number of words, which is the trie's strength compared with scanning a list, and it holds regardless of the alphabet size when children are stored in a dictionary.
- Listing the words under a prefix: `O(L + size of the subtree)`.
- Space: up to `O(total number of characters)` in the worst case, when no two words share a prefix, and much less when prefixes are shared. Each node adds the memory overhead of a Python object and a dictionary.
- Compare with a hash set: `O(L)` to hash and compare a string, so an exact lookup is `O(L)` in both, but the set cannot answer prefix queries without scanning.

### Why a trie is not always smaller

A trie shares prefixes, but sharing only helps when many words begin the same way. For a list of random strings, or for any set where prefixes are mostly unique, the trie stores roughly one node per character and ends up far larger than the strings themselves. Natural-language word lists share many prefixes, but in Python even that is not enough to offset the per-node overhead. The practical options for large vocabularies are a radix tree, which reduces the number of nodes, a trie built on packed arrays instead of objects, or a sorted array with binary search, which is compact and fast for static data. Specialized structures such as a DAWG (directed acyclic word graph) also share suffixes, and finite-state transducers are used in search engines for the same reason.

### Deletion and pruning

Deleting a word means unmarking its last node. If that node has children or is itself the end of another word, nothing else changes. Otherwise it is now useless, so remove it, and check its parent in the same way, climbing until you reach a node that is still needed. If you skip the pruning, the structure stays correct but wastes memory, which matters for long-lived tries that see many removals.

### Choosing an alphabet

- Lowercase letters only: an array of 26 or a dictionary.
- Mixed case and Unicode: a dictionary, and normalize the text first (case folding, Unicode normalization) so that visually identical strings share a path.
- Binary data and numbers: use one bit or one byte per level. An IP routing table uses a trie on the address bits, and "longest prefix match" finds the most specific route.

### Top-K autocomplete

Collecting every word under a short prefix, sorting by popularity, and keeping five is expensive when the prefix matches thousands of words. Storing the best few completions at every node moves that work to insertion time. The cost is memory (a small list on each node) and extra work when a score changes, because every node on the path of that word must be updated. When scores only increase, the update is cheap: replace the term's entry in each node's list and keep the best few. When scores can decrease, a term that dropped out of a node's list might need to be replaced by a term that is not stored there, which requires searching the subtree.

### Recursion and depth

The depth of a trie equals the length of the longest word. Words are normally short, so recursive traversal is usually safe, but input that comes from users can contain very long strings. Limit the length of inserted strings, as the industry snippet does, before they reach the trie.

### Pruning makes backtracking fast

In the word-search example, the trie lets the search stop at the first letter that cannot lead to any word. Without the trie, each of the words would need its own search from every cell, and many paths would be explored that no word could follow. The same pruning idea applies to other backtracking problems that build strings, such as generating words from a rack of letters.

### Tries and other prefix structures

- Sorted list with `bisect`: compact and fast for static data, but insertion is `O(n)`.
- Hash map from every prefix to a list of words: fast queries but memory grows with the sum of all prefix lengths.
- Suffix arrays and suffix automata: handle substring queries (not just prefixes), and are covered in the string algorithms phase.
- Database `LIKE 'abc%'` queries use a B-tree index on the column, which behaves like a trie of pages.

## Industry-level snippet

A shop's search box suggests completions as the customer types. The most popular searches matter more than rarely used ones, and suggestions must appear within a few milliseconds, so the service cannot scan all stored searches for every keystroke. It keeps a trie in memory where every node stores the top five completions for its prefix, so a suggestion request walks one character per keystroke and then reads a short list.

Search counts live in a database table, so they survive restarts. Recording a search increments the count in a transaction, and only after the commit is the trie updated with the new count. On startup the trie is rebuilt from the table. Counts only ever increase, which is what makes the per-node update simple and correct.

Search terms come from the public, so the code is defensive. Terms are normalized (case folded, spaces collapsed), limited in length, and restricted to letters, digits, and spaces. The number of distinct terms is capped, so a stream of unique junk queries cannot make the trie consume all of memory. A rejected term is never written to the database.

```python
# autocomplete.py
import logging
import re
import sqlite3

logger = logging.getLogger(__name__)

TOP_K = 5
MAX_TERM_LENGTH = 64
TERM_PATTERN = re.compile(r"^[a-z0-9 ]+$")

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS search_term (
    term TEXT PRIMARY KEY,
    hits INTEGER NOT NULL CHECK (hits >= 1)
)
"""

COUNT_SQL = (
    "INSERT INTO search_term (term, hits) VALUES (?, 1) "
    "ON CONFLICT (term) DO UPDATE SET hits = hits + 1"
)


class InvalidTermError(ValueError):
    """Raised for a term that is empty, too long, or has unsupported characters."""


class TermLimitError(Exception):
    """Raised when the index already holds the maximum number of distinct terms."""


def normalize_prefix(raw: str) -> str:
    """Case fold and collapse whitespace. An empty result is allowed."""
    prefix = " ".join(raw.casefold().split())
    if len(prefix) > MAX_TERM_LENGTH:
        raise InvalidTermError(f"terms may have at most {MAX_TERM_LENGTH} characters")
    if prefix and not TERM_PATTERN.match(prefix):
        raise InvalidTermError("only letters a-z, digits, and spaces are supported")
    return prefix


def normalize_term(raw: str) -> str:
    term = normalize_prefix(raw)
    if not term:
        raise InvalidTermError("a search term must not be empty")
    return term


class _Node:
    __slots__ = ("children", "top")

    def __init__(self) -> None:
        self.children: dict[str, _Node] = {}
        self.top: list[tuple[int, str]] = []  # up to TOP_K entries, best first


class Autocomplete:
    """A trie where every node remembers the best completions for its prefix."""

    def __init__(self) -> None:
        self._root = _Node()
        self._hits: dict[str, int] = {}

    def __len__(self) -> int:
        return len(self._hits)

    def __contains__(self, term: str) -> bool:
        return term in self._hits

    def set_hits(self, term: str, hits: int) -> None:
        """Add a term or raise its count. Counts may only increase."""
        if hits < self._hits.get(term, 0):
            raise ValueError("hits may only increase")
        self._hits[term] = hits
        node = self._root
        self._update(node, term, hits)  # the empty prefix lists the overall best terms
        for char in term:
            child = node.children.get(char)
            if child is None:
                child = _Node()
                node.children[char] = child
            node = child
            self._update(node, term, hits)

    @staticmethod
    def _update(node: _Node, term: str, hits: int) -> None:
        entries = [entry for entry in node.top if entry[1] != term]
        entries.append((hits, term))
        entries.sort(key=lambda entry: (-entry[0], entry[1]))  # most hits, then A to Z
        del entries[TOP_K:]
        node.top = entries

    def suggest(self, prefix: str, limit: int = TOP_K) -> list[str]:
        """Up to `limit` best completions. Cost is O(len(prefix) + limit)."""
        if not 1 <= limit <= TOP_K:
            raise ValueError(f"limit must be between 1 and {TOP_K}")
        node = self._root
        for char in normalize_prefix(prefix):
            child = node.children.get(char)
            if child is None:
                return []
            node = child
        return [term for _, term in node.top[:limit]]


class SearchSuggestions:
    """Search counts in a database table, with an in-memory trie in front."""

    def __init__(
        self, connection: sqlite3.Connection, max_terms: int = 100_000
    ) -> None:
        if max_terms < 1:
            raise ValueError("max_terms must be at least 1")
        self._connection = connection
        self._max_terms = max_terms
        self._index = Autocomplete()
        for term, hits in connection.execute("SELECT term, hits FROM search_term"):
            self._index.set_hits(term, hits)

    def record_search(self, raw_term: str) -> int:
        """Count one search for the term. Returns its new total."""
        term = normalize_term(raw_term)
        if term not in self._index and len(self._index) >= self._max_terms:
            raise TermLimitError(f"the index already has {self._max_terms} terms")
        try:
            with self._connection:
                self._connection.execute(COUNT_SQL, (term,))
                row = self._connection.execute(
                    "SELECT hits FROM search_term WHERE term = ?", (term,)
                ).fetchone()
        except sqlite3.Error:
            logger.exception("could not record a search for %r", term)
            raise
        hits = int(row[0])
        self._index.set_hits(term, hits)  # only after the commit succeeded
        return hits

    def suggest(self, prefix: str, limit: int = TOP_K) -> list[str]:
        return self._index.suggest(prefix, limit)
```

```python
# test_autocomplete.py
import random
import sqlite3
from collections.abc import Iterator

import pytest

from autocomplete import (
    SCHEMA_SQL,
    TOP_K,
    Autocomplete,
    InvalidTermError,
    SearchSuggestions,
    TermLimitError,
    normalize_term,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    conn.execute(SCHEMA_SQL)
    yield conn
    conn.close()


def test_suggestions_are_ranked_by_hits_then_alphabetically() -> None:
    index = Autocomplete()
    index.set_hits("phone case", 10)
    index.set_hits("phone charger", 30)
    index.set_hits("phone", 30)
    index.set_hits("photo frame", 5)
    expected = ["phone", "phone charger", "phone case", "photo frame"]
    assert index.suggest("pho") == expected
    assert index.suggest("phone c") == ["phone charger", "phone case"]
    assert index.suggest("phone", limit=2) == ["phone", "phone charger"]


def test_unknown_prefix_and_empty_prefix() -> None:
    index = Autocomplete()
    index.set_hits("apple", 3)
    index.set_hits("banana", 9)
    assert index.suggest("zzz") == []
    assert index.suggest("") == ["banana", "apple"]


def test_only_the_best_few_are_kept_per_prefix() -> None:
    index = Autocomplete()
    for number in range(8):
        index.set_hits(f"a{number}", number + 1)
    assert index.suggest("a") == ["a7", "a6", "a5", "a4", "a3"]
    assert len(index.suggest("a")) == TOP_K


def test_a_term_can_climb_into_the_top_list() -> None:
    index = Autocomplete()
    for number in range(6):
        index.set_hits(f"a{number}", 10 + number)
    assert "a0" not in index.suggest("a")
    index.set_hits("a0", 100)
    assert index.suggest("a")[0] == "a0"


def test_counts_cannot_decrease() -> None:
    index = Autocomplete()
    index.set_hits("apple", 5)
    with pytest.raises(ValueError):
        index.set_hits("apple", 4)


def test_random_terms_match_a_brute_force_ranking() -> None:
    rng = random.Random(31)
    index = Autocomplete()
    model: dict[str, int] = {}
    alphabet = "abc"
    for _ in range(400):
        term = "".join(rng.choice(alphabet) for _ in range(rng.randint(1, 5)))
        model[term] = model.get(term, 0) + rng.randint(1, 4)
        index.set_hits(term, model[term])
        prefix = "".join(rng.choice(alphabet) for _ in range(rng.randint(0, 3)))
        expected = sorted(
            (t for t in model if t.startswith(prefix)), key=lambda t: (-model[t], t)
        )[:TOP_K]
        assert index.suggest(prefix) == expected


@pytest.mark.parametrize(
    ("raw", "expected"),
    [("  Red   SHOES ", "red shoes"), ("iPhone 15", "iphone 15"), ("a", "a")],
)
def test_terms_are_normalized(raw: str, expected: str) -> None:
    assert normalize_term(raw) == expected


@pytest.mark.parametrize("raw", ["", "   ", "caf\u00e9", "rock&roll", "x" * 65])
def test_invalid_terms_are_rejected(raw: str) -> None:
    with pytest.raises(InvalidTermError):
        normalize_term(raw)


def test_recorded_searches_are_counted_and_persisted(
    connection: sqlite3.Connection,
) -> None:
    service = SearchSuggestions(connection)
    assert service.record_search("Red Shoes") == 1
    assert service.record_search("red  shoes") == 2
    service.record_search("red shirt")
    assert service.suggest("red") == ["red shoes", "red shirt"]
    reloaded = SearchSuggestions(connection)
    assert reloaded.suggest("red") == ["red shoes", "red shirt"]


def test_invalid_searches_are_not_written(connection: sqlite3.Connection) -> None:
    service = SearchSuggestions(connection)
    with pytest.raises(InvalidTermError):
        service.record_search("<script>")
    count = connection.execute("SELECT COUNT(*) FROM search_term").fetchone()[0]
    assert count == 0


def test_the_number_of_distinct_terms_is_capped(
    connection: sqlite3.Connection,
) -> None:
    service = SearchSuggestions(connection, max_terms=2)
    service.record_search("one")
    service.record_search("two")
    with pytest.raises(TermLimitError):
        service.record_search("three")
    assert service.record_search("two") == 2  # existing terms still count
    assert service.suggest("three") == []


def test_a_failed_write_leaves_the_index_unchanged(
    connection: sqlite3.Connection,
) -> None:
    service = SearchSuggestions(connection)
    service.record_search("alpha")
    connection.execute("DROP TABLE search_term")
    with pytest.raises(sqlite3.OperationalError):
        service.record_search("beta")
    assert service.suggest("") == ["alpha"]
```

Because the top completions are precomputed, a request does work proportional to the length of what the customer has typed so far, however many million terms are stored. The price is paid on writes, which update one short list per character of the term.

## Common mistakes

- Marking only the nodes that have no children as words. A word can be a prefix of another word, so keep an explicit end-of-word flag.
- Returning `True` from `search` when the walk succeeds but the last node is not marked as a word. That is the `starts_with` answer.
- Confusing `search` and `starts_with`, or forgetting that `starts_with("")` is always true.
- Using a trie when a `set` (exact membership) or a sorted list with `bisect` (static prefix queries) would be simpler and much smaller.
- Using array-based nodes for a large alphabet, or for sparse data, and wasting memory on empty slots.
- Letting untrusted input create unlimited nodes. Bound the string length and the number of distinct terms.
- Skipping normalization, so `"Phone"`, `"phone"`, and a spelling with an accented letter take different paths.
- Not pruning on delete, so removed words leave dead branches behind.
- Collecting all words under a short prefix on every keystroke instead of precomputing the best few per node.
- Forgetting to undo the visited mark when backtracking in a grid search, so valid words are missed.
- Searching the grid once per word instead of once with a trie, which repeats the same exploration.
- Updating the in-memory index before the database commit, so a failed write leaves them out of step.

## Practice exercises

1. Insert `tea`, `ten`, `to`, `inn`, and `in` into a trie by hand and draw it, marking the ends of words. Then say what `starts_with("te")` and `search("te")` return.
2. Write `count_words_with_prefix(trie, prefix)` that returns the number of words under a prefix in `O(len(prefix))` by storing a count in every node.
3. Write `shortest_unique_prefix(words)` that returns, for each word, the shortest prefix that identifies it uniquely, using node counts.
4. Write `replace_words(dictionary, sentence)` that replaces each word in a sentence by the shortest dictionary root that is a prefix of it, using a trie.
5. Write `longest_word_built_from_others(words)` that returns the longest word that can be built one letter at a time with every prefix also in the list, using a trie.
6. Write `add_and_search_word` support for the wildcard `"."` that matches any single letter, using depth-first search over the trie.
7. Write `max_xor_pair(numbers)` using a binary trie on the bits of each number, to find the pair with the largest XOR in `O(n * 32)`.
8. Write `suggestions_by_prefix(words, prefix, k)` using a sorted list and `bisect`, and compare its memory and speed with the trie for a 100,000-word list.
9. Add `delete` to the radix tree, merging a node with its only child when a deletion leaves it with a single child and no word marker.
10. Write `ip_longest_prefix_match(routes, address)` that stores route prefixes (such as `192.168.0.0/16`) in a binary trie on address bits and returns the most specific route for an address.
11. Measure the memory of a dictionary-based trie and a `set` for the same 50,000 words using `tracemalloc`, and report the ratio.
12. Extend `Autocomplete` so that scores can also decrease, by rebuilding a node's top list from its subtree when a term drops out of it, and state the cost.
13. Write `find_words` for a larger grid with 1,000 words and measure the speedup of the trie version over searching each word separately.
14. Extend `SearchSuggestions` with a `forget(term)` method that deletes the term in a transaction and removes it from the in-memory index, and describe what you must do to the top lists on the path.

## Summary

A trie stores strings in a tree with one character per edge, so words with a common beginning share a path, and an end-of-word flag marks complete words. Insert, search, and prefix tests cost `O(L)` for a string of length `L`, whatever the number of stored words, and collecting all words under a prefix costs the size of that subtree. Dictionary-based nodes suit any alphabet and sparse data, while array-based nodes suit small, dense alphabets. Tries support autocomplete (especially with the best completions stored at each node), longest common prefix, and pruned backtracking in word searches. A compressed (radix) trie merges single-child chains into labeled edges, cutting the node count, which matters because Python trie nodes are expensive, over 200 bytes each. For a fixed dictionary, a sorted list with `bisect` is often the better answer. In services, normalize and bound untrusted input, cap the number of distinct terms, and update memory only after the database commit.
