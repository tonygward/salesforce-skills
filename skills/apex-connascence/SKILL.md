---
name: apex-connascence
description: "Analyse and reduce coupling in Apex and LWC using connascence — a strength-ranked taxonomy of dependency between components. Use this skill when asking how coupled two pieces of code are, why a change rippled into unexpected places, whether a dependency is fragile, or how to make code easier to change: name the kind of connascence, judge it by strength/degree/locality, and convert strong forms to weaker ones. Triggers on 'reduce coupling', 'this is fragile', 'why did changing X break Y', 'tighten/loosen the dependency', or prioritising refactors. Pairs with reviewing-apex (coupling as one smell category) and clean-apex-functions (argument objects, DRY). Do NOT use for general smell review (reviewing-apex) or authoring method structure (clean-apex-functions)."
metadata:
  version: "1.0"
  source: "Connascence — Meilir Page-Jones, 'What Every Programmer Should Know About Object-Oriented Design'; strength ordering and rules of degree/locality popularised by Jim Weirich — adapted to Apex"
---

# Apex Connascence

Two components are **connascent** if changing one forces a change in the other to keep the system correct. Connascence is a vocabulary for coupling that lets you *name* a dependency, *rank* how fragile it is, and *aim* a refactor — always toward a weaker form. It underpins many Clean Code rules: an argument object (`clean-apex-functions`) is just converting connascence of *position* into connascence of *name*; replacing a magic picklist string with a constant converts connascence of *meaning* into connascence of *name*.

---

## Judge every dependency on three axes

- **Strength** — how hard the coupling is to refactor. Weaker forms (Name, Type) are cheap to change; stronger forms (Algorithm, Value, Identity) are expensive and error-prone. **Refactor toward weaker forms.**
- **Degree** — how many components share the dependency. A name two methods agree on is low-degree; a picklist string hardcoded in forty places is high-degree. **Reduce degree.**
- **Locality** — how far apart the coupled elements are. Strong connascence inside one short method is fine; the *same* strength across class or package boundaries is a defect. **The further apart, the weaker the connascence must be.**

---

## The forms, weakest → strongest

### Static (visible at compile time)

| Form | The agreement | Apex example | Weaken it by |
|------|---------------|--------------|--------------|
| **Name** (CoN) | Multiple places must agree on a name | Method/field/variable names | (weakest — leave it; rename via tooling) |
| **Type** (CoT) | Must agree on a type | SObject field types, method signatures | Mostly compiler-enforced in Apex |
| **Meaning** (CoM) | Must agree what a *value* means | `'Closed Won'`, `'012…AAE'` RecordType Id, `status == 2`, a Boolean flag arg | Constants, **enums**, Custom Metadata (CoM → CoN) |
| **Position** (CoP) | Must agree on *order* | Positional method params, dynamic-SOQL field order, CSV column order | **Argument object** / named fields (CoP → CoN) |
| **Algorithm** (CoA) | Both ends must run the *same* algorithm | Apex↔external JSON/serialisation contract, hashing/signing, a calc duplicated two ways | Single shared method; one source of the contract (DRY) |

### Dynamic (visible only at runtime — generally stronger and harder)

| Form | The agreement | Apex example | Weaken it by |
|------|---------------|--------------|--------------|
| **Execution order** (CoE) | Steps must run in a set order | Trigger order of execution; `configure()` must run before `process()`; recursion guard set before re-entrant DML | Make operations order-independent; enforce order inside one method (Stepdown Rule) |
| **Timing** (CoTi) | *When* things run matters | Callout-before-DML ("uncommitted work pending"), Mixed DML, `@future`/Queueable racing, platform-event delivery timing | Make async boundaries explicit; never depend on wall-clock or dispatch order |
| **Value** (CoV) | Several values must change *together* | `Start_Date__c`/`End_Date__c` invariant, `Amount = Quantity × UnitPrice`, rollup vs detail totals | Encapsulate the invariant in one place (domain class / validation rule) |
| **Identity** (CoI) | Must reference the *same* instance | Two methods must mutate the *same* in-memory `Account`; re-querying yields a different instance → lost update | Single source of truth; pass one instance; Unit of Work |

> Static forms break loudly (compile error). Dynamic forms break *silently in production* — which is why CoE/CoTi/CoV/CoI are the highest-value findings in Apex.

---

## The two rules that drive refactoring

**Rule of Degree** — convert high-degree connascence to low-degree. One picklist value referenced in forty classes (degree 40) becomes one constant referenced forty times (degree 1: everyone depends on the constant, not on each other).

**Rule of Locality** — as the distance between elements grows, use weaker forms. Connascence of position between two lines of one method is harmless. The same positional coupling between an Apex controller and an LWC, or between your code and an external API, is fragile and must be weakened (to Name) or hidden behind an interface.

Together: **minimise connascence overall, and especially across encapsulation boundaries.** Strong connascence is acceptable only when it is also local.

---

## Worked Apex conversions

```apex
// CoM (meaning) across the codebase — every caller must "know" what 'Closed Won' means
if (opp.StageName == 'Closed Won') { ... }
// → CoN: one name, degree collapses to 1
if (opp.StageName == OpportunityStages.CLOSED_WON) { ... }
```

```apex
// CoP (position) — callers must remember argument order; reordering breaks every call site
createTask(whatId, ownerId, subject, dueDate, priority, true);
// → CoN: named fields, order no longer load-bearing
createTask(new TaskRequest{ whatId = whatId, ownerId = ownerId, subject = subject, ... });
```

```apex
// CoI (identity) — silent lost update: handler mutates a fresh query, not the trigger's record
for (Account a : [SELECT Id, Rating FROM Account WHERE Id IN :Trigger.newMap.keySet()]) {
    a.Rating = 'Hot';   // mutates a different instance than Trigger.new — never saved
}
// → operate on the one instance the framework will persist
for (Account a : Trigger.new) { a.Rating = 'Hot'; }
```

---

## How this relates to the other skills

- **`reviewing-apex`** lists coupling as one smell category; this skill is the strength-ranked lens for *prioritising* those findings (lead with the strong, non-local, high-degree ones).
- **`clean-apex-functions`** — argument objects and DRY are CoP→CoN and CoA reductions; flag arguments are CoM.
- **`salesforce-naming-things`** — CoN is the destination of almost every conversion, so good names are what make weak coupling readable.

---

## Quick Checklist

- [ ] Named the kind of connascence for each suspect dependency
- [ ] Ranked findings by strength × degree × distance — strong + non-local + high-degree first
- [ ] No strong connascence (Position, Algorithm, or any dynamic form) across class/package/integration boundaries
- [ ] Magic values (CoM) replaced by constants / enums / Custom Metadata
- [ ] Positional coupling (CoP) on long signatures replaced by an argument object
- [ ] Dynamic coupling (order, timing, shared invariants, shared identity) made explicit or encapsulated, not assumed
- [ ] High-degree dependencies routed through one definition to collapse degree
