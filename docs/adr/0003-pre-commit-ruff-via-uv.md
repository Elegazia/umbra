# ADR 0003: Run ruff in pre-commit through uv-managed local hooks

- Status: Accepted
- Date: 2026-11-15
- Deciders: Project owner
- Related: none

## Context

Umbra uses [ruff](https://docs.astral.sh/ruff/) as its linter and formatter and [uv](https://docs.astral.sh/uv/) to manage the Python environment. Ruff's version is pinned in `uv.lock`, and continuous integration installs exactly that lock (`uv sync --locked`) before running `ruff check`, `ruff format --check` and `pytest`.

[pre-commit](https://pre-commit.com/) runs the same checks earlier, when `git commit` runs on the developer's machine, so problems are fixed before they reach a pull request. pre-commit can get ruff in two ways, which differ in where ruff's version is defined. It can install ruff from the published hook repository `astral-sh/ruff-pre-commit`, pinned by a `rev:` tag in `.pre-commit-config.yaml`, into an environment that pre-commit builds and caches itself. Or it can run ruff from the project environment, as `uv run ruff …` in a local hook.

A file that passes the commit hook must also pass CI, and the other way round. Ruff's formatter output and lint rules change between releases, so two different ruff versions can disagree about the same file.

## Decision drivers

Hard constraint:

- D1. Parity. The hook, CI and the developer's terminal give identical lint and format results for the same file.

Preferences, in order:

- D2. One source of truth for tool versions. Each version is declared in one place (`uv.lock`), so an upgrade cannot leave a stale copy behind.
- D3. Low contributor friction. A fresh clone needs few setup steps.
- D4. Hook speed. Commits are not noticeably slower.
- D5. Compatibility with hosted hook runners, such as pre-commit.ci, which runs hooks on pull requests and pushes fixes back.

## Considered options

### A. Published hook repository with a pinned `rev`

```yaml
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.17.4
  hooks:
    - id: ruff-check
      args: [--fix]
    - id: ruff-format
```

pre-commit clones the hook repository at that tag and installs ruff into an isolated environment cached under `~/.cache/pre-commit`.

- Good, because it works on any machine with pre-commit installed, without uv (D3).
- Good, because the cached ruff-only environment is fast (D4), and it is the standard setup for pre-commit.ci (D5).
- Bad, because ruff's version is declared twice (fails D2). Parity holds only while `rev` matches `uv.lock`: `pre-commit autoupdate` bumps `rev` but not the lock, and a dependency update does the reverse, so D1 depends on discipline. The resulting failures cannot be traced to any code change.

### B. Local hooks that run ruff through uv

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

`language: system` tells pre-commit to run `entry` as it is, without building an environment. `uv run` executes ruff from the project environment, syncing it from `uv.lock` first if needed. `require_serial` stops pre-commit from starting several `uv run` processes in parallel.

- Good, because the hook, CI and the terminal all run the ruff version in `uv.lock`, so parity holds by construction (D1), and the version is declared once (D2).
- Good, because uv is already required to work on the project, so it adds no setup step (D3).
- Bad, because `uv run` checks the environment against the lock before each hook, although this takes milliseconds when it is in sync (D4).
- Bad, because hosted runners such as pre-commit.ci do not provide uv and are reported to block network access while hooks run (secondary), so these hooks would have to be skipped there (D5).

### C. Published hook repository with automated version sync

Option A, plus automation that keeps `rev` equal to the version in `uv.lock`: either a hook that rewrites `rev` from the lock file, or a dependency bot configured to update both together.

- Good, because it keeps the conventional setup and its advantages for D3, D4 and D5.
- Bad, because parity holds only as long as the automation runs (D1), and the version is still declared twice, kept equal by tooling (D2).
- Bad, because it adds another component that has to be configured and trusted.

### D. No pre-commit; rely on editor integration and CI

The editor formats on save and CI enforces the checks.

- Bad, because editor plugins can bundle their own ruff version (fails D1), and problems only surface after pushing, which costs a CI round trip.

## Decision

Chosen: option B, running ruff through uv-managed local hooks. File-hygiene checks that uv does not cover (trailing whitespace, end-of-file newlines, YAML and TOML syntax, merge-conflict markers, large files and private keys) use the published `pre-commit/pre-commit-hooks` repository, pinned to a release tag.

B is the only option that satisfies D1 by construction, rather than by discipline or automation, and the only one that meets D2. Its costs fall on D3 and D5, and neither applies at present: uv is already a prerequisite for the project, and no hosted hook runner is in use. Option C would recover D3 and D5, but only by adding a component whose sole job is to restore the guarantee that B already gives.

## Consequences

Upgrading ruff is one change (`uv lock --upgrade-package ruff`), which updates the hook, CI and the terminal together. A file that passes the hook passes CI's lint and format steps.

Contributors must install uv. The README's setup section lists it, together with `uv run pre-commit install`, which every clone needs because git hooks are not versioned.

Hooks can be bypassed with `git commit --no-verify`, and they exist only in clones where they were installed. CI therefore remains the check that enforces the rules.

When a hook modifies files, the commit is aborted. The developer re-stages the fixed files and commits again.

No follow-up work is needed.

### Revisit when

- the project accepts outside contributors who do not use uv (reconsider option C);
- a hosted hook runner such as pre-commit.ci is adopted (reconsider option C, or skip the two ruff hooks there with the runner's `skip` setting);
- `uv run` overhead makes commits noticeably slow, more than about a second per hook on a typical commit (reconsider option A or C).

## Background

### Git hooks

Git runs scripts from `.git/hooks/` at fixed points: `pre-commit` before a commit is created, `commit-msg` after the message is written, and `pre-push` before a push. If `pre-commit` exits with a non-zero status, the commit is aborted; nothing is written to history and the staging area is unchanged. Because `.git/` is not part of the repository, hooks have to be installed in each clone.

### The pre-commit framework

pre-commit manages the `pre-commit` hook script from a declarative `.pre-commit-config.yaml`. For each hook it builds and caches an environment, or uses the system for `language: system`, and passes the staged file names as arguments. Before running the hooks it temporarily stashes unstaged changes, so the hooks see exactly what will be committed. A hook that modifies files counts as a failure, so that the developer reviews and re-stages the change.

### Lock files and reproducibility

A lock file records the exact version and hash of every dependency, including indirect ones. Installing from it reproduces the same environment on every machine. `uv sync --locked` fails if the lock no longer matches `pyproject.toml`, which catches a forgotten lock update in CI instead of silently resolving new versions. Every other place that declares a version is a copy that can drift away from the lock.

### Shift-left checking

The earlier a defect is found, the cheaper it is to fix: in the editor, then at commit time, then in CI, then in review. pre-commit moves lint and format checks to commit time. CI still runs them, because local checks can be skipped and can differ between machines.

## Sources

| Source | Type | Accessed |
|---|---|---|
| [pre-commit documentation](https://pre-commit.com/) | Primary | 2026-11-15 |
| [astral-sh/ruff-pre-commit](https://github.com/astral-sh/ruff-pre-commit) | Primary | 2026-11-15 |
| [uv documentation: locking and syncing](https://docs.astral.sh/uv/concepts/projects/sync/) | Primary | 2026-11-15 |
| [pre-commit.ci](https://pre-commit.ci/) | Primary | 2026-11-15 |
| [Adam Johnson: pre-commit hooks for unsupported tools](https://adamj.eu/tech/2023/02/09/pre-commit-hooks-unsupported-tools/) | Secondary | 2026-11-15 |
| [check-jsonschema issue 21 (pre-commit.ci network restrictions)](https://github.com/python-jsonschema/check-jsonschema/issues/21) | Secondary | 2026-11-15 |
