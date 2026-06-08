---
name: clean-apex-error-handling
description: "Apply Clean Code error-handling principles to Apex. Use this skill when writing or refactoring Apex that can fail: prefer exceptions over status-code/Boolean returns, define custom exception classes that carry context, never return or pass null (return empty collections instead), separate error handling from business logic, and write the try/catch scope first. Triggers when generating callouts, DML-heavy services, @AuraEnabled methods, or any code with try/catch, and when a user asks about exceptions, null handling, or error patterns. Do NOT use for method structure generally (clean-apex-functions) or callout test mocking (generating-apex-test-httpmocks)."
metadata:
  version: "1.0"
  source: "Clean Code (Robert C. Martin), Chapter 7 — Error Handling"
---

# Clean Apex Error Handling

Error handling is important, but if it obscures the logic it is wrong. The goal is code where the business logic reads cleanly and the error handling is separable from it. Apex error handling is one of the most inconsistent areas across Salesforce projects — these rules make it predictable.

---

## Rules

### 1. Use exceptions, not return codes or status flags

Returning a `Boolean` success flag or an error code forces every caller to check immediately and tangles the happy path with error checks. Throw an exception instead; the calling code stays clean.

```apex
// Bad — caller must remember to check, logic is obscured
public Boolean updateAccountStatus(Id accountId, String status) {
    Account acc = getAccount(accountId);
    if (acc == null) {
        return false;
    }
    acc.Status__c = status;
    update acc;
    return true;
}

// Good — happy path is clear, failure is exceptional
public void updateAccountStatus(Id accountId, String status) {
    Account account = getAccount(accountId);   // throws if not found
    account.Status__c = status;
    update account;
}
```

### 2. Write the try-catch-finally scope first

A `try` block is like a transaction: the `catch` must leave the program in a consistent state regardless of what failed in the `try`. When writing code that can throw, start with the try/catch structure so you define up front what the caller can expect on failure.

```apex
public List<RenewalRecord> retrieveRenewals(String segment) {
    try {
        return parseRenewals(callRenewalApi(segment));
    } catch (CalloutException e) {
        throw new RenewalServiceException(
            'Renewal API call failed for segment ' + segment, e
        );
    }
}
```

### 3. Define custom exceptions in terms of the caller's needs

Create meaningful custom exception types named for the failure, not for its mechanism. This lets callers catch the category they care about and gives logs a useful name. Avoid catching bare `Exception` and avoid throwing bare `AuraHandledException('Error')` with no context.

```apex
// Bad
throw new AuraHandledException('Error');

// Good — named, contextual, chains the cause
public class RenewalServiceException extends Exception {}

throw new RenewalServiceException(
    'Could not calculate renewal date for opportunity ' + opp.Id, e
);
```

For `@AuraEnabled` methods, throw `AuraHandledException` with a message the UI can show, but only at the boundary — let the service layer throw typed exceptions, and translate at the controller.

```apex
@AuraEnabled
public static RenewalDto getRenewal(Id opportunityId) {
    try {
        return RenewalService.calculateRenewal(opportunityId);
    } catch (RenewalServiceException e) {
        throw new AuraHandledException(e.getMessage());
    }
}
```

### 4. Provide context with every exception

An exception with no context is hard to diagnose in a production log. Include what was being attempted and the key identifiers, and chain the original cause as the second constructor argument so the stack trace survives.

```apex
// Bad
catch (DmlException e) {
    throw new RenewalServiceException('Update failed');   // lost the cause
}

// Good
catch (DmlException e) {
    throw new RenewalServiceException(
        'Failed to update ' + renewals.size() + ' renewal opportunities', e
    );
}
```

### 5. Don't return null

Returning `null` pushes a null check onto every caller, and one forgotten check is a production `NullPointerException`. Return an empty collection, an empty result object, or throw.

```apex
// Bad
public List<Contact> getContacts(Id accountId) {
    if (accountId == null) {
        return null;
    }
    return [SELECT Id FROM Contact WHERE AccountId = :accountId];
}

// Good
public List<Contact> getContacts(Id accountId) {
    if (accountId == null) {
        return new List<Contact>();
    }
    return [SELECT Id FROM Contact WHERE AccountId = :accountId];
}
```

A SOQL query assigned to a `List` never returns null in Apex — it returns an empty list — so returning the query result directly is already null-safe.

### 6. Don't pass null

Passing `null` into a method is even worse than returning it. Validate at the public boundary and throw a clear exception, rather than letting `null` propagate into the logic.

Throw a typed exception, not bare `Exception` — Apex won't even let you instantiate `System.Exception` directly, and a named type lets callers catch the category they care about (rule 3).

```apex
// Good — fail fast at the boundary with a typed exception
public void scheduleRenewal(Opportunity opportunity) {
    if (opportunity == null) {
        throw new RenewalServiceException('opportunity must not be null');
    }
    ...
}
```

### 7. Separate error handling from business logic

If a method's body is mostly try/catch plumbing, extract the body so the method *is* the error boundary and the extracted method is pure logic. Error handling is one thing; a method that handles errors should do nothing else.

```apex
public void sendRenewalNotices(List<Opportunity> renewals) {
    try {
        deliverNotices(renewals);          // the one thing it does
    } catch (EmailException e) {
        logFailure(renewals, e);           // error handling, separated
    }
}
```

---

## Apex-Specific Notes

- **DML partial success.** For bulk DML, decide deliberately between all-or-nothing (`insert records;`) and partial success (`Database.insert(records, false)` then inspect `SaveResult`). Don't silently swallow `SaveResult` errors — surface or log them with context.
- **Governor-limit exceptions** (`LimitException`) cannot be caught and recovered in the normal sense — design to stay within limits rather than catching them.
- **`finally`** is the right place to release locks or reset state, but Apex has no `IDisposable` — most cleanup is logging or flag resets.
- **Don't catch what you can't handle.** Catching an exception only to re-throw it unchanged adds nothing. Either add context, recover, or let it propagate.

---

## Quick Checklist

- [ ] Throws exceptions instead of returning success flags / codes
- [ ] Custom exception types named for the failure
- [ ] Every thrown exception carries context + chained cause
- [ ] `AuraHandledException` only at the controller boundary
- [ ] Never returns `null` (empty collection or throw)
- [ ] Validates and rejects `null` arguments at the boundary
- [ ] Error handling separated from business logic
- [ ] Bulk DML `SaveResult` errors surfaced, not swallowed
