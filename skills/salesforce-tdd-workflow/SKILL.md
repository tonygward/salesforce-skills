---
name: salesforce-tdd-workflow
description: "Drive Apex development from tests using the RED-GREEN-REFACTOR cycle. Use this skill when building new Apex behaviour test-first, when a user asks to 'TDD', 'test-drive', or 'write the test first', or when deciding what test to write next. Covers the three laws of TDD, the RGR loop, ARRANGE-ACT-ASSERT structure, writing the test in reverse order (assert first), test naming, the FIRST properties, and the four rules of simple design — all adapted to the Apex execution model and governor limits. Pairs with salesforce-tests-as-specification (test naming as living spec) and the clean-apex-* skills (code quality). Do NOT use for after-the-fact test backfilling of existing code, or for callout mock structure specifically (generating-apex-callout-test)."
metadata:
  version: "1.0"
  source: "TDD by Beck — RED-GREEN-REFACTOR; three laws (Martin); Arrange-Act-Assert (Wake); FIRST + four rules of simple design (Martin/Beck) — adapted to Apex"
---

# TDD for Salesforce

This skill is about *process* — the order in which you write code and tests, what test to write next, and when to stop. Test-class mechanics live in `generating-apex-test`; code quality lives in the `clean-apex-*` skills. This is the loop that ties them together.

**The single rule everything else serves: write no production code without a failing test.**

---

## The Three Laws of TDD

1. You may not write **production code** until you have written a **failing** unit test.
2. You may not write **more of a test** than is sufficient to fail — *a compilation failure counts as a failure*.
3. You may not write **more production code** than is sufficient to pass the one failing test.

In Apex: a test that references a class or method that does not yet exist is a compilation failure — that is your "red". Create just enough of the class to compile and fail an assertion, then make it pass.

---

## The Loop: RED → GREEN → REFACTOR

### RED — write a failing test that designs the behaviour
- Write one small test for the next slice of behaviour you want.
- The test designs the *external behaviour* from the caller's side — don't constrain the implementation or test through internal methods.
- Run it. It must fail, and fail for the right reason (assertion, not a compile error you forgot about). **A test you haven't seen fail proves nothing.**

### GREEN — make it pass as fast as possible
- Write the least code that makes the test pass. Do exactly what the test demands, no more.
- Hardcoding a return value is a legitimate first step — the next test forces generalisation.
- Commit whatever design sins you must to get green. This is allowed *because* the next phase cleans them up.

### REFACTOR — make it the simplest design that passes
- With a green bar protecting you, remove the sins. Apply the four rules of simple design (below) and the `clean-apex-*` rules.
- Run the tests after every change. Refactoring with a red bar is just editing.

> Run the full test suite after every green and every refactor.

Then loop. Each pass adds one small slice of behaviour.

---

## Change Only ONE Thing at a Time

When a test goes red unexpectedly, you must be able to point at the *one* change that caused it.

- Don't add behaviour and refactor in the same step.
- Don't rename and change logic in the same step.
- Don't upgrade a dependency and edit code in the same commit.

If you changed two things and something breaks, you've lost the thread.

---

## Test Structure: ARRANGE → ACT → ASSERT

Every test has three parts, in this order:

```apex
@isTest
static void shouldApplyBulkDiscount_WhenQuantityExceedsThreshold() {
    // ARRANGE — set up the subject using only the public surface; provide fakes for dependencies
    Order testOrder = TestDataFactory.createOrderWithQuantity(150);

    // ACT — send one message to trigger the behaviour under test
    Test.startTest();
    Pricing result = PricingService.priceOrder(testOrder);
    Test.stopTest();

    // ASSERT — check the resulting state
    Assert.areEqual(0.10, result.discountRate, 'Orders over 100 units should get the 10% bulk rate');
}
```

Apex-specific mapping:
- **ARRANGE** ends before `Test.startTest()`. Set up data and mocks here so they don't count against the governed code's limits.
- **ACT** is the single call wrapped by `Test.startTest()` / `Test.stopTest()`. That boundary resets governor limits so the assertion measures the code under test, not the setup.
- **ASSERT** comes after `Test.stopTest()`. For async (future/queueable/batch), `stopTest()` forces execution to complete — assert after it.

---

## Write the Test in Reverse: ASSERT First

Write the three sections in **reverse order — ASSERT first, then ACT, then ARRANGE.**

1. Start with the assertion. This forces you to decide what success *means* before anything else — the outcome drives the design.
2. Write the ACT line that would produce the value you're asserting on. This names the method and its signature.
3. Write the ARRANGE that sets up the inputs ACT needs.

Writing forward (arrange first) tempts you to set up data and then cast around for something to assert. Writing backward keeps every line in service of the outcome.

---

## What Test to Write Next

- **One concept per test.** A test method exercises a single behaviour. Multiple asserts are fine *if* they all describe the same concept.
- **Triangulate from simple to general.** Start with the simplest case that fails, get it green, then add a case that forces generalisation. Hardcoding `return false` is a legitimate GREEN step — the next test makes it untenable.
- **Cover ranges, not just points.** Where behaviour changes at a boundary, test on both sides of each boundary.
- **Follow the requirement, don't over-solution.** Write tests for the rules you were given, not for elaborations you imagined.

**Worked Apex slice (leap year):**

```apex
// RED: references a method that doesn't exist yet — compilation failure is the red
@isTest static void yearNotDivisibleByFourIsNotLeap() {
    Assert.isFalse(LeapYearService.isLeapYear(2023), '2023 is not divisible by 4');
}

// GREEN: simplest thing that passes — hardcode it; the next test forces generalisation
public static Boolean isLeapYear(Integer year) {
    return false;
}

// next RED forces real logic:
//   yearDivisibleByFourIsLeap -> 2024 should be true -> Math.mod(year, 4) == 0
// then add the 100 and 400 rules one test at a time (see salesforce-tests-as-specification)
```

---

## Test Naming

Name the behaviour, not the method. A test list should read as a specification.

- Use `should...When...`: `shouldReturnFreePostage_WhenOrderOver25AndNoExceptionalItems`
- Group tests by the *scenario* they describe, not the method they call
- `@TestSetup` for shared ARRANGE so each test stays focused on what's unique to it

See `salesforce-tests-as-specification` for how names form a living spec across a full rule set.

---

## FIRST — Properties of Good Tests

| Property | Meaning | Apex angle |
|----------|---------|-----------|
| **Fast** | Tests run quickly so you run them often | No unnecessary DML/SOQL; a slow test is a smell |
| **Independent** | No test sets up another; any order works | No shared mutable static state; `@TestSetup` not cross-test coupling |
| **Repeatable** | Same result in any environment | **Never `SeeAllData=true`** — build your own data |
| **Self-validating** | Pass/fail is automatic | Real `Assert` calls, not `System.debug` and eyeballing |
| **Timely** | Written just before the code | This is the whole point of the RGR loop |

---

## The Four Rules of Simple Design

When refactoring, aim for code that, in priority order:

1. **Passes all its tests.** Correctness first.
2. **Communicates intent.** Reveals what it does to the next reader (see `salesforce-naming-things`).
3. **Expresses every concept once and only once.** No duplication of logic, SOQL, or rules.
4. **Contains nothing extra.** No speculative generality, no dead code, no unused parameters.

The goal is code that gets *simpler* with each feature you add, because the test harness lets you keep refactoring toward that.

---

## LWC Notes

The same loop applies with Jest: write a failing `it(...)`, render the component or call the function, make it pass, then refactor markup/JS. Keep the watch runner (`npm run test:unit:watch`) on so red/green is instant.

---

## Apex-Specific Cautions

- **Governor limits are part of the design.** "Get it passing fast" never licenses SOQL/DML in a loop. Bulkify in the GREEN-to-REFACTOR transition, and add a bulk test (200+ records) as one of your cases.
- **`Test.startTest`/`stopTest` is the ACT boundary** and the async flush point — don't scatter the act across it.
- **Assert class only** — `Assert.areEqual`, `Assert.isTrue`, `Assert.fail`; not legacy `System.assert*`.
- **Fakes over mocks** — a hand-rolled fake selector or the Apex Stub API keeps tests readable; reach for a mocking framework only when verifying interactions you can't observe through state.
- **Deliver the `.cls-meta.xml`** with every test class.

---

## Quick Checklist

- [ ] No production code written without a failing test first
- [ ] Saw the test fail (red) before making it pass
- [ ] Only enough code written to pass that one test
- [ ] ARRANGE / ACT / ASSERT clearly separated; ACT wrapped by `startTest` / `stopTest`
- [ ] Wrote the assertion first (reverse-order drafting)
- [ ] One concept per test; boundaries tested on both sides
- [ ] Exactly one behavioural change per cycle / commit
- [ ] Refactor step performed (four rules of simple design) — not skipped
- [ ] No `SeeAllData=true`; data built in-test or `@TestSetup`
- [ ] Bulk (200+) case included; no SOQL/DML in loops
