# OpenShift Loki Update — Reference

## Version discovery commands

```bash
# Latest upstream Grafana Loki release
gh api repos/grafana/loki/releases/latest --jq '.tag_name'

# OpenShift upstream-v* branches (highest semver wins)
gh api repos/openshift/loki/branches --paginate \
  --jq '.[].name' | grep '^upstream-v' | sort -V | tail -1

# Check if openshift branch exists for target version
gh api repos/openshift/loki/branches/upstream-v{TARGET_VERSION} 2>/dev/null && echo "exists" || echo "missing"

# Check fork branch
gh api repos/JoaoBraveCoding/loki/branches/upstream-v{TARGET_VERSION} 2>/dev/null && echo "exists" || echo "missing"
```

## OCP-specific commits on upstream-v3.6.5

These sit on top of the upstream release commits. Apply equivalent changes for each new version.

| # | SHA | Message | URL |
|---|-----|---------|-----|
| 1 | e53306f | Added OWNERS | https://github.com/openshift/loki/commit/e53306f191ffa03226efd36c73d9b9679deb57c1 |
| 2 | 7aba85b | Provide OCP rhel9 based dockerfiles | https://github.com/openshift/loki/commit/7aba85b4ab5b2d9a7af39edb23d7a674d378dbd1 |
| 3 | bc0290b | Remove inspect tools | https://github.com/openshift/loki/commit/bc0290b8293cfd858507a9623f3e7cba2a9e63ad |
| 4 | 3e5aaf9 | Create Dockerfile used by ART build | https://github.com/openshift/loki/commit/3e5aaf9c7acab2beed73133d6e3bd8e736dbe2a1 |

Full commit log: https://github.com/openshift/loki/commits/upstream-v3.6.5/

## openshift/release PR #74481 — templating guide

Reference pattern: https://github.com/openshift/release/pull/74481

**Do not copy-paste PR #74481 files.** They target `upstream-v3.6.5`. For each new `upstream-v{TARGET}`, template from the current `upstream-v{OCP_LOKI_VERSION}` files on openshift/release master.

### Files to produce (4 changes)

| # | File | Action |
|---|------|--------|
| 1 | `ci-operator/config/openshift/loki/openshift-loki-upstream-v{TARGET}.yaml` | New — templated from previous version config |
| 2 | `ci-operator/jobs/openshift/loki/openshift-loki-upstream-v{TARGET}-presubmits.yaml` | New — templated from previous presubmits |
| 3 | `ci-operator/jobs/openshift/loki/openshift-loki-upstream-v{TARGET}-postsubmits.yaml` | New — templated from previous postsubmits |
| 4 | `core-services/image-mirroring/openshift-logging/mapping_logging_loki_quay` | Modified — required |

### Templating procedure

1. Read the three files for `upstream-v{OCP_LOKI_VERSION}` from openshift/release master
2. Create new files named for `upstream-v{TARGET_VERSION}`
3. Replace every version-bound identifier (see checklist below)
4. Verify regex escaping — semver dots must be escaped in prow `branches` patterns
5. Review `build_root`, `base_images`, `releases` against current OCP standards (may differ from reference PR)

### Version substitution checklist

Given `OCP_LOKI_VERSION=3.6.5` and `TARGET_VERSION=3.7.2`:

| Field | Old value | New value |
|-------|-----------|-----------|
| Config/job filename | `upstream-v3.6.5` | `upstream-v3.7.2` |
| `zz_generated_metadata.branch` | `upstream-v3.6.5` | `upstream-v3.7.2` |
| `promotion.to.tag` | `v3.6.5` | `v3.7.2` |
| Postsubmit job name | `branch-ci-openshift-loki-upstream-v3.6.5-images` | `branch-ci-openshift-loki-upstream-v3.7.2-images` |
| Presubmit images job | `pull-ci-openshift-loki-upstream-v3.6.5-images` | `pull-ci-openshift-loki-upstream-v3.7.2-images` |
| Presubmit test job | `pull-ci-openshift-loki-upstream-v3.6.5-test` | `pull-ci-openshift-loki-upstream-v3.7.2-test` |
| Branch regex (exact) | `^upstream-v3\.6\.5$` | `^upstream-v3\.7\.2$` |
| Branch regex (prefix) | `^upstream-v3\.6\.5-` | `^upstream-v3\.7\.2-` |

General rule: any literal `{OCP_LOKI_VERSION}` or `upstream-v{OCP_LOKI_VERSION}` in filenames, metadata, job names, tags, or regex must become `{TARGET_VERSION}` / `upstream-v{TARGET_VERSION}` with dots escaped where used in regex.

### Image mirroring (required)

File: `core-services/image-mirroring/openshift-logging/mapping_logging_loki_quay`

On each version bump:

1. Strip `quay.io/openshift-logging/loki:latest` from the current promoted loki line (`v{OCP_LOKI_VERSION}`)
2. Strip `quay.io/openshift-logging/promtail:latest` from the current promoted promtail line (`v{OCP_LOKI_VERSION}`)
3. Add loki line: `registry.ci.openshift.org/logging/loki:v{TARGET} quay.io/openshift-logging/loki:v{TARGET} quay.io/openshift-logging/loki:latest`
4. Add promtail line: `registry.ci.openshift.org/logging/promtail:v{TARGET} quay.io/openshift-logging/promtail:v{TARGET} quay.io/openshift-logging/promtail:latest`

Example from PR #74481 (3.5.7 was previous latest, 3.6.5 became new latest):

```
 registry.ci.openshift.org/logging/loki:v3.4.3 quay.io/openshift-logging/loki:v3.4.3
 registry.ci.openshift.org/logging/promtail:v3.4.3 quay.io/openshift-logging/promtail:v3.4.3
-registry.ci.openshift.org/logging/loki:v3.5.7 quay.io/openshift-logging/loki:v3.5.7 quay.io/openshift-logging/loki:latest
-registry.ci.openshift.org/logging/promtail:v3.5.7 quay.io/openshift-logging/promtail:v3.5.7 quay.io/openshift-logging/promtail:latest
+registry.ci.openshift.org/logging/loki:v3.5.7 quay.io/openshift-logging/loki:v3.5.7
+registry.ci.openshift.org/logging/promtail:v3.5.7 quay.io/openshift-logging/promtail:v3.5.7
+registry.ci.openshift.org/logging/loki:v3.6.5 quay.io/openshift-logging/loki:v3.6.5 quay.io/openshift-logging/loki:latest
+registry.ci.openshift.org/logging/promtail:v3.6.5 quay.io/openshift-logging/promtail:v3.6.5 quay.io/openshift-logging/promtail:latest
```

## Example: v3.6.5 ci-operator config shape

```yaml
base_images:
  base:
    name: "4.18"
    namespace: ocp
    tag: base-rhel9
build_root:
  image_stream_tag:
    name: builder
    namespace: ocp
    tag: rhel-9-golang-1.25-openshift-4.23
images:
- additional_architectures:
  - arm64
  dockerfile_path: Dockerfile.ocp
  from: base
  to: loki
- additional_architectures:
  - arm64
  dockerfile_path: Dockerfile.promtail.ocp
  from: base
  to: promtail
promotion:
  to:
  - namespace: logging
    tag: v3.6.5
releases:
  latest:
    release:
      channel: stable
      version: "4.21"
resources:
  '*':
    requests:
      cpu: 100m
      memory: 200Mi
tests:
- as: test
  steps:
    test:
    - as: unit
      commands: GOFLAGS="" make test
      from: src
      resources:
        requests:
          cpu: 100m
          memory: 200Mi
zz_generated_metadata:
  branch: upstream-v3.6.5
  org: openshift
  repo: loki
```

Verify `build_root.tag`, `base_images`, and `releases.latest` against the current openshift/release master — these change between OCP versions.

## Branch naming

| Location | Pattern | Example |
|----------|---------|---------|
| openshift/loki | `upstream-v{VERSION}` | `upstream-v3.7.2` |
| JoaoBraveCoding/loki (fork) | `upstream-v{VERSION}` | `upstream-v3.7.2` |
| grafana/loki (source) | tag `v{VERSION}` | `v3.7.2` |
| JoaoBraveCoding/release (fork) | `loki-upstream-v{VERSION}` | `loki-upstream-v3.7.2` |

Agent pushes to the fork branches above. The user opens PRs manually — do not run `gh pr create`.

```bash
# Check if release branch already pushed to fork
gh api repos/JoaoBraveCoding/release/branches/loki-upstream-v{TARGET_VERSION} 2>/dev/null && echo "exists" || echo "missing"
```
