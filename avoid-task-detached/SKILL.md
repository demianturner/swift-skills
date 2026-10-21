---
name: avoid-task-detached
description: Refactor Task.detached used for CPU work in Swift 6.2+ TCA effects. Use when preserving effect cancellation and actor isolation matters; do not apply to intentionally independent tasks.
---

# Avoid unnecessary `Task.detached` in TCA effects

When a reducer launches `Task.detached` merely to move expensive work off the main actor, move that work into an async dependency operation and await it from the `.run` effect. For CPU-intensive synchronous work, an explicitly `@concurrent` async function can run the processing outside the caller's actor isolation in Swift 6.2+.

Before changing code, check why the detached task exists, who owns its lifetime, and how the effect is cancelled. A genuinely independent task may need a separate lifetime; this skill does not require removing it.

## Preserve execution and cancellation

- Keep the reducer responsible for starting the effect and receiving the result as an action. Put processing behind a dependency so it remains replaceable in tests.
- Use `@concurrent` when CPU work must leave the caller's actor. Do not add it to async operations that already suspend efficiently, such as `URLSession` requests.
- Pass values across isolation boundaries in a way Swift's sendability rules permit. Keep main-actor state out of the processing function.
- Preserve effect cancellation and any existing request identity or stale-result checks. CPU loops may need cooperative cancellation checks; awaiting a function does not interrupt synchronous work automatically.
- Use an actor when processing must protect shared mutable state. Actor isolation alone does not guarantee execution on a particular thread.

`@concurrent` selects concurrent execution outside actor isolation; it does not reserve a background thread. If the operation performs substantial blocking I/O, assess an appropriate I/O strategy separately instead of assuming `@concurrent` solves thread occupancy.

Check the affected effect's success, failure, and cancellation behavior, and confirm expensive processing no longer runs on the main actor. When compiler or language settings differ from Swift 6.2, verify the supported isolation spelling before applying this pattern.
