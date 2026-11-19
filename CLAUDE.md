# CLAUDE.md

Shared reusable workflows (`.github/workflows/`) and composite actions (one folder per action at
the repo root) called by my other repos. Callers run this code with their own secrets, so treat
every change as security-sensitive.

Plan and decision log: the "Shared CI" artifact,
https://claude.ai/code/artifact/65605ad2-92dc-4804-bdb4-234c8d689b90. Read it before starting
work and update it when a decision or the roadmap changes.

## Rules for workflows and actions

- Pin every third-party action to a full commit SHA with a `# vX.Y.Z` comment. Pick releases
  that are at least two weeks old. Pin tool versions too (no `latest`).
- Every workflow sets a least-privilege `permissions:` block (`contents: read` by default).
  A caller must grant the widest permissions any called job needs; called jobs can only narrow.
- Never use `secrets: inherit`. Declare each secret under `on.workflow_call.secrets`.
- Bind inputs through `env:`. Never put `${{ inputs.* }}` or other untrusted expressions inside
  `run:`.
- `actions/checkout` uses `persist-credentials: false` unless a later step needs to push.
- `concurrency` belongs in the caller, not the reusable workflow.
- Reusable workflows can't read files in this repo. Bundled scripts go in a composite action and
  are reached through `${{ github.action_path }}`.
- The repo must stay public.

## Validating changes

- `mise run lint` on every change. It runs `actionlint` and `zizmor` at the versions pinned in
  `mise.toml` (installed per-repo by `mise install`). CI runs the same checks on every PR.
- `mise.toml` is the only place tool versions are set; CI installs them with `jdx/mise-action`.
  Dependabot doesn't update it, so bump versions by hand.

## Git

- Commit and push only when asked.
- Conventional Commits (`feat`, `fix`, `chore`, `docs`).
- Push over SSH: `git push origin <branch>` with `origin` set to `git@github.com:drod3763/ci.git`.
  The gh token lacks `workflow` scope, so HTTPS pushes that touch workflows fail.
- main is protected: open a PR, wait for the `ci` check, squash merge.
- Releases are semver tags (`v1.0.0`) so callers' Dependabot can propose bumps.
