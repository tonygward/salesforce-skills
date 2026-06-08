# Exercise: Order Shipping Calculator (Apex TDD)

**Adapted from** the "Shipping costs" kata. Reframed as a Salesforce order-fulfilment exercise to practise test-driving a rules engine with interacting constraints and availability logic.

## The exercise

You have the raw data for a customer's order. Test-drive an Apex class that returns the **minimum** valid shipping cost, given the rules below. Build it one rule at a time, RED→GREEN→REFACTOR.

Two things make this harder than the billing exercise, and that's the point: (1) some delivery options are *unavailable* in some situations, so "minimum" means "minimum of the valid options", and (2) item attributes (size, weight, category) change which options exist.

## Suggested Apex domain mapping

- An inner `OrderItem` class (price in pence, weight in grams, max dimension in mm, category enum: standard/largeItem/heavyItem/videoGame/luxury, and a flag for exceptional).
- A `Destination` (mainland UK / off-mainland UK / outside UK) — model the regions explicitly; the rules hinge on them.
- A `ShippingService.cheapestValidCost(List<OrderItem>, Destination)` returning pence, or throwing a typed `ShippingUnavailableException` when no option is valid (exercises clean-apex-error-handling: don't return null/-1).
- Build the set of *available* options per item, price each, then take the minimum — rather than a giant nested conditional.

## Rules (restated)

Coverage
- Deliver within the UK only — includes Northern Ireland, Channel Islands, Isle of Man, Scottish islands. Anything else → not deliverable.

Base postage
- Regular: £4.99 per item; **free** for orders totalling over £25 **where there are no exceptional items**.
- First class on the mainland: +£2.99 per item.
- Next-day within the UK: +£11.99 per item — **mainland only**.

Exceptional items (these override the free-postage rule)
- Large item (any dimension over 30cm): +£19.90 per item; **first class is the only option**.
- Heavy item (over 15kg): +£39.90 per item; **first class only, mainland only**.
- First class **off** the mainland: +£9.00 per item.
- Video games: next-day +£3.90 per item + £1.50 per kg — **mainland only**.
- Luxury watches/jewellery: always shipped as **separate individual items**, **mainland only**, **next-day only**, +£36.99 per item.

"Minimum" means: across the delivery options actually available for that order/item, pick the cheapest.

## Suggested test progression (assertion first each time)

1. Single standard item, mainland, order under £25 → £4.99.
2. Order over £25, no exceptional items → free postage (£0).
3. Order over £25 **with** an exceptional item → free rule does **not** apply.
4. First-class mainland on a standard item → £4.99 + £2.99.
5. Off-mainland address → next-day option absent; assert it's not selectable.
6. Outside the UK → `ShippingUnavailableException`.
7. Large item → only first class offered; +£19.90; assert regular is not an option.
8. Heavy item off the mainland → unavailable (mainland-only); assert the exception or exclusion.
9. Video game next-day, 2kg → +£3.90 + £3.00.
10. Luxury item ×2 → two separate shipments, each next-day +£36.99; off-mainland luxury → unavailable.
11. Mixed basket (standard + large + heavy) → minimum valid combination, not a flat sum.

Fail first, pass minimally, refactor. By steps 5–8 the availability logic should be pushing you to separate "which options exist" from "what each costs".

## What to watch for

- The real difficulty is **availability**, not arithmetic: model "valid options" as a set you filter, then minimise — don't bury availability inside the cost arithmetic.
- Boundaries everywhere: exactly £25 vs just over; exactly 30cm vs just over; exactly 15kg vs just over. Test **both sides**.
- "No exceptional items" gates the free-postage rule — an easy case to miss; test it explicitly (step 3).
- When no option is valid, throw a typed exception with context (clean-apex-error-handling) rather than returning a sentinel.
- Luxury "always separate items" interacts with per-item pricing — assert the shipment count, not just the total.
