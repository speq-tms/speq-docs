# speq-docs

Canonical product, architecture, and delivery documentation for SPEQ. Focus is `speq-cli`.
This repository is also the **issue tracker of record for the whole ecosystem**.

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

## How we work

Full process: `docs/delivery/release-flow.md`. Read it before starting delivery work. Summary:

- **Issues live here**, in `speq-tms/speq-docs`, for every repository in the ecosystem. The target repository
  is given by the `area/*` label and stated in the issue body.
- **Milestone title == RC branch name.** Milestone `v1.1.0` means branch `v1.1.0` in each affected repository.
  `backlog` is not a release and has no branch.
- **Find the current RC** — GitHub state is authoritative, not any checked-in file:

  ```bash
  gh api repos/speq-tms/speq-docs/milestones \
    --jq '.[] | select(.state=="open" and .title != "backlog") | .title'
  git ls-remote --heads origin 'v*'
  ```

- **Branch from the RC, never from `main`:** `git switch -c docs/<scope>-<name> origin/<RC>`.
- **PR base is the RC**, never `main`. One final PR takes the RC into `main`.
- **Closing keywords never fire in this flow.** `Closes #10` works only when a PR merges into the
  repository's default branch, and every PR here targets the RC. Write `Part of speq-tms/speq-docs#10`
  and close the issue manually after merge — in every repository, this one included.
- Tick the checkbox in the epic issue when a child issue lands.

## Invariants
- Documentation must match implemented behavior; verify against `speq-cli` source before asserting behavior.
- No numeric filename prefixes; navigation lives in `README.md`.
- Language policy: English for public-facing files, Russian for internal specs under `docs/`.
- Update the `README.md` repository map whenever a document is added, moved, or removed.
