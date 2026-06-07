---
name: salesforce-tests-as-specification
description: Write Salesforce tests that act as a living specification of behaviour (Apex and LWC). Use when turning requirements or business rules into tests, naming test methods, or structuring test classes around behaviour.
---

# Tests as Specification

A good test suite is executable documentation: each test names one rule, and reading the test
names alone tells you exactly what the system promises to do. When a rule changes, the test that
encodes it changes — the spec can never drift from the code.

## One rule → one test

Translate each business rule into a single, independently-named test. The leap-year rules make a
clean example — the requirement decomposes into four rules:

1. Years **not divisible by 4** are **not** leap years.
2. Years **divisible by 4 but not 100** **are** leap years.
3. Years **divisible by 100 but not 400** are **not** leap years.
4. Years **divisible by 400** **are** leap years.

Each becomes one test whose name *is* the rule:

```apex
@isTest
private class LeapYearServiceTest {

    @isTest static void yearsNotDivisibleByFourAreNotLeapYears() {
        Assert.isFalse(LeapYearService.isLeapYear(2023), '2023 is not divisible by 4');
    }

    @isTest static void yearsDivisibleByFourButNotHundredAreLeapYears() {
        Assert.isTrue(LeapYearService.isLeapYear(2024), '2024 is divisible by 4, not by 100');
    }

    @isTest static void yearsDivisibleByHundredButNotFourHundredAreNotLeapYears() {
        Assert.isFalse(LeapYearService.isLeapYear(1900), '1900 is divisible by 100, not 400');
    }

    @isTest static void yearsDivisibleByFourHundredAreLeapYears() {
        Assert.isTrue(LeapYearService.isLeapYear(2000), '2000 is divisible by 400');
    }
}
```

Run the class and the test names alone read back the specification.

## Structure each test as Given / When / Then

Make the three phases visible (a.k.a. Arrange–Act–Assert):

```apex
@isTest static void discountAppliesToOrdersOverThreshold() {
    // Given an order above the discount threshold
    Order o = TestDataFactory.orderWithTotal(150);

    // When pricing is calculated
    Test.startTest();
    PricingService.apply(o);
    Test.stopTest();

    // Then the loyalty discount is applied
    Assert.areEqual(15, o.Discount__c, 'Orders over 100 get a 10% discount');
}
```

## Guidelines

- **Name the behaviour, not the method.** `refundIsRejectedWhenOrderIsClosed`, not `testRefund`.
- **One reason to fail per test.** Multiple unrelated asserts hide which rule broke.
- **Cover the boundaries the spec implies** — 2000 vs 1900 vs 2100 are the interesting cases, not
  three random leap years.
- **Write the failing test first** (see `salesforce-tdd-workflow`) so each rule is proven to be
  enforced by code, not assumed.
- Works the same in LWC Jest: `it('marks years divisible by 400 as leap years', ...)`.

## Checklist

- [ ] Every business rule maps to a named test.
- [ ] Test names read as sentences describing the rule.
- [ ] Each test has a single reason to fail.
- [ ] Boundary cases from the spec are covered, not arbitrary values.
