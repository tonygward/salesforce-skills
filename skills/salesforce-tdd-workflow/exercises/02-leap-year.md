# Exercise: Leap Year (Apex TDD)

**Adapted from** "TDD kata E". Determine whether a year is a leap year. Small and finite, but a perfect boundary-testing drill: three nested exception rules, each needing tests on both sides.

## The exercise

Test-drive a function that returns whether a given year is a leap year.

`LeapYear.isLeapYear(Integer year)` returning `Boolean`.

Rules: a leap year is divisible by 4 — **except** centuries (divisible by 100), which are not — **except** those divisible by 400, which are.

## Suggested Apex domain mapping

- Pure static method, `Boolean` return, no DML — the simplest possible unit-test target.
- Note: Apex has no built-in `isLeapYear`, so this is genuinely useful as a utility, not just a drill. (`Date.daysInMonth(year, 2) == 29` is an alternative oracle you could assert against.)

## Suggested test progression (assertion first)

1. `isLeapYear(2001)` → `false` (not divisible by 4).
2. `isLeapYear(2012)` → `true` (divisible by 4).
3. `isLeapYear(2100)` → `false` (century, not divisible by 400).
4. `isLeapYear(2000)` → `true` (divisible by 400).

Four tests fully specify the rule. Each one adds exactly one branch.

## Boundary discipline

This kata is *about* boundaries — test both sides of each rule:
- divisible by 4 vs not: 2012 / 2013
- century vs non-century divisible-by-4: 1900 / 1904
- divisible-by-400 vs century-not-400: 2000 / 2100

## What to watch for

- The rule reads as nested exceptions; the cleanest expression is a single boolean (`divisible by 400, OR (divisible by 4 AND not divisible by 100)`) — let the four tests drive you there rather than writing nested `if`s.
- Don't stop at three tests; the 400 case (step 4) is the one people forget, and it's the whole reason the rule is interesting.
