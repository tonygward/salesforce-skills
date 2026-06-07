---
name: salesforce-naming-things
description: Naming and readability conventions for Salesforce code (Apex classes/methods/variables, LWC components, and test methods). Use when naming or renaming anything, or when reviewing code for clarity.
---

# Naming Things in Salesforce

> "There are only two hard things in Computer Science: cache invalidation and naming things."
> — often attributed via Martin Fowler / Phil Karlton

Names are the cheapest documentation you will ever write. Spend effort here.

## Principles

- **Be descriptive using simple, clear language.** A longer, obvious name beats a short, cryptic
  one. `eligibleAccounts` over `accs` or `lstA`.
- **Reveal intent, not mechanics.** Name *what it is / does*, not *how it's stored*.
- **No unexplained abbreviations.** `quantity` not `qty`, `customer` not `cust`.
- **Booleans read as yes/no questions.** `isLeapYear`, `hasApprovalRights`, `canEdit`.
- **If a name is hard to choose, the design is probably unclear** — that's a signal, not a nuisance.

## Salesforce conventions

| Element | Convention | Example |
|---|---|---|
| Apex class | PascalCase, noun | `LeapYearService`, `AccountTriggerHandler` |
| Apex method | camelCase, verb phrase | `isLeapYear`, `calculateDiscount` |
| Apex variable/field | camelCase | `eligibleAccounts`, `totalAmount` |
| Apex constant | UPPER_SNAKE_CASE | `MAX_BATCH_SIZE` |
| Apex interface | PascalCase, capability | `Schedulable`, `DiscountStrategy` |
| LWC component folder/js | camelCase | `leapYearBadge` |
| LWC in markup | kebab-case with `c-` | `<c-leap-year-badge>` |
| LWC JS variable/method | camelCase | `handleYearChange` |
| Custom object/field API | PascalCase + `__c` | `Booking__c`, `Start_Date__c` |
| Test class | `<ClassUnderTest>Test` | `LeapYearServiceTest` |

## Test method names are the most valuable names of all

A test name should read as a sentence describing one behaviour, so a failing test tells you
*exactly* what broke without opening the body:

```apex
// Good — reads as a specification
@isTest static void yearsDivisibleByHundredButNotFourHundredAreNotLeapYears() { ... }

// Bad — tells you nothing on failure
@isTest static void test1() { ... }
@isTest static void leapYearTest() { ... }
```

See `salesforce-tests-as-specification` for how these names form a living spec.

## Checklist

- [ ] Name states intent; a new reader understands it without context.
- [ ] No abbreviations a newcomer wouldn't know.
- [ ] Booleans phrased as questions (`is/has/can/should`).
- [ ] Test methods read as sentences describing one behaviour.
