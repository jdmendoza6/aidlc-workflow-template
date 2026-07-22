# Operations Phase — Index

**Purpose**: Progressive deployment from working code to production with hard validation gates.

**Status**: ACTIVE — fully defined workflow with 8 stages (0-7).

## Stage Files

| Stage | File | Purpose |
|-------|------|---------|
| Orchestrator | `operations-workflow.md` | Main workflow coordinator, stage overview, gate rules |
| Stage 0 | `prerequisites-test-development.md` | Auto-discovery, test suite validation, project structure |
| Stage 1 | `environment-strategy.md` | Deployment targets, environments, branch strategy questions |
| Stage 2 | `local-scripts-validation.md` | Create/validate scripts, run locally, full E2E |
| Stage 3 | `staging-simulation.md` | Match target locally, deploy scripts on matching env, full E2E |
| Stage 4 | `staging-deployment.md` | Deploy to cloud, first-try success, full E2E against cloud |
| Stage 5 | `ci-cd-pipeline.md` | Pipeline automation, push-to-deploy, trigger verification |
| Stage 6 | `multi-environment.md` | UAT/Prod, branch mapping, approval gates, isolation |
| Stage 7 | `operational-readiness.md` | Monitoring, alerting, runbooks, rollback, incident response |
| Rules | `progression-framework-rules.md` | 11 enforcement rules, gate logic, failure recovery |

## Entry Point

The AI loads `operations-workflow.md` as the main orchestrator when entering the OPERATIONS phase. Individual stage files are loaded on-demand as each stage is reached.

## Progression Principle

```
Stage 0 → Stage 1 → Stage 2 → Stage 3 → Stage 4 → [Stage 5] → [Stage 6] → [Stage 7]
         (always)   (always)   (always)   (always)   (conditional) (conditional) (conditional)
```

Nothing skips a stage. Everything (code, scripts, config, IaC) progresses through all stages.
