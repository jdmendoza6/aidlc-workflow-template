# Operations Phase — Main Orchestrator

## Purpose

The OPERATIONS phase takes an application from CONSTRUCTION (working code + tests) through progressive deployment to production. It enforces hard progression gates (no skipping), is technology-agnostic, and follows the same patterns as INCEPTION and CONSTRUCTION.

## Design Principles

1. **Progressive validation**: Each stage must fully pass before the next begins. No skipping.
2. **Match the target**: Stage 2 simulation MUST match the actual target environment.
3. **Technology agnostic**: Defines principles, stages, and gates — not specific commands.
4. **Fault isolation**: If something fails, you know EXACTLY which stage it failed at and why.
5. **Everything progresses**: Code, build scripts, deploy scripts, seed data, config, IaC — ALL go through all stages.
6. **Safe by default**: Tests never touch production data. Security groups never open to 0.0.0.0/0.
7. **Exhaust automated before manual**: ALL automated steps completed before requesting manual Console actions.
8. **First-try success**: Scripts proven at Stage 2 should succeed first-try on cloud (Stage 3+).

---

## Stage Overview

```
OPERATIONS PHASE
├── Stage 0: Prerequisites & Test Development (ALWAYS)
├── Stage 1: Environment Strategy (ALWAYS)
├── Stage 2: Local Scripts & Validation (ALWAYS) — Framework Stage 1
├── Stage 3: Staging Simulation Locally (ALWAYS) — Framework Stage 2
├── Stage 4: Staging Cloud Deployment (ALWAYS) — Framework Stage 3
├── Stage 5: CI/CD Pipeline (CONDITIONAL)
├── Stage 6: Multi-Environment (CONDITIONAL) — Framework Stage 4
└── Stage 7: Operational Readiness (CONDITIONAL)
```

---

## Stage Execution Rules

### Stage 0: Prerequisites & Test Development (ALWAYS EXECUTE)

**Purpose**: Ensure the application has an automated test suite, organized structure, and defined seed data strategy before any deployment work begins.

**Execution**:
1. Load all steps from `operations/prerequisites-test-development.md`
2. Auto-discover project state (scan for tests, structure, DB, Dockerfile, CI/CD)
3. Present findings for user validation
4. If gaps found (no tests, no structure): develop them
5. **Gate**: Automated test suite passes 100%, structure documented
6. **Wait for Explicit Approval** — DO NOT PROCEED until user confirms

### Stage 1: Environment Strategy (ALWAYS EXECUTE)

**Purpose**: Define deployment targets, environments, branch strategy, DB strategy, automation level.

**Execution**:
1. Load all steps from `operations/environment-strategy.md`
2. If auto-answering: consider environmental context (EC2, IAM role, existing IaC) before defaulting to minimal
3. Ask environment strategy questions (6 decision-point questions)
4. Document answers in `aidlc-docs/operations/environment-strategy.md`
5. Generate deployment plan based on answers
6. **MANDATORY**: If ALL cloud stages (4-7) would be skipped, present degenerate result safeguard confirmation
7. **Wait for Explicit Approval** — DO NOT PROCEED until user confirms

### Stage 2: Local Scripts & Validation (ALWAYS EXECUTE)

**Purpose**: Create build/test/run/deploy scripts and validate everything works locally.

**Execution**:
1. Load all steps from `operations/local-scripts-validation.md`
2. Auto-discover existing scripts (scan project for build.sh, test.sh, deploy/, Dockerfile, etc.)
3. Present findings for validation ("I found these scripts, are they the ones to use?")
4. Gap analysis: identify missing scripts vs. deployment plan requirements
5. Create missing scripts OR validate existing ones
6. Run ALL scripts locally — every one must pass
7. Run FULL test suite locally (all tests, not a subset)
8. **Gate**: All scripts pass, full test suite passes, health check responds
9. **Wait for Explicit Approval** — DO NOT PROCEED until user confirms

### Stage 3: Staging Simulation Locally (ALWAYS EXECUTE)

**Purpose**: Simulate the real deployment architecture locally, matching the actual target.

**Execution**:
1. Load all steps from `operations/staging-simulation.md`
2. Determine simulation requirements per target (see matching rules)
3. Set up local simulation environment
4. Run deploy scripts on matching environment
5. Run FULL E2E test suite against local simulation
6. **Gate**: All deploy scripts pass first-try, full E2E passes
7. **Wait for Explicit Approval** — DO NOT PROCEED until user confirms

**Matching Rules**:
| Deployment Target | Stage 3 Simulation | NOT This |
|-------------------|-------------------|----------|
| ECS (container) | Docker container locally | — |
| EC2 on-prem (bare metal) | Run on matching local machine (same OS) | ❌ Docker |
| RDS SQL Server | SQL Server container locally | ❌ SQLite |
| Lambda | Local invoke (SAM) | ❌ Docker |
| Amplify | Run amplify.yml commands locally | — |

### Stage 4: Staging Cloud Deployment (ALWAYS EXECUTE)

**Purpose**: Deploy to actual cloud infrastructure and validate with full E2E.

**Execution**:
1. Load all steps from `operations/staging-deployment.md`
2. Deploy IaC (CDK/CloudFormation/Terraform)
3. Deploy application to cloud target
4. Verify health check on cloud endpoint
5. Run FULL E2E test suite against cloud endpoint
6. **Gate**: First-try deployment success, full E2E passes against cloud
7. **Wait for Explicit Approval** — DO NOT PROCEED until user confirms

**Critical Rule**: If deployment fails at Stage 4, STOP. Fix locally at Stage 3 first. Then redeploy. Do NOT iterate on cloud.

### Stage 5: CI/CD Pipeline (CONDITIONAL)

**Execute IF**: Automation level requires push-to-deploy pipeline.
**Skip IF**: Manual deployment with scripts is sufficient.

**Execution**:
1. Load all steps from `operations/ci-cd-pipeline.md`
2. Create/validate pipeline configuration (CodePipeline, GitHub Actions, etc.)
3. Deploy pipeline infrastructure
4. Test pipeline trigger (push to branch → pipeline executes → deploys)
5. Verify pipeline produces same result as manual deployment
6. **Gate**: Push triggers pipeline, pipeline deploys successfully, E2E passes
7. **Wait for Explicit Approval** — DO NOT PROCEED until user confirms

### Stage 6: Multi-Environment (CONDITIONAL)

**Execute IF**: Multiple environments needed (staging + UAT + production).
**Skip IF**: Single environment is sufficient.

**Execution**:
1. Load all steps from `operations/multi-environment.md`
2. Deploy additional environments (UAT, production)
3. Verify branch→environment mapping
4. Test approval gates between environments
5. Run FULL E2E per environment
6. Verify environment isolation
7. **Gate**: Each environment independently passes E2E, isolation confirmed
8. **Wait for Explicit Approval** — DO NOT PROCEED until user confirms

### Stage 7: Operational Readiness (CONDITIONAL)

**Execute IF**: Production deployment requires monitoring, alerting, runbooks.
**Skip IF**: Not going to production yet, or ops readiness not required.

**Execution**:
1. Load all steps from `operations/operational-readiness.md`
2. Set up monitoring and alerting
3. Create runbooks for common operations
4. Document rollback procedures
5. Create incident response plan
6. **Gate**: Monitoring active, runbooks documented, rollback tested
7. **Wait for Explicit Approval** — DO NOT PROCEED until user confirms

---

## Hard Gate Rules

1. Stage N **MUST** pass before Stage N+1 can begin
2. If a script/artifact is MODIFIED, re-validate from the earliest affected stage
3. The AI **REFUSES** to deploy to a higher stage if the current stage hasn't passed
4. **Opt-out**: User can explicitly say "skip gate X" — logged in audit.md with acknowledgment
5. **ALL unblocked automated steps** executed before presenting manual steps
6. **First-try success required** at Stage 4+: Scripts proven at Stage 3 should succeed first-try
7. **"Eventually worked" ≠ validated**: Multi-retry success means fix locally and redo

---

## Parallel Deployment Paths

When multiple targets exist, each has its OWN independent progression:

```
ECS Path:     Stage 2 → Stage 3 (Docker) → Stage 4 (ECR→ECS→ALB) → Stage 5/6
On-Prem Path: Stage 2 → Stage 3 (matching machine) → Stage 4 (CodeDeploy→EC2) → Stage 5/6
```

Passing one path does NOT validate the other. Both must independently pass all stages.

---

## State Tracking

Update `aidlc-state.md` with operations progress using this format:

```markdown
### OPERATIONS PHASE
- [x] Stage 0: Prerequisites (test suite: X tests pass, structure documented)
- [x] Stage 1: Environment Strategy (answers documented)
- [x] Stage 2: Local Scripts (all pass, E2E: X tests)
- [ ] Stage 3: Staging Simulation
  - [x] ECS path: Docker container, E2E passes
  - [ ] On-Prem path: Matching machine, E2E passes
- [ ] Stage 4: Cloud Deployment
  - [ ] ECS path: ALB responds, E2E passes
  - [ ] On-Prem path: CodeDeploy succeeds, E2E passes
- [ ] Stage 5: CI/CD Pipeline
- [ ] Stage 6: Multi-Environment
- [ ] Stage 7: Operational Readiness
```

---

## Completion Message Format

After each stage completes, present:

```markdown
## Stage X: [Name] — COMPLETE ✅

**Evidence**:
- [List what was validated with specific numbers/outputs]

**Gate Status**: PASSED
- [Criteria met with evidence]

**Next Stage**: Stage X+1: [Name]

Choose:
1. **Request Changes** — specify what needs modification
2. **Continue to Stage X+1** — proceed to next stage
```

---

## Pattern Alignment

| Pattern | INCEPTION/CONSTRUCTION | OPERATIONS |
|---------|----------------------|-----------|
| Questions file | `requirement-verification-questions.md` | `environment-strategy-questions.md` |
| Plan with checkboxes | `code-generation-plan.md` | `deployment-plan.md` |
| Approval gate | "Request Changes / Continue" | Same 2-option format |
| Audit logging | Every input + response logged | Same |
| State tracking | `aidlc-state.md` with checkboxes | Same (adds per-path gates) |
| Content validation | Validate before file creation | Validate scripts on matching target before deploy |
| Wait for approval | DO NOT PROCEED until confirmed | Same |
