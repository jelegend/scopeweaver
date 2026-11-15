# ScopeWeaver — downstream fork

This checkout is a **downstream fork** of [`cskwork/scopeweaver`](https://github.com/cskwork/scopeweaver).
It carries local changes on top of upstream releases and pulls in new upstream
versions as they are tagged.

## Remotes

| remote | URL | role |
|---|---|---|
| `upstream` | `cskwork/scopeweaver` | read-only; source of new releases |
| `fork` | `jelegend/scopeweaver` | our copy; `main` carries the local patches |

`main` here is upstream `main` plus local commits — never rebased, so upstream
merges stay fast-forward-friendly.

## Local commits on top of upstream

Currently 3 commits on top of `v0.9.1`:

1. `feat(dashboard): add findings-over-time chart with combined series`
   — new whitelisted `FindingTrend` aggregate + chart, incl. a Combined series.
2. `fix(dashboard): bucket legacy token usage by run time and de-saturate cache read`
   — Legacy LLM-usage daily buckets now group by `activity.created_at` instead of
   task creation date, and cache-read is de-saturated via the per-call ledger.
3. `feat(i18n): localize the findings-trend chart in all six languages`

## Pulling a new upstream release

```bash
cd /Users/aayush.mani/Desktop/AI-exp/sw-src

# 1. Get the new upstream code and tags
git fetch upstream --tags

# 2. See what landed
git log --oneline fork/main..upstream/main      # new upstream commits
git tag --sort=-v:refname | head -5             # new release tags

# 3. Merge upstream into our main (keeps local commits on top)
git checkout main
git merge upstream/main

# 4. Resolve conflicts if any (see below), then verify
go build ./...
(cd web && npx tsc --noEmit && npm test && npm run build:static)

# 5. Publish
git push fork main
```

### If a merge conflicts

Conflicts are most likely in the files this fork touches:

- `db/exploration.go` — `TokenDailyAll` (the ledger join)
- `db/findings.go` — `FindingTrend`
- `server/server.go` — `findingTrend` handler + route
- `web/src/app/(main)/dashboard/page.tsx` — legacy/new token computation
- `web/src/app/(main)/dashboard/finding-trend-chart.tsx` — chart component
- `web/src/i18n/messages/*.ts` — `m2889`–`m2893`
- `web/src/i18n/i18n.test.mjs` — catalog key count

For the i18n key count: if upstream adds keys, the expected number in
`i18n.test.mjs` must be updated to the new total (our 5 keys are already
included in the current 3291).

## Rebuilding and deploying

```bash
cd /Users/aayush.mani/Desktop/AI-exp/sw-src

# Frontend static export -> embedded UI
(cd web && npm run build:static)
rsync -a --delete web/out/ server/webui/dist/

# Linux binary with the UI embedded
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -tags embedui -trimpath \
  -ldflags "-s -w -X main.version=vX.Y.Z-fork" -o /tmp/scopeweaver ./cmd/artex
```

Then install to the host and restart the **user** unit (not a system unit):

```bash
sudo install -o t3svc -g t3svc -m 755 /tmp/scopeweaver /home/t3svc/sw/app/scopeweaver
sudo -u t3svc env XDG_RUNTIME_DIR=/run/user/$(id -u t3svc) \
  systemctl --user restart scopeweaver.service
```

Pause long-running tasks before swapping the binary and resume afterwards, so no
in-flight write is lost.

## Notes

- The `dev` branch is unused here; upstream promotes `dev` -> `main` before
  release, so tracking `main` tracks releases.
- Upstream CI (`checks`, `e2e`) only runs on upstream PRs. This fork is verified
  locally with `go build ./...`, `npx tsc --noEmit`, `npm test` and
  `npm run build:static`.