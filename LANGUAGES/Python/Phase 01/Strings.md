## Overview

Text is everywhere in software: names, messages, URLs, log lines, file contents, JSON, and every request and response a web service handles. In Python, text is represented by the `str` type. Learning strings well means learning how to slice them, search them, format them, and, most importantly for real systems, how text differs from the raw bytes that travel over networks and into files.

By the end of this note you will be able to:

- Create strings with single, double, and triple quotes, and use escape sequences and raw strings.
- Index and slice strings.
- Use the most common string methods.
- Format output with f-strings, `format()`, and format specifications.
- Explain what Unicode is and how `str` differs from `bytes`.
- Encode and decode text correctly.

> [!note] Strings are immutable sequences of Unicode characters
> Python 3 strings hold Unicode code points, so one string can mix English, Arabic, Bengali, and Chinese. A string never changes after creation. Operations that look like they modify a string return a new one.

## Core concepts

### Creating strings

```python
a = 'single quotes'
b = "double quotes"
c = """triple quotes can
span several lines"""
```

Single and double quotes are identical in meaning. Pick one style (the formatter in Chapter 0.9 prefers double quotes) and use the other when the text itself contains a quote.

### Indexing and slicing

Strings are **sequences**: ordered collections of characters, each at a numbered position starting from `0`. Negative numbers count from the end.

```text
 text:     P   y   t   h   o   n
 index:    0   1   2   3   4   5
 negative: -6  -5  -4  -3  -2  -1
```

A **slice** `text[start:stop:step]` takes a range. The `start` position is included, the `stop` position is excluded, and any part may be omitted.

| Slice | Meaning |
| --- | --- |
| `s[0]` | First character |
| `s[-1]` | Last character |
| `s[1:4]` | Characters at positions 1, 2, 3 |
| `s[:3]` | First three characters |
| `s[3:]` | Everything from position 3 |
| `s[::2]` | Every second character |
| `s[::-1]` | The string reversed |

### Immutability

You cannot assign to a position in a string. `word[0] = "b"` raises `TypeError`. To "change" a string, build a new one.

### Common methods

| Group | Methods |
| --- | --- |
| Case | `upper()`, `lower()`, `title()`, `capitalize()`, `swapcase()`, `casefold()` |
| Whitespace | `strip()`, `lstrip()`, `rstrip()` |
| Searching | `find()`, `index()`, `count()`, `startswith()`, `endswith()`, `in` |
| Testing | `isdigit()`, `isalpha()`, `isalnum()`, `isspace()`, `isupper()`, `islower()` |
| Changing | `replace()`, `removeprefix()`, `removesuffix()`, `zfill()`, `center()`, `ljust()`, `rjust()` |
| Splitting and joining | `split()`, `rsplit()`, `splitlines()`, `partition()`, `join()` |

### Escape sequences

A backslash inside a string starts an **escape sequence**, a way to write characters that are hard to type.

| Sequence | Meaning |
| --- | --- |
| `\n` | New line |
| `\t` | Tab |
| `\\` | A literal backslash |
| `\'` and `\"` | A literal quote |
| `\xhh` | Character with a two-digit hexadecimal code |
| `\uXXXX` | Unicode character with a four-digit code |
| `\N{name}` | Unicode character by name |

A **raw string** has an `r` before the opening quote and treats backslashes literally. It is ideal for Windows paths and regular expressions.

### Formatting: f-strings

An **f-string** has an `f` before the opening quote. Anything inside `{}` is evaluated and inserted. After a colon you can add a **format specification**.

| Spec | Meaning | Example | Result |
| --- | --- | --- | --- |
| `.2f` | Two decimal places | `f"{3.14159:.2f}"` | `3.14` |
| `,` | Thousands separator | `f"{1234567:,}"` | `1,234,567` |
| `>10` | Right-align in width 10 | `f"{'hi':>10}"` | `        hi` |
| `<10` | Left-align | `f"{'hi':<10}"` | `hi        ` |
| `^10` | Center | `f"{'hi':^10}"` | `    hi    ` |
| `05d` | Zero-pad to width 5 | `f"{42:05d}"` | `00042` |
| `.1%` | Percentage | `f"{0.256:.1%}"` | `25.6%` |
| `x`, `b` | Hexadecimal, binary | `f"{255:x}"` | `ff` |
| `.2e` | Scientific notation | `f"{12345.678:.2e}"` | `1.23e+04` |

Two useful conversions: `!r` shows the `repr` of a value, and `{name=}` prints both the expression and its value, which is excellent for quick debugging.

### Other formatting styles

`"{} and {}".format(a, b)` is the older method and still common in existing code. `"%s and %d" % (a, b)` is older still and appears in legacy code. Write new code with f-strings.

### Unicode, bytes, and encoding

A **code point** is a number assigned to a character by the Unicode standard. `str` holds code points. A **byte** is a number from 0 to 255, and `bytes` holds those. Files, networks, and databases move **bytes**, never abstract characters. **Encoding** turns text into bytes and **decoding** turns bytes back into text. The dominant encoding is **UTF-8**, which uses one byte for basic English letters and up to four for other characters.

| Type | Holds | Literal | Example |
| --- | --- | --- | --- |
| `str` | Unicode code points | `"café"` | 4 characters |
| `bytes` | Raw bytes | `b"caf\xc3\xa9"` | 5 bytes in UTF-8 |

## Basic syntax examples

### Quotes

```text
>>> 'text' == "text"
True
>>> "She said \"hi\""
'She said "hi"'
>>> "It's fine"
"It's fine"
```

### Indexing and slicing

```text
>>> s = "Python"
>>> s[0]
'P'
>>> s[-1]
'n'
>>> s[1:4]
'yth'
>>> s[:2]
'Py'
>>> s[2:]
'thon'
>>> s[::-1]
'nohtyP'
>>> s[::2]
'Pto'
>>> len(s)
6
>>> "th" in s
True
>>> s[10]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IndexError: string index out of range
```

Slices are forgiving: `s[2:100]` quietly stops at the end instead of raising an error. Single-position indexing is not forgiving.

### Immutability

```text
>>> word = "cat"
>>> word[0] = "b"
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'str' object does not support item assignment
>>> "b" + word[1:]
'bat'
```

### Concatenation and repetition

```text
>>> "abc" + "def"
'abcdef'
>>> "Hello" * 3
'HelloHelloHello'
>>> "-" * 10
'----------'
```

### Case, whitespace, and testing

```text
>>> "hello".upper()
'HELLO'
>>> "hello world".title()
'Hello World'
>>> "hello world".capitalize()
'Hello world'
>>> "  hello  ".strip()
'hello'
>>> "42".isdigit()
True
>>> "abc".isalpha()
True
>>> "abc123".isalnum()
True
>>> "   ".isspace()
True
```

### Searching and replacing

```text
>>> "banana".find("n")
2
>>> "banana".find("z")
-1
>>> "banana".index("z")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: substring not found
>>> "banana".count("a")
3
>>> "banana".replace("a", "o")
'bonono'
>>> "report.csv".endswith(".csv")
True
>>> "report.csv".removesuffix(".csv")
'report'
>>> "v1.2".removeprefix("v")
'1.2'
```

`find` returns `-1` when nothing matches, while `index` raises an error. Choose based on whether "not found" is normal or exceptional.

### Splitting and joining

```text
>>> "a,b,c".split(",")
['a', 'b', 'c']
>>> "a  b   c".split()
['a', 'b', 'c']
>>> "line1\nline2".splitlines()
['line1', 'line2']
>>> "key=value=x".partition("=")
('key', '=', 'value=x')
>>> "-".join(["2026", "03", "01"])
'2026-03-01'
```

`split()` with no argument splits on any run of whitespace and ignores leading and trailing spaces. `join` is called on the separator and takes a list of strings.

### Padding and alignment

```text
>>> "7".zfill(3)
'007'
>>> "hi".center(6, "*")
'**hi**'
>>> "Hello".ljust(8, ".")
'Hello...'
>>> "5".rjust(3)
'  5'
```

### Comparing strings

```text
>>> "a" < "b"
True
>>> "Z" < "a"
True
>>> sorted(["banana", "apple", "Cherry"])
['Cherry', 'apple', 'banana']
>>> sorted(["banana", "apple", "Cherry"], key=str.lower)
['apple', 'banana', 'Cherry']
```

Strings compare by Unicode code point, so every uppercase English letter sorts before every lowercase one unless you say otherwise.

### f-strings

```text
>>> name = "Ada"
>>> score = 93.456
>>> f"{name} scored {score:.1f}"
'Ada scored 93.5'
>>> f"{1234567:,}"
'1,234,567'
>>> f"{1234.5:,.2f}"
'1,234.50'
>>> f"{0.256:.1%}"
'25.6%'
>>> f"{'left':<8}|{'right':>8}|"
'left    |   right|'
>>> f"{'centered':^12}"
'  centered  '
>>> f"{42:05d}"
'00042'
>>> f"{255:x} {255:b} {255:#x}"
'ff 11111111 0xff'
>>> f"{3.14159:8.2f}"
'    3.14'
>>> f"{name!r}"
"'Ada'"
>>> f"{name=}"
"name='Ada'"
>>> f"{2 + 3}"
'5'
>>> f"{{literal braces}}"
'{literal braces}'
```

Any expression may appear inside the braces, and doubling the braces prints a literal brace.

> [!tip] Python 3.12 and newer
> Since Python 3.12 you may reuse the same kind of quote inside an f-string expression, such as `f"{data["key"]}"`. Older versions require different quotes inside and outside.

### The older format styles

```text
>>> "{} is {} years old".format("Ada", 36)
'Ada is 36 years old'
>>> "{name} lives in {city}".format(name="Ada", city="London")
'Ada lives in London'
>>> format(3.14159, ".2f")
'3.14'
>>> "%s is %d" % ("Ada", 36)
'Ada is 36'
```

### Escapes and raw strings

```text
>>> print("line1\nline2")
line1
line2
>>> len("a\nb")
3
>>> "\u00e9"
'é'
>>> "\N{GREEK SMALL LETTER ALPHA}"
'α'
>>> len("C:\new")
5
>>> len(r"C:\new")
6
>>> r"C:\new"
'C:\\new'
```

In `"C:\new"` the `\n` became a newline, so the text is five characters long. The raw version keeps all six. When Python shows a string it prints a single backslash as `\\`, which is how it writes a literal backslash.

A raw string cannot end with an odd number of backslashes, so `r"C:\folder\"` is a syntax error. Use `"C:\\folder\\"` or the `pathlib` module (Chapter 5.5).

### Multiline strings

```python
message = """Dear customer,
Your order has shipped.
Thank you!"""
print(message)
```

```text
Dear customer,
Your order has shipped.
Thank you!
```

Everything between the triple quotes is kept, including the line breaks and any indentation.

### Unicode

```text
>>> len("café")
4
>>> ord("A")
65
>>> chr(233)
'é'
>>> "Straße".casefold()
'strasse'
>>> "STRASSE".lower() == "straße".lower()
False
>>> "STRASSE".casefold() == "straße".casefold()
True
```

`casefold()` is the correct tool for case-insensitive comparison of international text.

### Bytes and encoding

```text
>>> data = "café".encode("utf-8")
>>> data
b'caf\xc3\xa9'
>>> len("café"), len(data)
(4, 5)
>>> data.decode("utf-8")
'café'
>>> "café".encode("latin-1")
b'caf\xe9'
>>> b"abc"[0]
97
>>> bytes([72, 105])
b'Hi'
>>> str(b"hi")
"b'hi'"
```

The accented letter takes two bytes in UTF-8. Indexing `bytes` gives integers, not one-character strings. The last line shows a common trap: `str(some_bytes)` does not decode anything, it gives you the printed form of the bytes object. Use `.decode()`.

### Encoding errors

```text
>>> "naïve".encode("ascii")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
UnicodeEncodeError: 'ascii' codec can't encode character '\xef' in position 2: ordinal not in range(128)
>>> "naïve".encode("ascii", errors="replace")
b'na?ve'
>>> b"\xff".decode("utf-8")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xff in position 0: invalid start byte
```

The `errors=` argument controls what happens to problem characters: `"strict"` (the default) raises an error, `"replace"` substitutes a placeholder, and `"ignore"` drops them silently, which hides data loss and is rarely the right choice.

## Deeper behavior

### Strings are built, not changed

Every operation that appears to modify a string creates a new one. That is why building a long string with `+=` in a loop is slow: each step copies everything accumulated so far. Collect the pieces in a list and join once:

```python
parts = []
for number in range(5):
    parts.append(f"item-{number}")
result = ", ".join(parts)
print(result)
```

```text
item-0, item-1, item-2, item-3, item-4
```

### `len` counts code points, not what you see

```text
>>> len("e\u0301")
2
>>> "e\u0301" == "\u00e9"
False
```

The first string is the letter `e` followed by a combining accent. It looks exactly like `é` but is two code points, and it is not equal to the single-code-point version. Text from different sources can represent the same visible character differently. **Normalization** converts text to one standard form:

```text
>>> import unicodedata
>>> composed = "\u00e9"
>>> decomposed = "e\u0301"
>>> composed == decomposed
False
>>> unicodedata.normalize("NFC", decomposed) == composed
True
```

Normalize user-supplied text before comparing it, storing it as a key, or checking for duplicates.

### Always state the encoding when reading or writing files

Python decides which encoding to use when you open a text file based on your system's settings unless you say otherwise, and that default differs between operating systems and versions. Code that works on your laptop may corrupt text on a server. Always write `open(path, encoding="utf-8")` (Chapter 5.4).

### Slicing makes a new string

A slice copies the selected characters. It is cheap for short strings but remember that large strings sliced repeatedly in a loop can add up. It also never raises an error for out-of-range slice positions.

### Interning and `is`

CPython may reuse the same string object for identical short strings. This is an optimization, not a guarantee. Always compare strings with `==`, never with `is` (Chapter 1.2).

### Strings are iterable

A `for` loop over a string visits each character:

```python
for letter in "abc":
    print(letter)
```

```text
a
b
c
```

### f-strings and security

An f-string builds a string. It does not make that string safe. Never create SQL queries, shell commands, or HTML by inserting user input with an f-string. Use parameterized queries (Chapter 12.6), argument lists (Chapter 12.4), and proper escaping. Likewise, do not call `.format()` on a template string that a user can control.

### The `repr` of a string

`print(text)` shows the characters. `print(repr(text))` shows the string as Python code, including quotes and escapes. When hunting for stray spaces or hidden newlines, use `repr`.

```text
>>> text = "hello \n"
>>> print(text)
hello 
>>> print(repr(text))
'hello \n'
```

## Industry-level snippet

User-facing services constantly clean, shorten, and protect text. This module provides four small utilities that appear in real backends: producing a URL-friendly slug from a title, masking an email address for logs, truncating text for previews, and decoding an incoming byte payload safely.

```python
"""Text utilities for a user-facing service."""

import logging
import unicodedata

logger = logging.getLogger(__name__)


def slugify(title: str, max_length: int = 60) -> str:
    """Turn a title into a lowercase, ASCII, hyphen-separated slug.

    Args:
        title: Any user-supplied title, possibly with accents or punctuation.
        max_length: The longest slug to return. Must be at least 1.

    Returns:
        A slug such as "cafe-creme-deja-vu".

    Raises:
        ValueError: If max_length is invalid or the title has no usable
            ASCII letters or digits.
    """
    if max_length < 1:
        raise ValueError("max_length must be at least 1")

    # Split accented letters into a base letter plus a combining mark,
    # then drop everything that is not plain ASCII.
    decomposed = unicodedata.normalize("NFKD", title)
    ascii_text = decomposed.encode("ascii", errors="ignore").decode("ascii")

    words: list[str] = []
    current: list[str] = []
    for char in ascii_text.lower():
        if char.isalnum():
            current.append(char)
        elif current:
            words.append("".join(current))
            current = []
    if current:
        words.append("".join(current))

    slug = "-".join(words)[:max_length].rstrip("-")
    if not slug:
        raise ValueError("title contains no usable characters")
    return slug


def mask_email(email: str) -> str:
    """Hide most of an email's local part so it is safe to log.

    Raises:
        ValueError: If the text does not look like local@domain.
    """
    local, separator, domain = email.strip().partition("@")
    if not separator or not local or not domain:
        raise ValueError("not a valid email address")
    return f"{local[0]}***@{domain}"


def truncate(text: str, max_length: int, suffix: str = "...") -> str:
    """Shorten text to at most max_length characters, adding a suffix if cut."""
    if max_length < len(suffix):
        raise ValueError("max_length must be at least the length of the suffix")
    if len(text) <= max_length:
        return text
    return text[: max_length - len(suffix)].rstrip() + suffix


def decode_payload(raw: bytes) -> str:
    """Decode a request body as UTF-8, replacing invalid bytes instead of failing."""
    try:
        return raw.decode("utf-8")
    except UnicodeDecodeError as exc:
        logger.warning(
            "Payload is not valid UTF-8 (first bad byte at position %d); "
            "replacing invalid bytes",
            exc.start,
        )
        return raw.decode("utf-8", errors="replace")
```

Try it:

```python
print(slugify("  Café Crème: Déjà Vu!! "))
print(mask_email("ada.lovelace@example.com"))
print(truncate("The quick brown fox jumps over the lazy dog", 20))
print(decode_payload("naïve".encode("utf-8")))
print(decode_payload(b"bad \xff byte"))
```

```text
cafe-creme-deja-vu
a***@example.com
The quick brown f...
naïve
bad � byte
```

Design notes:

- Normalization with `NFKD` followed by an ASCII encode removes accents in a predictable way. Titles with no ASCII letters at all, such as purely Chinese text, raise an error, so a real system would need a fallback, such as an ID.
- `partition("@")` splits at the first `@` and reports whether the separator was found, which avoids a try/except around `split`.
- Masking uses only the first character and never logs the full address.
- `decode_payload` prefers strict decoding and degrades gracefully, logging the position of the first bad byte instead of the data itself.
- Every function states its failure mode in its docstring and raises a specific error.

## Common mistakes

1. **Trying to modify a string in place.** `s[0] = "x"` fails. Build a new string.
2. **Building large strings with `+=` in a loop.** Use `"".join(parts)`.
3. **Forgetting that methods return new strings.** `name.strip()` on its own line does nothing useful. Write `name = name.strip()`.
4. **Confusing `str` and `bytes`.** Text on disk and over the network is bytes. Decode on the way in, encode on the way out.
5. **Using `str(some_bytes)` to decode.** It produces `"b'...'"`. Use `.decode("utf-8")`.
6. **Opening files without `encoding="utf-8"`.** Behavior then depends on the machine.
7. **Using `errors="ignore"` casually.** It silently destroys data.
8. **Ending a raw string with a backslash.** `r"C:\dir\"` is a syntax error.
9. **Comparing user text without normalizing or casefolding.** Visually identical strings may not be equal.
10. **Building SQL, shell commands, or HTML with f-strings and user input.** This causes injection vulnerabilities.
11. **Assuming `len` equals the number of visible characters.** Combining marks and other sequences break that assumption.
12. **Using `find` and forgetting that `-1` is a valid return.** `if text.find("x"):` is true for `-1` and false for position `0`. Use `in`, or compare with `-1`.
13. **Mixing quote styles inconsistently** within a project.

## Practice exercises

1. Given `s = "programming"`, write expressions for: the first letter, the last letter, the first three letters, the last three letters, every second letter, and the reversed string.
2. Write a program that asks for a full name such as `"  ada LOVELACE  "` and prints it cleaned: trimmed, in title case, with a single space between words.
3. Check whether a string is a palindrome, ignoring case and spaces. Test it with `"A man a plan a canal Panama"`.
4. Count how many times each vowel appears in a sentence of your choice, using string methods and a loop.
5. Use an f-string to print a receipt line with a left-aligned product name in a width of 15, a right-aligned quantity in a width of 5, and a right-aligned price with two decimals and a thousands separator in a width of 12.
6. Predict the lengths of `"C:\temp"`, `r"C:\temp"`, and `"C:\\temp"`, then verify.
7. Take the text `"naïve café"`. Print its length as a string, its length in UTF-8 bytes, and the bytes themselves. Explain the difference.
8. Create the same visible character in two ways (one code point and two code points). Show that they are not equal, then make them equal with normalization.
9. Write a function that takes a file name such as `"Annual Report 2026.PDF"` and returns the name without its extension, in lowercase, with spaces replaced by underscores.
10. Decode the bytes `b"caf\xe9"` as UTF-8 and observe the error. Find the encoding in which it decodes to `"café"`, and explain what went wrong.
11. Use `unicodedata` and `casefold` to write `same_text(a, b)`, which treats `"Straße"` and `"STRASSE"` as equal, and treats composed and decomposed accents as equal.
12. Extend `truncate` to cut at the last whole word before the limit when possible, and write three test cases, including a text with no spaces.

## Summary

- A `str` is an immutable sequence of Unicode code points. Create it with single, double, or triple quotes.
- Indexing starts at `0` and negative indexes count from the end. Slices `[start:stop:step]` exclude the stop position and do not raise errors when out of range.
- Methods such as `strip`, `split`, `join`, `replace`, `find`, `startswith`, and `removesuffix` always return new values.
- F-strings are the standard way to format text. Format specs control precision, alignment, padding, separators, and percentages. `{x=}` and `!r` help with debugging.
- Escape sequences write special characters. Raw strings (`r"..."`) keep backslashes literal.
- Unicode defines characters, encodings such as UTF-8 turn them into bytes, and `str` and `bytes` are different types. Encode on the way out, decode on the way in.
- Always specify `encoding="utf-8"` for files, normalize and casefold when comparing international text, and never put user input into SQL, shell commands, or HTML through string formatting.
- Build long strings with `"".join(...)` rather than repeated `+=`.
