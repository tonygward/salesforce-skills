# Exercise: Usage-Rated Billing Engine (Apex TDD)

**Adapted from** the "Mobile phone billing" kata. Reframed as a Salesforce pricing/billing exercise to practise test-driving a complex business-rule engine in Apex.

## The exercise

You have a month's raw call and text records for one account. Test-drive an Apex class that calculates the bill. Build it one rule at a time, RED→GREEN→REFACTOR, following `salesforce-tdd-workflow`.

This is deliberately a rules-heavy calculator with many boundaries — the point is to practise triangulating from simple cases to the full rule set without the design collapsing into nested `if`s.

## Suggested Apex domain mapping

- An inner `UsageRecord` class (type: call/text, duration in seconds, destination type, peak/off-peak, timestamp) rather than raw SObjects, so the engine is unit-testable without DML.
- A `Bundle` concept modelled as a strategy/interface per plan (Easyplan, EasyplanPlus, EasyplanDeluxe), so adding a plan doesn't mean editing a `switch` — this directly exercises clean-apex's "prefer polymorphism" rule.
- A `BillingService.calculate(List<UsageRecord>, Bundle)` returning a `Bill` wrapper (total in pence, plus a breakdown).
- Money in **integer pence**, never `Double` — rounding rules below depend on it.

## Rules (restated)

Rounding
- Call times round **up** to the next whole minute.
- The final bill rounds to the nearest whole penny.

Easyplan
- First 300 texts free; each text beyond 300 costs 10p.
- First 500 minutes bundled at £10; each minute over costs 10p.

EasyplanPlus
- £20/month, includes 1000 minutes and 600 texts.
- Beyond that: minutes 15p each, texts 12p each.

EasyplanDeluxe
- First 2000 minutes cost £30; subsequent minutes 23p each.
- No free texts; every text to an in-network mobile costs 8p.

Cross-cutting rules (apply across plans)
- Calls to **other** networks don't count toward the bundle; charged at the bundle's normal per-minute rate.
- Calls to **08xx** numbers don't count toward the bundle; charged at the normal rate **plus 7.5%**.
- Texts to other networks aren't bundled; cost the basic bundle text price **plus 6%**.
- Operator-service calls always cost 12p — **except** on Deluxe, where they cost 19p.
- Bundle rates apply only to **off-peak** calls (weekdays 6pm–11pm, weekends 7am–11pm). Peak calls are charged outside the bundle.

## Suggested test progression (write the assertion first each time)

1. Empty usage → bill equals the plan's base charge (£0 / £20 / £30).
2. One off-peak in-bundle call under the allowance → still just the base charge.
3. One call of 60s and one of 61s → 1 min and 2 min respectively (round-up boundary).
4. Exactly at the allowance (500 / 1000 / 2000 mins; 300 / 600 texts) → no overage.
5. One unit over each allowance → correct per-unit overage rate.
6. A call to another network → billed at normal rate, allowance untouched.
7. An 08xx call → normal rate + 7.5%, assert the exact pence.
8. A text to another network → base text price + 6%.
9. An operator call on each plan → 12p, and 19p on Deluxe.
10. A peak-time call → charged outside the bundle; a weekend 7am call → off-peak boundary.
11. Final-penny rounding → construct a mix whose raw total has fractional pence.

Each step should fail first, pass minimally, then refactor. By step 6–7 the per-plan `if` chain should be pushing you toward the `Bundle` strategy.

## What to watch for

- Boundaries are the whole exercise: test **both sides** of every threshold (allowance, peak window edges, the 60s round-up).
- Keep money in pence to avoid floating-point drift; only convert for display.
- When the third plan's rules force a second `switch`, that's the REFACTOR signal to introduce the strategy interface — don't pre-build it.
