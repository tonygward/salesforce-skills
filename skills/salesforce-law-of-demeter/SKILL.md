---
name: salesforce-law-of-demeter
description: "Apply the Law of Demeter and Tell-Don't-Ask to Salesforce code — Apex and LWC JavaScript. Don't reach through objects, and command collaborators instead of querying their internals to decide. Use this skill when code chains accessors across several objects (a.b.c.d), when SObject relationship traversal walks deep into related records, when an LWC reaches deep into @wire data or another component's internals, when a method pulls data out of an object only to make a decision the object could make itself, or when you want to reduce the ripple from a structural change: talk only to immediate collaborators, and tell objects what to do rather than asking for their state. Triggers on 'train wreck', 'this getter chain is fragile', 'why did changing X break this', 'reduce coupling to internals', 'move this logic onto the object', or 'tell don't ask'. Pairs with salesforce-connascence (chains multiply connascence) and salesforce-primitive-obsession (give behaviour a home). Do NOT use for general smell review (reviewing-apex) or method-internal structure (clean-salesforce-functions)."
metadata:
  version: "2.0"
---

# Salesforce Law of Demeter

The Law of Demeter — *"don't talk to strangers"* — says a method should only call methods on: itself, its own fields, its parameters, and objects it creates. It should **not** reach *through* one object to get at another and call methods on that. The everyday symptom is the **train wreck**: `order.getCustomer().getAddress().getCountry().getCode()`. Each `.` past the first couples the caller to a structure it has no business knowing.

Its behavioural twin is **Tell, Don't Ask**: don't pull state out of an object to make a decision the object is better placed to make — *tell* it to act. Both push behaviour toward the data it operates on, which is exactly where `salesforce-primitive-obsession` wants it too.

The Apex examples below carry the principle; the **LWC (JavaScript) Notes** section translates it to component code, where the same train wreck appears in templates and `@wire` traversal.

---

## Why it bites in Apex

- **Relationship traversal is a train wreck generator.** `opp.Account.Owner.Manager.Name` couples the caller to four objects and the entire relationship path between them; reparent or restructure any link and every such chain breaks — and the compiler won't always warn you.
- **Each extra `.` multiplies connascence.** A chain spreads *connascence of name and type* across every intermediate object (`salesforce-connascence`); a change to any hop ripples to the caller. One dot per concept keeps that ripple short.
- **Ask-then-decide scatters policy.** Logic that should live on a class ends up in callers and services that interrogate it, producing the anemic-object / feature-envy smell `reviewing-apex` flags.

A clarification specific to Apex: a **single** SObject query that *projects* related fields (`SELECT Account.Owner.Name FROM Opportunity`) is a legitimate, bulk-friendly query — that's the database doing a join, not your Apex walking objects. Demeter is about **chained method/relationship calls in code paths and logic**, not about which fields one SOQL statement selects. The smell is traversal *threaded through business logic*, especially repeated per record.

---

## Rules

### 1. One dot per concept — don't reach through objects

Call methods on your immediate collaborators only. If you need something two hops away, ask the nearest object to provide it (or to do the work), so the path stays the collaborator's secret.

```apex
// Bad — train wreck: caller knows Order → Customer → Address → Country
String code = order.getCustomer().getAddress().getCountry().getCode();
if (code == 'GB') { applyUkTax(order); }

// Good — ask the immediate collaborator; the path is hidden
if (order.isUkBased()) { applyUkTax(order); }

// inside Order:
public Boolean isUkBased() {
    return customer.isUkBased();   // Order talks only to Customer
}
```

The fix is not a longer method that does the same walk in one place — it is **delegation**: each object answers for what it owns and forwards to the next.

### 2. Tell, Don't Ask — command, don't interrogate

If you find yourself getting an object's data, deciding something, then acting on that object, the decision belongs *inside* the object. Replace the query-and-branch with a command.

```apex
// Bad — pull the total out, decide outside, push a change back in
if (cart.getTotal() > 100) {
    cart.setDiscount(cart.getTotal() * 0.1);
}

// Good — tell the cart; it owns the rule
cart.applyBulkDiscount();

// inside Cart:
public void applyBulkDiscount() {
    if (total > 100) { discount = total * 0.1; }
}
```

This keeps related data and the rules about it together (high cohesion), and it removes the temporal coupling of "remember to set the discount after reading the total."

### 3. Distinguish objects from data structures

Tell-Don't-Ask applies to **objects** (behaviour-rich, hide their data). It does *not* apply to genuine **data structures / DTOs** — SObjects, wrapper records, `@AuraEnabled` response shapes — whose whole purpose is to expose fields. Reading `account.AnnualRevenue` is fine; an SObject is a data structure. The smell is *asking a behaviour-rich object for its internals to make its decision for it*. Don't add ceremony to bags of data, and don't let real domain objects degrade into bags of data (`reviewing-apex` — anemic domain model).

### 4. Where the law genuinely bends

- **SOQL field projection** across relationships in one query is allowed and preferred — it is a join, not a code-path chain (see above).
- **Fluent builders** (`new Query().selectField(x).whereEquals(y).build()`) return `this` each call; that is one object talking to itself, not a Demeter violation.
- **Standard library chains** on `String`/`List`/`Map` are idiomatic and not the target.

The law targets reaching across *your own domain's* collaborators in logic, not every chained call in the language.

---

## LWC (JavaScript) Notes

The law bites just as hard in components — the train wreck moves into templates and wired data.

- **Template train wrecks.** `{record.data.fields.Account.value.Owner.Name}` couples the markup to the entire SObject/UI-API shape; one structural change and the template silently renders blank. Expose a flat, intent-revealing getter and bind to that.

```js
// Bad — template reaches through the whole UI-API graph
//   <p>{record.data.fields.Owner.value.fields.Name.value}</p>

// Good — one getter owns the path; the template talks to its own component
get ownerName() {
  return this.record?.fields?.Owner?.value?.fields?.Name?.value;
}
//   <p>{ownerName}</p>
```

The getter still walks the path, but it does so in **one place the component owns**, so a UI-API change is a one-line fix, not a hunt across markup. Optional chaining (`?.`) handles the partial-load reality of wired data.

- **Don't reach into child components.** Querying a child and walking its internals — `this.template.querySelector('c-child').state.items.length` — is a cross-component train wreck. Instead, **tell** the child via a public `@api` method (`child.refresh()`) or let it **tell** the parent via a `CustomEvent`. Parents command children; children notify parents. Never read a child's private fields.
- **Tell, Don't Ask across components.** If a parent reads a child's data, decides, then pushes a change back into the child, the decision belongs in the child. Give the child an `@api` command (`cart.applyBulkDiscount()`) and let it own the rule — the same move as the Apex `Cart` above.
- **Wire/Apex data is a data structure.** Reading fields off a wired record or an `@AuraEnabled` response shape is fine — these are DTOs, not behaviour-rich objects. The smell is *threading deep traversal through component logic*, not accessing a field. Flatten the access into a getter and keep decisions on the component (or in a shared module), not smeared across handlers and markup.

---

## Apex-Specific Notes

- **Wrap deep relationship logic on a domain class.** If business rules repeatedly walk `Opp.Account.Owner...`, add an `OpportunityModel`/domain wrapper exposing `isOwnedBySenior()` so the traversal lives in one place and trigger/service code tells, not walks.
- **Query the path once, pass the answer.** Project the related fields you need in a single bulk SOQL, then have objects expose intent-revealing methods over that data — never re-traverse per record in a loop (governor limits *and* Demeter).
- **Behaviour belongs with the value.** A getter chain that ends in a comparison (`...getCountry().getCode() == 'GB'`) usually means the concept (`Country`) should own that question — combine with `salesforce-primitive-obsession` to give it a home.
- **SObjects stay data structures.** Don't wrap every SObject field access; do consolidate the *decisions* made from those fields onto a domain class.

---

## Quick Checklist

- [ ] No train wrecks — methods call only self, own fields, parameters, and objects they create
- [ ] Multi-hop needs answered by **delegation**, not a chain reproduced in one method
- [ ] Query-then-decide-then-act on the same object replaced by a **command** on it (Tell, Don't Ask)
- [ ] Decisions live **with the data** they concern (high cohesion, no feature envy)
- [ ] Real DTOs/SObjects left as data structures — no ceremony, but no domain logic leaking into them either
- [ ] Cross-relationship reads done as a **single SOQL projection**, not per-record traversal in code
