# Swift skills

Focused Codex skills for Swift and The Composable Architecture (TCA) refactors. Install only the skills you want; each folder contains its own `SKILL.md`.

## Model Swift domain state

[model-swift-domain-state](model-swift-domain-state/SKILL.md) helps replace related booleans and optionals with an enum when they describe mutually exclusive states of one operation. It keeps genuinely independent state separate and calls out transition, cancellation, and stale-response behavior to preserve during a refactor.

Install or update this skill:

```sh
skill_dir="${CODEX_HOME:-$HOME/.codex}/skills/model-swift-domain-state"
mkdir -p "$skill_dir"
curl -fsSL https://raw.githubusercontent.com/demianturner/swift-skills/main/model-swift-domain-state/SKILL.md -o "$skill_dir/SKILL.md"
```

## Organise TCA modules

[organise-tca-modules](organise-tca-modules/SKILL.md) groups a feature module's files by responsibility: client, domain, model, resources, and view. It keeps tests in the module's test target and avoids adding empty folders or moving shared types into a feature merely to match a template.

Install or update this skill:

```sh
skill_dir="${CODEX_HOME:-$HOME/.codex}/skills/organise-tca-modules"
mkdir -p "$skill_dir"
curl -fsSL https://raw.githubusercontent.com/demianturner/swift-skills/main/organise-tca-modules/SKILL.md -o "$skill_dir/SKILL.md"
```

## Avoid unnecessary detached tasks

[avoid-task-detached](avoid-task-detached/SKILL.md) guides refactors where a TCA effect uses `Task.detached` only to move CPU-intensive work off the main actor. It favors an async dependency operation with explicit isolation, while preserving cancellation and leaving intentionally independent tasks alone.

Install or update this skill:

```sh
skill_dir="${CODEX_HOME:-$HOME/.codex}/skills/avoid-task-detached"
mkdir -p "$skill_dir"
curl -fsSL https://raw.githubusercontent.com/demianturner/swift-skills/main/avoid-task-detached/SKILL.md -o "$skill_dir/SKILL.md"
```
