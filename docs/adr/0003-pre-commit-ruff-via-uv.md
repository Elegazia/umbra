# ADR 0003: Run ruff in pre-commit through uv-managed local hooks

- **Status:** Accepted
- **Date:** 2026-11-15
- **Deciders:** Project owner
- **Related:** none

## Context

Umbra uses [ruff](https://docs.astral.sh/ruff/) as its linter and formatter, and [uv](https://docs.astral.sh/uv/) to manage the Python environment. Ruff's version is pinned in `uv.lock`, and continuous integration installs exactly that lock (`uv sync --locked`) before running `ruff check`, `ruff format --check` and `pytest`.

[pre-commit](https://pre-commit.com/) runs the same checks earlier, at `git commit` time on the developer's machine, so problems are fixed before they reach a pull request. pre-commit can obtain ruff in two ways, and they differ in where ruff's version is defined:

- from the published hook repository `astral-sh/ruff-pre-commit`, pinned by a `rev:` tag in `.pre-commit-config.yaml`, in an environment pre-commit builds and caches itself; or
- from the project environment, by running `uv run ruff …` as a local hook.

The requirement is that a file which passes the commit hook also passes CI, and vice versa. Ruff's formatter output and lint rules change between releases, so two different ruff versions can disagree on the same file.

## Decision drivers

Hard constraints:

1. **D1 (hard): Parity.** The hook, CI and the developer's terminal must produce identical lint and format results for the same file.

Preferences, ranked:

2. **D2: Single source of truth for tool versions.** Each tool version is declared in one place (`uv.lock`), so upgrades cannot leave copies behind.
3. **D3: Low contributor friction.** A new clone should need few setup steps.
4. **D4: Hook speed.** Commits should not be noticeably slowed.
5. **D5: Compatibility with hosted hook runners** (for example pre-commit.ci, which runs hooks on pull requests and pushes fixes back).

## Options considered

### Option A: Published hook repository with a pinned `rev`

```yaml
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.17.4
  hooks:
    - id: ruff-check
      args: [--fix]
    - id: ruff-format
```

pre-commit clones the hook repository at that tag and installs ruff into an isolated, cached environment under `~/.cache/pre-commit`.

| Driver | Assessment |
|---|---|
| D1 | Holds only while `rev` matches the version in `uv.lock`. `pre-commit autoupdate` bumps `rev` but not `uv.lock`, and a dependency update does the reverse, so parity depends on discipline. |
| D2 | Two declarations of ruff's version. |
| D3 | Works on any machine with pre-commit, without uv. |
| D4 | Fast: a ruff-only environment, built once and cached. |
| D5 | The standard setup for pre-commit.ci. |

- **Benefits:** the conventional setup; isolation from the project environment; hosted-runner support.
- **Costs:** version drift between the hook and CI, causing failures that no code change explains.

### Option B: Local hooks that run ruff through uv

```yaml
- repo: local
  hooks:
    - id: ruff-check
      name: ruff check
      entry: uv run ruff check --fix
      language: system
      types_or: [python, pyi]
      require_serial: true
    - id: ruff-format
      name: ruff format
      entry: uv run ruff format
      language: system
      types_or: [python, pyi]
      require_serial: true
```

`language: system` tells pre-commit to run `entry` as-is, without building an environment. `uv run` executes ruff from the project environment, syncing it from `uv.lock` first if needed. `require_serial` stops pre-commit from starting several `uv run` processes in parallel.

| Driver | Assessment |
|---|---|
| D1 | Holds by construction: the hook, CI and terminal all run the ruff version in `uv.lock`. |
| D2 | One declaration (`uv.lock`). |
| D3 | Requires uv on the machine. uv is already required to work on the project at all, so this adds no new step. |
| D4 | `uv run` checks the environment against the lock before each hook (milliseconds when it is in sync). |
| D5 | Hosted runners such as pre-commit.ci do not provide uv, and are reported to block network access at hook runtime *(secondary)*, so these hooks would need to be skipped there. |

- **Benefits:** guaranteed parity; one version to upgrade.
- **Costs:** depends on uv; not usable on hosted hook runners without skipping.

### Option C: Published hook repository plus an automated version sync

Option A, plus automation that keeps `rev` equal to the version in `uv.lock`: a hook that rewrites `rev` from the lock file, or a dependency bot configured to update both together.

| Driver | Assessment |
|---|---|
| D1 | Holds as long as the automation runs. |
| D2 | Two declarations, kept equal by tooling. |
| D3 | As A. |
| D4 | As A. |
| D5 | As A. |

- **Benefits:** the conventional setup with drift controlled.
- **Costs:** an extra moving part to configure and trust.

### Option D: No pre-commit; rely on editor integration and CI

The editor formats on save, and CI enforces the checks.

- **Fails D1 in practice:** editor plugins can bundle their own ruff version. Problems surface only after pushing, which costs a CI round trip.

## Decision

**Option B: run ruff through uv-managed local hooks.** File-hygiene checks that uv does not provide (trailing whitespace, end-of-file newlines, YAML and TOML syntax, merge-conflict markers, large files, private keys) use the published `pre-commit/pre-commit-hooks` repository, pinned to a release tag.

**Rationale (against the drivers):** B is the only option that satisfies D1 by construction rather than by discipline or automation, and it is the only one that meets D2. Its costs fall on D3 and D5, and neither applies today: uv is already a prerequisite for the project, and no hosted hook runner is in use. Option C would recover D3 and D5, at the price of an extra component whose only job is to restore the guarantee B already gives.

## Consequences

- **Easier:** upgrading ruff is one change (`uv lock --upgrade-package ruff`), which updates the hook, CI and the terminal together. A file that passes the hook passes CI's lint and format steps.
- **Harder:** contributors must install uv. The README's setup section lists it, together with `uv run pre-commit install`, which every clone needs because git hooks are not versioned.
- **Risks and mitigations:**
  - *Hooks are bypassable* (`git commit --no-verify`) and exist only on clones where they were installed. CI remains the enforcing check.
  - *A hook modifies files and aborts the commit:* re-stage the fixed files and commit again.
- **Follow-up work:** none.

### Revisit triggers

- The project accepts outside contributors who do not use uv → reconsider Option C.
- A hosted hook runner (such as pre-commit.ci) is adopted → reconsider Option C, or skip the two ruff hooks there with the runner's `skip` setting.
- `uv run` overhead makes commits noticeably slow (more than about a second per hook on a typical commit) → reconsider Option A or C.

## Background

### Git hooks

Git runs scripts from `.git/hooks/` at fixed points: `pre-commit` before a commit is created, `commit-msg` after the message is written, `pre-push` before a push. A non-zero exit from `pre-commit` aborts the commit; nothing is written to history and the staging area is unchanged. Because `.git/` is not part of the repository, hooks must be installed in each clone.

### The pre-commit framework

pre-commit manages the `pre-commit` hook script from a declarative `.pre-commit-config.yaml`. For each hook it builds and caches an environment (or uses the system for `language: system`) and passes the **staged** file names as arguments. Before running hooks it temporarily stashes unstaged changes, so hooks see exactly what will be committed. A hook that modifies files counts as a failure, so the developer reviews and re-stages the change.

### Lock files and reproducibility

A lock file records the exact version and hash of every dependency, including indirect ones. Installing from it reproduces the same environment on every machine. `uv sync --locked` fails if the lock no longer matches `pyproject.toml`, which catches a forgotten lock update in CI instead of silently resolving new versions. Every additional place that declares a version is a copy that can fall out of sync with the lock.

### Shift-left checking

Defects are cheaper to fix the earlier they are found: in the editor, then at commit, then in CI, then in review. pre-commit moves lint and format checks to commit time. CI still runs them, because local checks can be skipped and can differ between machines.

## Sources

| Source | Type | Accessed |
|---|---|---|
| [pre-commit documentation](https://pre-commit.com/) | Primary | 2026-11-15 |
| [astral-sh/ruff-pre-commit](https://github.com/astral-sh/ruff-pre-commit) | Primary | 2026-11-15 |
| [uv documentation: locking and syncing](https://docs.astral.sh/uv/concepts/projects/sync/) | Primary | 2026-11-15 |
| [pre-commit.ci](https://pre-commit.ci/) | Primary | 2026-11-15 |
| [Adam Johnson: pre-commit hooks for unsupported tools](https://adamj.eu/tech/2023/02/09/pre-commit-hooks-unsupported-tools/) | Secondary | 2026-11-15 |
| [check-jsonschema issue 21 (pre-commit.ci network restrictions)](https://github.com/python-jsonschema/check-jsonschema/issues/21) | Secondary | 2026-11-15 |
