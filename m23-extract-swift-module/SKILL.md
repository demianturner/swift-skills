---
name: m23-extract-swift-module
description: Decide whether and how to extract Swift code into a separate module or Swift package, including cleanup of temporary @_exported and typealias bridges. Use for module boundary design, package splits, dependency direction, public API and linking choices, and reviews that propose new targets. Not for routine file moves inside one target.
metadata:
  short-description: Extract Swift modules with a proven boundary
---

# Extract a Swift module

A module boundary is a permanent tax: an import line, a public surface, a build-graph edge, a resource bundle, a test target, and a place where cross-module optimisation stops. Extract to remove a named cost, not to tidy a diagram.

## Decide whether to extract

Extract only when at least one is true:

- **Several consumers** need the code without depending on each other (macOS app plus iOS app, or two features).
- **Independent change cadence**: it changes for reasons unrelated to its neighbours, so its churn keeps invalidating them.
- **A real substitution point**: two implementations exist or are planned, behind a stable interface.
- **Enforced direction**: something must be prevented from importing the app, UI frameworks, or a concrete backend. Compilation enforces it; review does not.
- **Measured build cost**: a target is a demonstrated compile-time bottleneck, or tests rebuild far more than they need.
- **Publication**: the code must be reused outside this repository, with a version and support commitment.

Do not extract because a folder looks crowded, a type seems important, one app consumes it, or reuse is predicted but unproven. Unused reusability pays the tax forever. If the answer is no, improve layout inside the existing target and follow `m23-organise-tca-modules` for the `Client/`, `Domain/`, `Model/`, `Resources/`, `View/` split before proposing a new target.

## Test cohesion before naming the module

Write the purpose as one sentence with no "and". Two clauses mean two modules or none.

- Group by capability and change reason, not by type kind. `NoteDomain` is a boundary; `Models` and `Helpers` are filing cabinets.
- Things that change together stay together. If a format and its model always change in one commit, splitting them creates a two-step edit forever.
- Describe the module without naming its callers. "Used by the editor" is a location, not a responsibility.
- Prefer a narrow surface over a small file count. Ten types with three public entry points beats one target per type.

Stop when the boundary has one owner and no upward dependencies. Further splitting beyond that buys nothing.

## Check coupling in both directions

List every edge that will cross the new line before moving code.

- **Afferent** (importers of the new module): more than one is the usual justification.
- **Efferent** (what the new module must import): each is a permanent promise.

Then remove avoidable efferent edges:

- Replace a concrete client with a value type, a protocol owned by the module, or a closure the caller supplies.
- Keep file-system, network, and UI types out of domain modules. Domain code should compile for macOS and iOS without an app framework.
- Never put a third-party type in the public API. It forces the dependency on every consumer and blocks replacing it later.

If the new module needs most of its old neighbours, the boundary is in the wrong place. Move the boundary, not the neighbours.

## Fix dependency direction

Arrows point from volatile, composed code toward stable, reusable code:

```
App (composition root, concrete wiring)
  └── Features (reducers, views, feature clients)
        └── Domain and client protocols (pure values and rules)
              └── Platform-free foundations (no SwiftUI/AppKit/UIKit)
```

- Sibling features never depend on each other. Share by pushing the shared piece down a level, or let the app pass it in.
- Lower layers never reach up for configuration. Accept it as a parameter.
- Concrete implementations of a shared protocol sit with the consumers that choose between them; only the app selects them.
- Third-party code lives at the leaf behind a small adapter your code owns.
- SwiftPM rejects cyclic targets, so a cycle is a design signal rather than a manifest problem to work around.

## Design the public API deliberately

Everything `public` is a contract.

- Start `internal`. Promote exactly what consumers need, when they need it.
- Across targets in one package, prefer `package` access: it crosses targets without promising anything to external clients.
- `public` does not propagate the memberwise initializer. Declare the initializers consumers need and keep invariants in them.
- Expose values, small protocols, and one entry-point type — not mutable shared state, caches, or concrete backends.
- Keep the seam testable: injected `Clock`, `UUID` and clients instead of hidden globals or raw `UUID()`.
- State concurrency as part of the contract: `Sendable` conformances, the isolation of each entry point, and whether calls are reentrant or cancellable.
- Avoid `@_exported import` outside a documented umbrella module, and avoid `@frozen` unless you ship a stable binary ABI. See Remove migration scaffolding.
- Avoid single-implementation protocols exposed publicly. Keep the concrete type until a second implementation or a test seam exists.

Check: the public API should read like a short description of the module. If it reads like a dump of internals, shrink it.

## Remove migration scaffolding

An extraction compiles long before it is finished, because Swift lets old code keep working through shims. Each shim is unfinished work that looks like a design decision.

| Shim | Why it appears | Why it must go |
| --- | --- | --- |
| `@_exported import Module` | consumers keep their old import and still see the moved types | hides the real dependency, hands importers names they never imported, and makes cleanup a breaking change later |
| `typealias Old = Module.Type`, including `public typealias` | call sites keep the old spelling while the type moves | leaves two names for one type, so readers cannot tell who owns it and searches find both |
| `public extension` on another module's type that only forwards into the new module | keeps a helper at its old address | splits one responsibility across two modules and keeps the old edge alive |
| a target whose only content is re-exports or aliases | avoids touching imports during the move | adds a build-graph node and a public surface with no capability of its own |

`@_exported` is an underscored, undocumented attribute. Treat it as a migration tool, not a design tool: the compiler may change it, and it communicates an intent the module graph does not record.

Rules:

- Record the removal point when you add a shim, in the same change: which files carry it and what will replace it. Put it in the task list or the pull request description, not in a permanent document.
- Remove shims in the change that completes the move. "Later" is how they survive for years.
- Resolve each hit by editing the call sites: import the owning module, use the real type name, delete the alias or re-export.
- Test code counts. A `typealias` in a fixture that exists only to avoid renaming test call sites is scaffolding too.
- Keep a re-export only when re-exporting is the module's documented purpose, such as an umbrella or compatibility module named as such in the module map or README. Documentation and intent are the difference between a façade and a leftover.
- Never let a shim become the public API by default. If consumers end up importing the shim's module for a type it does not own, the boundary has failed.

Before calling the extraction complete, search for every shim and account for each hit:

```sh
rg -n --glob '*.swift' '^\s*@_exported import' .
rg -n --glob '*.swift' '^\s*(@?public |@?package )?typealias\s+\w+\s*=\s*\w+\.' .
```

The second pattern catches aliases qualified with a module name, which is the shape a move produces. A bare `typealias` in the same module is ordinary Swift, not scaffolding.

Then open the manifest for each target those results point at and confirm the target has a capability of its own. Do not dismiss a hit as "probably fine" without stating why it stays.

## Choose linking mode

Use `.automatic`, the SwiftPM default, for app-internal Swift code; it produces a static library for an app target. Choose `.dynamic` only for runtime plugin loading, a prebuilt binary shipped to other teams, or measured runtime sharing — never to chase build speed. Never let two static products each vendor a copy of the same dependency, which can produce duplicate symbols at link time. Full comparison in `references/linking-and-api.md`.

Targets with resources get their own bundle, read via `Bundle.module`, and that bundle is absent when tests build a different target.

## Verify testability and build cost

Extraction should make at least one of these measurably better:

- The module's tests build without linking the app or UI layer.
- A dependency can be faked at the seam, making untestable behaviour testable with Swift Testing.
- The changed code compiles a smaller graph: build only the affected package scheme, per `m23-resolve-xcode-package-state`.

Do not trade incremental build speed for a faster clean build. Boundaries enable parallel compilation, but each change invalidates every downstream module, so many small modules can lengthen the edit–build loop. Prefer a handful of capability modules over one per type.

## Avoid over-modularisation

Symptoms the split went too far:

- Modules with one or two types, or named `Common`, `Utils`, `Shared`, `Extensions`, `Core`.
- A protocol with one implementation and no test or runtime reason, in a module of its own.
- Every feature change needs a new `public` declaration elsewhere.
- One behaviour change touches five manifests, or a chain of re-exports.
- Import lists longer than the code they enable.

Repairs: merge sibling micro-modules into one capability module, delete pass-through modules that only re-export, replace a single-implementation protocol with the concrete type, move a shared type back to its real owner.

**Small types stay together.** Types under about 50 lines generally belong in the same file as their close relatives, not in standalone files, and a group of them almost never justifies its own module. Do not carry a one-type-per-file habit across the boundary. Split a file when the type has its own reason to change, not to satisfy a count.

## Extraction procedure

1. Write the boundary sentence, the consumers, and the in/out edges. If the sentence needs "and", re-scope.
2. Identify every call site, plus anything that reaches app-only types, file paths, or UI.
3. Name the module for its capability (`NoteDomain`, `ProjectPersistence`) and place it at the dependency level its imports allow: shared logic in a cross-platform package, presentation in a platform package.
4. Add the target, its dependencies, its test target, and any resources in one change. See `references/package-manifest.md` for shape and failure modes.
5. Move code without behaviour changes. Widen access only as the compiler demands. Keep small types with their relatives.
6. Move the tests and import the new module. Delete duplicated helpers instead of copying them.
7. Close the workspace in Xcode before editing the manifest. Move the files while Xcode is closed, resolve dependencies once and wait for resolution to finish, then reopen the same workspace on the scheme the user was using and build the smallest affected scheme. See "Close, resolve, reopen" below.
8. Report what moved, what became public and why, what remains duplicated, and which shims were added or removed. State any re-export or alias that survives and why it is permanent. Do not call the boundary validated without a successful build of the affected target.

## Close, resolve, reopen

Changing a package manifest while Xcode has the workspace open makes Xcode notice an external change. It then blocks the developer with modal dialogs, typically:

- The workspace "has been modified by another application": keep the Xcode version or use the version on disk.
- The workspace "appears to be open in another running Xcode process": open it anyway or cancel.

Both dialogs wait for a click. An unattended run stalls at that point, and whichever button a returning developer chooses decides whether the package graph matches the files on disk. Do the package work in this order instead:

1. Close the workspace in Xcode before writing any manifest. Use the IDE's own close command so unsaved editor changes are handled first; quit Xcode when no other project is open.
2. Confirm no Xcode process still has the workspace open before writing manifests. On macOS, `pgrep -x Xcode` and the workspace's window list are enough. Do not use `killall Xcode` while other projects are open.
3. Edit the manifest and move the files. Build or test only after resolution, so the workspace is not reopened mid-edit.
4. Resolve dependencies once with the scheme the developer will use and wait for the command to finish. Resolution normally runs on the command line, because the Xcode integration's tools do not expose it. Follow `m23-resolve-xcode-package-state` when the manifest changed or resolution reports an error.
5. Reopen the same workspace. Select the base app scheme the developer was using before the close, not a package scheme or a second window. If the previous scheme no longer exists, say which scheme you selected and why.
6. Build the smallest affected target and run the moved tests when behaviour changed. Report what was closed, resolved, reopened, and built.

When the user is away from the machine, perform steps 1–5 without waiting for confirmation. Tell the user what you closed and reopened in the final report. Do not leave the workspace closed when the task ends. If a close or reopen fails, report the exact command and the error rather than reopening a second window as a workaround.

If the Xcode integration is available through an MCP bridge, use its workspace close and open actions so the developer's own window state is restored. A command-line `open` on the workspace is the fallback. Do not open the `.xcodeproj` instead of the `.xcworkspace`, and do not open a second copy while the first is still running.

## References

- `references/package-manifest.md` — target declarations, resources, access control, verification commands, common manifest failures.
- `references/linking-and-api.md` — static versus dynamic detail, ABI notes, and public API review checklist.
