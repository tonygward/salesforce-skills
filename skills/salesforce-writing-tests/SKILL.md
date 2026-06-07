---
name: salesforce-writing-tests
description: How to write high-quality Salesforce tests (Apex test classes and LWC Jest). Use when creating, reviewing, or improving test code, or when discussing code coverage, assertions, test data, mocking, or bulk testing in Salesforce.
---

# Writing Good Salesforce Tests

Tests exist to **reduce defects** and let you change code with confidence — not to hit the
75% coverage gate. Coverage is a side effect of good tests, never the goal.

## GUTS, BUTS, NUTS

- **GUTS** – Good Unit Tests: assert real behaviour, fail when the code is wrong.
- **BUTS** – Bad Unit Tests: run code for coverage but assert nothing (or assert trivia).
  A passing BUTS gives false confidence — worse than no test.
- **NUTS** – No Unit Tests: every change is a guess.

> A test with no meaningful assertion is a BUTS, even at 100% coverage.

## What makes a test GOOD (FIRST)

- **Fast** – no needless DML/SOQL; share setup with `@testSetup`.
- **Independent** – no order dependency, no reliance on org data. Never use
  `@isTest(SeeAllData=true)`; build your own data with a test data factory.
- **Repeatable** – same result every run, in any org, on any day. Inject clocks/dates
  rather than calling `Date.today()` / `Datetime.now()` directly in logic under test.
- **Self-validating** – clear pass/fail via assertions; no manual log reading.
- **Timely** – written alongside the code (ideally test-first — see
  `salesforce-tdd-workflow`).

## Apex essentials

```apex
@isTest
private class LeapYearServiceTest {

    @testSetup
    static void setup() {
        // shared, minimal data for the class
    }

    @isTest
    static void yearsDivisibleByFourButNotHundredAreLeapYears() {
        Test.startTest();
        Boolean result = LeapYearService.isLeapYear(2024);
        Test.stopTest();

        // Use the modern Assert class, not System.assertEquals
        Assert.isTrue(result, '2024 is divisible by 4 and not 100, so it is a leap year');
    }
}
```

- Wrap the code under test in `Test.startTest()` / `Test.stopTest()` to get a fresh set of
  governor limits and to force queued async work to run.
- Prefer the `Assert` class (`Assert.areEqual`, `Assert.isTrue`, `Assert.isNotNull`) over the
  older `System.assert*` methods. **Always pass a message** explaining the expected behaviour.
- Always test three shapes: **positive**, **negative/error**, and **bulk (200 records)** —
  triggers and bulkified code must survive 200 records in one transaction.
- Use `System.runAs(user)` to test sharing, CRUD/FLS, and permission-dependent logic.
- Mock external work: `Test.setMock(HttpCalloutMock.class, ...)` for callouts and the
  **Stub API** (`Test.createStub`) to mock collaborators so you test one unit in isolation.

## LWC (Jest) essentials

```js
import { createElement } from 'lwc';
import LeapYearBadge from 'c/leapYearBadge';

describe('c-leap-year-badge', () => {
    afterEach(() => { while (document.body.firstChild) document.body.firstChild.remove(); });

    it('marks 2024 as a leap year', () => {
        const element = createElement('c-leap-year-badge', { is: LeapYearBadge });
        element.year = 2024;
        document.body.appendChild(element);

        return Promise.resolve().then(() => {
            const label = element.shadowRoot.querySelector('.result');
            expect(label.textContent).toBe('Leap year');
        });
    });
});
```

- Mock wire adapters and Apex imports with `@salesforce/sfdx-lwc-jest`; never hit a real org.
- Assert on rendered DOM / dispatched events — the user-visible behaviour, not internals.

## Checklist

- [ ] Every test asserts the behaviour it claims to (no BUTS).
- [ ] No reliance on org data; data built in-test or in `@testSetup`.
- [ ] Bulk (200-record) path covered for triggers and DML-heavy logic.
- [ ] Negative/exception paths covered with `Assert` on the error.
- [ ] Callouts and collaborators mocked, not invoked for real.
