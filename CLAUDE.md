# AI Workflows — Claude Ambient Instructions

This repository stores agent workflows. When asked to prepare an OpenShift Loki image update, follow the workflow below.

## OpenShift Loki image update

**Trigger:** User mentions Loki version update, OpenShift logging image, upstream-v branch, or openshift/release CI for loki.

**Workflow file:** [workflows/openshift-loki-update/WORKFLOW.md](workflows/openshift-loki-update/WORKFLOW.md)

**Reference:** [workflows/openshift-loki-update/reference.md](workflows/openshift-loki-update/reference.md)

### Quick summary

1. Compare grafana/loki latest release vs highest openshift/loki `upstream-v*` branch
2. If update needed: create `upstream-v{VERSION}` on JoaoBraveCoding/loki from grafana tag, apply 4 OCP commits separately, push, exit
3. Wait for team to create openshift/loki branch (re-check on next run)
4. Open openshift/release PR: template 3 CI files + update image mirroring from previous `upstream-v*` version (pattern: PR #74481; do not copy-paste verbatim)

Always detect the current phase first. Do not skip the wait step after pushing the fork branch.
