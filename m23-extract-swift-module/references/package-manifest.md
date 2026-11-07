# Package manifest shapes

Illustrative only. Match the existing manifest's tools version, platform floors, settings, and naming in the repository you are editing.

## Add a capability module next to existing ones

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "JournalDomain",
    platforms: [.macOS(.v15), .iOS(.v18)],
    products: [
        // .automatic is the right default for app-internal code.
        .library(name: "JournalDomain", targets: ["JournalDomain"]),
    ],
    dependencies: [
        // Keep third-party dependencies at the leaves.
    ],
    targets: [
        .target(
            name: "JournalDomain",
            dependencies: []
        ),
        .testTarget(
            name: "JournalDomainTests",
            dependencies: ["JournalDomain"]
        ),
    ]
)
```

Include the test target in the same change as the source target. A module without its tests moved is an unfinished extraction.

## Depend on another local package

```swift
.target(
    name: "NoteDetail",
    dependencies: [
        .product(name: "SharedJavaScript", package: "SharedJavaScript"),
        .product(name: "NoteDomain", package: "NoteDomain"),
    ]
)
```

Use `.product(name:package:)` when the dependency is an external product. Use `.target(name:)` for a target in the same package, `.byName(name:)` only when the name is unambiguous.

## Linking mode

```swift
.library(
    name: "PluginHost",
    type: .dynamic,
    targets: ["PluginHost"]
)
```

| Mode | When |
| --- | --- |
| `.automatic` (default) | Everything app-internal. Lets the build system choose per consumer. |
| `.static` | You need to guarantee one linked copy in a non-SwiftPM consumer, or you are publishing a static binary. |
| `.dynamic` | Runtime plugin loading, a prebuilt binary for other teams, or measured runtime sharing. Requires an ABI you will not break. |

Do not switch to `.dynamic` to chase build speed. Measure the project, then decide.

## Resources

```swift
.target(
    name: "NoteEditorWebView",
    resources: [.process("Resources")]
)
```

- Access with `Bundle.module`.
- `.process` flattens and optimises; `.copy` preserves directory structure. Use `.copy` when an asset must keep its folder layout.
- A resource bundle belongs to its target. Tests that build a different target will not find it, so avoid resource lookups in code paths the unit tests exercise through another target.
- Xcode shows the generated bundle as a separate target, so a resource-only target still adds build-graph cost. Weigh that against keeping the resource in its consumer.

## Access control across targets

| Modifier | Reach |
| --- | --- |
| `internal` | This target only. Start here. |
| `package` | Targets in the same package. Prefer over `public` for intra-package sharing. |
| `public` | Any importer. A compatibility promise. |

SwiftPM has no "internal to a product" access level, so `package` is the closest tool for several targets that share a library product.

## Verify in this order

Close the workspace in Xcode before editing the manifest. Editing a manifest while Xcode has the workspace open makes Xcode raise "modified by another application" and "open in another Xcode process" dialogs that wait for a click. Reopen the same workspace only after resolution finishes.

```sh
# Only after a manifest change, with Xcode closed.
xcodebuild -resolvePackageDependencies -project Example.xcodeproj -scheme Example

# Reopen the workspace on the developer's original scheme, then build the
# affected package scheme, not the app.
xcodebuild -scheme JournalDomain -destination 'platform=macOS,arch=arm64' build

# Run the moved tests.
xcodebuild -scheme JournalDomain -destination 'platform=macOS,arch=arm64' test
```

Substitute the real scheme and a supported destination discovered from the current workspace. A successful build does not mean the tests ran, and vice versa. Do not follow success with a cleanup, reset, or full app build. Do not leave the workspace closed when the task ends.

## Common manifest failures

| Symptom | Cause | Fix |
| --- | --- | --- |
| `no such module` after the move | Target lacks a direct product dependency, or the product spelling differs from `Package.swift`. | Add the dependency; do not rely on transitive imports. |
| `duplicate product` | The same `.product(...)` appears twice in one target, or two project references duplicate the package. | Remove the actual duplicate. A library and its test target both listing it is not a duplicate. |
| `public` initializer missing | Memberwise initializers are internal. | Declare the initializer consumers need, or expose a factory. |
| Cycle reported | Two targets depend on each other. | The boundary is wrong. Move the shared piece down a level. |
| Duplicate symbols at link time | Two static products each embed the same static dependency. | Keep a single owner of that dependency. |
