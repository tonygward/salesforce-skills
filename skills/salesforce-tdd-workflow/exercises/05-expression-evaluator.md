# Exercise: Expression Evaluator (Apex TDD)

**Adapted from** "TDD kata D". Evaluate a string containing an arithmetic expression. The hardest kata in the set — it grows from trivial into a small parser, so it's the best practice for letting design *emerge* under test rather than big-design-up-front.

## The exercise

Test-drive a function that evaluates an arithmetic expression given as a string.

`Evaluator.evaluate(String expression)` returning a number.

Example: `"12 + 5"` → `17`. Must handle `+ - * /`.

## Suggested Apex domain mapping

- Start with `Integer`/`Decimal` returns; move to `Decimal` once the decimal-point extension lands.
- Resist building a tokeniser/parser early — the first few tests don't need one. Let precedence (extension a) force the tokenise→parse split.
- Pure static method; no DML. Throw a typed `ExpressionException` on malformed input (clean-apex-error-handling) rather than returning null.

## Suggested test progression (assertion first)

1. `evaluate('1')` → `1`.
2. `evaluate('1 + 2')` → `3`.
3. `evaluate('5 - 3')` → `2`.
4. `evaluate('4 * 3')` → `12`.
5. `evaluate('10 / 2')` → `5`.
6. `evaluate('1 + 2 + 3')` → `6` (left-to-right chaining).

By step 6 a flat left-to-right fold works. The extensions below are what force a real parser.

## Bonus extensions (each is a mini-project — add tests first)

- **a) Precedence** — `"12 - 3 * 2"` → `6`, not `18`. This is the big one: it forces you to separate tokenising from evaluation and introduce operator precedence (shunting-yard or recursive descent).
- **b) Decimals** — `"1.5 + 2"` → `3.5`. Switch the return type to `Decimal`; watch division.
- **c) Brackets** — `"(12 - 3) * 2"` → `18`. Recursive evaluation of sub-expressions.
- **d) Exponents and roots** — right-associativity for `^` is a new wrinkle.
- **e) Function definitions** — define a syntax allowing user functions. Open-ended; stop here unless you want a language.

## What to watch for

- Do **not** reach for a parser at step 1. The discipline is to let extension (a) make the flat approach untenable — that's the lesson about emergent design.
- Division: decide integer vs decimal early (extension b), and handle divide-by-zero with a typed exception, tested.
- Brackets (c) turn the evaluator recursive; that's a REFACTOR moment, driven by the failing bracket test, not anticipated.
- Each extension is a fresh RED→GREEN→REFACTOR cycle; commit a green bar before starting the next.
