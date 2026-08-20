# speq-docs

Canonical product, architecture, and delivery documentation for SPEQ. Focus is `speq-cli`.

## Responsibilities
- Keep CLI specs aligned with implemented behavior in `speq-cli`.
- Capture release decisions and delivery process.
- Mark planned-but-unimplemented behavior explicitly.

## Structure
- `docs/cli/` — CLI reference (`docs/cli/README.md`) and `docs/cli/features/` specs. Primary source of truth.
- `docs/product/` — vision, OSS/Pro boundary, release scope.
- `docs/architecture/` — target architecture, CLI refactoring.
- `docs/delivery/` — release flow, publishing playbook, gh-pages.
- `archive/` — superseded documents. Never cite as current behavior.

## Key references
- `docs/cli/README.md`
- `docs/cli/phase2-core-expansion.md`
- `docs/cli/features/*.md`
- `docs/delivery/release-flow.md`

## Invariants
- Documentation must match implemented behavior; verify against `speq-cli` source before asserting behavior.
- No numeric filename prefixes; navigation lives in `README.md`.
- Language policy: English for public-facing files, Russian for internal specs under `docs/`.
- Update the `README.md` repository map whenever a document is added, moved, or removed.
