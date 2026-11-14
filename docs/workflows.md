# Workflows

Every consumer repository runs exactly two workflows, both named after the event
and imported from `gardusi-gustavo/package-yml`.

| File | Trigger |
| --- | --- |
| `.github/workflows/event-pull-request.yml` | `pull_request: branches: [main]` |
| `.github/workflows/event-release.yml` | `release: types: [published]` |
| `.github/workflows/agent.yml` | `issue_comment`: `/agent code`, `/agent adjust` |

There is no `ci.yml`, no `release.yml`, and no combined `pipeline.yml`.

## Why flat files, not folders

GitHub does not traverse subdirectories under `.github/workflows`, and a reusable
workflow can only be referenced as `.github/workflows/<file>.yml`. A path like
`event/pull-request.yml` never runs.

The folder structure lives in the **source**:

```text
package-yml/
  templates/event/
    pull-request.yml
    release.yml
```

and is flattened in consumers by the `event-` prefix.

## The callers

**`event-pull-request.yml`**

```yaml
name: event-pull-request
on:
  pull_request:
    types: [opened, ready_for_review, synchronize, reopened]
    branches: [main]
permissions:
  contents: read
jobs:
  ci:
    if: github.event.pull_request.draft == false
    uses: gardusi-gustavo/package-yml/.github/workflows/event-pull-request.yml@release/v1
```

The caller job stays `ci` on purpose: the required check context is `ci / ci`,
and renaming the job would break branch protection.

**`event-release.yml`**

```yaml
name: event-release
on:
  release:
    types: [published]
permissions:
  contents: write
jobs:
  release:
    uses: gardusi-gustavo/package-yml/.github/workflows/event-release.yml@release/v1
```

## Repository surface

```text
.github/workflows/event-pull-request.yml
.github/workflows/event-release.yml
scripts/test.sh                 optional
scripts/release.sh              optional
infra/docker/ci.dockerfile      optional, opt-in via ci.container
infra/config/repo.json          language, release type, commands
```

No `schedule`, `issue_comment`, `repository_dispatch`, or `push: branches`
trigger belongs in a consumer.

## Experimental

Everything else lives in `package-yml` and is inert:

```text
.github/workflows/experimental-update.yml
.github/workflows/experimental-backup.yml
.github/workflows/experimental-ingest.yml
.github/workflows/experimental-pages.yml
```

All are `workflow_call`; none fires on its own. Manual dispatch is the test, and
a schedule is added only after it passes. One issue tracks propagation.

## Reusable logic

| Caller event | Reusable workflow |
| --- | --- |
| pull request | `event-pull-request.yml` |
| release | `event-release.yml` |

Logic changes in those files reach every repository without a pull request,
because callers reference `release/v1`.

## Language setup

`event-pull-request.yml` infers the language from the repository name
(`package-<lang>`) or `infra/config/repo.json` `.ci.language`, then installs the
toolchain: Python, Node, Go, Rust, Java/Kotlin/Scala, .NET, Ruby, PHP, Elixir,
Lua, Swift, Zig, or Terraform. C++ and shell need nothing. `scripts/setup.sh`
overrides.

## Resolution order

1. `infra/docker/ci.dockerfile`, when `.ci.container` is `true`;
2. `scripts/test.sh`;
3. `.ci.command` or `.ci.script`;
4. fail with a clear message.

`event-release.yml` uses `scripts/release.sh` when present; otherwise it archives the
tag and publishes a GitHub release from `dist/*`.

## Ref strategy

| Ref | Consumer PRs on logic change | Use |
| --- | --- | --- |
| `release/v1` branch | none | current |
| `v1.x.y` tag | one wave | freeze |
| commit SHA | one wave | maximum assurance |

## Drift check

```bash
python3 scripts/sync_workflows.py --check
python3 scripts/sync_workflows.py --apply --yes
```

`--check` compares the two event files in every repository with
`templates/event/`, and reports leftover `pipeline.yml`, `pr.yml`, or
`release.yml`. Expected steady state: every repository reports `ok`.

`--apply` writes the callers, removes the legacy files, pushes `workflow/callers`
over SSH, and opens a pull request. It is idempotent.

## Rules

- Never edit the two event files by hand; change the template or run the sync.
- Never copy reusable logic into a repository.
- Experimental and scheduled workflows stay in `package-yml`.
- The builder may not modify `.github/workflows/**`.


## Comment commands

The agent is manual by design: only an explicit command invokes the model.

| Comment | Result |
| --- | --- |
| `/agent code` on an issue | create `agent/issue-<n>` and a draft PR |
| `/agent adjust <text>` on a PR | change the existing PR branch |

Any other comment is ignored. At most one draft exists per issue because the
branch is deterministic. Marking the draft ready for review triggers CI.
