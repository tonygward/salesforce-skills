---
name: clean-apex-comments
description: "Apply Clean Code comment principles to Apex. Use this skill when writing, reviewing, or cleaning up comments in Apex code: prefer expressing intent in code over commenting, delete commented-out code and redundant/noise comments, never auto-Javadoc every method, and keep only the few legitimate comment types (legal headers, intent, warnings, TODOs tied to a work item, public-API ApexDoc). Triggers when generating Apex with comments, when a file is cluttered with comments or dead code, or when a user asks about commenting standards. Do NOT use for naming (clean-apex-naming) or method structure (clean-apex-functions)."
metadata:
  version: "1.0"
  source: "Clean Code (Robert C. Martin), Chapter 4 — Comments"
---

# Clean Apex Comments

A comment is an admission that the code failed to express itself. The goal is not to comment well but to make comments unnecessary. Comments lie over time: code moves and changes, comments do not follow it, and a stale comment is worse than none.

Default position: **express intent in code, not in a comment.** Reach for a comment only when the code genuinely cannot carry the meaning.

---

## Express Yourself in Code First

Before writing a comment, try to make it redundant by improving the code — usually by extracting a well-named method or introducing an explanatory variable.

```apex
// Bad — comment compensates for an opaque condition
// Check if the account is an active enterprise customer eligible for renewal
if (acc.Status__c == 'Active' && acc.Tier__c == 'Enterprise' && acc.Contract_End__c != null) { ... }

// Good — the code says it
if (isRenewableEnterpriseAccount(acc)) { ... }
```

---

## Legitimate Comments (keep these)

These are the cases where a comment earns its place:

1. **Legal / licence headers** — required on managed-package and ISV code. Keep them concise; link to the full licence rather than pasting it.
2. **Explanation of intent** — *why* a decision was made, when the reason is not obvious from the code. (e.g. "Sorted descending because the downstream batch expects newest-first.")
3. **Warning of consequences** — flag a non-obvious hazard. (e.g. "Runs synchronously — do not call from a trigger context, will exceed CPU limits on bulk loads.")
4. **TODO comments** — acceptable **only** when tied to a tracked work item. A freestanding `// TODO: fix this` is noise; `// TODO [W-04821]: replace with Platform Event once the bus is live` is actionable.
5. **ApexDoc on public APIs** — public/global methods in a shared library or managed package benefit from ApexDoc so consumers get signatures and intent without reading the body.

---

## Bad Comments (remove or prevent these)

### Commented-out code — delete it

This is the single most common offender in Apex codebases. Old SOQL queries, superseded field references, and disabled logic left "just in case." Version control already remembers it. Delete it.

```apex
// Bad
// List<Account> accs = [SELECT Id, Name, Old_Field__c FROM Account];   <-- delete
List<Account> accounts = [SELECT Id, Name FROM Account WHERE IsActive__c = true];
```

### Redundant comments — delete them

A comment that just restates the code adds reading cost and risks going stale.

```apex
// Bad
// increment the counter
counter++;

// the account id
Id accountId;
```

### Mandated / noise ApexDoc on internal methods — don't generate it

Do not auto-generate ApexDoc on every private or internal method. Boilerplate like `@description Gets the name / @return the name` is noise and lies the moment the method changes. Reserve ApexDoc for genuine public APIs (see legitimate list above).

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

// Good — the signature already says everything; no comment
private String getAccountName(Account account) {
    return account.Name;
}
```

### Journal / attribution comments — delete them

`// Modified 2019-03-12 by JB` and change-log blocks belong in Git history, not the source file.

### Position markers and closing-brace comments — delete them

`// ===== HELPERS =====` banners and `} // end for` markers signal that the class or method is too big. Fix the size, don't annotate it.

---

## Apex-Specific Notes

- **Git is your journal.** Salesforce orgs that adopt source-driven development (scratch orgs, SFDX, version control) no longer need in-file history comments — the repo carries lineage. Delete legacy change-log blocks on sight.
- **ApexDoc generators** (e.g. for documentation sites) need structured tags on public surfaces — that is a legitimate public-API case, not a licence to annotate everything.
- **`@deprecated`** is a real annotation, not a comment — use it on superseded global methods rather than a `// don't use this` comment.

---

## Decision Flow

```
Need to write a comment?
  │
  ├─ Can I rename or extract to make it unnecessary?  → do that instead
  │
  ├─ Is it legal / intent / warning / tracked-TODO / public-API doc?  → keep, concise
  │
  └─ Otherwise  → don't write it
```

---

## Quick Checklist

- [ ] No commented-out code (Git remembers it)
- [ ] No comments that merely restate the code
- [ ] No auto-ApexDoc on private/internal methods
- [ ] No journal/attribution/change-log blocks
- [ ] No banner or closing-brace markers
- [ ] TODOs reference a work item
- [ ] Comments that remain explain *why*, not *what*
