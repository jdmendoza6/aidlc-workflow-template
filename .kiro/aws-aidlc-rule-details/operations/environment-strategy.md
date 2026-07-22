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

## Question 6: Stage 3 Simulation Scope
What should Stage 3 (local staging simulation) include?

A) Full architecture locally (DB engine container + app on matching machine + web server)
B) Partial simulation (only components that differ from Stage 2)
C) Other (please describe)

[Answer]:
```

---

## Step 3: Wait for Answers

**DO NOT PROCEED** until all 6 questions are answered.

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
2. **Ask all 6 questions at once** — don't drip-feed questions
3. **Generate plan from answers** — don't ask more questions after this stage
4. **Each deployment target gets its own path** — independent progression
5. **Security is non-negotiable** — no 0.0.0.0/0, test data isolation baked in
6. **The plan drives ALL subsequent stages** — reference it throughout
