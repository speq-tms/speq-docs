# Contributing to speq-docs

## What belongs here

- Product direction, OSS/Pro boundary, release scope.
- Architecture of the ecosystem and of `speq-cli`.
- Feature specifications for the CLI.
- Delivery process: release flow, publishing, documentation site.

What does **not** belong here: runtime implementation notes that duplicate code comments, per-repository READMEs,
and anything that becomes stale the moment code changes without a way to verify it.

## Structure

```
docs/
  cli/            # CLI reference + features/ specs — primary source of truth
  product/        # vision, OSS/Pro boundary, release scope
  architecture/   # target architecture, refactoring plans
  delivery/       # release flow, publishing playbook, gh-pages
archive/          # superseded documents, historical context only
```

New documents go into one of the four `docs/` sections. Do not add numeric prefixes to filenames — ordering lives
in [README.md](README.md), not in the file names. Use lowercase kebab-case.

## Language policy

- **English**: `README.md`, `CONTRIBUTING.md`, issue and PR templates, anything published to the public site.
- **Russian**: internal specifications under `docs/`.

Keep a single language per document. Do not mix.

## Accuracy rules

- A specification must state whether the behavior it describes is **implemented** or **planned**.
- When a spec and the code disagree, either fix the code or fix the spec — do not leave the contradiction.
- When the DSL changes, synchronize `speq-cli`, `speq-contracts`, and `speq-vscode-extension`, and update the
  affected spec in the same release candidate.

## Branching and commits

See [docs/delivery/release-flow.md](docs/delivery/release-flow.md). Documentation-only changes use `docs:`.

## Pull request checklist

- [ ] The repository map in `README.md` is updated if a document was added, moved, or removed.
- [ ] The document states implementation status where it describes CLI behavior.
- [ ] Terminology is consistent (`speq`, `.speq`, mode names, command and flag spellings).
- [ ] Language policy is respected.
- [ ] Cross-links to related documents are present and resolve.

## Tracking

Work is tracked with GitHub Issues in this repository.

- `type/*` — bug, feature, docs, chore.
- `area/*` — cli, contracts, examples, runner, docs, release.
- `status/*` — blocked, needs-decision.
- `priority/*` — p0, p1, p2.

Milestones follow CLI release versions.
