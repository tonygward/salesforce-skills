---
name: clean-apex-functions
description: "Apply Clean Code function principles to Apex methods. Use this skill when writing or refactoring Apex methods, trigger handlers, or service-layer logic: keep methods small and doing one thing, one level of abstraction per method, no flag (Boolean) arguments, few parameters (prefer an argument object), no side effects, and command-query separation. Triggers when generating Apex methods, when a method grows long or takes many arguments, or when a user asks to refactor or simplify Apex logic. Do NOT use for naming-only changes (salesforce-naming-things) or error-handling structure (clean-apex-error-handling)."
metadata:
  version: "1.0"
  source: "Clean Code (Robert C. Martin), Chapter 3 — Functions"
---

# Clean Apex Functions

The first rule of methods: they should be small. The second rule: they should be smaller than that. A method should do one thing, do it well, and do it only.

---

## Rules

### 1. Small — and do one thing

Target methods under ~20 lines. Flag anything over 30. A method does "one thing" if everything in it sits at a single level of abstraction and you cannot meaningfully extract another named method from it.

A reliable test: if you can extract a block and give the new method a name that is not merely a restatement of the original method's name, the original was doing more than one thing.

```apex
// Bad — fetches, filters, transforms, and persists in one method
public void processRenewals(List<Opportunity> opps) {
    List<Opportunity> toRenew = new List<Opportunity>();
    for (Opportunity opp : opps) {
        if (opp.StageName == 'Closed Won' && opp.CloseDate < Date.today().addDays(-330)) {
            toRenew.add(opp);
        }
    }
    List<Opportunity> renewals = new List<Opportunity>();
    for (Opportunity opp : toRenew) {
        Opportunity renewal = opp.clone(false, true, false, false);
        renewal.CloseDate = opp.CloseDate.addYears(1);
        renewal.StageName = 'Prospecting';
        renewals.add(renewal);
    }
    insert renewals;
}

// Good — each method is one level of abstraction
public void processRenewals(List<Opportunity> opportunities) {
    List<Opportunity> eligible = filterRenewable(opportunities);
    List<Opportunity> renewals = buildRenewals(eligible);
    insert renewals;
}
```

### 2. One level of abstraction per method

Don't mix high-level policy with low-level mechanics in the same method. `processRenewals` above reads as policy; the SOQL field comparisons and `clone()` calls belong one level down. This is the Stepdown Rule — the code should read top-to-bottom, each method followed by those one level beneath it.

### 3. Few arguments — prefer an argument object

Zero arguments is ideal, one or two is fine, three is suspect, and more than three needs strong justification. When parameters travel together, wrap them in a class.

```apex
// Bad
public void createTask(Id whatId, Id ownerId, String subject,
                       Date dueDate, String priority, Boolean isReminder) { ... }

// Good
public class TaskRequest {
    public Id whatId;
    public Id ownerId;
    public String subject;
    public Date dueDate;
    public String priority;
    public Boolean isReminderOn;
}

public void createTask(TaskRequest request) { ... }
```

In coupling terms this trades *connascence of position* (callers must remember the argument order) for the much weaker *connascence of name* — see `apex-connascence`.

### 4. No flag arguments

A Boolean parameter that switches behaviour means the method does more than one thing. Split it into two intention-revealing methods.

```apex
// Bad
processAccounts(accounts, true);   // what does true mean at the call site?

public void processAccounts(List<Account> accounts, Boolean isInsert) {
    if (isInsert) { ... } else { ... }
}

// Good
public void processInsertedAccounts(List<Account> accounts) { ... }
public void processUpdatedAccounts(List<Account> accounts) { ... }
```

This applies equally to trigger handlers — dispatch on context with separate handler methods (`onAfterInsert`, `onAfterUpdate`) rather than passing operation flags around.

### 5. No side effects

A method should not do something hidden that its name does not promise. A method called `checkPassword` that also initialises a session lies to its caller and creates temporal coupling.

```apex
// Bad — getter that mutates
public Decimal getDiscountedTotal() {
    this.total = this.total * 0.9;   // hidden mutation
    return this.total;
}

// Good — no hidden state change
public Decimal calculateDiscountedTotal() {
    return this.total * 0.9;
}
```

### 6. Command-Query Separation

A method should either **do** something (command, returns void, may have side effects) or **answer** something (query, returns a value, no side effects) — never both.

```apex
// Bad — sets and reports success in one call; ambiguous at call site
if (account.setAttribute('status', 'Active')) { ... }

// Good
if (account.hasAttribute('status')) {
    account.setAttribute('status', 'Active');
}
```

### 7. Don't Repeat Yourself

Duplicated logic is a defect multiplier. Repeated SOQL field lists, repeated validation blocks, and copy-pasted trigger logic should be extracted to a single method. In Apex this also protects you from governor limits, since duplicated queries are duplicated against limits.

### 8. Prefer exceptions to error-code returns

Returning status flags forces the caller to check immediately and clutters call sites. Throw instead. (Detailed handling structure is covered by `clean-apex-error-handling`.)

---

## Apex-Specific Notes

- **Bulkify, don't loop DML/SOQL.** "Do one thing" never justifies a query or DML statement inside a loop. Operate on collections; query once before the loop, `insert`/`update` once after.
- **Trigger handlers** stay thin: the handler routes context to service methods. No business logic in the trigger file itself.
- **`@AuraEnabled` methods** should delegate to a service method, not contain the logic, so the logic stays testable and reusable outside the LWC boundary.

---

## Quick Checklist

- [ ] Method under ~20 lines (hard flag at 30)
- [ ] Does one thing at one level of abstraction
- [ ] Three or fewer parameters (else argument object)
- [ ] No Boolean flag arguments
- [ ] No hidden side effects
- [ ] Command or query, not both
- [ ] No duplicated logic / repeated SOQL
- [ ] No SOQL or DML inside loops
