---
name: m23-organise-tca-modules
description: Organise files within a Swift Package TCA feature module by responsibility. Use when placing or refactoring module files, without changing package boundaries merely to match a folder pattern.
metadata:
  short-description: Organise TCA files by responsibility
---

# Organise TCA modules

Within `Sources/<Module>/`, group files by their main responsibility:

| Folder | Contents |
| --- | --- |
| `Client/` | File system, network, and platform adapters that perform work for the feature. |
| `Domain/` | Reducers, actions, state transitions, and feature rules. |
| `Model/` | Small data types, errors, and values used by the feature. |
| `Resources/` | Localized strings, catalogs, and other bundled assets. |
| `View/` | SwiftUI screens, rows, controls, and presentation-only styles. |

Keep tests under `Tests/<Module>Tests/`. Create only the folders the module actually needs. When a type serves several features, place it in the appropriate shared package rather than duplicating it inside one feature module.

When moving existing files, preserve their behavior and check the package manifest and resource references for paths affected by the move. Verify the affected package builds and its relevant tests pass.
