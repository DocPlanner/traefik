---
name: syncing-traefik-fork
description: Use when syncing the DocPlanner traefik fork to a newer upstream traefik release (e.g. v3.7.5 → v3.7.6, or moving to a new minor like v3.8) — rebasing the k8s-provider performance patches and the fork-image CI commit onto the new upstream base and republishing docplanner/production-k8s-provider-perf so the GHCR image rebuilds.
---

# Syncing the DocPlanner traefik fork to upstream

## Overview

This fork carries a small **stack of patches on top of an upstream traefik release**:
a set of Kubernetes-provider **performance patches** plus **one CI commit**
(`.github/workflows/fork-image.yaml`) that builds the multi-arch GHCR image. The
production branch `docplanner/production-k8s-provider-perf` is simply:

```
<upstream release tag>  +  N perf patches  +  fork-image CI commit (must stay on top)
```

Syncing = move that whole stack onto a newer upstream release via `git rebase --onto`,
**re-verify every patch is still logically correct** (not just that it applies cleanly),
run the production build checks, then force-republish the branch (which retriggers the image build).

**Core principle:** a patch applying cleanly does NOT mean it is still correct. Upstream may
have changed the same files or the assumptions a patch relies on. You MUST review each patch
against the new code.

## Repository layout (know this first)

- Remotes: `upstream` = `traefik/traefik`, `origin` = `DocPlanner/traefik`.
- `origin/v3.7` — our tracking ref for upstream's current v3.x maintenance branch. The patch stack sits on top of whatever commit this points to. **The branch name always equals the minor of the release you're tracking**: syncing to `v3.7.6` → branch `v3.7`; the first time you sync to a new minor like `v3.8.0` you create/track branch `v3.8`. Everywhere below that says `v3.7`, substitute the minor of `$NEW_BASE`.
- `origin/docplanner/production-k8s-provider-perf` — **the deliverable**: the rebased stack. Pushing here triggers `fork-image.yaml`.
- `origin/pr/k8s-*` — the individual perf patches as standalone branches (one per patch), kept so they can be re-proposed/inspected.
- The `fork-image` CI commit must remain the **top** commit of the stack, must never be sent in an upstream PR, and must **never** end up on the `v3.x` tracking branch. Confirm it is absent there before the fast-forward push: `git ls-tree origin/v3.7 .github/workflows/fork-image.yaml` must print nothing.

Throughout, `<ver>` is the dotted release without the `v` prefix, e.g. `3.7.6` (used only for worktree/branch naming).

Inspect the live stack any time with:
```bash
git log --oneline origin/v3.7..origin/docplanner/production-k8s-provider-perf
```

## Tools required

`go` (version per `.go-version`/`go.mod`), `docker`, `golangci-lint`, `shellcheck`, `misspell`, `gh`.
If running inside a sandboxed agent, build/test/lint/push commands need to bypass the sandbox
(they write to `GOCACHE`/`.git` and reach non-allowlisted hosts) — run them with the sandbox disabled.

## Procedure

### 1. Fetch upstream and pick the target

```bash
git fetch upstream --tags
git tag -l 'v3.*' | sort -V | tail            # find the new release, e.g. v3.7.6
```

**Pin to the release TAG (`v3.7.6`), not the moving `upstream/v3.7` branch tip.** The branch tip
often carries post-release doc commits; a tag gives a deterministic, versioned base for the image.

### 2. Determine the current base of the stack

Refresh origin first so the refs you read are current (a teammate may have moved them):
```bash
git fetch origin
```
The stack sits on whatever `origin/v3.7` currently points to. Derive the base, don't assume a fixed commit:
```bash
NEW_BASE=v3.7.6                                # the new release tag
OLD_BASE=$(git rev-parse origin/v3.7)
git log --oneline "$OLD_BASE"..origin/docplanner/production-k8s-provider-perf   # = the exact stack to replay
```
Sanity-check that range: it must be **exactly** the N perf patches + the `fork-image` CI commit on top and
nothing else. If it shows extra/missing commits, `origin/v3.7` isn't the real base — use the parent of the
oldest patch instead: `OLD_BASE=<oldest-patch-sha>^`.

Confirm `OLD_BASE` is an ancestor of the new tag:
```bash
git merge-base --is-ancestor "$OLD_BASE" "$NEW_BASE" && echo OK || echo "NOT ANCESTOR — stop"
```
If it prints `NOT ANCESTOR` (e.g. a minor-version jump where upstream rebuilt the branch), **stop and
reassess**: the simple `--onto` replay assumes a clean lineage. You can still rebase onto the new tag, but
expect conflicts in every patch and review each with extra care.

### 3. Rebase the stack in an isolated worktree

```bash
git worktree add .worktrees/sync-<ver> -b sync/<ver> origin/docplanner/production-k8s-provider-perf
cd .worktrees/sync-<ver>
git -c commit.gpgsign=false rebase --onto "$NEW_BASE" "$OLD_BASE" HEAD
```
- `-c commit.gpgsign=false` is **required** — the perf commits aren't yours to sign and signing will abort the rebase with `gpg: No secret key`.
- Resolve any conflicts patch-by-patch. Conflicts are most likely in `pkg/provider/kubernetes/**`.

Verify the result: `git log --oneline -N+1` shows the N perf patches + the CI commit on top of the new tag.

### 4. Verify each patch is still logically correct (the important part)

First, get the true footprint of the stack — it spans **more than ingress**. The patches touch the
`crd`, `gateway`, `knative`, `ingress`, `ingress-nginx`, and shared `k8s` provider packages (plus, for the
config-hashing patch, a dependency *outside* the provider tree). Derive the exact paths instead of assuming:
```bash
git diff --name-only "$OLD_BASE" origin/docplanner/production-k8s-provider-perf -- pkg/ cmd/   # files the stack touches
```
Then list **what upstream changed in those areas** — scope the diff to the whole provider tree, and remember
patches can depend on code elsewhere (e.g. the config watcher):
```bash
git log --oneline "$OLD_BASE".."$NEW_BASE" -- pkg/provider/kubernetes/
git log --oneline "$OLD_BASE".."$NEW_BASE" -- pkg/server/configurationwatcher.go   # config-hash patch dep
```
`gateway`, `knative`, and `ingress-nginx` are the **fastest-moving** upstream areas — scrutinise upstream
changes there hardest (new informers, new fields on resolved endpoints, new resource kinds). For each
upstream commit, check whether it invalidates a patch's assumption. Known hot spots:
- **Cache/dedup patches — there are several** ("deduplicate backend service loads" in `ingress-nginx/build.go`, "deduplicate status updates per rebuild" in `ingress-nginx/kubernetes.go`, "cache parsed router annotations" in `ingress/`): for each, confirm the cache **key still captures every input** to the cached result. If upstream made the result depend on a new field, a stale key = a silent correctness bug even though it compiles. (Real example: upstream added endpointslice **fencing**; the backend-load cache stayed correct only because the `Fenced` value lives *inside* the cached `endpoint` struct, keyed by the same service/port tuple.)
- **"remove redundant config hashing"**: confirm the config watcher still dedupes downstream (`grep reflect.DeepEqual pkg/server/configurationwatcher.go`). If it stopped, this patch would cause config churn.
- **`client.go` patches** (informer index, `StripManagedFields` transform): confirm they still wire into **every** informer factory in **every** provider — `crd`, `gateway`, `ingress`, `ingress-nginx`, **and `knative`** — including any factory upstream added. Grep each provider's `client.go` for `WithTransform(k8s.StripManagedFields)` and `AddIndexers` and confirm coverage matches the new set of factories.
- **event-handler patches** (condition-change / node-update filtering): confirm they're consistent with how upstream now consumes those fields.

Write a one-line verdict per patch, covering **all** of them (don't skip a patch just because it rebased clean). Build + tests catch structural breakage; **this step catches the semantic breakage they don't.**

### 5. Production build checks

Run from the worktree (sandbox disabled). Capture long output to a file and grep it rather than re-running.
```bash
go build ./...
make generate      ; git status --porcelain    # expect NO output (clean tree)
make generate-crd  ; git status --porcelain     # expect NO output (generated code clean)
make test-unit
make validate-files                              # vendor + misspell + shellcheck
make lint
make binary-linux-amd64                          # what the production image build runs; produces dist/linux/amd64/traefik
```
After `make generate`/`generate-crd`, `git status --porcelain` must print **nothing**. If it lists files,
the generated output drifted: inspect the diff — if it's a real regeneration (CRD/types changed), commit it
into the appropriate patch or as a fork commit; if it's the macOS sed artifact below, revert it.

Known environment caveats (NOT failures of the sync — don't let them block you, but confirm them):
- **`make generate-crd`** fails on macOS at the final doc step with `sed: -I or -i may not be used with stdin` (BSD vs GNU sed). The *generated code* is still produced and clean; it passes on Linux CI. It may leave a stray `---` atop the CRD doc — revert just that file: `git checkout -- docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml`.
- **`make test-unit`** can flake on `TestInMemoryRateLimit` (`pkg/middlewares/ratelimiter`) — a wall-clock-timing test, sensitive to CPU contention (e.g. lint running concurrently). It's an untouched package. Re-run it in isolation (`go test -count=1 ./pkg/middlewares/ratelimiter/...`) to confirm it's environmental.
- **`make lint`** may report issues only in **upstream files the patches don't touch** if your local `golangci-lint` is newer than the version pinned by `golangci-lint-action` in `.github/workflows/validate.yaml`. Prove it's a version artifact: `golangci-lint run ./pkg/provider/kubernetes/...` and confirm none of the findings are on lines the patches added. Don't "fix" upstream files.

### 6. Sync the v3.x branch and publish

`v3.7` (or the current minor) fast-forwards to the new tag; the production branch is a force update
(history was rewritten by the rebase). **Pushing the production branch triggers the GHCR image build — get explicit approval before pushing.**
```bash
# Re-read the remote prod tip RIGHT BEFORE pushing so the lease reflects reality.
git fetch origin
LEASE=$(git rev-parse origin/docplanner/production-k8s-provider-perf)

git push origin "$NEW_BASE":refs/heads/v3.7                              # fast-forward
git push origin sync/<ver>:refs/heads/docplanner/production-k8s-provider-perf \
    --force-with-lease=refs/heads/docplanner/production-k8s-provider-perf:"$LEASE"
```
`--force-with-lease` aborts if someone moved the remote prod branch since you read `$LEASE` — never use a
plain `git push --force` here.

### 7. Confirm the image build

The run may not register instantly after the push; grab the latest run id for the workflow, then watch it:
```bash
RUN_ID=$(gh run list --repo DocPlanner/traefik --workflow=fork-image.yaml --limit 1 --json databaseId -q '.[0].databaseId')
gh run watch "$RUN_ID" --repo DocPlanner/traefik --exit-status
```
A non-zero exit from `gh run watch --exit-status` means the build failed — investigate before declaring done.
Image lands at `ghcr.io/docplanner/traefik:perf-latest` and `:<short-sha>`.

### 8. Clean up

```bash
cd <repo-root> && git worktree remove .worktrees/sync-<ver>
```

## Quick reference

| Step | Command |
|------|---------|
| Fetch | `git fetch upstream --tags && git fetch origin` |
| Stack to replay | `git log --oneline origin/v3.7..origin/docplanner/production-k8s-provider-perf` |
| Stack footprint | `git diff --name-only origin/v3.7 origin/docplanner/production-k8s-provider-perf -- pkg/ cmd/` |
| Rebase | `git -c commit.gpgsign=false rebase --onto v3.7.x origin/v3.7 HEAD` |
| Upstream changes to review | `git log --oneline <old>..<new> -- pkg/provider/kubernetes/ pkg/server/configurationwatcher.go` |
| Prod build | `make binary-linux-amd64` |
| Publish prod | `git push origin sync/<ver>:refs/heads/docplanner/production-k8s-provider-perf --force-with-lease=...:"$LEASE"` |
| Watch image | `gh run watch "$RUN_ID" --repo DocPlanner/traefik --exit-status` |

## Common mistakes

| Mistake | Reality |
|---------|---------|
| Rebasing onto `upstream/v3.7` branch tip | Use the release **tag** for a deterministic versioned image. |
| Reading `origin/v3.7` or the prod tip without re-fetching | `git fetch origin` first — stale refs give the wrong base / a bad `--force-with-lease`. |
| "It rebased cleanly, so it's correct" | Clean apply ≠ correct. Do step 4 — review against upstream's changes to the same files. |
| Reviewing only ingress in step 4 | The stack also touches `gateway`, `knative`, `ingress-nginx`, `crd`, shared `k8s`, and `configurationwatcher.go`. Use the footprint diff; scrutinise the fast-moving providers hardest. |
| Letting `fork-image.yaml` reach the `v3.x` branch | It belongs only on the prod branch. Confirm `git ls-tree origin/v3.7 .github/workflows/fork-image.yaml` is empty before the FF push. |
| Letting GPG abort the rebase | Use `-c commit.gpgsign=false`. |
| Treating the ratelimiter flake / macOS sed / lint-version noise as a real failure | They're environmental — confirm per step 5, don't chase them. |
| "Fixing" lint findings in upstream files | If a finding isn't on a line the patches added, it's pre-existing upstream / a linter-version artifact. Leave it. |
| Dropping or reordering the `fork-image` CI commit | It must stay the **top** commit of the stack. |
| Plain `git push --force` | Use `--force-with-lease` to avoid clobbering unexpected remote state. |
| Pushing the production branch without asking | It triggers a real GHCR image build — confirm first. |
