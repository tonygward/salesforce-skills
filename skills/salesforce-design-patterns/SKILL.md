---
name: salesforce-design-patterns
description: "Apply the Decorator and Strategy design patterns to Salesforce code — Apex and LWC JavaScript — and prefer composition over inheritance. Use this skill when behaviour needs to vary or stack at run time, when a subclass hierarchy is exploding to cover combinations, when an if/switch on a type keeps reopening, when an LWC grows variant flags or a parallel component per option, or when you want to add behaviour to an object without editing it: wrap with a Decorator, swap with a Strategy, compose rather than inherit. Triggers on 'add behaviour without changing the class', 'too many subclasses', 'too many component variants', 'vary this at run time', 'select behaviour by config/picklist', 'should this be inheritance', or 'apply a pattern here'. Realises the Open/Closed Principle from salesforce-solid-design and pairs with salesforce-connascence. Do NOT force a pattern where a single method suffices, and do NOT use for method-internal structure (clean-salesforce-functions) or general smell review (reviewing-apex)."
metadata:
  version: "2.0"
---

# Salesforce Design Patterns

Patterns are not a goal — they are names for shapes that *emerge* when you let behaviour vary by composition instead of by editing existing code. This skill covers the two that recur most in Salesforce business logic — **Strategy** (swap one behaviour) and **Decorator** (stack many behaviours) — both expressions of the same rule: **favour composition over inheritance.** Both realise the Open/Closed Principle (`salesforce-solid-design`): you extend the system by adding a class, never by reopening a tested one. The Apex examples carry the patterns; the **LWC (JavaScript) Notes** section shows the same shapes in component code, where Strategy lives in a config map and composition is the framework's native idiom (slots and child components).

Let patterns arrive through refactoring. The first `if` is not a Strategy and a single wrapper is not a Decorator (`clean-salesforce-functions` DRY-after-Rule-of-Three; `reviewing-apex` speculative generality). Reach for them when the *combinations* or *variants* start to multiply.

---

## Composition over inheritance — the core idea

Inheritance fixes behaviour at compile time and forces every combination into its own subclass. The classic explosion: a `Beverage` with optional Mocha, Soy, and Whip becomes `HouseBlendWithMochaAndWhip`, `EspressoWithSoy`, and so on — a class per combination. Composition assembles behaviour at run time from small parts instead.

Prefer composition when you see:
- A subclass hierarchy growing to cover *combinations* of options.
- A base class you must edit every time a new variant appears.
- Subclasses that override to *remove* behaviour (a Liskov violation — `salesforce-solid-design`).

Both patterns below replace an inheritance tree with objects that hold a reference to a collaborator.

---

## Rules

### 1. Strategy — swap one behaviour at run time

A Strategy captures one varying behaviour behind an interface so the caller can choose the implementation without changing its own code. Use it when an `if`/`switch` on a type or picklist keeps reopening for new cases.

```apex
// Bad — every new method edits this body and its tests
public Decimal fee(String method, Decimal weight) {
    if (method == 'Standard')  return weight * 1.5;
    if (method == 'Express')   return weight * 3.0;
    if (method == 'Overnight') return weight * 5.0 + 10;
    throw new IllegalArgumentException(method);
}

// Good — each variant is a class; the calculator never reopens
public interface ShippingStrategy {
    Decimal calculateFee(Decimal weight);
}
public class ExpressShipping implements ShippingStrategy {
    public Decimal calculateFee(Decimal weight) { return weight * 3.0; }
}
public class ShippingCalculator {
    public Decimal feeFor(ShippingStrategy strategy, Decimal weight) {
        return strategy.calculateFee(weight);
    }
}
```

**Apex selection.** Map the picklist value to the Strategy class with **Custom Metadata Types** + `Type.forName(className).newInstance()`, so a new strategy ships as a CMDT row, not a deploy. This keeps the *Open/Closed* boundary at config, not code.

### 2. Decorator — stack behaviour without editing the class

A Decorator implements the same interface as the object it wraps, adds its own behaviour, then delegates to the wrapped object. Because each decorator both *is* and *holds* the type, you can nest them to any depth. Use it to add behaviour to a class without changing its source — the textbook OCP move.

```apex
// The shared abstraction
public interface Beverage {
    String description();
    Decimal cost();
}

// A concrete component
public class Espresso implements Beverage {
    public String  description() { return 'Espresso'; }
    public Decimal cost()        { return 1.99; }
}

// A decorator: same interface, wraps a Beverage, adds cost
public class Mocha implements Beverage {
    private final Beverage wrapped;
    public Mocha(Beverage wrapped) { this.wrapped = wrapped; }
    public String  description() { return wrapped.description() + ', Mocha'; }
    public Decimal cost()        { return wrapped.cost() + 0.20; }
}

// Stack them at run time — no class per combination
Beverage order = new Whip(new Mocha(new Espresso()));
order.cost();          // 1.99 + 0.20 + 0.35
order.description();    // 'Espresso, Mocha, Whip'
```

**Apex uses that pay off:**
- **Cross-cutting concerns on a service.** Wrap a `LeadService` in a `LoggingLeadService` or `RetryingLeadService` that implements the same interface, adds logging/retry, then delegates. The core service stays untouched and untested-around.
- **Layered `HttpCalloutMock`.** A decorator mock can add latency, count invocations, or inject a failure on the Nth call while delegating to a base mock (`generating-apex-test-httpmocks`).

A decorator must stay **substitutable** for what it wraps (`salesforce-solid-design`, LSP) — same contract, additive behaviour only. Don't let a decorator throw where the component would not.

### 3. Don't force the pattern

A pattern earns its keep only when variation is real. Indicators you do **not** need one yet:
- One implementation, no second in sight → a plain class.
- A single conditional that has never changed → leave the `if`.
- A wrapper used in exactly one place that will never stack → inline it.

Introducing Strategy/Decorator prematurely adds an interface and indirection (stronger *connascence of name* spread wider — `salesforce-connascence`) for no extension benefit.

---

## Strategy vs Decorator — which one

| Question | Pattern |
|---|---|
| Pick **one of N** interchangeable behaviours? | **Strategy** |
| **Add to / stack** behaviour on top of an existing object? | **Decorator** |
| Behaviour chosen by config/picklist? | **Strategy** (+ CMDT) |
| Wrap a service/mock to layer logging, retry, metrics? | **Decorator** |

---

## LWC (JavaScript) Notes

Composition is LWC's native grain — the framework is built from small components assembled at run time. The patterns translate directly; you rarely need class inheritance.

- **Strategy as a lookup map.** When a handler grows an `if`/`switch` on a type, mode, or picklist, replace it with an object literal mapping key → function. Adding a variant is adding an entry, not editing the dispatch.

```js
// Bad — the switch reopens for every new format
formatValue(value, type) {
  switch (type) {
    case 'currency': return `$${value.toFixed(2)}`;
    case 'percent':  return `${value}%`;
    case 'date':     return new Date(value).toLocaleDateString();
    default:         return String(value);
  }
}

// Good — each strategy is an entry; the formatter never reopens
const FORMATTERS = {
  currency: v => `$${v.toFixed(2)}`,
  percent:  v => `${v}%`,
  date:     v => new Date(v).toLocaleDateString(),
};
formatValue(value, type) {
  return (FORMATTERS[type] ?? String)(value);
}
```

For behaviour an admin should control, the strategy key can come from **Custom Metadata** surfaced through `@wire` or an `@AuraEnabled` call — the JS analogue of the Apex CMDT registry.

- **Compose with child components and slots, not inheritance.** LWC has no useful component inheritance; you assemble UI by nesting components and projecting markup through `<slot>`. A `c-card` that exposes a default slot plus `header`/`actions` slots is the Decorator idea expressed in markup — callers stack content and behaviour around a core without editing it.
- **Decorator for wrapping behaviour.** The same wrap-and-delegate shape applies to plain JS: wrap an Apex-call function in a `withRetry(fn)` or `withToast(fn)` higher-order function that adds the concern and delegates. Each wrapper keeps the same call signature (substitutable) and adds behaviour only.
- **Prefer functions/modules over class hierarchies.** Where Apex reaches for an `interface`, LWC JS usually reaches for a function, a module export, or a passed-in callback. Same Open/Closed payoff — extend by adding a function, not editing a tested one — with less ceremony.
- **Don't force it.** A single `if` that has never changed, or one component variant, is not a pattern. Reach for the map or the slot only when variants/combinations actually multiply (`reviewing-apex` speculative generality).

---

## Apex-Specific Notes

- **Interfaces are the seam.** Apex has true `interface` types; both patterns hang off one. Classes/methods are `final` by default, so design the interface deliberately — you cannot retrofit a seam from outside.
- **No DI container.** Compose by constructor — pass the wrapped object or the strategy into the constructor. Offer a no-arg constructor wiring the real default so production call sites stay terse.
- **CMDT is the Strategy registry.** `Type.forName(...).newInstance()` turns a metadata row into a behaviour; admins extend without a deploy.
- **Decorator ≠ trigger handler framework.** If you already route by context in a handler, that's dispatch, not Decorator. Reach for Decorator when you need to *layer* concerns over a single collaborator.
- **Watch the cost of indirection.** Each decorator is a virtual call; deep stacks are fine for business logic but keep them out of tight bulk loops where a flat computation is clearer.

---

## Quick Checklist

- [ ] Behaviour varies → chosen a **Strategy** (one-of-N) or **Decorator** (stack), not a growing `switch` or subclass tree
- [ ] The pattern hangs off an **interface**; the core class is never reopened to extend it
- [ ] Composed via **constructor injection**; a no-arg default wires the real implementation
- [ ] Strategy variants resolved from **CMDT** where admins should extend without deploy
- [ ] Every decorator stays **substitutable** for what it wraps (same contract, additive only)
- [ ] The variation is real — not a single `if` or one-off wrapper dressed up as a pattern
