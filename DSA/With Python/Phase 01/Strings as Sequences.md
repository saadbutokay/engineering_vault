## Overview

A string is a sequence of characters, and most of what you learned about lists carries over: you can index it, slice it, loop over it, and ask for its length. The big difference is that Python strings are immutable. You can never change a string in place, only build a new one. That one fact explains the most common string performance problem, building a long string with `+=` in a loop, and it shapes how you write string algorithms.

Strings are also where beginners first meet text encoding. A character on your screen is not always one item in a Python string, and two strings that look identical can compare as different. Handling this correctly is what separates a toy program from a backend that accepts usernames, names, and addresses from real people around the world.

This note covers immutability and the cost of concatenation, `str.join`, character frequency, comparison, palindromes, anagrams, `ord` and `chr`, ASCII versus Unicode, parsing numbers by hand, run-length compression, and validating formatted input.

## Core concepts

### Strings are immutable sequences

A `str` supports the sequence operations: `len(s)`, `s[i]`, `s[a:b]`, `for char in s`, and `char in s`. It does not support item assignment: `s[0] = "x"` raises `TypeError`. Every method that seems to modify a string (`upper`, `replace`, `strip`) returns a new string and leaves the original untouched.

### The cost of repeated concatenation

`a + b` builds a new string of length `len(a) + len(b)`, copying both. Doing this in a loop copies the growing result again and again, which is `O(n^2)` in the worst case. The standard fix is to collect the pieces in a list and join once.

- `parts.append(piece)` is `O(1)` amortized.
- `"".join(parts)` is `O(total length)`, one allocation.

### Character frequency

Counting how often each character appears is the foundation of anagram checks, palindrome permutations, and many interview problems. Use a dictionary (or `Counter`) for any character set, or a fixed list of 26 counters when the input is limited to lowercase ASCII letters.

### Comparison

Strings compare lexicographically, character by character, by code point (the number returned by `ord`). `"apple" < "banana"` is true because `"a" < "b"`. Uppercase letters have smaller code points than lowercase ones, so `"Z" < "a"`. Comparing two strings costs up to `O(min(len(a), len(b)))`, not `O(1)`.

### Palindromes and anagrams

- A palindrome reads the same forwards and backwards. Check it with two pointers moving inward.
- Two strings are anagrams if they contain the same characters with the same counts. Compare sorted copies in `O(n log n)`, or compare frequency tables in `O(n)`.

### ord and chr

`ord(char)` returns the integer code point of a one-character string. `chr(number)` does the reverse. Subtracting `ord("a")` turns a lowercase letter into an index from 0 to 25, which is how fixed-size frequency tables work.

### ASCII and Unicode

- ASCII is the old standard with 128 characters: English letters, digits, punctuation, and control codes.
- Unicode assigns a number (a code point) to over a million characters from every writing system. A Python `str` is a sequence of code points.
- Encodings such as UTF-8 turn code points into bytes. In UTF-8 an ASCII character takes 1 byte and other characters take 2 to 4 bytes.
- What a person sees as one character may be more than one code point, for example a letter followed by a combining accent.

### Parsing numbers by hand

Converting `"1234"` to the integer 1234 means processing characters left to right: `value = value * 10 + digit`. Writing this once teaches you how `int()` works and how to validate input strictly instead of trusting it.

### Run-length compression

Run-length encoding replaces each run of the same character by the character and its count: `"aaabcc"` becomes `"a3b1c2"`. It is a good exercise in scanning a string while tracking a current run.

### Validating formatted input

Many real inputs have a fixed format: an IPv4 address (four numbers separated by dots), a date, an invoice number. Validation means splitting the input, checking each piece against explicit rules, and rejecting anything else.

## Basic syntax examples

### Indexing, slicing, and common methods

```python
word = "Python"

print(word[0], word[-1])  # P n
print(word[1:4])  # yth
print(word[::-1])  # nohtyP, a reversed copy
print(len(word))  # 6

print(word.upper(), word.lower())  # PYTHON python
print("  padded  ".strip())  # padded
print("a,b,c".split(","))  # ['a', 'b', 'c']
print("-".join(["2026", "11", "08"]))  # 2026-11-08
print("ab" * 3)  # ababab
print("th" in word)  # True
print(word.startswith("Py"), word.find("th"), word.find("zz"))  # True 2 -1
print(word.replace("th", "TH"))  # PyTHon
```

### Strings cannot be changed in place

```python
name = "kiwi"
try:
    name[0] = "K"  # type: ignore[index]
except TypeError as error:
    print(error)  # 'str' object does not support item assignment

name = "K" + name[1:]  # build a new string and rebind the name
print(name)  # Kiwi
```

### Building a string efficiently

```python
def numbers_to_text(numbers: list[int]) -> str:
    """Join the pieces once instead of concatenating in a loop."""
    parts: list[str] = []
    for number in numbers:
        parts.append(str(number))
    return ",".join(parts)


print(numbers_to_text([3, 1, 4, 1, 5]))  # 3,1,4,1,5
```

### ord and chr

```python
print(ord("a"), ord("A"), ord("0"))  # 97 65 48
print(chr(97), chr(ord("a") + 1))  # a b
print(ord("d") - ord("a"))  # 3, the position of "d" in the alphabet
```

### Character frequency

```python
def char_frequency(text: str) -> dict[str, int]:
    """O(n) time, O(k) space for k distinct characters."""
    counts: dict[str, int] = {}
    for char in text:
        counts[char] = counts.get(char, 0) + 1
    return counts


def letter_counts(word: str) -> list[int]:
    """Lowercase ASCII only: a fixed table of 26 slots, so O(1) extra space."""
    counts = [0] * 26
    for char in word:
        counts[ord(char) - ord("a")] += 1
    return counts


print(char_frequency("banana"))  # {'b': 1, 'a': 3, 'n': 2}
print(letter_counts("banana")[:3])  # [3, 1, 0]
```

### Palindrome with two pointers

```python
def is_palindrome(text: str) -> bool:
    """Ignore case and characters that are not letters or digits.

    O(n) time and O(1) extra space: no cleaned copy of the string is built.
    """
    left, right = 0, len(text) - 1
    while left < right:
        if not text[left].isalnum():
            left += 1
        elif not text[right].isalnum():
            right -= 1
        elif text[left].casefold() != text[right].casefold():
            return False
        else:
            left += 1
            right -= 1
    return True


print(is_palindrome("A man, a plan, a canal: Panama"))  # True
print(is_palindrome("hello"))  # False
```

### Anagrams

```python
from collections import Counter


def is_anagram_sorted(first: str, second: str) -> bool:
    """O(n log n): sorting makes equal multisets look identical."""
    return sorted(first) == sorted(second)


def is_anagram(first: str, second: str) -> bool:
    """O(n): compare character counts."""
    return len(first) == len(second) and Counter(first) == Counter(second)


print(is_anagram("listen", "silent"))  # True
print(is_anagram("rat", "car"))  # False
```

### Parsing an integer by hand

```python
def parse_int(text: str) -> int:
    """Parse an optional sign followed by ASCII digits. Stricter than int()."""
    text = text.strip()
    if not text:
        raise ValueError("empty string")
    sign = 1
    start = 0
    if text[0] in "+-":
        sign = -1 if text[0] == "-" else 1
        start = 1
    if start == len(text):
        raise ValueError(f"no digits in {text!r}")
    value = 0
    for char in text[start:]:
        if not ("0" <= char <= "9"):
            raise ValueError(f"invalid character {char!r} in {text!r}")
        value = value * 10 + (ord(char) - ord("0"))
    return sign * value


print(parse_int("  -1234 "))  # -1234
```

Trace of `parse_int("-123")`:

```text
sign = -1, digits start at index 1
char "1": value = 0 * 10 + 1   = 1
char "2": value = 1 * 10 + 2   = 12
char "3": value = 12 * 10 + 3  = 123
result: -1 * 123 = -123
```

### Run-length encoding

```python
def rle_encode(text: str) -> str:
    """'aaabcc' -> 'a3b1c2'. O(n) because the pieces are joined once.

    Digits in the input would make the output ambiguous, so they are rejected.
    """
    if any("0" <= char <= "9" for char in text):
        raise ValueError("input must not contain digits")
    if not text:
        return ""
    parts: list[str] = []
    current = text[0]
    run = 1
    for char in text[1:]:
        if char == current:
            run += 1
        else:
            parts.append(f"{current}{run}")
            current = char
            run = 1
    parts.append(f"{current}{run}")
    return "".join(parts)


def rle_decode(encoded: str) -> str:
    parts: list[str] = []
    i = 0
    while i < len(encoded):
        char = encoded[i]
        i += 1
        start = i
        while i < len(encoded) and "0" <= encoded[i] <= "9":
            i += 1
        if start == i:
            raise ValueError(f"missing run length after {char!r}")
        parts.append(char * int(encoded[start:i]))
    return "".join(parts)


print(rle_encode("aaabcc"))  # a3b1c2
print(rle_decode("a3b1c2"))  # aaabcc
```

### Validating a formatted input

```python
def is_valid_ipv4(text: str) -> bool:
    """Four decimal numbers from 0 to 255 separated by dots, no leading zeros."""
    pieces = text.split(".")
    if len(pieces) != 4:
        return False
    for piece in pieces:
        if not piece or not all("0" <= char <= "9" for char in piece):
            return False
        if len(piece) > 1 and piece[0] == "0":
            return False
        if int(piece) > 255:
            return False
    return True


print(is_valid_ipv4("192.168.1.10"))  # True
print(is_valid_ipv4("192.168.01.10"))  # False, leading zero
print(is_valid_ipv4("256.1.1.1"))  # False
```

## Deeper behavior

### Immutability and what it implies

Immutability makes strings safe to share and hash. A `str` can be a dictionary key because its contents can never change under the dictionary's feet, and the hash value is computed once and cached inside the string object. The price is that every "change" allocates a new string. Operations like `replace` and `strip` cost `O(n)` because they copy.

Slicing also copies: `text[1:]` builds a new string of length `n - 1`. A loop that repeatedly slices the front of a long string does `n + (n - 1) + (n - 2) + ...` work, which is quadratic. Walk the string with an index instead.

### Why join wins

Conceptually `result += piece` builds a new string every time. CPython has an optimization that can extend a string in place when nothing else refers to it, so a simple loop is often fast on CPython, but that is an implementation detail that does not hold on every Python or in every situation. `"".join(parts)` first adds up the lengths, allocates once, and copies each piece once. It is guaranteed linear. Join takes any iterable of strings, so a generator expression works too: `"".join(str(n) for n in numbers)`.

### Code points, bytes, and what len measures

`len(text)` counts code points, not bytes and not what a person would call characters.

```python
import unicodedata

print(len("e\u0301"), len("\u00e9"))  # 2 1, both look like the letter e with an accent
print("e\u0301" == "\u00e9")  # False, different code point sequences
print(unicodedata.normalize("NFC", "e\u0301") == "\u00e9")  # True after normalizing
print(len("\u00e9".encode("utf-8")))  # 2, the byte length differs from len()
```

The two spellings of the accented letter are the same to a human and different to `==`. Normalization (`unicodedata.normalize`) converts text to a standard form so comparisons work. NFC composes characters where possible. NFKC also folds compatibility characters such as full-width Latin letters into plain ones: normalizing `"\uff21\uff2c\uff29"` gives `"ALI"`.

### Case-insensitive comparison

`lower()` is not enough for every language. `casefold()` is the aggressive version meant for caseless matching.

```python
print("stra\u00dfe".casefold() == "STRASSE".casefold())  # True
print("stra\u00dfe".lower() == "STRASSE".lower())  # False
```

### isdigit, isdecimal, and int

String predicates are Unicode-aware, which can surprise you when validating input.

```python
print("\u00b2".isdigit(), "\u00b2".isdecimal())  # True False, superscript two
print(int("\u09e7\u09e8\u09e9"))  # 123, Bengali digits are accepted by int()
print("\u09e7\u09e8\u09e9".isascii())  # False
```

`int()` accepts decimal digits from any script. If your system must accept only ASCII digits, as in an invoice number or an API field, check the range `"0" <= char <= "9"` explicitly or test `text.isascii()` first, as the strict parsers above do.

### Comparison cost and sorting strings

Equality and ordering compare character by character and stop at the first difference, so comparing two long strings that share a long prefix is expensive. Sorting `n` strings of length up to `m` costs `O(m * n log n)` in the worst case, not `O(n log n)`, because each comparison can cost `O(m)`.

### Frequency tables and space

A dictionary uses space proportional to the number of distinct characters. For a known small alphabet, a fixed list or `array` is smaller and faster. Counting is also the basis of many string algorithms: two strings are anagrams if their count tables match, and a string can be rearranged into a palindrome if at most one character has an odd count.

### Interning

CPython may keep a single copy of short identifier-like strings and reuse it ("interning"), so two equal strings are sometimes the same object. Never use `is` to compare strings. Always use `==`.

## Industry-level snippet

A web application lets people choose a username. Real users will try `Ali`, `ali`, a full-width `\uff21\uff2c\uff29`, or a name with a trailing space, and the system must treat all of those as the same name, so only one account can use it. The standard technique is to store two forms: the display name the user typed (cleaned up) and a normalized key with a unique index. Normalization combines Unicode NFKC, stripping, and `casefold`. Validation uses explicit character rules rather than hoping the database rejects bad data.

Registration runs inside a transaction. If the unique index reports a duplicate, the function translates the database error into a clear application error. Any other database failure is logged and re-raised.

```python
# usernames.py
import logging
import sqlite3
import unicodedata
from dataclasses import dataclass

logger = logging.getLogger(__name__)

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS app_user (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    username_key TEXT NOT NULL UNIQUE
)
"""

MIN_LENGTH = 3
MAX_LENGTH = 20
ALLOWED = frozenset("abcdefghijklmnopqrstuvwxyz0123456789_")
RESERVED = frozenset({"admin", "root", "support"})


class InvalidUsernameError(ValueError):
    """Raised when a username does not satisfy the naming rules."""


class UsernameTakenError(Exception):
    """Raised when the normalized username already exists."""


@dataclass(frozen=True, slots=True)
class Username:
    display: str
    key: str


def normalize_username(raw: str) -> Username:
    """Clean the display name and derive the case-insensitive unique key."""
    display = unicodedata.normalize("NFKC", raw).strip()
    key = display.casefold()
    if not MIN_LENGTH <= len(key) <= MAX_LENGTH:
        raise InvalidUsernameError(
            f"username must be {MIN_LENGTH} to {MAX_LENGTH} characters long"
        )
    unsupported = sorted(set(key) - ALLOWED)
    if unsupported:
        raise InvalidUsernameError(f"unsupported characters: {unsupported!r}")
    if key in RESERVED:
        raise InvalidUsernameError(f"{key!r} is reserved")
    return Username(display=display, key=key)


def create_schema(connection: sqlite3.Connection) -> None:
    connection.execute(SCHEMA_SQL)


def register_user(connection: sqlite3.Connection, raw_username: str) -> int:
    """Create the account atomically and return its id."""
    username = normalize_username(raw_username)
    try:
        with connection:
            cursor = connection.execute(
                "INSERT INTO app_user (username, username_key) VALUES (?, ?)",
                (username.display, username.key),
            )
    except sqlite3.IntegrityError as error:
        logger.info("username %r is already taken", username.key)
        raise UsernameTakenError(username.key) from error
    except sqlite3.Error:
        logger.exception("could not register username %r", username.key)
        raise
    if cursor.lastrowid is None:
        raise RuntimeError("database did not return a row id")
    logger.info("registered user %d as %r", cursor.lastrowid, username.display)
    return cursor.lastrowid
```

```python
# test_usernames.py
import sqlite3
from collections.abc import Iterator

import pytest

from usernames import (
    InvalidUsernameError,
    UsernameTakenError,
    create_schema,
    normalize_username,
    register_user,
)


@pytest.fixture
def connection() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(":memory:")
    create_schema(conn)
    yield conn
    conn.close()


def test_normalization_strips_and_casefolds() -> None:
    username = normalize_username("  Ali_99 ")
    assert username.display == "Ali_99"
    assert username.key == "ali_99"


def test_fullwidth_letters_fold_to_ascii() -> None:
    username = normalize_username("\uff21\uff2c\uff29_1")
    assert username.display == "ALI_1"
    assert username.key == "ali_1"


@pytest.mark.parametrize(
    "raw",
    ["ab", "x" * 21, "a b c", "caf\u00e9", "bad-name", "Admin", "   "],
)
def test_invalid_usernames_are_rejected(raw: str) -> None:
    with pytest.raises(InvalidUsernameError):
        normalize_username(raw)


def test_register_returns_distinct_ids(connection: sqlite3.Connection) -> None:
    first = register_user(connection, "alice")
    second = register_user(connection, "bobby")
    assert first != second


def test_duplicate_differing_only_by_case_is_rejected(
    connection: sqlite3.Connection,
) -> None:
    register_user(connection, "Alice")
    with pytest.raises(UsernameTakenError):
        register_user(connection, "aLICE")
    count = connection.execute("SELECT COUNT(*) FROM app_user").fetchone()[0]
    assert count == 1


def test_invalid_name_never_reaches_the_database(
    connection: sqlite3.Connection,
) -> None:
    with pytest.raises(InvalidUsernameError):
        register_user(connection, "no spaces")
    count = connection.execute("SELECT COUNT(*) FROM app_user").fetchone()[0]
    assert count == 0
```

## Common mistakes

- Building a long string with `+=` in a loop. Collect pieces in a list and call `"".join(pieces)`.
- Trying to assign to a character, `s[0] = "x"`. Build a new string instead.
- Slicing in a loop or recursion (`s = s[1:]`), which copies the remainder every time.
- Comparing strings with `is`. Use `==`; identity depends on interning.
- Using `lower()` for caseless comparison across languages. Use `casefold()`.
- Comparing text from different sources without normalizing it first, so visually identical strings are not equal.
- Assuming one character is one byte, or that `len` equals the number of visible characters.
- Using `isdigit()` or `int()` to validate that input contains only ASCII digits. Superscripts and other scripts pass or fail in surprising ways.
- Forgetting the empty string. `""` is a valid input to every string function you write.
- Assuming a frequency table of 26 slots handles uppercase letters, spaces, or digits. `ord(char) - ord("a")` goes negative or too large and either raises `IndexError` or silently indexes from the end.
- Decoding run-length strings when the original text may contain digits, which makes the format ambiguous.

## Practice exercises

1. Write `reverse_words(sentence)` that reverses the order of words and collapses repeated spaces. Do it with `split` and `join`.
2. Write `is_palindrome_permutation(text)` that tells whether the letters of `text` can be rearranged into a palindrome. Use a frequency table and the odd-count rule.
3. Write `first_unique_char(text)` that returns the index of the first character that appears exactly once, or `-1`. Make it `O(n)`.
4. Write `compress(text)` that returns the run-length encoding only if it is shorter than the original, and the original otherwise.
5. Write `longest_common_prefix(words)` for a list of strings. State the time complexity in terms of the number of words and their lengths.
6. Write `parse_float(text)` by hand for inputs such as `"-12.50"`. Reject inputs with two dots, no digits, or stray characters.
7. Implement `is_valid_date(text)` for the format `YYYY-MM-DD`, including the number of days in each month and leap years, without `datetime`.
8. Write `caesar_cipher(text, shift)` that shifts only ASCII letters and keeps case and punctuation unchanged. Use `ord` and `chr` with modular arithmetic.
9. Write `are_isomorphic(first, second)` that checks whether the characters of one string can be consistently replaced to produce the other, for example `"egg"` and `"add"`.
10. Show with `unicodedata.normalize` that two differently written accented words are equal after NFC. Then write `same_text(a, b)` that compares them ignoring case and normalization form.
11. Time `+=` in a loop and `"".join` for 10,000, 100,000, and 1,000,000 pieces. Record the ratio at each size and explain what you see.
12. Count how many bytes `"hello"`, an accented word of your choice, and a word in a non-Latin script take in UTF-8 and compare with `len`.
13. Extend `normalize_username` to reject names that start with a digit or an underscore, and add parametrized tests for the new rule.
14. Write `mask_email(address)` that keeps the first character of the local part and the whole domain, replacing the rest of the local part with `*`, and validates that there is exactly one `@`.

## Summary

A Python string is an immutable sequence of Unicode code points. Indexing is `O(1)`, but slicing and concatenation copy, so building text piece by piece should collect parts in a list and use `str.join`. Frequency tables, two-pointer scans, and sorted or counted comparisons solve palindromes and anagrams, `ord` and `chr` connect characters to numbers, and parsing by hand teaches strict validation. Unicode means `len` is not the number of visible characters, equal-looking text may need normalization, `casefold` is the right tool for caseless matching, and `int()` accepts digits from many scripts. Real systems store a clean display form and a normalized key, and let the database enforce uniqueness inside a transaction.
