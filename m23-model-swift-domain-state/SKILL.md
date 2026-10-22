---
name: m23-model-swift-domain-state
description: Refactor related Swift booleans and optionals into domain enums when their combinations represent mutually exclusive operation states. Use for state modelling and reducer transitions, not routine boolean cleanup.
metadata:
  short-description: Model related Swift state with enums
---

# Model Swift domain state

Use an enum when several stored fields jointly describe one operation and permit combinations the domain cannot handle. Name the valid phases and attach data that exists only in a particular phase to that case.

## Decide the boundary

- Trace every read, write, and transition before changing the representation. Identify valid combinations, including states reached during cancellation and late responses.
- Combine fields only when their meanings exclude one another. Loading, saving, monitoring, and file availability may overlap and need separate models. Independent settings may remain booleans; presence or absence alone may remain an optional.
- Keep data outside the phase enum when it must survive transitions, such as cached rows shown during refresh or after failure.

For example, `isLoading` plus an optional error may become `.idle`, `.loading`, and `.failed(Error)` if loading and failure are exclusive. An active save request plus a `saveAgain` flag may become `.saving(Request)` and `.savingWithFollowUp(Request)` when a follow-up requires an active save.

## Refactor the transitions

- Make transitions assign one case directly. Avoid writable compatibility properties that reconstruct the old combinations.
- Derive read-only convenience values for presentation when useful, such as `isLoading` or a display error.
- Preserve request identity, cancellation, shared references, and guards against stale responses. An enum does not provide those safeguards by itself.
- Keep case names and associated values tied to domain meaning. Do not introduce an enum merely to reduce the number of booleans.

## Check behavior

Test the transitions affected by the change, especially retry after failure, edits during saving, newer external changes, and closure with work pending when those cases exist. Confirm that previously valid overlapping states still work and that invalid combinations can no longer be constructed.
