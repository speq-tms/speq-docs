---
name: CLI specification
about: Specify a new or changed CLI capability before it is implemented
title: "[spec] "
labels: ["type/feature", "area/cli"]
assignees: ""
---

## Capability

What should `speq-cli` be able to do?

## Motivation

Which real testing scenario does this unblock?

## Proposed DSL / CLI surface

```yaml
# manifest or test spec changes
```

```
# command and flag changes
```

## Exit code and reporting impact

Does this change exit codes, `summary.json`, or Allure output?

## Cross-repo impact

- [ ] `speq-cli` — runtime
- [ ] `speq-contracts` — schema
- [ ] `speq-examples` — acceptance example (required for new features)
- [ ] `speq-vscode-extension` — diagnostics
- [ ] `speq-github-runner`

## Acceptance criteria

- [ ] Behavior is implemented and covered by `cargo test`
- [ ] Acceptance example added to `speq-examples/test-repo-mode-jsonplaceholder`
- [ ] Spec written under `docs/cli/features/` and linked from `docs/cli/README.md`
