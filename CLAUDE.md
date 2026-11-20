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

## Trust model for this repo's own CI

`ci.yml` runs on `pull_request`, so everything it executes comes from the PR: the workflow file,
`mise.toml` (tool versions and the `lint` task) and the code being checked. A PR can change any
of them, including turning a check into a no-op. That is true of every `pull_request` workflow,
and moving a check out of `ci.yml` neither adds nor removes that ability.

So the required `ci` check catches mistakes; it is not a security boundary. The boundary is:

- Review: main only changes through a PR that the owner reviews and merges. Review the whole
  diff, including `mise.toml` and `package.json`, not only `.github/`.
- No privilege for PR code: `contents: read`, no secrets, `persist-credentials: false`, and never
  `pull_request_target` or reading "trusted" config from the base branch, which would run PR
  code with elevated privileges.
- Pinned tools and actions, so an upstream compromise can't change what runs.
- Locally, read before you run. mise trusts this directory by path, not by content, so on
  someone else's branch `mise install` and `mise run lint` execute whatever that branch defines,
  as you, with your credentials, including your `gh` token. mise reads config and tasks from
  several files (`mise.toml`, `.mise.toml`, `mise.local.toml`, `.mise/`, `.config/mise/` and
  more), so review the whole diff, not a list of files. A fresh clone is not isolation: it still
  runs as you and can read your home directory and credential stores. For an unreviewed branch,
  rely on CI or use a container, VM or Codespace without your credentials.
- Run mise with `paranoid` on (`mise settings set paranoid true`). mise then trusts each config
  file by its contents, so after a checkout that edits or adds mise config it refuses to load
  that config until you review it and run `mise trust` again. This includes hooks that run
  when you `cd` into the directory with mise activated. Without it, a trusted path loads
  whatever the branch puts there, with no prompt. Paranoid mode doesn't check paths listed in
  `trusted_config_paths` or `MISE_TRUSTED_CONFIG_PATHS`, so keep this checkout and its parent
  directories out of both.

Checks stay defined once, in `mise.toml`, so local runs and CI run the same thing. Don't
duplicate the list in `ci.yml`.

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
