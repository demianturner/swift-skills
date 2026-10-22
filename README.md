# Swift Agent Skills

Focused Codex skills for Swift and The Composable Architecture (TCA) refactors. Run these commands from the project where you want to use the skills. Each install selects only the named skill for Codex; the update command refreshes that project's copy.

## Model Swift domain state

[m23-model-swift-domain-state](m23-model-swift-domain-state/SKILL.md) helps replace related booleans and optionals with an enum when they describe mutually exclusive states of one operation. It keeps genuinely independent state separate and calls out transition, cancellation, and stale-response behavior to preserve during a refactor.

Install:

```sh
npx --yes skills add https://github.com/demianturner/swift-skills --skill m23-model-swift-domain-state --agent codex --yes
```

Update:

```sh
npx --yes skills update m23-model-swift-domain-state --project --yes
```

## Organise TCA modules

[m23-organise-tca-modules](m23-organise-tca-modules/SKILL.md) groups a feature module's files by responsibility: client, domain, model, resources, and view. It keeps tests in the module's test target and avoids adding empty folders or moving shared types into a feature merely to match a template.

Install:

```sh
npx --yes skills add https://github.com/demianturner/swift-skills --skill m23-organise-tca-modules --agent codex --yes
```

Update:

```sh
npx --yes skills update m23-organise-tca-modules --project --yes
```

## Avoid unnecessary detached tasks

[m23-avoid-task-detached](m23-avoid-task-detached/SKILL.md) guides refactors where a TCA effect uses `Task.detached` only to move CPU-intensive work off the main actor. It favors an async dependency operation with explicit isolation, while preserving cancellation and leaving intentionally independent tasks alone.

Install:

```sh
npx --yes skills add https://github.com/demianturner/swift-skills --skill m23-avoid-task-detached --agent codex --yes
```

Update:

```sh
npx --yes skills update m23-avoid-task-detached --project --yes
```
