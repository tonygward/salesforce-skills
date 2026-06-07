---
name: clean-apex-naming
description: "Apply Clean Code naming principles to Apex. Use this skill when writing or renaming Apex variables, methods, classes, properties, or constants — names should reveal intent, avoid disinformation and encodings (no Hungarian notation or member prefixes), use Salesforce problem-domain vocabulary, and stay consistent (one word per concept). Triggers when generating new Apex, reviewing names in existing Apex, or when a user asks to improve readability or follow naming standards. Do NOT use for LWC/Jest naming or for non-naming refactors — see clean-apex-functions or reviewing-apex."
metadata:
  version: "1.0"
  source: "Clean Code (Robert C. Martin), Chapter 2 — Meaningful Names"
---

# Clean Apex Naming

Names are the highest-leverage readability decision in Apex. A name should tell the reader why a thing exists, what it does, and how it is used. If a name needs a comment to explain it, the name has failed.

---

## Rules

### 1. Use intention-revealing names

The name should answer the big questions on its own. If you need a trailing comment to say what a variable holds, rename the variable instead.

```apex
// Bad
Integer d;                          // elapsed time in days
List<Account> list1 = new List<Account>();

// Good
Integer elapsedTimeInDays;
List<Account> activeAccounts = new List<Account>();
```

A method name should make the call site readable without the reader opening the method body.

```apex
// Bad
if (acc.checkStatus() == 4) { ... }

// Good
if (account.isFlaggedForReview()) { ... }
```

### 2. Avoid disinformation

Do not use a name whose conventional meaning differs from the actual contents. The most common Apex offender is calling something a `List` or `Map` when it is not, or naming a `Set` "accountList".

```apex
// Bad — it's a Set, not a List
Set<Id> accountList;

// Good
Set<Id> accountIds;
```

### 3. Avoid encodings — no Hungarian notation, no member prefixes

Apex is strongly typed and the IDE shows you the type. Type-encoded prefixes (`strName`, `lstAccounts`, `mapIdToAcc`, `iCount`, `bIsActive`) are pure noise and rot the moment the type changes.

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

Do not prefix instance variables with `m_` or `_`. If a class is so big you cannot tell members from locals without a prefix, the class is too big — split it.

```apex
// Bad
private String m_description;
private Id _ownerId;

// Good
private String description;
private Id ownerId;
```

### 4. Use pronounceable, searchable names

Avoid abbreviations that cannot be spoken or grep'd. A name like `genymdhms` is unsearchable; `generationTimestamp` is both pronounceable and findable.

Single-letter names are acceptable **only** for short-lived loop counters in a tiny scope. The longer the scope, the longer the name should be.

```apex
// Acceptable — trivial scope
for (Account a : accounts) { total += a.AnnualRevenue; }

// Required — wider scope
for (Account accountToProcess : accountsRequiringRecalculation) {
    recalculate(accountToProcess);
    auditLog.add(accountToProcess.Id);
}
```

### 5. Pick one word per concept — and don't pun

Choose one verb per abstract concept and use it everywhere. Do not scatter `get`, `fetch`, `retrieve`, and `load` across service classes that all do the same thing. Conversely, do not reuse one word for two different ideas (a "pun") — e.g. `add` meaning both list-append and arithmetic.

```apex
// Bad — three names for one concept across the codebase
AccountSelector.getAccounts();
ContactSelector.fetchContacts();
CaseSelector.retrieveCases();

// Good — consistent verb
AccountSelector.getAccounts();
ContactSelector.getContacts();
CaseSelector.getCases();
```

### 6. Use solution-domain and problem-domain names

Use computer-science terms where the reader is a programmer (`accountQueue`, `eventObserver`). Use Salesforce and business-domain vocabulary everywhere else — prefer Salesforce's own nouns over invented generic ones.

```apex
// Bad — generic, hides the domain
List<SObject> items;
String thing;
Id theId;

// Good — speaks Salesforce and the business
List<Opportunity> renewalOpportunities;
String billingCountryCode;
Id parentAccountId;
```

### 7. Add meaningful context — but no gratuitous context

Give names enough context to be unambiguous (a bare `state` is unclear; `billingState` is not). But do not prefix every name with the application or class name — `Persimmon_AccountName` adds nothing inside the Persimmon codebase.

```apex
// Bad — ambiguous
String state;

// Good — contextualised
String billingState;

// Bad — gratuitous prefix
String mcpPersonalisationCustomerEmailAddress;

// Good
String customerEmail;   // inside a class already scoped to the feature
```

---

## Apex-Specific Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Class | PascalCase noun / noun phrase | `AccountRenewalService` |
| Method | camelCase verb phrase | `calculateRenewalDate()` |
| Boolean method/var | reads as a predicate | `isActive`, `hasRenewal()`, `canEdit()` |
| Local variable | camelCase | `renewalOpportunity` |
| Constant | UPPER_SNAKE_CASE | `MAX_BATCH_SIZE` |
| Custom exception | PascalCase ending `Exception` | `RenewalCalculationException` |
| Trigger handler | `{Object}TriggerHandler` | `AccountTriggerHandler` |
| Selector / Service | `{Object}Selector` / `{Object}Service` | `ContactSelector` |
| Test class | `{ClassUnderTest}Test` | `AccountRenewalServiceTest` |

---

## Quick Checklist

- [ ] No type-encoding prefixes (`str`, `lst`, `map`, `b`, `i`)
- [ ] No member prefixes (`m_`, `_`)
- [ ] No abbreviations that can't be spoken or searched
- [ ] Booleans read as predicates (`is`/`has`/`can`)
- [ ] One verb per concept across the codebase
- [ ] Salesforce/business nouns, not generic `item`/`obj`/`thing`
- [ ] Single-letter names only in trivial loop scope
- [ ] No comment needed to explain what a name holds
