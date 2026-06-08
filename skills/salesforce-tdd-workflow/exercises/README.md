# Apex TDD Exercises

Seven katas reworked as Salesforce/Apex test-driving exercises, adapted from an XP TDD workshop. These are **practice specs, not skills** — you work through them by hand to sharpen test-driven development. Each restates the kata in our own words, maps it to an Apex domain, and gives a test progression to follow assertion-first.

Ordered roughly easiest → hardest:

| File | Exercise | Practises |
|------|----------|-----------|
| `01-fizzbuzz.md` | FizzBuzz | Rules-as-data, avoiding conditionals, CMDT-style config thinking |
| `02-leap-year.md` | Leap year | Pure boundary testing; one branch per test |
| `03-word-wrap.md` | Word wrap | Triangulation on a string algorithm; greedy fill |
| `04-word-frequency.md` | Word frequency histogram | Normalisation rules; output-structure design |
| `05-expression-evaluator.md` | Expression evaluator | Emergent design — trivial grows into a parser |
| `06-billing-engine.md` | Usage-rated billing engine | Strategy via polymorphism; rounding/peak boundaries |
| `07-shipping-calculator.md` | Order shipping calculator | Availability vs cost; typed exceptions; interacting constraints |

The first five are the classic algorithm katas (good general drills). The last two are **Salesforce-shaped** — usage-rated pricing and order fulfilment with availability rules are real Apex work and are wall-to-wall boundary conditions, so they're the highest-value practice if you only do two.

## How to use them

Work each one with the `salesforce-tdd-workflow` skill open:
- One rule at a time, RED → GREEN → REFACTOR.
- Write the **assertion first** each time.
- Test **both sides of every boundary** — these katas are mostly boundaries.
- Let the second or third rule *force* a refactor (strategy table, parser, availability model); don't pre-build it.

They also exercise the `clean-apex-*` skills: naming, small single-responsibility methods, no magic numbers (prices belong in constants/Custom Metadata), and error handling that throws typed exceptions instead of returning sentinels.

## Suggested rotation

The workshop's own homework was: pick one kata, do it daily in 15 minutes, delete it each time, and vary one thing each run (different approach, different order, leave the refactoring out and see what happens). That repetition — not finishing once — is where the muscle memory comes from.

## On provenance

The katas come from an XP TDD workshop pack. FizzBuzz, leap year, word wrap, word frequency, and expression evaluation are standard, widely-published exercises and not proprietary; the billing and shipping rules are restated in our own words and remapped to a Salesforce domain. The Apex modelling and test progressions are original.
