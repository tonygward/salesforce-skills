# Exercise: Word Frequency Histogram (Apex TDD)

**Adapted from** "TDD kata C". Count how often each word appears in a piece of text. Good for practising output-data-structure design decisions and normalisation rules — and it maps to real Apex work (text analysis, tag clouds, dedup keys).

## The exercise

Test-drive a function that returns the frequency of each word in a text.

`WordFrequency.count(String text)` returning `Map<String, Integer>`.

Example input:
```
There's a hole in the bucket, dear Liza, dear Liza,
There's a hole in the bucket, dear Liza, a hole.
```
should report `a → 3`, `bucket → 2`, `dear → 3`, `hole → 3`, `in → 2`, `the → 2`, `there's → 2`, `liza → 3`, and so on (counts are case-folded, so `Liza` is reported as `liza`).

## Suggested Apex domain mapping

- Return a `Map<String, Integer>`; the caller can sort keys for display.
- `text.split('\\s+')` for tokenising; `String.toLowerCase()` for case folding.
- Strip trailing punctuation per token before counting.
- Pure static method — unit-testable without DML.

## Design decisions to make (then test)

The kata explicitly asks how you'll handle these — decide deliberately and pin each with a test:
- **Punctuation** — `bucket,` and `bucket` count as the same word? (Strip it.)
- **Apostrophes** — is `There's` one word? (Yes — don't split on `'`.)
- **Numbers** — count them as words or ignore?
- **Case** — do `Go` and `go` merge? (Lowercase before counting.)

## Suggested test progression (assertion first)

1. `count('')` → empty map.
2. `count('hello')` → `{hello → 1}`.
3. `count('hello hello')` → `{hello → 2}`.
4. `count('Go go')` → `{go → 2}` (case folding).
5. `count('bucket, bucket')` → `{bucket → 2}` (punctuation stripped).
6. `count("There's")` → `{there's → 1}` (apostrophe kept).
7. The full Liza verse → assert several key counts.

## What to watch for

- Steps 4–6 are the whole exercise: each normalisation rule is one new test that forces one new transformation. Add them one at a time.
- Don't strip the apostrophe while stripping other punctuation — step 5 and step 6 pull in opposite directions; that tension is the point.
- Decide case-folding *before* counting, not after, or `Go`/`go` land in separate buckets.
