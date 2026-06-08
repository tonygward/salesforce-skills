---
name: apex-primitive-obsession
description: "Replace raw primitives that stand in for domain concepts with small, immutable Apex value objects. Use this skill when an Id, String, Decimal, or Boolean is carrying domain meaning (a record Id, a picklist value, an email, a money amount, a percentage), when the same primitive-plus-validation appears in several places, or when related primitives always travel together: wrap the concept in a class that validates on construction, is immutable, and attracts the behaviour that belongs to it. Triggers on 'this String is really an X', 'magic picklist value', 'validate this in one place', 'these fields always go together', 'make this immutable', or 'stop passing raw Ids around'. Ties to apex-connascence (primitive obsession is connascence of meaning) and clean-apex-functions (argument objects, value objects). Do NOT use for naming-only changes (salesforce-naming-things) or method structure (clean-apex-functions)."
metadata:
  version: "1.0"
---

# Apex Primitive Obsession

Primitive obsession is using a `String`, `Decimal`, `Id`, or `Boolean` to represent a concept that deserves its own type — an email, a money amount, a country code, an account tier. The primitive carries no rules and no behaviour, so the validation and logic that belong *to the concept* get smeared across every caller. The fix is a **value object**: a small, immutable class that validates once on construction and becomes the home for that concept's behaviour.

This is the same problem `apex-connascence` calls **connascence of meaning** — every place that knows `'Active'` is the live status, or that a 9-char string is a country code, is coupled by a shared convention. Wrapping the concept converts it to the far weaker **connascence of name** (everyone just depends on the type).

---

## Why it hurts in Apex

- **Picklist and status strings** (`'Closed Won'`, `'GBP'`, `'Tier 1'`) get compared by literal in dozens of classes; one rename ripples everywhere and the compiler catches none of it.
- **Validation is duplicated.** The same `email.contains('@')` or `amount >= 0` check is copy-pasted, and inevitably one copy is missing.
- **Method signatures lie.** `transfer(Id from, Id to, Decimal amount)` accepts a Contact Id where an Account Id is meant, and pays in the wrong currency — all type-valid, all wrong.
- **Behaviour has no home.** Logic that operates on "money" or "an email" lands in a util class or a service instead of on the concept itself.

---

## Rules

### 1. Wrap the concept in a value object

When a primitive carries domain meaning, give it a type. Validate in the constructor so an invalid instance cannot exist.

```apex
// Bad — a raw String that is really an email, validated ad hoc everywhere
public void invite(String email) {
    if (!email.contains('@')) { throw new IllegalArgumentException('Bad email'); }
    // ...
}

// Good — invalid state is unrepresentable; validation lives once
public class EmailAddress {
    private final String value;
    public EmailAddress(String value) {
        if (String.isBlank(value) || !value.contains('@')) {
            throw new IllegalArgumentException('Invalid email: ' + value);
        }
        this.value = value.toLowerCase();
    }
    public override String toString() { return value; }
    public Boolean equals(Object o) {
        return (o instanceof EmailAddress) && ((EmailAddress) o).value == value;
    }
    public override Integer hashCode() { return value.hashCode(); }
}
```

`invite(EmailAddress email)` now cannot be called with a malformed string, and no caller repeats the check.

### 2. Make value objects immutable

A value object is defined by its value, so its value must not change after construction. Mark fields `final`, set them only in the constructor, and return new instances instead of mutating.

```apex
public class Money {
    public final Decimal amount;
    public final String currency;
    public Money(Decimal amount, String currency) {
        if (amount == null || currency == null) {
            throw new IllegalArgumentException('amount and currency required');
        }
        this.amount = amount;
        this.currency = currency;
    }
    // Operations return new instances — never mutate
    public Money plus(Money other) {
        if (other.currency != this.currency) {
            throw new IllegalArgumentException('Currency mismatch');
        }
        return new Money(this.amount + other.amount, this.currency);
    }
}
```

Immutability buys you: safe sharing (no caller can corrupt your copy), easy reasoning (state is fixed for the object's life), and freedom from a whole class of aliasing bugs. Two `Money` objects with the same amount and currency are equal — define `equals`/`hashCode` so they behave as values, not references.

### 3. Attract behaviour to the object

Once a concept has a type, move the logic that operates on it *onto* the type. This is where primitive obsession meets **Tell, Don't Ask** (`apex-law-of-demeter`) — instead of asking for the raw value and deciding outside, tell the object to do the work.

```apex
// Bad — logic about percentages lives on every caller
Decimal discounted = price - (price * discountPercent / 100);

// Good — the concept owns its behaviour
public class Percentage {
    private final Decimal value;
    public Percentage(Decimal value) {
        if (value < 0 || value > 100) { throw new IllegalArgumentException('0..100'); }
        this.value = value;
    }
    public Decimal of(Decimal base)        { return base * value / 100; }
    public Decimal applyAsDiscount(Decimal base) { return base - of(base); }
}
```

### 4. Group primitives that travel together

When the same cluster of primitives is passed around as a set — street, city, postcode, country — that clump *is* a concept. Make it one. (This is the argument-object move from `clean-apex-functions`, applied to data that recurs across the codebase, not just one signature.)

```apex
// Bad — a data clump passed limb by limb
createShipment(String street, String city, String postcode, String country) { ... }

// Good — an Address value object
public class Address {
    public final String street, city, postcode, country;
    public Address(String street, String city, String postcode, String country) { ... }
}
createShipment(Address shipTo) { ... }
```

---

## When a primitive is fine

Don't wrap for the sake of it. A raw primitive is correct when:
- It has no rules and no behaviour of its own — a free-text note, a row counter, a loop index.
- It is genuinely a number or string at the boundary (a log message, a raw HTTP body) with no domain identity.
- The wrapper would only ever hold the value and forward it — that is ceremony, not a value object (`reviewing-apex` speculative generality).

The trigger is *meaning + rules or behaviour*, not merely "it's a primitive."

---

## Apex-Specific Notes

- **No `record`/struct sugar.** Apex value objects are plain classes; keep them small (a few `final` fields, a constructor, a handful of behaviour methods).
- **Override `equals` and `hashCode`** if instances go into Sets/Maps or are compared — otherwise Apex compares by reference and two equal-valued objects won't match.
- **Picklist values → typed constants or an enum/value object.** Replace scattered `'Closed Won'` literals with a `StageName` value object or `enum`, so a relabel is one edit and the compiler guards call sites.
- **Ids are already typed at run time, not compile time.** `Id` doesn't distinguish Account from Contact at compile time; a thin `AccountId` wrapper restores that safety on critical paths where a mix-up is costly.
- **Don't over-wrap in bulk loops.** Constructing a value object per row across 50k records is usually fine, but if a hot path shows allocation pressure, wrap at the boundary and pass primitives internally.

---

## Quick Checklist

- [ ] Primitive that carries domain meaning + rules/behaviour → wrapped in a value object
- [ ] Validation happens **once**, in the constructor; an invalid instance cannot exist
- [ ] Value object is **immutable** (`final` fields, operations return new instances)
- [ ] `equals`/`hashCode` defined where instances are compared or keyed
- [ ] Behaviour about the concept lives **on the type**, not smeared across callers (Tell, Don't Ask)
- [ ] Recurring data clumps grouped into one type
- [ ] Picklist/status literals replaced by typed constants, enum, or value object
- [ ] Not wrapping primitives that have no rules or behaviour
