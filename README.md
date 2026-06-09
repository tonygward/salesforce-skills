# Tony's Salesforce Skills

Public Salesforce AI skills for use with Claude Code and compatible skill runners.

## Installation

```bash
npx skills add tonygward/salesforce-skills --yes
```

## Usage

Once installed, invoke a skill by name in your Claude Code session:

```
/salesforce-naming-things
/salesforce-code-comments
/salesforce-writing-tests
/salesforce-tests-as-specification
/salesforce-tdd-workflow
/generating-apex-test-httpmocks
/clean-salesforce-functions
/clean-apex-error-handling
/reviewing-apex
/salesforce-connascence
/salesforce-solid-design
/salesforce-design-patterns
/salesforce-primitive-obsession
/salesforce-law-of-demeter
/lwc-component-design
```

## Skills

### Salesforce Fundamentals

| Skill | Description |
|---|---|
| `salesforce-naming-things` | Naming conventions for Apex and LWC — intent-revealing names, no encodings, Salesforce conventions |
| `salesforce-code-comments` | When and how to comment Salesforce code — the why-not-how rule, ApexDoc/JSDoc, what to delete |

### Testing

| Skill | Description |
|---|---|
| `salesforce-writing-tests` | Write high-quality Apex and LWC Jest tests — FIRST principles, Assert class, bulk testing, mocking |
| `salesforce-tests-as-specification` | Tests as a living spec — one rule per test, Given/When/Then, boundary cases |
| `salesforce-tdd-workflow` | TDD workflow for Salesforce — red/green/refactor, three laws of TDD, one change at a time |
| `generating-apex-test-httpmocks` | Generate Apex test classes for HTTP callout code using the self-shunt pattern |

### Clean Code

Adapted from *Clean Code* (Robert C. Martin) — authoring skills applied as you write, `reviewing-apex` for existing code.

| Skill | Source | Use when |
|---|---|---|
| `clean-salesforce-functions` | Ch 3 — Functions | Writing or refactoring Apex methods or LWC JS functions/handlers — small, one thing, few args |
| `clean-apex-error-handling` | Ch 7 — Error Handling | Callouts, DML services, exceptions, null handling |
| `reviewing-apex` | Ch 17 + 6, 9, 10 | Reviewing or critiquing existing Apex code |

### Coupling

| Skill | Source | Use when |
|---|---|---|
| `salesforce-connascence` | Page-Jones / Weirich — connascence | Judging how coupled code is, why a change rippled, or prioritising coupling refactors by strength/degree/locality |

### Design

| Skill | Source | Use when |
|---|---|---|
| `salesforce-solid-design` | Robert C. Martin — SOLID | Designing class responsibilities and interfaces, making a class testable/mockable, or refactoring a class that does too much (SRP/OCP/LSP/ISP/DIP) |
| `salesforce-design-patterns` | GoF / Head First — Decorator & Strategy | Behaviour needs to vary or stack at run time (Apex or LWC), a subclass tree / component-variant set is exploding, or a type/picklist `switch` keeps reopening — compose over inherit |
| `salesforce-primitive-obsession` | Refactoring / Clean Code Ch 6 | An `Id`/`String`/`Decimal` (or a JS magic string) is really a domain concept — wrap it in an immutable value object, or hoist literals to constants |
| `salesforce-law-of-demeter` | Lieberherr & Holland / Tell-Don't-Ask | Getter, relationship, or LWC template/`@wire` chains (`a.b.c.d`) are fragile, or code asks an object/child component for state to make a decision it should make itself |

### Lightning Web Components

| Skill | Source | Use when |
|---|---|---|
| `lwc-component-design` | LWC framework idioms | Building or refactoring an LWC — `@api` surface, reactivity, `@wire` vs imperative Apex, lifecycle hooks, data-down/events-up, composition with slots |

> **Apex and LWC:** naming, comments, tests, `clean-salesforce-functions`, `salesforce-connascence`, `salesforce-design-patterns`, `salesforce-primitive-obsession`, `salesforce-law-of-demeter`, and `lwc-component-design` all apply to both Apex and LWC JavaScript. **Apex-specific:** `salesforce-solid-design` (OO class design), `clean-apex-error-handling`, `reviewing-apex`, and `generating-apex-test-httpmocks`.


## Exercises

`salesforce-tdd-workflow` ships with seven Apex TDD katas in
[`skills/salesforce-tdd-workflow/exercises/`](skills/salesforce-tdd-workflow/exercises/).
These are **practice specs, not skills** — work them by hand, assertion-first, one
RED → GREEN → REFACTOR cycle at a time, to build test-driving muscle memory.

| Exercise | Practises |
|---|---|
| FizzBuzz | Rules-as-data, avoiding conditionals, CMDT-style config thinking |
| Leap year | Pure boundary testing; one branch per test |
| Word wrap | Triangulation on a string algorithm; greedy fill |
| Word frequency | Normalisation rules; output-structure design |
| Expression evaluator | Emergent design — trivial grows into a parser |
| Billing engine | Strategy via polymorphism; rounding/peak boundaries |
| Shipping calculator | Availability vs cost; typed exceptions; interacting constraints |

The last two are Salesforce-shaped (usage-rated pricing, order fulfilment) and are the
highest-value practice. See the [exercises README](skills/salesforce-tdd-workflow/exercises/README.md)
for how to use them.

## Contributing

Each skill lives in `skills/<name>/SKILL.md`.
