---
name: salesforce-tdd-workflow
description: Test-Driven Development workflow for Salesforce (Apex and LWC) - red/green/refactor, the three laws of TDD, and changing only one thing at a time. Use when implementing new functionality test-first or when asked to follow TDD.
---

# TDD for Salesforce

Build behaviour in tiny, verified steps. Each step is driven by a failing test.

## The Three Laws of TDD (Uncle Bob)

1. You may not write **production code** until you have written a **failing** unit test.
2. You may not write **more of a test** than is sufficient to fail — *and a compilation
   failure counts as a failure*.
3. You may not write **more production code** than is sufficient to pass the one failing test.

In Salesforce terms: an Apex test that references a class/method that does not yet exist is a
*compilation failure* — that is your "red". Create just enough of the class to compile and fail
an assertion, then make it pass.

## Red → Green → Refactor

1. **Red** – write a small failing test for the next slice of behaviour.
2. **Green** – write the simplest code that makes it pass. Hard-coding a return value is a
   legitimate first step; the next test forces you to generalise.
3. **Refactor** – clean up names, remove duplication, simplify. **Refactoring is thinking** —
   it is not an optional step you skip "because there's nothing to refactor". Skipping it is how
   designs rot.

> Run the full test suite after every green and every refactor.

## Change only ONE thing at a time

The single most important discipline. When a test goes red unexpectedly, you must be able to
point at the *one* change that caused it. So:

- Don't add behaviour and refactor in the same step.
- Don't rename and change logic in the same step.
- Don't upgrade a dependency and edit code in the same commit.

If you changed two things and something breaks, you've lost the thread.

## Worked Apex slice

```apex
// RED: test references a method that doesn't compile yet
@isTest static void yearNotDivisibleByFourIsNotLeap() {
    Assert.isFalse(LeapYearService.isLeapYear(2023), '2023 is not divisible by 4');
}

// GREEN: simplest thing that passes (and fails the *next* test, forcing generalisation)
public static Boolean isLeapYear(Integer year) {
    return false;
}

// next RED forces real logic:
//   yearDivisibleByFourIsLeap -> 2024 should be true
//   -> generalise to: Math.mod(year, 4) == 0
// then add 100 and 400 rules one test at a time (see salesforce-tests-as-specification)
```

## LWC notes

The same loop applies with Jest: write a failing `it(...)`, render the component or call the
function, make it pass, then refactor markup/JS. Keep the watch runner (`sfdx-lwc-jest --watch`)
on so red/green is instant.

## Checklist

- [ ] Test written and seen to fail *before* the production code.
- [ ] Only enough code written to pass that one test.
- [ ] Refactor step actually performed (names, duplication) — not skipped.
- [ ] Exactly one behavioural change per cycle/commit.
