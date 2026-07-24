# Stage 5: CI/CD Pipeline

## Purpose

Automate Stage 4 deployment via a push-to-deploy pipeline. After this stage, pushing to the designated branch triggers build → test → deploy automatically. The pipeline must produce the same result as manual deployment.

---

## When to Execute

**Execute IF**:
- Deployment automation level is "Full" or "Semi-automated" (from Stage 1 answers)
- Multiple team members deploy (automation prevents drift)
- Deployment frequency justifies automation

**Skip IF**:
- Manual deployment with scripts is sufficient
- Single developer, infrequent deployments
- User explicitly chooses "Manual with scripts"

---

## Step 1: Identify Pipeline Components

Based on deployment plan (Stage 1):

| Component | Purpose | Examples |
|-----------|---------|----------|
| Source | Trigger on code change | CodeStar Connection, GitHub webhook |
| Build | Compile + test + package | CodeBuild, GitHub Actions, Jenkins |
| Deploy | Push to target | ECS deploy, CodeDeploy, S3 sync |
| Approval | Gate between environments | Manual approval action, Slack notification |
| Artifacts | Intermediate outputs | Docker image, zip bundle, build output |

---

## Step 2: Create Pipeline Configuration

### For AWS CodePipeline (CDK)

Pipeline infrastructure is defined in IaC (CDK/Terraform). Key elements:

```markdown
Pipeline Stages:
1. Source: GitHub/CodeCommit → trigger on branch push
2. Build: CodeBuild → execute buildspec.yml (build, test, package)
3. [Approval]: Manual gate (for UAT/Prod only)
4. Deploy: ECS deploy action / CodeDeploy action
```

### Pipeline Artifacts Needed

| Artifact | Path | Purpose |
|----------|------|---------|
| buildspec.yml | repo root | CodeBuild instructions |
| taskdef-template.json | repo root | ECS task definition template |
| appspec-ecs.yaml | repo root | ECS deployment spec |
| appspec-onprem.yml | repo root | On-prem deployment spec |
| imagedefinitions.json | build output | ECS image reference |

---

## Step 3: Validate Pipeline Artifacts Locally

Before deploying the pipeline, validate that pipeline artifacts produce correct results locally:

```bash
# Simulate buildspec.yml commands locally
# (same commands that CodeBuild will execute)
bash scripts/build.sh
bash scripts/test.sh
bash scripts/docker-build.sh

# Verify imagedefinitions.json output format
echo '[{"name":"container-name","imageUri":"ACCOUNT.dkr.ecr.REGION.amazonaws.com/REPO:TAG"}]'

# Verify taskdef-template.json is valid JSON
python3 -c "import json; json.load(open('taskdef-template.json'))"

# Verify appspec files are valid YAML
python3 -c "import yaml; yaml.safe_load(open('appspec-ecs.yaml'))"
```

---

## Step 4: Deploy Pipeline Infrastructure

```bash
# Deploy pipeline stack (CDK example)
cdk deploy PIPELINE_STACK_NAME --require-approval never

# Verify pipeline was created
aws codepipeline get-pipeline --name PIPELINE_NAME
```

**Prerequisites** (may be manual):
- CodeStar Connection to source repository (GitHub OAuth — manual Console step)
- IAM roles for pipeline execution
- S3 bucket for artifacts

**Rule**: If prerequisites require manual steps, document them clearly and present AFTER all automated validation is complete.

---

## Step 5: Test Pipeline Trigger

```bash
# Make a trivial change and push to trigger branch
echo "# pipeline test $(date)" >> README.md
git add README.md
git commit -m "test: trigger pipeline"
git push origin TRIGGER_BRANCH

# Monitor pipeline execution
aws codepipeline get-pipeline-execution \
  --pipeline-name PIPELINE_NAME \
  --pipeline-execution-id EXECUTION_ID

# Wait for completion
# Expected: pipeline succeeds, application deployed, health check passes
```

---

## Step 6: Verify Pipeline Result Matches Manual

After pipeline deploys:

```bash
# Run same E2E as Stage 4 to verify pipeline-deployed app works identically
bash scripts/test-e2e-remote.sh http://ENDPOINT

# Expected: SAME test count as Stage 4
```

The pipeline-deployed application must behave identically to the manually-deployed one.

---

## Step 7: Document Results

```markdown
# Stage 5: CI/CD Pipeline — Results

## Pipeline Configuration
- Platform: [CodePipeline/GitHub Actions/etc.]
- Trigger: Push to [branch]
- Stages: [Source → Build → (Approval) → Deploy]

## Pipeline Deployment
- Stack: [stack name]
- Status: [CREATE_COMPLETE]

## Trigger Test
- Commit: [SHA]
- Branch: [branch name]
- Pipeline execution: [ID]
- Result: [Succeeded/Failed]
- Duration: [time]

## Post-Pipeline E2E
- Tests: [X] pass (must match Stage 4 count)
- Endpoint: [URL]

## Manual Prerequisites
- [ ] CodeStar Connection: [ARN or status]
- [ ] [Other manual steps if any]
```

---

## Step 8: Completion Message

```markdown
## Stage 5: CI/CD Pipeline — COMPLETE ✅

**Evidence**:
- Pipeline deployed: [pipeline name]
- Trigger test: Push to [branch] → pipeline executed → deployed successfully
- E2E post-pipeline: [X] tests pass (matches Stage 4)
- Duration: [build time] + [deploy time]

**Gate Status**: PASSED
- Push triggers pipeline automatically
- Pipeline builds, tests, and deploys correctly
- Result matches manual deployment

**Next Stage**: Stage 6: Multi-Environment [or Stage 7 / Complete]

Choose:
1. **Request Changes** — specify what needs modification
2. **Continue to Stage 6** — proceed to multi-environment
```

---

## Key Rules for This Stage

1. **Pipeline MUST call the same scripts validated at Stage 2/3/4** — the pipeline is an ORCHESTRATOR, not a reimplementation. It calls `build.sh`, `test.sh`, `deploy-to-ec2.sh`, `test-e2e-remote.sh` — the EXACT same scripts. NO inline commands that duplicate or rewrite what those scripts do.
2. **NO inline build/deploy logic in pipeline config** — if the pipeline has `npm ci && tar czf ...` inline, it violates the progression framework. Those commands belong in scripts that were validated locally. The pipeline YAML should contain ONLY: checkout, setup credentials, call scripts.
3. **Same result as manual** — pipeline-deployed app must pass same E2E as Stage 4
4. **Pipeline artifacts validated locally first** — buildspec commands must work on local machine
5. **Manual prerequisites documented** — CodeStar Connection, etc. clearly stated
6. **Trigger test required** — not enough to deploy pipeline; must verify push-to-deploy works
7. **Approval gates for non-staging** — staging auto-deploys, UAT/Prod require approval
8. **Automated steps first** — validate artifacts, deploy stack, THEN present manual steps
9. **Security group management for CI runners** — if runners need SSH access, use DYNAMIC IP whitelisting (get runner IP → add to SG → deploy → revoke). NEVER open 0.0.0.0/0 for runner access.
