# Lab 2 Starter: Availability Calculator

A small reservation component. Given a room's bookings and the day's business hours,
`AvailabilityCalculator.freeSlots` computes when the room is free. It is the code you
work in for Lab 2.

It ships with a generated test suite that passes, and a property-based test harness
(jqwik) with one example property. Everything is green. Your job in Lab 2 is to decide
whether green actually means correct.

**Read `ARCHITECTURE.md` before the code.**

## Build and test

```
mvn test
```

`mvn test` runs both files, the ordinary example-based tests (`AvailabilityCalculatorTest`)
and the property-based tests (`AvailabilityProperties`). A code-coverage report is written
to `target/site/jacoco/index.html`.

## Continuous integration

This repository has CI configured in `.github/workflows/ci.yml`. GitHub disables workflows on a
fresh fork, so enable them once on your fork (the handout shows where). After that, every
push runs `mvn test`. You will watch the gate go red when your new property finds the bug, then
green once you fix it.

## Where things are

- Component: `src/main/java/edu/cmu/cs214/availability/`
- Example-based tests: `src/test/java/edu/cmu/cs214/availability/AvailabilityCalculatorTest.java`
- Property-based tests: `src/test/java/edu/cmu/cs214/availability/AvailabilityProperties.java`
- Setup: `SETUP.md`

See the Lab 2 handout on the course page for the three milestones you show a TA.

## Lab 2 write-up

### Milestone 3: auditing the generated suite

`AvailabilityCalculatorTest` held 100% instruction and 100% branch coverage of
`AvailabilityCalculator` and stayed green over a real bug: the scan loop emitted a
free gap only when the *next* booking triggered it, so the stretch from the last
booking's end to `dayEnd` was never reported. Three concrete weaknesses let that
through.

**1. Every scenario pins the last booking's end to `DAY_END` — controllability.**
Five of the six tests (`fullyBookedDayHasNoFreeSlots`,
`bookingUntilEndOfDayLeavesTheMorningFree`, `gapsBetweenBookingsAreReturned`,
`unsortedBookingsAreHandled`, `overlappingBookingsAreMerged`) build bookings whose
latest end is exactly `DAY_END = 1020`. The cursor therefore always finishes on
`dayEnd`, the trailing gap is always zero-length, and the missing code has nothing
to omit. The suite never drove the one input shape that exposes the bug: a last
booking that ends *before* closing time.

**2. The booking list is never empty and never fully clipped away — controllability.**
Every test passes at least one booking that survives clipping, so the scan loop always
executes at least once. The most extreme form of the bug — no bookings at all, where
the loop body never runs and the entire business day is dropped — is unreachable from
any of these inputs. This is exactly the case jqwik shrank to:
`Scenario[dayStart=0, dayEnd=1, bookings=[]]`. The same hole covers bookings that lie
entirely outside business hours and clip to nothing.

**3. The one test that does reach the bug can't see it — observability.**
`returnedSlotsNeverOverlapABooking` books `[600, 660)`, which ends at 660, well before
`DAY_END = 1020`. It is the only test that actually executes the buggy state: the
cursor stops at 660 and `[660, 1020)` is silently dropped. But its assertion is
`for (slot : free) assertFalse(slot.overlaps(booking))` — it iterates only over what
was *returned* and checks that nothing returned is busy. Omission is invisible to it:
returning fewer slots only makes it easier to pass, and returning nothing at all
passes vacuously. It ran the bug and its assertions could not observe the wrong
result. `fullyBookedDayHasNoFreeSlots` has the same shape, asserting only
`.isEmpty()`, which cannot tell "correctly empty" from "always empty".

**Why high coverage did not save it.** Coverage records which lines and branches
*executed*, not what the assertions could *see*, and it cannot record code that was
never written. Both of those matter here. The trailing-gap emission is a missing
statement, so there is no line for a coverage tool to mark red — the report was
green because everything that exists ran, not because everything that should exist
was there. And weakness 3 shows the other half: the buggy state was reached and
still went unreported, because the only assertion watching it was one-directional.
The provided property has the same one-directional shape, asserting
`free ⊆ ¬booked` (soundness) but never `free = ¬booked` (soundness and
completeness). Adding the completeness half is what caught the bug.

## Tools and models

Claude Code (Anthropic), model Claude Opus 5 (1M context). Editors: IntelliJ IDEA
and VS Code for the agent session.
