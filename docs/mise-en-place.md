# Mise en place

Prepare every repository so workflows can be enabled later without rework. Nothing
scheduled runs yet.

## Per repository

| File | Status | Notes |
| --- | --- | --- |
| `.github/workflows/event-pull-request.yml` | done in 40 repos | pull-request CI |
| `.github/workflows/event-release.yml` | done in 40 repos | release event |
| `.github/workflows/agent.yml` | done in 39 repos | `/agent` comment |
| `.github/CONTRIBUTING.md` | done in 39 | from `package-md` |
| `.github/SECURITY.md` | done in 39 | from `package-md` |
| `.github/CODE_OF_CONDUCT.md` | done in 39 | from `package-md` |
| `scripts/test.sh` or `infra/config/repo.json` | per repo | CI entrypoint |
| `infra/docker/ci.dockerfile` | optional | opt-in via `ci.container` |

The split and the removal of every non-PR/tag workflow are staged on the
`workflow/callers` branch of each repository.

## Experimental workflows

They live in `package-yml` only:

```text
.github/workflows/experimental-update.yml   pull-based caller sync
.github/workflows/experimental-backup.yml
.github/workflows/experimental-ingest.yml
.github/workflows/experimental-pages.yml
```

All are `workflow_call`; none has a schedule or fires on its own.

The caller for a future self-update is `templates/experimental/update.yml`. It is
not distributed.

## Order

1. Two callers (`event-pull-request.yml`, `event-release.yml`) and community files (mostly done).
2. Test entrypoint (`scripts/test.sh` or `.ci.script`).
3. Validate one experimental workflow with a manual dispatch.
4. Only then add a `schedule:` to one class at a time.
5. Track the propagation in a single issue in `package-yml`.

## Rules

- Consumer repositories run only `event-pull-request.yml` and `event-release.yml`.
- No `schedule`, `issue_comment`, `repository_dispatch`, or `push: branches` in a consumer.
- The bot prepares a branch and a PR; a human reviews and merges.
- No PAT, no cross-repo token.
