---
name: reviewing-apex
description: "Review existing Apex against Clean Code smells and heuristics. Use this skill when asked to review, critique, assess, or find problems in Apex code, or to suggest refactors: detects duplication, dead code, magic numbers (hardcoded IDs/picklist values), feature envy, Law-of-Demeter train wrecks, God classes, oversized methods, weak tests, and naming smells. Triggers on 'review this Apex', 'code review', 'what's wrong with this class', 'refactor suggestions', or a pull-request-style assessment. Do NOT use when generating new code from scratch — use clean-apex-naming, clean-apex-functions, clean-apex-comments, or clean-apex-error-handling for authoring."
metadata:
  version: "1.0"
  source: "Clean Code (Robert C. Martin) — Ch 17 (Smells & Heuristics), Ch 10 (Classes), Ch 9 (Unit Tests), Ch 6 (Objects & Data Structures)"
---

# Reviewing Apex

A checklist-driven review skill. Read the target Apex, then walk the smell categories below and report findings with severity, location, and a concrete fix. This is the counterpart to the authoring skills — those prevent smells, this one catches them in code that already exists.

---

## How to Review

1. Identify what you're reviewing (class, trigger, test, whole feature).
2. Walk each category below in order. For each hit, record: **smell**, **where**, **why it matters in Apex**, **fix**.
3. Prioritise by severity: governor-limit and correctness risks first, then maintainability, then style.
4. Report as a prioritised list, not a line-by-line dump. Lead with the few things that matter most.

---

## General Smells

### G5 — Duplication
The most important smell to hunt. Repeated SOQL field lists, copy-pasted validation, near-identical trigger branches. In Apex, duplicated queries also duplicate against governor limits. **Fix:** extract to a shared method or selector; apply the template method pattern for near-duplicates.

### G25 — Magic numbers and hardcoded values
The dominant Apex offender. Hardcoded record type IDs, picklist string literals, profile/user IDs, `15`/`18`-char Ids, batch sizes scattered in code.

```apex
// Smell
if (acc.RecordTypeId == '012xx0000001AbcAAE') { ... }
if (opp.StageName == 'Closed Won') { ... }

// Fix
if (acc.RecordTypeId == AccountRecordTypes.ENTERPRISE_ID) { ... }
if (opp.StageName == OpportunityStages.CLOSED_WON) { ... }
```
Hardcoded Ids are also environment-fragile — they break on deploy to another org. **Fix:** Custom Metadata, constants, or `Schema` describe calls.

### G23 — Prefer polymorphism to switch/if-else chains
Long `if/else` or `switch` on a type or status field, repeated in several places, signals missing polymorphism. **Fix:** a strategy/handler per case behind a common interface.

### G9 — Dead code
Methods never called, `if` branches that cannot execute, unreachable returns. **Fix:** delete it; Git remembers.

### G14 — Feature envy
A method that reaches repeatedly into another object's fields is envious of that object's responsibilities. Common in fat trigger handlers that manipulate child-record internals. **Fix:** move the behaviour to the class that owns the data.

### G19 — Use explanatory variables
A dense boolean or arithmetic expression buried in a condition. **Fix:** assign sub-expressions to well-named locals.

### G28 / G29 — Encapsulate conditionals; avoid negatives
`if (isRenewable(acc))` beats an inline three-clause condition; `if (isActive)` beats `if (!isInactive)`.

---

## Law of Demeter (Ch 6)

### Train wrecks
Chained calls reaching through several objects expose structure that should be hidden.

```apex
// Smell — train wreck
String city = order.getAccount().getBillingAddress().getCity();

// Fix — tell, don't ask; or expose intent on the nearest object
String city = order.getBillingCity();
```
In Apex this also appears as deep parent-field SOQL traversal (`Account.Parent.Parent.Owner.Profile.Name`) — acceptable in a query, but don't then navigate it further in code.

---

## Class Smells (Ch 10)

### God class / Single Responsibility violation
A class with many unrelated public methods, or one that changes for many different reasons. Measure by **responsibilities, not lines** — a class with five methods can still be a God class if they serve unrelated concerns. Classic Apex case: one `Utils` or `Helper` class that does everything. **Fix:** split by responsibility into focused classes (selector, service, domain, etc.).

### Classes should be small
If you can't give the class a concise name without "and" or a weasel word (`Manager`, `Processor`, `Utils`), it likely has too many responsibilities.

### Cohesion
Methods should use the class's instance variables. When a subset of methods uses one subset of fields and another subset uses different fields, the class wants to be two classes.

---

## Function Smells (Ch 3, summarised at Ch 17)

| Smell | Trigger |
|-------|---------|
| F1 — Too many arguments | More than 3 parameters |
| F3 — Flag arguments | Boolean parameter that switches behaviour |
| G30 — Does more than one thing | Multiple levels of abstraction in one method |
| G34 — Descends more than one level of abstraction | Mixing policy and mechanics |

*(For fixes, defer to `clean-apex-functions`.)*

---

## Apex Architecture Smells

These have no direct Clean Code chapter but are the highest-impact Apex-specific findings:

- **SOQL or DML inside a loop** — the cardinal governor-limit sin. Always a high-severity finding.
- **No bulkification** — trigger or service assumes a single record. Test mentally with 200+ records.
- **Logic in the trigger file** — business logic belongs in a handler/service, not the `trigger` body.
- **Logic in `@AuraEnabled` controller** — controllers should delegate to a service so logic is reusable and testable.
- **Hardcoded org-specific Ids** — see G25; environment-fragile.
- **Missing `with sharing` / `without sharing` declaration** — security posture left implicit.

---

## Test Smells (Ch 9)

Apply when reviewing `*Test` classes:

- **T1 — Insufficient tests** — only the happy path; no negative or bulk cases.
- **One concept per test** — a test method asserting many unrelated things. Split it.
- **`SeeAllData=true`** — depends on org data; not repeatable. High-severity.
- **No assertions / weak assertions** — a test that runs code without asserting outcomes proves nothing. Range assertions where the value is deterministic are a smell.
- **F.I.R.S.T. violations** — tests that are slow, interdependent, non-repeatable, or that don't self-validate.
- **Legacy `System.assert*`** — should be the `Assert` class.

---

## Comment Smells (Ch 4)

- **C5 — Commented-out code** — delete it. (Very common in Apex.)
- **C3 — Redundant comments** — restate the code; delete.
- **Noise ApexDoc on private methods** — boilerplate that will go stale.

*(For the full policy, defer to `salesforce-code-comments`.)*

---

## Report Format

Structure findings like a pull-request review:

```
## Review: AccountRenewalService.cls

### High severity
1. SOQL inside loop (lines 34–41) — will hit query limits on bulk loads.
   Fix: query Contacts once into a Map<Id, List<Contact>> before the loop.
2. Hardcoded RecordType Id (line 58) — breaks on deploy to other orgs.
   Fix: replace with AccountRecordTypes.ENTERPRISE_ID constant.

### Medium severity
3. processRenewals() does four things (lines 12–48) — filter, build, persist, notify.
   Fix: extract filterRenewable / buildRenewals / sendNotices.

### Low severity / style
4. Variable lstAccs (line 20) — Hungarian prefix; rename to accounts.
```

Lead with what matters. Don't bury a governor-limit bug under a naming nit.
