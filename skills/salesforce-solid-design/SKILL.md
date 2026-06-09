---
name: salesforce-solid-design
description: "Apply the SOLID object-oriented design principles to Apex classes and interfaces. Use this skill when designing class responsibilities, introducing or shaping interfaces and abstract/virtual classes, deciding what depends on what, or refactoring a class that has grown to do too much: keep one reason to change per class (SRP), extend by adding code not editing it (OCP), keep subclasses substitutable (LSP), keep interfaces narrow (ISP), and depend on abstractions you can inject and mock (DIP). Triggers on 'design this class', 'should this be an interface', 'how do I make this testable / mockable', 'this class does too much', 'extend without modifying', or layering trigger/service/selector/domain code. Pairs with clean-salesforce-functions (method-level structure), salesforce-connascence (coupling vocabulary), and reviewing-apex (catching violations in existing code). Do NOT use for method-internal structure (clean-salesforce-functions), naming (salesforce-naming-things), or error-handling structure (clean-apex-error-handling)."
metadata:
  version: "1.0"
---

# Salesforce SOLID Design

SOLID is five class- and module-level design principles. Where `clean-salesforce-functions` keeps a single method honest, SOLID keeps the *boundaries between classes* honest: who owns what, what depends on what, and what you can change without a ripple. In Apex the payoff is concrete — code that obeys DIP can be unit-tested without a single SOQL query or DML statement, which is the difference between a 2-second test run and a slow, fragile one.

The principles are a lens, not a quota. Apply them where a class is changing for several reasons, where a `switch` keeps growing, or where a test forces you to insert records just to exercise logic. Don't manufacture interfaces for classes that have one implementation and always will (`salesforce-connascence`, `reviewing-apex` — speculative generality is a smell).

---

## Rules

### 1. SRP — Single Responsibility

A class should have one reason to change. The classic Apex violation is a trigger handler that validates, queries, applies business rules, and persists — four reasons to change in one file. Separate the layers:

```apex
// Bad — one class owns validation, querying, policy, and DML
public class AccountTriggerHandler {
    public void onAfterUpdate(List<Account> accounts) {
        for (Account a : accounts) {
            if (a.AnnualRevenue == null) { a.addError('Revenue required'); }
        }
        List<Contact> contacts = [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accounts];
        // ...tier recalculation policy inline...
        update contacts;
    }
}

// Good — each class has one reason to change
public class AccountTriggerHandler {            // routing only
    public void onAfterUpdate(List<Account> accounts) {
        new AccountTierService().recalculateTiers(accounts);
    }
}
public class AccountTierService { ... }         // business policy
public class ContactSelector { ... }            // querying
```

The standard Apex layering — **trigger → handler → service → selector/domain** — is SRP made structural. The handler routes context, the service holds policy, the selector owns SOQL, the domain object owns record-level behaviour. Each changes for its own reason.

### 2. OCP — Open for extension, closed for modification

You should be able to add a new variant without editing existing, tested code. The tell is a `switch`/`if-else` on a picklist or type that you reopen every time the business adds a case.

```apex
// Bad — every new shipping method edits this method and its tests
public Decimal calculateFee(String method, Decimal weight) {
    if (method == 'Standard') { return weight * 1.5; }
    else if (method == 'Express') { return weight * 3.0; }
    else if (method == 'Overnight') { return weight * 5.0 + 10; }
    throw new IllegalArgumentException(method);
}

// Good — add a class (and a CMDT row) to extend; this code never reopens
public interface ShippingStrategy {
    Decimal calculateFee(Decimal weight);
}
public class ExpressShipping implements ShippingStrategy {
    public Decimal calculateFee(Decimal weight) { return weight * 3.0; }
}
public class ShippingCalculator {
    public Decimal calculateFee(ShippingStrategy strategy, Decimal weight) {
        return strategy.calculateFee(weight);
    }
}
```

Pair this with **Custom Metadata Types** to map the picklist value to the Apex class name (`Type.forName(...)`), so admins extend behaviour by adding a CMDT row — no deploy. Don't reach for this until the second or third variant: the first `if` is not a strategy pattern (`clean-salesforce-functions` DRY-after-Rule-of-Three).

### 3. LSP — Liskov Substitution

A subtype must be usable anywhere its base type is, without surprising the caller. A subclass that strengthens preconditions, weakens postconditions, or throws where the base did not, breaks substitutability.

```apex
// Bad — subtype rejects input the base type accepts; callers break
public virtual class Discount {
    public virtual Decimal apply(Decimal amount) { return amount * 0.9; }
}
public class LoyaltyDiscount extends Discount {
    public override Decimal apply(Decimal amount) {
        if (amount < 100) { throw new DiscountException('Min 100'); }  // new precondition
        return amount * 0.8;
    }
}
```

If `LoyaltyDiscount` cannot stand in for `Discount` everywhere, the inheritance is wrong — model it as a separate type or a strategy chosen by the caller, not a subclass. A reliable check: every test written against the base class should pass when handed the subclass.

### 4. ISP — Interface Segregation

No client should be forced to depend on methods it does not use. A fat interface forces implementers to stub members that throw or return null — a smell visible in any class littered with `// not applicable`.

```apex
// Bad — read-only callers must still implement write methods
public interface DataStore {
    SObject getById(Id recordId);
    void save(SObject record);
    void deleteById(Id recordId);
}

// Good — split by client need; a selector implements only the read role
public interface Readable  { SObject getById(Id recordId); }
public interface Writable  { void save(SObject record); }
```

In Apex this matters most for **selectors vs. unit-of-work**: query classes implement a read interface, persistence classes implement a write interface. A service that only reads depends on `Readable` alone, so its test double stubs one method, not three.

### 5. DIP — Dependency Inversion

High-level policy should depend on an abstraction, not on a concrete low-level class — and the concrete class should be **injected**, not `new`-ed inside the policy. This is the single highest-leverage principle in Apex, because it is what makes a service unit-testable without the database.

```apex
// Bad — service is welded to SOQL; you cannot test it without inserting records
public class RenewalService {
    public void process() {
        List<Opportunity> opps = [SELECT Id, CloseDate FROM Opportunity WHERE StageName = 'Closed Won'];
        // ...policy...
    }
}

// Good — depend on an interface, inject the implementation
public interface OpportunitySelector {
    List<Opportunity> selectClosedWon();
}
public class RenewalService {
    private final OpportunitySelector selector;
    public RenewalService(OpportunitySelector selector) {   // injected
        this.selector = selector;
    }
    public void process() {
        List<Opportunity> opps = selector.selectClosedWon();
        // ...policy, now testable with a stubbed selector...
    }
}
```

In the test, pass a fake `OpportunitySelector` that returns an in-memory list — the policy is exercised with **zero DML and zero SOQL**, fast and deterministic. The same shape applies to callouts (inject an `HttpExecutor`) and to `@AuraEnabled` controllers (delegate to an injected service). Apex also supports `Test.createStub()` with `StubProvider` for generating these doubles. For the callout-specific mocking pattern see `generating-apex-test-httpmocks`; for the coupling rationale, depending on a *name* (the interface) instead of an *implementation* is connascence reduction — `salesforce-connascence`.

---

## Apex-Specific Notes

- **`virtual` and `abstract` are opt-in.** Apex classes and methods are final by default; a class must be declared `virtual` or `abstract` and a method `virtual`/`abstract` before it can be overridden. Design the extension point deliberately — you cannot retrofit overridability from a subclass.
- **Dependency injection is constructor-based.** Apex has no IoC container in the platform; inject through the constructor (or a setter for trigger-handler frameworks). Provide a default no-arg constructor that wires the real implementation so production callers stay simple.
- **CMDT is your OCP registry.** Custom Metadata Types + `Type.forName()` let you map data to strategy classes, so new variants ship as config, not redeploys.
- **`Test.createStub` / `StubProvider`** generate interface doubles without writing a fake class per test — pairs naturally with DIP.
- **Don't over-abstract.** One interface per concrete class with no second implementation in sight is speculative generality. Introduce the seam when the second case (or the testing need) actually arrives.

---

## Quick Checklist

- [ ] **SRP** — one reason to change per class; trigger/handler/service/selector layered, not merged
- [ ] **OCP** — new variants add a class (and ideally a CMDT row), not an edit to a growing `switch`
- [ ] **LSP** — every subclass passes the base class's tests; no strengthened preconditions or new exceptions
- [ ] **ISP** — no implementer forced to stub methods it does not use; read and write roles split
- [ ] **DIP** — policy depends on an injected interface; the service tests with zero SOQL/DML
- [ ] No interface introduced before a second implementation or a real testing seam exists
