---
name: lwc-component-design
description: "Design Lightning Web Components with the grain of the framework — reactivity, data flow, lifecycle, and composition. Use this skill when building or refactoring an LWC: deciding what is `@api` vs internal state, getting reactivity right (immutable reassignment, getters not stored derived state), choosing `@wire` vs imperative Apex, placing work in the correct lifecycle hook, communicating between components (props down / events up, never reaching into children), and composing with slots and child components instead of one mega-component. Triggers on 'build an LWC', 'this component does too much', 'why isn't my LWC re-rendering', '@wire vs imperative', 'pass data between components', 'parent/child communication', 'where should this go in the lifecycle', or 'split this component'. Pairs with clean-salesforce-functions (method structure inside the component), salesforce-naming-things (component/property naming), salesforce-law-of-demeter (don't reach through wired data or into children), salesforce-design-patterns (composition, strategy maps), and salesforce-writing-tests (Jest). Do NOT use for Apex class design (salesforce-solid-design), Apex method structure (clean-salesforce-functions), or writing the Jest tests themselves (salesforce-writing-tests)."
metadata:
  version: "1.0"
---

# LWC Component Design

Lightning Web Components reward code written *with the grain* of the framework: data flows down through properties, events flow up, the engine re-renders from observed state, and behaviour is assembled from small components rather than inherited. Most LWC pain — stale renders, untestable blobs, tangled parent/child coupling — comes from fighting that grain. This skill is about component-level design decisions; for the structure of the functions *inside* a component, defer to `clean-salesforce-functions`, and for naming to `salesforce-naming-things`.

---

## Rules

### 1. Public API (`@api`) is a contract — keep it small and intentional

A component's `@api` properties and methods are its public surface; everything else is private state. Expose the minimum. Each `@api` property is an input owned by the parent; each `@api` method is a command the parent may issue.

- **Inputs are `@api` properties; internal working state is plain fields.** Don't expose state the parent has no business setting.
- **Never mutate an `@api` property from inside the component.** It is owned upstream. Derive from it (a getter) or copy it into private state if you must change it.
- **A growing list of `@api` boolean flags that drive a `switch` is a smell** — that's a flag argument at the component boundary (`clean-salesforce-functions`, `salesforce-design-patterns`). Prefer a single `variant`/`mode` string mapped through a config, or separate components.

```js
// Bad — mutating the input the parent owns
@api records;
connectedCallback() {
  this.records.sort((a, b) => a.Name.localeCompare(b.Name)); // mutates parent's array
}

// Good — derive; never touch the input
@api records;
get sortedRecords() {
  return [...this.records].sort((a, b) => a.Name.localeCompare(b.Name));
}
```

### 2. Reactivity: reassign, don't deep-mutate; derive with getters

The engine re-renders when an observed field is **reassigned**. Mutating inside an object or array the field already points to may not be observed. Fields holding objects/arrays need `@track` (or a fresh reassignment) for nested changes to render.

- **Replace, don't poke.** `this.items = [...this.items, newItem]` re-renders; `this.items.push(newItem)` may not.
- **Don't store derived state.** Computing a value into a field means you must remember to recompute it whenever its inputs change — a temporal-coupling bug factory. Use a **getter** so it's always current. Getters are re-evaluated each render, so keep them cheap and **pure** (no mutation, no Apex calls — `clean-salesforce-functions` CQS).

```js
// Bad — derived state goes stale when firstName/lastName change
@api firstName;
@api lastName;
fullName = `${this.firstName} ${this.lastName}`; // computed once, never updates

// Good — always current
get fullName() {
  return `${this.firstName} ${this.lastName}`;
}
```

### 3. Choose `@wire` vs imperative Apex deliberately

- **`@wire`** for reactive, cacheable reads that should refresh when their reactive inputs change (`@wire(getContacts, { accountId: '$recordId' })`). The platform manages caching and re-invocation. Pair updates with `refreshApex` rather than manual re-fetch.
- **Imperative Apex** (`await getContacts({ accountId })`) when you need a call on a user action, ordering/sequencing, try/catch around the result, or a one-shot fetch that shouldn't re-run reactively.
- Don't fake one with the other: don't call imperative Apex inside a getter (runs every render), and don't fight `@wire` with flags to stop it firing — if the call is event-driven, make it imperative.

### 4. Put work in the right lifecycle hook

- **`constructor`** — cheap field init only. No DOM, no access to `@api` props (not set yet).
- **`connectedCallback`** — element inserted; `@api` props available. Kick off one-shot imperative loads, subscribe to message channels/events here.
- **`renderedCallback`** — runs after *every* render; guard with a flag for one-time DOM work and **never** unconditionally set reactive state here (infinite re-render). Last resort, not a data hook.
- **`disconnectedCallback`** — unsubscribe, clear timers; mirror every subscription made in `connectedCallback`.
- **`errorCallback(error, stack)`** — catch errors from descendant components (the LWC error boundary).

### 5. Data down, events up — and never reach across the boundary

Parents pass **data down** through properties and issue commands through `@api` methods. Children notify **up** by dispatching `CustomEvent`. Neither side reaches into the other's internals (`salesforce-law-of-demeter`).

- **Parent → child:** set properties; call `child.refresh()` via an `@api` method for imperative commands.
- **Child → parent:** `this.dispatchEvent(new CustomEvent('selectrecord', { detail: { recordId } }))`. Lowercase, no spaces; carry data in `detail`.
- **Never** `this.template.querySelector('c-child').someInternalField` to read or write a child's state — that's a cross-component train wreck.
- For distant or sibling components, use **Lightning Message Service**, not a chain of re-dispatched events through unrelated ancestors.

```js
// child — notify up, don't expect the parent to read your state
handleSelect() {
  this.dispatchEvent(new CustomEvent('select', { detail: { id: this.record.Id } }));
}
```

### 6. Compose with child components and slots; split when a component does too much

One reason to change per component (the SRP shape from `salesforce-solid-design`, applied to UI). When a component juggles unrelated concerns — fetching, a list, a form, and a modal — split it and compose. Use `<slot>` to let parents inject markup into a reusable shell rather than baking every variation in (`salesforce-design-patterns`, composition over inheritance — LWC has no useful component inheritance).

Signs a component should be split:
- The JS file has several unrelated `@wire`s / data sources.
- The template has large, independent regions toggled by flags.
- You're tempted to add a `mode`/`type` flag that changes most of the render.

### 7. Keep components thin — business logic belongs in Apex or shared modules

A component orchestrates UI; it should not own business rules. Push reusable, testable logic into imported ES modules (`c/utils`) or into Apex behind `@AuraEnabled`. This keeps logic unit-testable without rendering and keeps the component focused on presentation and wiring.

---

## Anti-patterns to flag

- **Mutating an `@api` input** in place (parent owns it; mutation won't reliably render and breaks the contract).
- **Stale derived state** stored in a field instead of a getter.
- **`renderedCallback` setting reactive state unconditionally** → infinite render loop.
- **Reaching into a child** via `querySelector(...).privateThing` instead of props-down/events-up.
- **Imperative Apex inside a getter** (fires every render) — and conversely **`@wire` throttled by flags** when the call is really event-driven.
- **God component**: one `.js` with many data sources and flag-toggled template regions.
- **Magic strings** for event names, picklist values, and channels duplicated across components (`salesforce-primitive-obsession` — hoist to a constants module).
- **Business logic in the component** that can't be tested without rendering.

---

## Quick Checklist

- [ ] `@api` surface is minimal and intentional; inputs are never mutated internally
- [ ] No flag-`@api` `switch`; variants chosen by `mode` + config or separate components
- [ ] Reactive updates **reassign** (`[...arr]`, new object); no in-place mutation expecting a render
- [ ] Derived values are **pure getters**, not stored fields that can go stale
- [ ] `@wire` for reactive cacheable reads (with `refreshApex`); imperative Apex for actions/sequencing
- [ ] Each lifecycle hook does its job; `renderedCallback` guarded; subscriptions cleaned up in `disconnectedCallback`
- [ ] Data flows **down via props**, events flow **up via `CustomEvent`**; no reaching into children
- [ ] Distant components communicate via **LMS**, not event relays
- [ ] Composed from child components and **slots**; oversized components split by responsibility
- [ ] Business logic lives in **Apex or shared modules**, not the component
- [ ] Event names / picklist literals / channels are **named constants**, not scattered strings
