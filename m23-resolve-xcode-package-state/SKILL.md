---
name: m23-resolve-xcode-package-state
description: Diagnose Xcode package resolution, missing products, and suspected stale SwiftPM or build state using incremental checks and the smallest affected build. Use after package dependency changes or package-related Xcode failures, including CodeSign errors and tool output that contradicts the manifest.
metadata:
  short-description: Resolve Xcode packages with minimal disruption
---

# Resolve Xcode package state

Find the concrete failure before choosing a repair. Continue routine diagnosis and reversible fixes autonomously within the user's authorization. Do not turn a package warning into a request for Reset Package Caches, Clean Build Folder, and a full app build.

## Operating rules

- Diagnose the specific error first; continue useful work instead of handing a generic “inconsistent package state” message to the user.
- Rule out stale tooling state before changing the package graph: duplicate Xcode windows on the same workspace, and stale build products left by an interrupted build.
- Resolve dependencies when declarations change or the diagnosed failure requires it. Wait for resolution to finish before building.
- Build the smallest affected package or target using a supported destination.
- Stop escalating when the relevant check passes.
- Avoid automatic package resets, build cleans, cache deletion, and full app builds. Require concrete evidence before considering a more disruptive action.

## Choose the next action from the evidence

Read the first actionable error and its surrounding log, inspect the changed manifest or project references, and confirm the active scheme and destination. Keep a brief record of the failure, action, and result in the conversation; do not create troubleshooting documents by default.

| Evidence | Next action |
| --- | --- |
| Source-only compiler warning or error | Fix the source and build the affected package. Package resolution is unnecessary unless dependencies changed. |
| `ignoring duplicate product` | Check for repeated `.product(...)` entries within the same target and duplicate project references. Remove actual duplicates; retain direct dependencies required by imports. A product used by both a library target and its test target is not itself a duplicate. |
| Missing module or package product after extraction | Check the importing file, target's direct product dependency, package path, product spelling, and app target linkage. Repair the declaration before resolving. |
| Dependency requires a newer platform | Check the scheme's destination and declared platforms. Select a supported destination. Do not change deployment targets or dependency versions merely to make an unsuitable destination build. |
| `Operation not permitted` or cache access denied | Use the permission recovery below. This is not evidence of cache corruption. |
| A dependency declaration was added, removed, or changed | Resolve once, wait for completion, then build the smallest affected target. |
| `CodeSign` fails naming a `.xctest`, `.app`, or framework under DerivedData, and the bundle is missing `Info.plist` | A stale build product from an interrupted build. Remove that one product, not Derived Data. See Remove a single stale build product. |
| Tool output contradicts the manifest: phantom products, a package missing from the graph, or a stale scheme list | Check for duplicate Xcode windows on the same workspace before editing anything. See Rule out stale tooling state. |
| A package resolves as an external dependency although it exists in the workspace | Resolution drift. Resolve once and confirm the workspace reclaims it as a local package. See Reconcile workspace resolution drift. |

## Rule out stale tooling state

Two failures looked like broken package state but were stale clients. Both are cheap and reversible, so check them before deeper package work.

- **Duplicate Xcode windows.** Two Xcode windows registered against the same `.xcworkspace` left the MCP bridge reading one window's outdated package graph, so its output described a graph that no longer existed. Closing the extra window restored correct output; the package graph itself was fine. Before changing a manifest, check how many windows have the workspace open and reduce it to one.
- **Interrupted build product.** An empty test bundle left in DerivedData by an earlier interrupted build had no `Info.plist`. Its presence failed CodeSign for the whole app, which points the reader at signing settings rather than at the real cause. Removing that one bundle fixed the build.

When tool output disagrees with the manifest, decide which one is stale before acting on either. Report which stale client or product you found and what removed it; do not describe the package graph as repaired when the graph never changed.

## Resolve only when needed

Use the existing project, scheme, package checkout location, and normal Xcode build output. Discover names from the current workspace; do not invent schemes or introduce alternate checkout directories during troubleshooting.

For an app project that consumes the changed packages, the confirmed command is:

```sh
xcodebuild -resolvePackageDependencies \
  -project Example.xcodeproj \
  -scheme Example
```

For a workspace that owns the package graph, use the workspace form:

```sh
xcodebuild -resolvePackageDependencies \
  -workspace Example.xcworkspace \
  -scheme Example
```

Substitute the real project and scheme. Prefer an already working invocation. If a workspace invocation reports that the workspace is invalid, confirm its path and environment permissions before falling back to the existing consuming project. Do not rewrite workspace configuration based only on that error.

Wait for resolution to finish before starting a build. Serialize resolution operations, and avoid builds or another Xcode process resolving the same graph concurrently. Capture the exit status and relevant error text.

If resolution fails, classify the error before retrying. Retry only after an action that addresses it. The `-disablePackageRepositoryCache` flag did not repair the observed permission failure; do not make it an unconditional retry or describe it as a proven cache repair.

## Recover from permission failures

The confirmed failure involved denied writes to SwiftPM manifest diagnostics and the Clang module cache in the user's cache directories. Running the same resolution command with the execution environment's approved access succeeded. No cache deletion was required.

When that pattern occurs:

1. Identify the denied path and operation in the error. Distinguish sandbox restrictions from filesystem ownership, a read-only volume, and network or authentication failures.
2. If the execution sandbox blocks the normal cache writes, request the narrow execution permission required for the same resolver command, using the available approval mechanism. Explain the denied paths and why resolution needs access. Reuse existing authorization where applicable.
3. Run resolution again after access is granted, then wait for success before building. If approval is denied, report the exact command and restriction; finish any independent work.

If a cache deletion itself fails with a permission error, stop that deletion. Do not escalate to `sudo`, recursive permission changes, or a broader deletion. First reconsider whether deletion is necessary: use the normal resolution path with appropriate access when possible. Only request access for a specific deletion if corruption of that exact artifact has been established and the user has authorized its removal. No deletion workaround was validated by the observed cases.

## Remove a single stale build product

DerivedData holds the products of every build, including interrupted ones. A partial product can fail a later build even when the source is correct.

When the error names a path, treat that path as the evidence:

1. Read the exact path from the error: the specific `.xctest`, `.app`, or framework under DerivedData. For example, an `ExampleTests.xctest` bundle.
2. Confirm the product is incomplete, for example a bundle with no `Info.plist` or a directory with no executable. Distinguish this from a product the current build legitimately needs.
3. Remove only that path.
4. Rebuild the smallest affected target and confirm the original error is gone.

This stays inside the escalation boundary: one named product, not the Derived Data directory, not the module caches, and not `Package.resolved`.

## Reconcile workspace resolution drift

When a package resolves as an external dependency although it exists in the workspace, or drops out of the graph:

- Confirm the package is declared in the workspace and present on disk.
- Resolve once with the workspace's own scheme, and wait for completion.
- Re-inspect the resolved graph. In the observed case a single resolve reclaimed the package as an in-workspace local package, replacing a stale external entry, with no manifest edit needed.

Do not delete `Package.resolved` to force this. The resolve is sufficient and preserves the pinned versions.

## Verify the smallest affected scope, then stop

For changes confined to a local package, select its generated scheme and a supported destination in the normal workspace. Build that package; include its test targets when the changed code is in tests. Do not substitute an app build merely because the package scheme previously selected an unsuitable destination.

Run affected tests when behavior changed or a failing test needs verification. A source warning or duplicate dependency cleanup ordinarily needs successful compilation and inspection of the new log for the original warning. A stale build product or duplicate window is verified by the affected build succeeding with the original error absent, not by re-running resolution. A successful build does not mean tests executed.

Once resolution succeeds and the relevant build/check passes, stop. Do not follow success with a reset, clean, repeated resolution, or full app build. Broaden verification only for affected app wiring, a new failure, or another concrete unresolved concern.

## Escalation boundary

Do not delete `Package.resolved`, purge shared SwiftPM caches, remove `SourcePackages`, clear all Derived Data, or clean the build folder as a routine ladder of retries. Removing one build product that the error names by path, as described in Remove a single stale build product, is inside scope; clearing Derived Data is not. Those repairs were not demonstrated as necessary in the observed cases. Preserve existing package versions and build settings unless the actual diagnosis requires changing them.

If a failure remains unexplained after the relevant declaration, destination, resolution, and permission checks, report the exact remaining evidence. Request only the smallest human action needed to answer that uncertainty. Do not prescribe the reset/clean/full-build bundle. If an applicable instruction explicitly requires a reset, explain that instruction and the conflict with this workflow instead of silently treating a reset as technically justified.
