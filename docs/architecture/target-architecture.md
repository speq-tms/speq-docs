# speq target architecture

## High-level

```text
YAML tests + manifest + env
          |
          v
       speq-cli
   (parse/validate/run/report)
      /               \
     v                 v
speq-github-runner   speq-vscode-extension
```

## Target repositories

## 1) `speq-cli` (OSS)

Recommended structure:

```text
speq-cli/
  src/
    cli/
    parser/
    runner/
    assertions/
    manifest/
    env/
    reporting/
  schemas/
  examples/
  docs/
```

## 2) `speq-github-runner` (OSS)

```text
speq-github-runner/
  action.yml
  scripts/
  workflows-examples/
  docs/
```

## 3) `speq-vscode-extension` (OSS)

```text
speq-vscode-extension/
  src/
    extension.ts
    tree/
    preview/
    diagnostics/
  syntaxes/
  package.json
  docs/
```

## Shared contracts

- Manifest schema and versions.
- Test YAML schema and semantics.
- CLI JSON result model.
- Exit code contract.
- Allure output contract.

## Compatibility strategy

- Backward compatibility for minor versions.
- Explicit migration on major schema changes.
- CLI provides actionable migration messages.
