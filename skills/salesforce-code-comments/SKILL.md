---
name: salesforce-code-comments
description: When and how to comment Salesforce code (Apex and LWC) - ApexDoc/JSDoc, the "why not how" rule, and avoiding noise. Use when writing or reviewing comments or documentation in Salesforce code.
---

# Comments in Salesforce Code

> "Code tells you **how**, comments tell you **why**." — Jeff Atwood

The code already shows *how* it works. A comment earns its place only by explaining *why* —
something the code itself cannot say.

## Prefer clearer code over a comment

The classic solution to code that is hard to understand is **not** to add a comment — it's to
make the code clearer: better names, a well-named helper method, a guard clause. Reach for a
comment only when the *why* genuinely lives outside the code.

```apex
// Noise — the code already says this
i = i + 1; // increment i

// Better — no comment needed
recordsProcessed++;

// Worth keeping — explains a non-obvious WHY
// 400-year rule: the Gregorian calendar drops 3 leap days every 400 years
// to stay aligned with the solar year. Without this, 1900 and 2100 are wrong.
if (Math.mod(year, 400) == 0) return true;
```

## Good reasons to comment in Salesforce

- **Business / regulatory rules** that aren't obvious from the code.
- **Governor-limit workarounds** (e.g. why work is chunked, why a query is shaped oddly).
- **Order-of-execution gotchas** (trigger recursion guards, `@future` vs Queueable choices).
- **Why a "wrong-looking" approach is intentional** (e.g. a `SeeAllData` exception, a hard-coded ID
  with a link to the reason).
- **TODO/FIXME** with a ticket reference, not a vague note.

## Public APIs: document the contract

Use **ApexDoc** on public/global Apex and **JSDoc** on exported LWC functions/`@api` properties —
this is the *why/what* for callers, not how.

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

## Avoid

- Comments that restate the code.
- Commented-out code — delete it; version control remembers.
- Stale comments — a wrong comment is worse than none. Update or remove when code changes.

## Checklist

- [ ] Each comment explains *why*, not *how*.
- [ ] Tried clearer naming/extraction before reaching for a comment.
- [ ] Public Apex/LWC APIs documented with ApexDoc/JSDoc.
- [ ] No commented-out code or stale comments left behind.
