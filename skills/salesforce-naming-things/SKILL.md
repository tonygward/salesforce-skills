---
name: salesforce-naming-things
description: Naming and readability conventions for Salesforce code (Apex and LWC) — Clean Code principles applied to Apex classes, methods, variables, constants, LWC components, and test methods. Use when naming or renaming anything in Apex or LWC, or when reviewing code for clarity. Do NOT use for comment style (salesforce-code-comments) or method structure (clean-salesforce-functions).
---

# Naming Things in Salesforce

> "There are only two hard things in Computer Science: cache invalidation and naming things."
> — often attributed via Martin Fowler / Phil Karlton

Names are the cheapest documentation you will ever write. A name should tell the reader why a thing exists, what it does, and how it is used — without needing a comment to explain it.

---

## Principles

- **Reveal intent, not mechanics.** Name *what it is / does*, not *how it's stored*. If a name needs a comment, rename it.
- **Be descriptive, use simple language.** A longer, obvious name beats a short, cryptic one. `eligibleAccounts` over `accs` or `lstA`.
- **No unexplained abbreviations.** `quantity` not `qty`, `customer` not `cust`. Names must be pronounceable and searchable.
- **Booleans read as yes/no questions.** `isLeapYear`, `hasApprovalRights`, `canEdit`.
- **Avoid disinformation.** Don't call a `Set` a "List". Don't use names whose conventional meaning differs from the actual contents.
- **One word per concept.** Pick one verb per abstract idea and use it everywhere — don't mix `get`, `fetch`, `retrieve`, and `load` for the same operation across a codebase.
- **If a name is hard to choose, the design is probably unclear** — that's a signal, not a nuisance.

---

## Apex Anti-Patterns to Avoid

### Type-encoding prefixes — don't use them

Apex is strongly typed and the IDE shows you the type. Prefixes (`strName`, `lstAccounts`, `mapIdToAcc`, `iCount`, `bIsActive`) are noise that rot the moment the type changes.

```apex
// Bad
String strAccountName;
List<Account> lstAccounts;
Map<Id, Account> mapIdToAccount;
Boolean bIsActive;

// Good
String accountName;
List<Account> accounts;
Map<Id, Account> accountsById;
Boolean isActive;
```

### Member prefixes — don't use them

```apex
// Bad
private String m_description;
private Id _ownerId;

// Good
private String description;
private Id ownerId;
```

### Single-letter names outside trivial loops

```apex
// Acceptable — trivial scope
for (Account a : accounts) { total += a.AnnualRevenue; }

// Required — wider scope
for (Account accountToProcess : accountsRequiringRecalculation) {
    recalculate(accountToProcess);
}
```

---

## Salesforce Conventions

| Element | Convention | Example |
|---|---|---|
| Apex class | PascalCase noun | `LeapYearService`, `AccountTriggerHandler` |
| Apex method | camelCase verb phrase | `isLeapYear`, `calculateDiscount` |
| Apex variable / field | camelCase | `eligibleAccounts`, `totalAmount` |
| Apex constant | UPPER_SNAKE_CASE | `MAX_BATCH_SIZE` |
| Apex interface | PascalCase, capability | `Schedulable`, `DiscountStrategy` |
| Custom exception | PascalCase ending `Exception` | `RenewalCalculationException` |
| Trigger handler | `{Object}TriggerHandler` | `AccountTriggerHandler` |
| Selector / Service | `{Object}Selector` / `{Object}Service` | `ContactSelector` |
| Test class | `{ClassUnderTest}Test` | `LeapYearServiceTest` |
| LWC component folder / JS | camelCase | `leapYearBadge` |
| LWC in markup | kebab-case with `c-` | `<c-leap-year-badge>` |
| LWC JS variable / method | camelCase | `handleYearChange` |
| Custom object / field API | PascalCase + `__c` | `Booking__c`, `Start_Date__c` |

---

## Test Method Names Are the Most Valuable Names of All

A test name should read as a sentence describing one behaviour. A failing test must tell you *exactly* what broke without opening the body.

```apex
// Good — reads as a specification
@isTest static void yearsDivisibleByHundredButNotFourHundredAreNotLeapYears() { ... }

// Bad — tells you nothing on failure
@isTest static void test1() { ... }
@isTest static void leapYearTest() { ... }
```

See `salesforce-tests-as-specification` for how these names form a living spec.

---

## Quick Checklist

- [ ] Name states intent; a new reader understands without context
- [ ] No type-encoding prefixes (`str`, `lst`, `map`, `b`, `i`) or member prefixes (`m_`, `_`)
- [ ] No abbreviations a newcomer wouldn't recognise
- [ ] No disinformation (Set called "List", etc.)
- [ ] Booleans phrased as questions (`is/has/can/should`)
- [ ] One verb per concept across the codebase
- [ ] Single-letter names only in trivial loop scope
- [ ] Test methods read as sentences describing one behaviour
- [ ] No comment needed to explain what a name holds
