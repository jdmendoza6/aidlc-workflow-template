# Stage 4: Staging Cloud Deployment

## Purpose

Deploy to actual cloud infrastructure and validate with the full E2E test suite. Scripts proven at Stage 3 (local simulation) should succeed first-try on cloud. If they don't, something is wrong with the cloud configuration — fix it, but the fix must be validated at Stage 3 first before retrying cloud.

---

## Pre-Deployment Checklist

Before ANY cloud deployment, verify:

```markdown
- [ ] Stage 3 PASSED for this path (evidence documented)
- [ ] All scripts passed first-try at Stage 3
- [ ] Same artifacts/images being deployed (image identity rule)
- [ ] IaC templates validated (cdk synth / terraform plan / cfn validate)
- [ ] No 0.0.0.0/0 in any security group
- [ ] Secrets/config values prepared in target service
- [ ] User has approved proceeding to cloud deployment
```

---

## Step 1: Deploy Infrastructure (IaC)

Deploy infrastructure stacks in dependency order:

```bash
# Example for CDK:
cdk deploy STACK_NAME_1 --require-approval never
cdk deploy STACK_NAME_2 --require-approval never
# ... continue in dependency order

# Example for Terraform:
terraform plan -out=tfplan
terraform apply tfplan
```

**Rules**:
- Deploy stacks in dependency order (VPC → Security → Database → Compute → Load Balancer)
- Wait for each stack to complete before deploying dependent stacks
- If a stack fails: STOP. Diagnose. Fix at Stage 3 if possible. Retry.
- Log every stack deployment result

---

## Step 2: Deploy Application

### For Container Path (ECS/Fargate)

```bash
# Tag and push the SAME image that passed Stage 3
docker tag IMAGE:local ACCOUNT.dkr.ecr.REGION.amazonaws.com/REPO:TAG
docker push ACCOUNT.dkr.ecr.REGION.amazonaws.com/REPO:TAG

# Force new deployment (if service already exists)
aws ecs update-service --cluster CLUSTER --service SERVICE --force-new-deployment

# Wait for stabilization
aws ecs wait services-stable --cluster CLUSTER --services SERVICE
```

**Image Identity Rule**: The image pushed to ECR MUST be the exact same image tested at Stage 3. Tag with git SHA or unique identifier. Never rebuild between stages.

### For On-Prem Path (CodeDeploy)

```bash
# Create deployment bundle (same artifacts tested at Stage 3)
# Push to S3 or use GitHub/CodeCommit source
aws deploy create-deployment \
  --application-name APP_NAME \
  --deployment-group-name GROUP_NAME \
  --s3-location bucket=BUCKET,bundleType=zip,key=KEY \
  --file-exists-behavior OVERWRITE

# Wait for deployment completion
aws deploy wait deployment-successful --deployment-id DEPLOYMENT_ID
```

### For Frontend (Amplify/S3)

```bash
# Build frontend (same commands as Stage 3)
dotnet publish FRONTEND_PROJECT -c Release -o publish/ui

# For S3: sync to bucket
aws s3 sync publish/ui/wwwroot s3://BUCKET --delete

# For Amplify: push to branch (triggers auto-build)
git push origin BRANCH
```

---

## Step 3: Verify Health

After deployment completes, verify the service is responding:

```bash
# Wait for target to be healthy
for i in $(seq 1 30); do
  if curl -sf http://ENDPOINT/health; then
    echo "Healthy after ${i} attempts"
    break
  fi
  sleep 10
done

# Verify response content
curl -s http://ENDPOINT/health | jq .
```

---

## Step 4: Run Full E2E Against Cloud

```bash
# Same test script, cloud endpoint
bash scripts/test-e2e-remote.sh http://CLOUD_ENDPOINT

# Expected: SAME number of tests pass as Stage 3
```

**If tests fail**:
1. STOP — do not iterate on cloud
2. Identify which tests fail and why
3. Determine if the issue is:
   - Cloud config (fix config, retry)
   - Script/code issue (fix at Stage 3 locally, then redeploy)
   - Infrastructure issue (fix IaC, redeploy stack)
4. After fix: re-run full E2E to confirm ALL tests pass

---

## Step 5: Handle Failures

### Deployment Failure Recovery

```markdown
IF deployment fails:
1. Check CloudFormation/ECS/CodeDeploy events for root cause
2. Classify the failure:
   a) Infrastructure issue (IAM, networking, resource limits)
      → Fix IaC, redeploy stack
   b) Application issue (startup crash, health check fail)
      → Fix at Stage 3 first, rebuild, redeploy
   c) Configuration issue (missing env var, wrong secret)
      → Fix config, retry deployment
3. After fix, re-run FULL E2E (not just health check)
4. Document the failure and fix in audit.md
```

### First-Try Expectation

Scripts proven at Stage 3 should succeed first-try on cloud. If they don't, the root cause is one of:
- **Cloud-specific** (IAM permissions, VPC networking, service limits) — fix on cloud, document
- **Missed at Stage 3** (incomplete simulation) — fix locally, improve simulation, redeploy
- **Race condition** (timing, eventual consistency) — add waits/retries in scripts, validate at Stage 3

---

## Step 6: Document Results

Create `aidlc-docs/operations/stage4-deployment-results.md`:

```markdown
# Stage 4: Cloud Deployment — Results

## Infrastructure Deployed
| Stack/Resource | Status | Details |
|----------------|--------|---------|
| [stack 1] | [CREATE_COMPLETE/FAILED] | [notes] |
| [stack 2] | [CREATE_COMPLETE/FAILED] | [notes] |

## Application Deployment
| Path | Method | Status | Endpoint |
|------|--------|--------|----------|
| ECS | ECR push + ECS update | [Success/Failed] | [URL] |
| On-Prem | CodeDeploy | [Success/Failed] | [URL] |

## Health Checks
| Endpoint | Status | Response Time |
|----------|--------|---------------|
| [URL]/health | [200/xxx] | [ms] |

## E2E Results
| Path | Tests Run | Tests Pass | Target |
|------|-----------|------------|--------|
| ECS | [X] | [X] | [ALB URL] |
| On-Prem | [X] | [X] | [EC2 URL] |

## First-Try Success
- ECS path: [Yes/No]
- On-Prem path: [Yes/No]
- If No: [what failed, root cause, fix applied]

## Security Verification
- [ ] No 0.0.0.0/0 in security groups
- [ ] All endpoints restricted to deployer IP
- [ ] Secrets stored in managed service (not plaintext)
- [ ] Test data isolated from application data
```

---

## Step 7: Completion Message

```markdown
## Stage 4: Staging Cloud Deployment — COMPLETE ✅

**Evidence**:
- Infrastructure: [X] stacks deployed successfully
- ECS path: [endpoint] — E2E [X] tests pass
- On-Prem path: [endpoint] — E2E [X] tests pass
- First-try: [Yes for all / details if No]
- Security: All SGs restricted, no 0.0.0.0/0

**Gate Status**: PASSED
- Cloud deployment succeeds
- Full E2E validates against cloud endpoints
- Same test count as Stage 3

**Next Stage**: Stage 5: CI/CD Pipeline [or Stage 6 if pipeline skipped]

Choose:
1. **Request Changes** — specify what needs modification
2. **Continue to Stage 5** — proceed to CI/CD pipeline
```

---

## Key Rules for This Stage

1. **Stage 3 must pass first** — no cloud deployment without local simulation success
2. **Image identity** — same artifact at Stage 3 and Stage 4 (tag with git SHA)
3. **First-try expected** — if it fails, root-cause it, don't just retry
4. **Full E2E, same count** — all tests, matching Stage 3 count
5. **Fix locally when possible** — only fix on cloud if it's cloud-specific
6. **Security non-negotiable** — no 0.0.0.0/0, restricted access, isolated test data
7. **Exhaust automated before manual** — run all automated validation before presenting manual steps
8. **Document failures** — every failure logged with root cause and fix
