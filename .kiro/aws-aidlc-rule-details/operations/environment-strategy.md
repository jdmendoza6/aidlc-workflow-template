# Stage 1: Environment Strategy

## Purpose

Define deployment targets, environments, branch strategy, database strategy, and automation level. This stage produces the deployment plan that guides all subsequent stages.

---

## Step 1: Load Prior Context

Before asking questions, load:
- Stage 0 prerequisites document (`aidlc-docs/operations/prerequisites.md`)
- Any existing CI/CD artifacts detected in Stage 0
- Reverse engineering artifacts (if brownfield)
- Requirements documents (if they exist)

---

## Step 2: Ask Environment Strategy Questions

These are genuine decision points — the user decides what they WANT for their deployment, not what already exists.

Create `aidlc-docs/operations/environment-strategy-questions.md`:

```markdown
## Question 1: Deployment Targets
What are the deployment targets for this application?

A) Cloud containers only (ECS/Kubernetes/Cloud Run)
B) On-premises servers (bare metal/VM)
C) Hybrid (cloud containers + on-premises)
D) Serverless (Lambda/Functions/App Runner)
E) Other (please describe)

[Answer]:

## Question 2: Environment Strategy
How many environments do you need?

A) 2 environments (staging + production)
B) 3 environments (staging + UAT + production)
C) 4+ environments (dev + staging + UAT + production)
D) Other (please describe)

[Answer]:

## Question 3: Branch Strategy
How should branches map to environments?

A) GitFlow (main=prod, develop=staging, release/*=UAT)
B) Trunk-based (main=prod, staging branch, feature branches)
C) Branch-per-environment (staging branch → staging, uat branch → UAT, main → prod)
D) Other (please describe)

[Answer]:

## Question 4: Deployment Automation
What level of CI/CD automation is needed?

A) Full automation (push-to-deploy for staging, approval gate for production)
B) Semi-automated (manual trigger, automated execution)
C) Manual with scripts (documented manual steps)
D) Other (please describe)

[Answer]:

## Question 5: Database Strategy
How should database schema/seed data be managed across environments?

A) ORM migrations (EF/Alembic/Flyway — code-first, versioned)
B) SQL scripts (manually managed, versioned in repo)
C) Auto-migration on app startup (dev/staging only, manual for prod)
D) Other (please describe)

[Answer]:

## Question 6: Database Rollout Approach
How should database changes be applied to each environment?

A) Auto-migrate on deploy (migration runs as part of deployment pipeline)
B) Manual migration with approval (DBA or lead runs migration separately before deploy)
C) Blue-green DB (deploy new schema alongside old, switch after validation)
D) Maintenance window (schedule downtime, apply migration, then deploy)
E) Other (please describe)

[Answer]:

## Question 7: Environment Variable & Configuration Strategy
Where should configuration and secrets be stored and how are they managed per environment?

A) Cloud secrets manager (AWS Secrets Manager, Parameter Store, Vault) — different values per env
B) Environment files (.env) deployed with the application — different file per env
C) CI/CD pipeline secrets (GitHub Secrets, CodePipeline env vars) — injected at deploy time
D) Combination: secrets in managed service, non-sensitive config in env files
E) Other (please describe)

[Answer]:

## Question 8: Rollback Strategy
How should a failed deployment be rolled back?

A) Automatic rollback on health check failure (pipeline reverts to previous version)
B) Manual rollback with script (operator runs rollback command)
C) Blue-green swap (keep previous version running, switch back if new version fails)
D) Redeploy previous commit (push previous git SHA through pipeline again)
E) Other (please describe)

[Answer]:

## Question 9: Stage 3 Simulation Scope
What should Stage 3 (local staging simulation) include?

A) Full architecture locally (DB engine container + app on matching machine + web server)
B) Partial simulation (only components that differ from Stage 2)
C) Other (please describe)

[Answer]:
```

---

## Step 3: Wait for Answers

**DO NOT PROCEED** until all 9 questions are answered.

If answers are ambiguous, ask ONE follow-up question for clarification. Do not ask multiple rounds of questions.

---

## Step 4: Generate Deployment Plan

Based on answers, create `aidlc-docs/operations/deployment-plan.md`:

```markdown
# Deployment Plan

## Deployment Targets
- [List targets from Q1]

## Environments
| Environment | Branch | Purpose | Approval Required |
|-------------|--------|---------|-------------------|
| Staging | [branch] | Integration testing | No (auto-deploy) |
| UAT | [branch] | User acceptance | Yes (manual gate) |
| Production | [branch] | Live traffic | Yes (manual gate) |

## Deployment Paths
[For each target, define the progression path]

### Path 1: [Target Name] (e.g., ECS/Fargate)
```
Stage 3: [local simulation approach]
Stage 4: [cloud deployment approach]
Stage 5: [pipeline approach]
Stage 6: [multi-env approach]
```

### Path 2: [Target Name] (e.g., On-Prem EC2)
```
Stage 3: [local simulation approach]
Stage 4: [cloud deployment approach]
Stage 5: [pipeline approach]
Stage 6: [multi-env approach]
```

## Infrastructure (IaC)
- Tool: [CDK/Terraform/CloudFormation/etc.]
- Language: [TypeScript/Python/HCL/etc.]
- Stacks: [list of infrastructure stacks needed]

## Database
- Dev: [engine]
- Staging+: [engine]
- Migration tool: [EF Core/Flyway/manual]
- Seed strategy: [from Stage 0]

## Branch→Environment Mapping
| Branch | Environment | Trigger | Pipeline |
|--------|-------------|---------|----------|
| [branch] | [env] | [push/manual] | [pipeline name] |

## Automation Level
- Staging: [auto/manual-trigger/manual]
- UAT: [auto-with-approval/manual]
- Production: [approval-required/manual]

## Security Requirements
- No 0.0.0.0/0 in any security group
- All endpoints restricted to [deployer IP/VPN CIDR]
- Secrets in [Secrets Manager/Parameter Store/vault]
- Tests never touch production data
```

---

## Step 5: Present Plan for Approval

```markdown
## Stage 1: Environment Strategy — COMPLETE ✅

**Deployment Plan Summary**:
- Targets: [list]
- Environments: [count] ([names])
- Branch strategy: [description]
- Automation: [level]
- Paths: [count] independent deployment paths

**Deployment Plan**: `aidlc-docs/operations/deployment-plan.md`

**Next Stages (per plan)**:
- Stage 2: Create/validate local scripts for [targets]
- Stage 3: Simulate [targets] locally
- Stage 4: Deploy to [cloud provider]
- Stage 5: [if applicable] Pipeline automation
- Stage 6: [if applicable] Multi-environment ([env names])

Choose:
1. **Request Changes** — modify the deployment plan
2. **Continue to Stage 2** — proceed to local scripts & validation
```

---

## Key Rules for This Stage

1. **These are decision points** — the user chooses what they want, not validates what exists
2. **Ask ALL 9 questions at once** — don't drip-feed questions, don't skip any
3. **DO NOT auto-answer these questions** — even if the user says "use best judgement", these are strategic decisions that need explicit answers. If the user delegates, make a clear choice, STATE it visibly to the user, and give them opportunity to override before proceeding.
4. **Generate plan from answers** — don't ask more questions after this stage
5. **Each deployment target gets its own path** — independent progression
6. **Security is non-negotiable** — no 0.0.0.0/0, test data isolation baked in
7. **The plan drives ALL subsequent stages** — reference it throughout
8. **Questions 6-8 (DB rollout, config strategy, rollback) inform Stage 4-7 implementation** — these answers determine how deploy scripts work, how config is loaded, and how failures are handled

---

## MANDATORY: Context-Aware Auto-Answering

**CRITICAL**: When the AI answers environment strategy questions on the user's behalf (e.g., user says "use best judgement"), the AI MUST consider environmental context before defaulting to the simplest/minimal option.

**Context signals that imply cloud deployment**:
| Signal | Implication |
|--------|-------------|
| Running on EC2 instance | Cloud deployment is expected (user has AWS access) |
| IAM admin role attached | Full deployment capability exists — use it |
| AWS CLI configured | Cloud services are available |
| CDK/Terraform/IaC exists in workspace | Infrastructure deployment is intended |
| Dockerfile exists | Container deployment likely |
| CI/CD artifacts exist (buildspec, appspec) | Pipeline/deployment is intended |
| Multi-branch git repo | Multi-environment is likely |

**Rules for auto-answering**:
1. If running on a cloud instance (EC2/Cloud9) with IAM role → default to cloud deployment, NOT local-only
2. If the project has IaC artifacts → default to infrastructure deployment
3. If the project has CI/CD artifacts → default to pipeline automation
4. **Never default to "local only, no cloud" when cloud capability is clearly available**
5. When in doubt, choose the MORE complete option — the progression framework handles validation

**Example**: On EC2 with admin role + static HTML game:
- WRONG: "No cloud deployment needed, just open the file locally"
- RIGHT: "Deploy to S3 + CloudFront (static hosting) since we're on AWS with full capability"

---

## MANDATORY: Degenerate Result Safeguard

**CRITICAL**: If the environment strategy answers (whether user-provided or auto-answered) result in ALL cloud stages being skipped (Stage 4, 5, 6, 7 all SKIP), the AI MUST present an explicit confirmation before proceeding — even when auto-answering.

```markdown
## ⚠️ Operations Phase Scope Check

Based on the environment strategy answers, the Operations phase will:
- ✅ Execute: Stage 2 (Local Scripts), Stage 3 (Local Simulation)
- ⏭️ SKIP: Stage 4 (Cloud Deployment), Stage 5 (Pipeline), Stage 6 (Multi-Env), Stage 7 (Readiness)

This means NO cloud deployment will occur. The Operations phase will validate
local execution only.

Is this intentional?
A) Yes — local-only is correct for this project
B) No — I want cloud deployment (let me re-answer the strategy questions)

[Answer]:
```

**This confirmation is MANDATORY** — it prevents the Operations phase from being trivialized into "just run scripts locally" without the user consciously choosing that outcome.

**Exception**: If the user explicitly stated "no cloud deployment" or "local only" in their original request, this confirmation can be skipped (the intent is already clear).
