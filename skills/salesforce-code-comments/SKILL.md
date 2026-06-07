---
name: salesforce-code-comments
description: When and how to comment Salesforce code (Apex and LWC) — the why-not-how rule, legitimate vs. noise comments, ApexDoc/JSDoc on public APIs, and what to delete. Use when writing, reviewing, or cleaning up comments in Apex or LWC. Do NOT use for naming (salesforce-naming-things) or method structure (clean-apex-functions).
---

# Comments in Salesforce Code

> "Code tells you **how**, comments tell you **why**." — Jeff Atwood

A comment is an admission that the code failed to express itself. The goal is not to comment well but to make comments unnecessary. Comments lie over time: code moves and changes, comments do not follow — a stale comment is worse than none.

**Default position:** express intent in code, not in a comment. Reach for a comment only when the *why* genuinely lives outside the code.

---

## Express Yourself in Code First

Before writing a comment, try to make it redundant — better names, a well-named helper method, a guard clause.

```apex
// Bad — comment compensates for an opaque condition
// Check if the account is an active enterprise customer eligible for renewal
if (acc.Status__c == 'Active' && acc.Tier__c == 'Enterprise' && acc.Contract_End__c != null) { ... }

// Good — the code says it
if (isRenewableEnterpriseAccount(acc)) { ... }
```

---

## Legitimate Comments (keep these)

1. **Business / regulatory rules** that aren't obvious from the code.
2. **Governor-limit workarounds** — why work is chunked, why a query is shaped oddly.
3. **Order-of-execution gotchas** — trigger recursion guards, `@future` vs Queueable choices.
4. **Warning of consequences** — a non-obvious hazard, e.g. "Do not call from a trigger context — will exceed CPU limits on bulk loads."
5. **Why a "wrong-looking" approach is intentional** — with a link to the tracked reason.
6. **TODO comments** — only when tied to a tracked work item: `// TODO [W-04821]: replace with Platform Event once the bus is live`. A freestanding `// TODO: fix this` is noise.
7. **ApexDoc on public / global APIs** — public/global methods in a shared library or managed package benefit from ApexDoc so consumers get intent without reading the body.
8. **JSDoc on exported LWC functions and `@api` properties** — documents the contract for callers.

---

## Bad Comments (remove or prevent)

### Commented-out code — delete it

The most common offender in Salesforce codebases. Version control remembers it. Delete it.

```apex
// Bad
// List<Account> accs = [SELECT Id, Name, Old_Field__c FROM Account];   ← delete
List<Account> accounts = [SELECT Id, Name FROM Account WHERE IsActive__c = true];
```

### Redundant comments — delete them

A comment that restates the code adds reading cost and will go stale.

```apex
// Bad
// increment the counter
counter++;

// the account id
Id accountId;
```

### Noise ApexDoc on internal methods — don't generate it

Do not auto-generate ApexDoc on every private or internal method. Reserve ApexDoc for genuine public APIs.

```apex
// Bad — noise on an internal method
/**
 * @description Returns the account name
 * @param account The account
 * @return String The name
 */
private String getAccountName(Account account) {
    return account.Name;
}

// Good — no comment needed
private String getAccountName(Account account) {
    return account.Name;
}
```

### Journal / attribution comments — delete them

`// Modified 2019-03-12 by JB` belongs in Git history, not the source file.

### Position markers and closing-brace comments — delete them

`// ===== HELPERS =====` banners and `} // end for` markers signal the class or method is too large. Fix the size, don't annotate it.

---

## Public APIs: document the contract

```apex
/**
 * Determines whether a year is a leap year under the Gregorian calendar.
 * @param year four-digit year, e.g. 2024
 * @return true if the year has 366 days
 */
public static Boolean isLeapYear(Integer year) { ... }
```

```js
/**
 * The year to evaluate. Bound from the parent component.
 * @type {number}
 */
@api year;
```

---

## Decision Flow

```
Need to write a comment?
  │
  ├─ Can I rename or extract to make it unnecessary?  → do that instead
  │
  ├─ Is it: business rule / limit workaround / warning / tracked-TODO / public-API doc?  → keep, concise
  │
  └─ Otherwise  → don't write it
```

---

## Apex-Specific Notes

- **`@deprecated`** is a real annotation — use it on superseded global methods rather than a `// don't use this` comment.
- **Git is your journal.** Source-driven development means in-file change-log blocks are redundant. Delete legacy attribution comments on sight.

---

## Quick Checklist

- [ ] Each comment explains *why*, not *how*
- [ ] Tried clearer naming or extraction before reaching for a comment
- [ ] No commented-out code (Git remembers it)
- [ ] No comments that restate the code
- [ ] No auto-ApexDoc on private / internal methods
- [ ] No journal / attribution / change-log blocks
- [ ] No banner or closing-brace markers
- [ ] TODOs reference a tracked work item
- [ ] Public Apex / LWC APIs documented with ApexDoc / JSDoc
