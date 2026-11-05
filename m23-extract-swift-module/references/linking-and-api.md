# Linking and public API reference

## Static versus dynamic Swift libraries

SwiftPM `.library(type:)` accepts `.automatic`, `.static`, or `.dynamic`. `.automatic` is the default and resolves to a static library inside an app target, so it is the right choice for app-internal code.

| Concern | Static | Dynamic |
| --- | --- | --- |
| Distribution size | Dead-stripped per app; unused symbols are dropped at link time | One copy on disk and in memory, but each dylib must be shipped and signed |
| Launch | No dyld work for the module | dyld must load, bind and relocate each image |
| Cross-module optimisation | Stops at the module boundary; whole-module optimisation applies within the module | Stops at the dylib boundary as well |
| Runtime identity | Separate linked copies in different consumers unless a shared dependency unifies them | Single runtime copy, relevant when type identity or singleton state must be shared |
| Update without relinking | Not possible | Possible, but requires a stable ABI you will not break; Swift's ABI for library evolution is opt-in and constrains types |
| Duplicate dependencies | Risk of duplicate symbols when two static products vendor the same library | One dynamic image can be shared, but version skew becomes a runtime failure |
| Test substitution | The fake is chosen at compile/link time by the composition root | The real dylib may be found before your fake; injection gets harder |

Rules that follow:

- Default to `.automatic`. Say why if you set anything else.
- Only add `BUILD_LIBRARY_FOR_DISTRIBUTION` and `@frozen` when a binary must survive a client rebuilt against an older version. Frozen layout is a promise; in-source app code pays for it and gains nothing.
- Keep one owner of a shared static dependency. If two products each embed it, expect duplicate symbols at link time.
- Plugins and bundles are the real dynamic-linking use case: a host loads code it did not compile. Design the plugin interface as a small, versioned surface with explicit failure when incompatible.

## Module-level build cost

- Each target is a compilation unit boundary. Changing a file inside a module recompiles that module, then relinks or re-emits every importer's interface.
- More modules mean more parallelism on a clean build and more edges on an incremental build. Both effects are real, and neither is visible from a file count.
- Swift's implicit module dependency discovery means module count alone does not lower build time; what helps is a smaller changed graph and fewer type-checking-heavy files in it.
- Resources, generated interfaces, and doc archives add per-target overhead. A target holding one small type is usually a net build loss.

## Public API review checklist

- Every `public` declaration is used by at least one consumer today.
- No third-party type appears in a public signature.
- No mutable shared state or cache is reachable through the public surface.
- Initializers enforce invariants; no public memberwise escape hatch unless intentional.
- `Sendable` and actor isolation are explicit for anything crossing a task boundary.
- Errors are typed and documented, not a generic `Error`.
- Nothing in the surface exposes file-system paths, bundle internals, or UI types unless that is the module's purpose.
- Names come from the domain, not from the implementation ("note storage", not "file helper").
- Deprecations, not deletions, for anything already published.
