# ai-workflows

Repository of AI agent workflows for automating recurring tasks.

## Workflows

| Workflow | Description |
|----------|-------------|
| [openshift-loki-update](workflows/openshift-loki-update/WORKFLOW.md) | Prepare OpenShift Loki image updates from upstream Grafana releases |

## Usage

### Claude ambient / Claude Code

Point Claude at this repository. `CLAUDE.md` at the repo root directs the agent to the relevant workflow.

### Cursor agents

Read the workflow markdown directly or add a project skill that references it:

```
workflows/<name>/WORKFLOW.md   — step-by-step instructions
workflows/<name>/reference.md  — URLs, commands, templates
```

## Layout

```
workflows/
  <workflow-name>/
    WORKFLOW.md      # Main instructions (phases, conditionals, exit points)
    reference.md     # Detailed reference (optional)
CLAUDE.md            # Entry point for Claude ambient sessions
```
