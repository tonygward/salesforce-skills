# Exercise: FizzBuzz (Apex TDD)

**Adapted from** "TDD kata A". The classic warm-up. In Apex it's trivial arithmetic — the value is in the *bonus constraints*, which push you toward a rules-table design that maps neatly onto Salesforce config-driven thinking.

## The exercise

Test-drive a function that, given a number, returns what a player should say in FizzBuzz:
- divisible by 3 → `"fizz"`
- divisible by 5 → `"buzz"`
- divisible by both → `"fizz-buzz"`
- otherwise → the number as a string

`FizzBuzz.say(Integer n)` returning `String`.

## Bonus constraints (the real exercise)

1. **No conditionals in the code** — no `if`, no ternary, no `switch`.
2. **New rules can be added without programming** (e.g. say "bang" for multiples of 7) — i.e. rules are data, not code.

## Suggested Apex domain mapping

- A `Rule` inner class holding a divisor and a word.
- A `List<Rule>` as the rule set; `say()` walks it, appending words whose divisor divides `n`, and falls back to the number when nothing matched.
- To honour "no conditionals", the fallback uses a default/empty-string-then-coalesce trick rather than an `if`.
- The "add rules without programming" constraint maps directly to **Custom Metadata Types** in a real org: each rule is a metadata record (divisor, word). Note that in the exercise; you don't need to build the CMDT, just structure the code so the rule set is injectable.

## Suggested test progression (assertion first)

1. `say(1)` → `"1"`.
2. `say(3)` → `"fizz"`.
3. `say(5)` → `"buzz"`.
4. `say(15)` → `"fizz-buzz"`.
5. `say(6)` → `"fizz"` (generalises the divisible-by-3 rule beyond 3).
6. Inject a rule (7 → "bang"); `say(7)` → `"bang"`, `say(21)` → `"fizz-bang"`.

## What to watch for

- Steps 2–3 tempt an `if`. Resist past step 4 and the rules-table design emerges naturally — that's the lesson.
- "fizz-buzz" ordering: assert the exact hyphenated string so rule order is pinned.
- Once it's a rule list, adding step 6's rule should require **zero** changes to `say()`.
