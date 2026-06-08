# Exercise: Word Wrap (Apex TDD)

**Adapted from** "TDD kata B". Wrap text at a given column width, respecting word boundaries. A pure string-algorithm kata — excellent for practising triangulation because each new test forces one more branch of the logic.

## The exercise

Test-drive a function that takes a string and a column width and returns the text wrapped so no line exceeds the width, breaking on spaces.

`WordWrap.wrap(String text, Integer width)` returning `String` (lines joined by `\n`).

Example: `"The boy stood on the burning deck"` wrapped at 8 →
```
The boy
stood on
the
burning
deck
```

## Suggested Apex domain mapping

- Pure static method, no SObjects, no DML — a clean unit-test target.
- Build lines into a `List<String>` and `String.join(lines, '\n')` at the end.
- Apex has `String.split`, `String.substring`, `String.length` — no regex needed for the core.

## Suggested test progression (assertion first)

1. `wrap('', 10)` → `''`.
2. `wrap('hello', 10)` → `'hello'` (shorter than width, unchanged).
3. `wrap('hello world', 5)` → `"hello\nworld"` (break on the space).
4. `wrap('a b c', 3)` → `"a b\nc"` (greedy fill — pack as many words as fit).
5. `wrap('word longerword', 4)` → handle a single word longer than the width.
6. Multiple spaces / trailing space → decide and pin the behaviour.

## Edge cases the kata calls out (pick how to cope, then test them)

- **Hyphens** — break on a hyphen as well as a space? Decide, then test both.
- **Very long words** — a word longer than the width: hard-break it, or let it overflow? Test your choice.
- **Punctuation** — punctuation attached to words shouldn't be split off.

## What to watch for

- The "very long word" case (step 5) is the one that breaks naive split-on-space solutions — let it drive a real generalisation.
- Greedy fill (step 4) is the heart of the algorithm; don't skip straight to it, let step 3 then 4 build it.
- Assert exact strings including the `\n` positions so line breaks are pinned.
