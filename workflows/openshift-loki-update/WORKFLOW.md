# OpenShift Loki Image Update Workflow

Prepare a Loki image update for OpenShift by syncing upstream Grafana releases into the fork, waiting for the openshift/loki branch, then pushing openshift/release CI changes to a branch.

This workflow is resumable. On each run, determine the current phase and continue from there.

## Agent constraints

- **Push branches only.** Do not open pull requests — the user opens PRs manually.
- After pushing, report the branch URL and stop. Optionally include a suggested PR title and summary for the user.

## Constants

| Item | Value |
|------|-------|
| Upstream Loki | https://github.com/grafana/loki |
| OpenShift Loki | https://github.com/openshift/loki |
| Fork Loki | https://github.com/JoaoBraveCoding/loki |
| Fork Release | https://github.com/JoaoBraveCoding/release |
| OpenShift Release | https://github.com/openshift/release |
| Reference PR | https://github.com/openshift/release/pull/74481 |
| Reference branch commits | https://github.com/openshift/loki/commits/upstream-v3.6.5/ |
| Fork remote name | `fork` (or `origin` if fork is origin) |
| Upstream remote name | `upstream` (grafana/loki) |
| OpenShift remote name | `openshift` (openshift/loki) |

## Phase 0: Determine versions

1. Fetch current OpenShift Loki version:
   - List branches matching `upstream-v*` at https://github.com/openshift/loki/branches
   - Parse semver from branch names (`upstream-v3.6.5` → `3.6.5`)
   - Highest semver = current OpenShift version (`OCP_LOKI_VERSION`)

2. Fetch latest upstream Grafana Loki release:
   - Check https://github.com/grafana/loki/releases
   - Use the latest non-prerelease tag (`v3.7.2` → `UPSTREAM_VERSION` = `3.7.2`)

3. Compare versions:
   - If `UPSTREAM_VERSION` <= `OCP_LOKI_VERSION`: **exit** — no update needed
   - If `UPSTREAM_VERSION` > `OCP_LOKI_VERSION`: set `TARGET_VERSION` = `UPSTREAM_VERSION` and continue

Record: `TARGET_VERSION`, `OCP_LOKI_VERSION`, branch names `upstream-v{OCP_LOKI_VERSION}` and `upstream-v{TARGET_VERSION}`.

## Phase 1: Prepare fork branch (JoaoBraveCoding/loki)

Skip this phase if fork branch `upstream-v{TARGET_VERSION}` already exists with all OCP commits.

### 1.1 Clone and set up remotes

```bash
git clone git@github.com:JoaoBraveCoding/loki.git
cd loki
git remote add upstream https://github.com/grafana/loki.git
git remote add openshift https://github.com/openshift/loki.git
git fetch --all --tags
```

### 1.2 Create branch from upstream release tag

Branch from the Grafana release tag, not from openshift/loki:

```bash
git checkout -b upstream-v{TARGET_VERSION} v{TARGET_VERSION}
```

Example for 3.7.2: `git checkout -b upstream-v3.7.2 v3.7.2`

### 1.3 Apply OCP-specific commits (one commit each)

Use the previous OpenShift branch as reference: `openshift/upstream-v{OCP_LOKI_VERSION}`.

Inspect the four OCP-only commits at the top of that branch (above the upstream release commits):

| Order | Commit message (upstream-v3.6.5) | What to do |
|-------|----------------------------------|------------|
| 1 | `Added OWNERS` | Add/update `OWNERS` for the new branch |
| 2 | `Provide OCP rhel9 based dockerfiles` | Add OCP RHEL9 Dockerfiles |
| 3 | `Remove inspect tools` | Remove inspect tools |
| 4 | `Create Dockerfile used by ART build` | Add Dockerfile used by ART build |

For each commit:

1. Diff the reference commit against its parent on `openshift/upstream-v{OCP_LOKI_VERSION}`:
   ```bash
   git log openshift/upstream-v{OCP_LOKI_VERSION} --oneline | head -10
   git show <commit-sha> --stat
   git diff <commit-sha>^..<commit-sha>
   ```
2. Apply equivalent changes adapted for `TARGET_VERSION` (paths or versions may differ)
3. Commit with the **same commit message** as the reference
4. Do **not** squash — keep four separate commits

Reference SHAs from upstream-v3.6.5 (for diff inspection only; do not cherry-pick blindly across major versions):

- `Added OWNERS` — e53306f
- `Provide OCP rhel9 based dockerfiles` — 7aba85b
- `Remove inspect tools` — bc0290b
- `Create Dockerfile used by ART build` — 3e5aaf9

### 1.4 Push fork branch

```bash
git push -u fork upstream-v{TARGET_VERSION}
```

### 1.5 Exit and wait

After pushing, **stop the workflow**. A team member creates `upstream-v{TARGET_VERSION}` in openshift/loki from your fork. This can take days.

Report status:

```
Phase 1 complete.
- Fork branch: https://github.com/JoaoBraveCoding/loki/tree/upstream-v{TARGET_VERSION}
- Waiting for openshift/loki branch upstream-v{TARGET_VERSION}
- Re-run this workflow later to continue
```

## Phase 2: Wait for openshift/loki branch

On each run after Phase 1:

```bash
git fetch openshift
git branch -r | grep "openshift/upstream-v{TARGET_VERSION}"
```

Or check https://github.com/openshift/loki/branches for `upstream-v{TARGET_VERSION}`.

- If branch **does not exist**: **exit** — still waiting
- If branch **exists**: continue to Phase 3

## Phase 3: Push openshift/release branch

Skip if branch `loki-upstream-v{TARGET_VERSION}` already exists on the fork with the CI and mirroring changes.

### 3.1 Clone openshift/release

```bash
git clone git@github.com:JoaoBraveCoding/release.git  # or fork openshift/release
cd release
git remote add upstream https://github.com/openshift/release.git
git fetch upstream
git checkout -b loki-upstream-v{TARGET_VERSION} upstream/master
```

### 3.2 Template CI and mirroring changes (4 file changes)

Use https://github.com/openshift/release/pull/74481 as the **pattern**, not as content to copy verbatim.

Start from the files for `upstream-v{OCP_LOKI_VERSION}` on current openshift/release master, then **template** them for `upstream-v{TARGET_VERSION}`. Do not copy-paste PR #74481 output directly — every version-specific string must match the new branch.

#### 3.2.1 Three new CI files

| File | Source template |
|------|-----------------|
| `ci-operator/config/openshift/loki/openshift-loki-upstream-v{TARGET_VERSION}.yaml` | `...-v{OCP_LOKI_VERSION}.yaml` |
| `ci-operator/jobs/openshift/loki/openshift-loki-upstream-v{TARGET_VERSION}-presubmits.yaml` | `...-v{OCP_LOKI_VERSION}-presubmits.yaml` |
| `ci-operator/jobs/openshift/loki/openshift-loki-upstream-v{TARGET_VERSION}-postsubmits.yaml` | `...-v{OCP_LOKI_VERSION}-postsubmits.yaml` |

Template all version-bound fields. At minimum replace:

| Location | From | To |
|----------|------|-----|
| Filenames | `upstream-v{OCP_LOKI_VERSION}` | `upstream-v{TARGET_VERSION}` |
| `zz_generated_metadata.branch` | `upstream-v{OCP_LOKI_VERSION}` | `upstream-v{TARGET_VERSION}` |
| `promotion.to.tag` | `v{OCP_LOKI_VERSION}` | `v{TARGET_VERSION}` |
| Prow job `name` fields | `upstream-v{OCP_LOKI_VERSION}` | `upstream-v{TARGET_VERSION}` |
| Prow `branches` regex (exact) | `^upstream-v{OCP_REGEX_OLD}$` | `^upstream-v{TARGET_REGEX}$` |
| Prow `branches` regex (prefix) | `^upstream-v{OCP_REGEX_OLD}-` | `^upstream-v{TARGET_REGEX}-` |

Where `{*_REGEX}` is the semver with dots escaped (e.g. `3.6.5` → `3\.6\.5`, `3.7.2` → `3\.7\.2`).

Also review non-version fields against current master — `build_root`, `base_images`, and `releases` may need updates independent of the version bump.

See [reference.md](reference.md) for the full templating checklist and field list.

#### 3.2.2 Image mirroring (required)

Update `core-services/image-mirroring/openshift-logging/mapping_logging_loki_quay`:

1. Remove `quay.io/openshift-logging/loki:latest` and `quay.io/openshift-logging/promtail:latest` from the `v{OCP_LOKI_VERSION}` lines
2. Add new lines for `v{TARGET_VERSION}` (loki and promtail) with the `latest` tags

Example pattern from PR #74481 (3.5.7 → 3.6.5):

```
-registry.ci.openshift.org/logging/loki:v3.5.7 quay.io/openshift-logging/loki:v3.5.7 quay.io/openshift-logging/loki:latest
-registry.ci.openshift.org/logging/promtail:v3.5.7 quay.io/openshift-logging/promtail:v3.5.7 quay.io/openshift-logging/promtail:latest
+registry.ci.openshift.org/logging/loki:v3.5.7 quay.io/openshift-logging/loki:v3.5.7
+registry.ci.openshift.org/logging/promtail:v3.5.7 quay.io/openshift-logging/promtail:v3.5.7
+registry.ci.openshift.org/logging/loki:v3.6.5 quay.io/openshift-logging/loki:v3.6.5 quay.io/openshift-logging/loki:latest
+registry.ci.openshift.org/logging/promtail:v3.6.5 quay.io/openshift-logging/promtail:v3.6.5 quay.io/openshift-logging/promtail:latest
```

Apply the same pattern using `OCP_LOKI_VERSION` and `TARGET_VERSION`.

### 3.3 Commit and push branch

```bash
git add ci-operator/config/openshift/loki/openshift-loki-upstream-v{TARGET_VERSION}.yaml
git add ci-operator/jobs/openshift/loki/openshift-loki-upstream-v{TARGET_VERSION}-presubmits.yaml
git add ci-operator/jobs/openshift/loki/openshift-loki-upstream-v{TARGET_VERSION}-postsubmits.yaml
git add core-services/image-mirroring/openshift-logging/mapping_logging_loki_quay

git commit -m "openshift/loki: add CI for upstream-v{TARGET_VERSION}"
git push -u origin loki-upstream-v{TARGET_VERSION}
```

Do **not** run `gh pr create` or open a PR. Report status and **stop**:

```
Phase 3 complete.
- Branch pushed: https://github.com/JoaoBraveCoding/release/tree/loki-upstream-v{TARGET_VERSION}
- User action: open PR against openshift/release master

Suggested PR title: openshift/loki: add CI for upstream-v{TARGET_VERSION}

Suggested PR summary:
- Add ci-operator config and presubmit/postsubmit jobs for openshift/loki upstream-v{TARGET_VERSION}
- Update image mirroring to promote v{TARGET_VERSION} as latest
- Templated from upstream-v{OCP_LOKI_VERSION} files; pattern reference: https://github.com/openshift/release/pull/74481
```

Future workflow steps will be added later.

## Phase detection (start of every run)

```
1. Determine TARGET_VERSION (Phase 0)
2. If no update needed → EXIT
3. If fork branch missing/incomplete → Phase 1
4. If fork pushed but openshift branch missing → Phase 2 (EXIT, wait)
5. If openshift branch exists and release branch not pushed → Phase 3
6. If release branch pushed → report status, EXIT (user opens PR)
```

## Progress checklist

Copy and update on each run:

```
Task Progress:
- [ ] Phase 0: Version check (OCP: ___, Upstream: ___, Target: ___)
- [ ] Phase 1: Fork branch upstream-v{TARGET} created
- [ ] Phase 1: Commit 1 — Added OWNERS
- [ ] Phase 1: Commit 2 — Provide OCP rhel9 based dockerfiles
- [ ] Phase 1: Commit 3 — Remove inspect tools
- [ ] Phase 1: Commit 4 — Create Dockerfile used by ART build
- [ ] Phase 1: Fork branch pushed
- [ ] Phase 2: openshift/loki upstream-v{TARGET} branch exists
- [ ] Phase 3: CI config templated (3 new files)
- [ ] Phase 3: Image mirroring updated
- [ ] Phase 3: openshift/release branch pushed
- [ ] User: PR opened against openshift/release
```

## Additional resources

- [reference.md](reference.md) — commit SHAs, file templates, API commands
