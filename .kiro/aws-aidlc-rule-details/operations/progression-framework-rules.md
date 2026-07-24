# Progression Framework — Enforcement Rules

## Purpose

This document defines the hard rules that the AI MUST follow during the Operations phase. These rules are derived from lessons learned during the MANSERV CI/CD deployment and are binding for all projects.

---

## CRITICAL: Database Progression Rule (applies from Requirements onward)

```
This rule applies EVEN DURING INCEPTION/CONSTRUCTION when defining requirements
and generating code. The database strategy MUST follow the progression:

Stage 2 (Local/Dev):    LIGHTWEIGHT DB — SQLite, H2, in-memory
Stage 3 (Simulation):   REAL DB ENGINE in container — SQL Server, PostgreSQL, MySQL
Stage 4 (Cloud):        REAL DB ENGINE on managed service — RDS, Cloud SQL

The application MUST be designed with provider switching from the start:
- EF Core: Configure SQLite provider for dev, SQL Server for prod
- Node.js: Use SQLite for dev, pg/mssql for prod (switched via env var)
- Java: H2 for dev, PostgreSQL/MySQL for prod

WRONG (in requirements/design): "Local dev uses Docker Compose with SQL Server"
RIGHT (in requirements/design): "Local dev uses SQLite; SQL Server for staging+"

If the AI writes requirements or generates code that puts the production DB engine
in the local dev environment, it is violating this rule.
```

---

## The 11 Enforcement Rules

### Rule 1: NO CLOUD DEPLOYMENT WITHOUT LOCAL VALIDATION

```
IF script/artifact will run on a remote target (EC2, ECS, Lambda, CodeBuild)
THEN it MUST be tested locally in a matching environment FIRST
NO EXCEPTIONS
```

**Enforcement**: Before any `aws deploy`, `cdk deploy`, or image push, the AI must:
1. Show evidence of local testing (command + output)
2. Show the test passed (exit code 0 + expected output)
3. Get explicit user approval
4. Only THEN proceed to cloud deployment

---

### Rule 2: NEVER SKIP PROGRESSION STAGES

```
Stage 0 → Stage 1 → Stage 2 → Stage 3 → Stage 4 → Stage 5 → Stage 6 → Stage 7
Each stage MUST pass. No skipping. No "it should work."
```

**Violation consequence**: If the AI skips a stage and deployment fails:
1. Stop immediately
2. Go BACK to the earliest affected stage
3. Fix the issue at that stage
4. Re-validate upward from that stage

---

### Rule 3: MATCH THE TARGET ENVIRONMENT FOR SIMULATION

```
Stage 3 simulation MUST match the actual deployment target:
- Docker for container targets (ECS, Kubernetes)
- Matching machine (same OS) for bare-metal targets (EC2, on-prem)
- Real DB engine for database targets (SQL Server container for RDS SQL Server)
- Local invoke for serverless (SAM for Lambda)
```

**CRITICAL**: Docker is a POOR simulation for bare-metal targets. It lacks systemd, has minimal packages, and behaves differently. NEVER use Docker to simulate on-prem deployment.

---

### Rule 4: VERIFY ASSUMPTIONS WITH DOCUMENTATION

```
Before using any AWS/cloud service feature:
1. Check documentation (MCP search) for constraints
2. Specifically look for "not supported" / "must be" / "requires"
3. Document the finding
THEN implement
```

Do not assume service behavior. RDS SQL Server doesn't support `databaseName` parameter. ECS CODE_DEPLOY requires ALB first. These were caught late — verify BEFORE implementing.

---

### Rule 5: ONE CHANGE PER DEPLOYMENT (when debugging)

```
When a deployment fails:
1. Identify the SINGLE root cause
2. Fix ONLY that issue
3. Test locally (Rule 1)
4. Deploy again
DO NOT fix multiple things at once (can't isolate which fix worked)
```

---

### Rule 6: SCRIPT TESTING CHECKLIST

Before any deployment script is considered "done":

```markdown
- [ ] Syntax valid (bash -n script.sh)
- [ ] Runs in matching target environment
- [ ] All required tools exist (curl, tar, dotnet, nginx, etc.)
- [ ] File permissions correct
- [ ] User context correct (can the runas user execute this?)
- [ ] Exit code is 0
- [ ] Expected side effects verified (files created, services started, health responds)
- [ ] Idempotent (safe to run multiple times)
```

---

### Rule 7: EXHAUST ALL AUTOMATED STEPS BEFORE REQUESTING MANUAL ACTION

```
BEFORE telling the user "the next step is manual":
1. List ALL remaining steps
2. For EACH step, determine: is it blocked? By what?
3. Execute ALL unblocked automated steps FIRST
4. ONLY AFTER all automated steps are done, present manual steps to user
```

The AI must NOT:
- Skip automated validation steps because "it should work"
- Direct user to manual Console steps when automated testing remains
- Declare something "blocked" without verifying the full dependency chain

---

### Rule 8: SYSTEMATIC STEP ENUMERATION

```
At EVERY decision point, the AI must:
1. State what has been DONE (with evidence)
2. List what REMAINS (all steps, not just next one)
3. Classify each remaining step: AUTOMATED/MANUAL, BLOCKED/UNBLOCKED
4. Execute the next UNBLOCKED AUTOMATED step
5. Repeat until only BLOCKED or MANUAL steps remain
```

This prevents:
- "Forgetting" that validation steps exist
- Jumping to manual steps prematurely
- Losing track of overall progress

---

### Rule 9: IMAGE/ARTIFACT IDENTITY

```
The container image / binary / bundle that passes Stage 3 MUST be the
EXACT SAME artifact deployed at Stage 4.

- Tag images with git SHA: app:abc1234
- Push THAT tag to registry
- Task definition / deployment references THAT tag
- Never rebuild between stages
- Verify: after deployment, confirm running artifact matches what was tested
```

---

### Rule 10: "EVENTUALLY WORKED" IS NOT VALIDATED

```
A deployment is VALIDATED only if:
1. Scripts pass local simulation (Stage 3) on FIRST attempt
2. The SAME scripts deploy to cloud (Stage 4) on FIRST attempt
3. E2E tests pass against the deployed target

If a deployment required multiple retries on cloud to succeed:
- It is NOT validated
- The scripts must be fixed and re-tested LOCALLY until they pass first-try
- THEN redeployed to cloud (expecting first-try success)
- ONLY THEN is it validated

"It works now" after 6 tries ≠ "It's validated"
"It works first-try from local to cloud" = "It's validated"
```

---

### Rule 11: PARALLEL PATHS NEED INDEPENDENT PROGRESSION

```
When multiple deployment targets exist (e.g., ECS + On-Prem):
- Each target has its OWN progression path
- Each path must independently pass Stage 3 before Stage 4
- Passing Stage 3 for Docker/ECS does NOT validate the On-Prem path
- Passing Stage 3 for On-Prem does NOT validate the ECS path

Example:
  ECS Path:     Stage 3 (Docker) → Stage 4 (ECR→ECS→ALB)
  On-Prem Path: Stage 3 (matching machine) → Stage 4 (CodeDeploy→EC2)

These are INDEPENDENT validations. Both must pass their own progression.
```

---

## Stage Gate Enforcement

The AI enforces gates with this logic:

```
FUNCTION can_proceed_to_next_stage(current_stage, path):
  IF current_stage.gate_criteria NOT all_met:
    RETURN "BLOCKED: [list unmet criteria]"
  IF current_stage.first_try_success == false AND current_stage >= 3:
    RETURN "BLOCKED: Must pass first-try. Fix at Stage N-1, then retry."
  IF current_stage.e2e_count != previous_stage.e2e_count:
    RETURN "BLOCKED: E2E count mismatch ([prev] vs [current])"
  RETURN "APPROVED: All gates passed. Ready for Stage N+1."
```

---

## Failure Recovery Protocol

When a stage fails:

```
1. STOP — do not retry immediately on cloud
2. IDENTIFY — root cause from error output
3. CLASSIFY:
   a) Script/code issue → fix at Stage 3 (local), then retry Stage 4
   b) Cloud-specific issue (IAM/networking) → fix on cloud, document
   c) Configuration issue → fix config, validate at Stage 3, retry Stage 4
4. FIX — apply the single fix (Rule 5)
5. VALIDATE — must pass first-try at current stage
6. PROCEED — only after first-try success
```

---

## Audit Requirements

ALL enforcement actions must be logged in audit.md:

```markdown
## [Stage] Gate Check
**Timestamp**: [ISO timestamp]
**Stage**: [stage number and name]
**Path**: [deployment path]
**Gate Criteria**:
- [criterion 1]: [PASS/FAIL + evidence]
- [criterion 2]: [PASS/FAIL + evidence]
**Decision**: [PROCEED/BLOCKED]
**Reason**: [why]
```

---

## Summary: The Pattern

Every mistake in the MANSERV deployment followed this anti-pattern:
```
Assumed it would work → Deployed to cloud → Discovered it doesn't → Fixed on cloud → Repeated
```

The correct pattern (enforced by these rules):
```
Verified locally → Proved it works (first-try) → Deployed to cloud → Confirmed (first-try success)
```

Every minute spent on local testing saves 10 minutes of debugging cloud failures.
