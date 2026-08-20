# speq-docs

Canonical product, architecture, and delivery documentation for **SPEQ** — declarative YAML-based API testing.

The focus of this repository is `speq-cli`, the single execution engine of the ecosystem. Interface layers
(`speq-github-runner`, `speq-vscode-extension`) are documented only where they constrain the CLI.

## Repository map

| Path | Contains |
| --- | --- |
| [docs/cli/](docs/cli/) | CLI reference and per-feature specifications — the primary source of truth |
| [docs/product/](docs/product/) | Vision, OSS/Pro boundary, release scope |
| [docs/architecture/](docs/architecture/) | Target architecture and CLI refactoring plan |
| [docs/delivery/](docs/delivery/) | Release flow, publishing playbook, GitHub Pages |
| [docs/delivery/rc/](docs/delivery/rc/) | Generated release-candidate reports — scope, changes, rollout order |
| [archive/](archive/) | Superseded documents, kept for historical context only |

## Start here

1. [docs/cli/project-layout.md](docs/cli/project-layout.md) — what a `.speq` project is, and the manifest reference.
2. [docs/cli/README.md](docs/cli/README.md) — command surface, exit codes, feature index.
3. [docs/product/vision.md](docs/product/vision.md) — what SPEQ is and who it is for.
4. [docs/architecture/target-architecture.md](docs/architecture/target-architecture.md) — how the repositories fit together.
5. [docs/delivery/release-flow.md](docs/delivery/release-flow.md) — how changes reach `main` and a release tag.

## CLI reference

| Document | Contains |
| --- | --- |
| [project-layout.md](docs/cli/project-layout.md) | `.speq` project layout, root discovery, manifest reference |
| [README.md](docs/cli/README.md) | Commands, flags, exit codes |

## CLI feature specifications

| Feature | Spec |
| --- | --- |
| Phase 2 core expansion (umbrella) | [phase2-core-expansion.md](docs/cli/phase2-core-expansion.md) |
| Data generation | [features/data-generation.md](docs/cli/features/data-generation.md) |
| Fixtures and builders | [features/fixtures-and-builders.md](docs/cli/features/fixtures-and-builders.md) |
| Retry and waiter policy | [features/retry-waiter-policy.md](docs/cli/features/retry-waiter-policy.md) |
| Module outputs | [features/module-outputs.md](docs/cli/features/module-outputs.md) |
| Parallel execution | [features/parallel-execution.md](docs/cli/features/parallel-execution.md) |
| ATDD flow (`status: pending`) | [features/atdd-flow.md](docs/cli/features/atdd-flow.md) |
| API coverage against OpenAPI | [features/coverage.md](docs/cli/features/coverage.md) |

## Ecosystem repositories

| Repository | Role |
| --- | --- |
| [speq-cli](https://github.com/speq-tms/speq-cli) | Core runtime and command execution (Rust) |
| [speq-contracts](https://github.com/speq-tms/speq-contracts) | Shared schemas and compatibility contracts |
| [speq-examples](https://github.com/speq-tms/speq-examples) | Canonical examples, fixtures, acceptance references |
| [speq-github-runner](https://github.com/speq-tms/speq-github-runner) | GitHub Action wrapper around the CLI |
| [speq-vscode-extension](https://github.com/speq-tms/speq-vscode-extension) | VS Code integration and diagnostics |
| [homebrew-tap](https://github.com/speq-tms/homebrew-tap) | Homebrew formula distribution |
| [speq-tms.github.io](https://github.com/speq-tms/speq-tms.github.io) | Public site and published documentation |

## Conventions

- Documentation must describe **implemented behavior**. Planned behavior is marked explicitly as such.
- Public-facing text (this README, quickstart, site content) is written in English; internal specifications are
  written in Russian. See [CONTRIBUTING.md](CONTRIBUTING.md).
- Work is tracked with GitHub Issues, labels, and milestones in this repository.
