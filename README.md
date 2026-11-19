# ci

Shared GitHub Actions code for my repos: reusable workflows and composite actions. A fix made
here reaches every caller through a normal Dependabot bump.

Plan, decisions and roadmap live in the "Shared CI" project log.

## Using it

Pin to a full commit SHA with the release tag as a comment. Dependabot reads the comment and
proposes bumps when a new tag is released.

Reusable workflows live in `.github/workflows/`:

```yaml
permissions:
  contents: read

jobs:
  lint:
    uses: drod3763/ci/.github/workflows/<name>.yml@<sha> # v1.0.0
```

Composite actions live at the repo root, one folder per action:

```yaml
steps:
  - uses: drod3763/ci/<name>@<sha> # v1.0.0
```

Add a Dependabot config for the `github-actions` ecosystem to each caller so the pins stay
current.

## Rules

Every caller runs this code with its own secrets, so everything here follows these rules:

- Third-party actions are pinned to a full commit SHA with a `# vX.Y.Z` comment.
- Every workflow has a least-privilege `permissions:` block (`contents: read` by default).
- No `secrets: inherit`. Callers pass each secret explicitly.
- Inputs are bound through `env:` and never expanded as `${{ inputs.* }}` inside `run:`.
- Reusable workflows can't see this repo's files. Anything that needs bundled scripts is a
  composite action that reads them through `${{ github.action_path }}`.
- Releases are semver tags (`v1.0.0`).
- The repo stays public: public repos can't call workflows in private repos.

This repo's own CI checks workflows (actionlint, zizmor), shell scripts (shellcheck, shfmt),
YAML formatting (yamlfmt), spelling (typos), Markdown (markdownlint-cli2 with
`@drod3763/markdownlint-config`) and that PR titles are Conventional Commits. To run the same
checks locally, `mise install` once, then `mise run lint`. `mise run fmt` fixes formatting.

## Planned contents

- JS/TS workflows (commitlint, format-check, lint, markdownlint, publish, test, typecheck),
  moving here from `drod3763/configs`.
- `brew-bump`: the Homebrew updater, once a second tap needs it.
- Rust and Go CI, added with the first project in each language.
