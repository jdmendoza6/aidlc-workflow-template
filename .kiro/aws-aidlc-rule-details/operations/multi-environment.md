# Stage 6: Multi-Environment

## Purpose

Deploy the same application to additional environments (UAT, Pre-Prod, Production), verify branch→environment mapping, test approval gates, and confirm environment isolation. Each environment must independently pass the full E2E test suite.

---

## When to Execute

**Execute IF**:
- Multiple environments defined in Stage 1 (staging + UAT + production)
- Environment segregation needs verification
- Approval gates need testing

**Skip IF**:
- Single environment is sufficient
- Multi-env is deferred to later

---

## Step 1: Verify Branch→Environment Mapping

Confirm the branch strategy from Stage 1:

```markdown
| Branch | Environment | Pipeline | Approval Gate |
|--------|-------------|----------|---------------|
| staging | Staging | Auto-deploy on push | None |
| uat | UAT | Auto-deploy on push | Manual approval |
| main | Production | Auto-deploy on push | Manual approval |
```

Verify branches exist:
```bash
git branch -r | grep -E "staging|uat|main"
```

---

## Step 2: Deploy Additional Environment Infrastructure

```bash
# Deploy UAT infrastructure (same stacks, different params)
cdk deploy --all --context env=uat --context githubBranch=uat --require-approval never

# Verify stacks created
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE | grep uat
```

**Key Differences Per Environment**:
| Concern | Staging | UAT | Production |
|---------|---------|-----|------------|
| Stack naming | `app-staging-*` | `app-uat-*` | `app-prod-*` |
| Branch trigger | `staging` | `uat` | `main` |
| Approval gate | None | Manual | Manual |
| DB instance | Shared RDS (different DB name) | Dedicated or shared | Dedicated |
| Config source | `/app/staging/config/*` | `/app/uat/config/*` | `/app/prod/config/*` |
| Security | Deployer IP only | Deployer IP + QA IPs | VPN/restricted |

---

## Step 3: Verify Environment Isolation

Each environment must be completely isolated:

```markdown
## Isolation Verification Checklist
- [ ] Separate VPC (or separate subnets)
- [ ] Separate security groups (no cross-env rules)
- [ ] Separate database (or separate DB name on shared instance)
- [ ] Separate config values (Parameter Store paths are env-scoped)
- [ ] Separate secrets (Secrets Manager entries are env-scoped)
- [ ] Separate ALB/endpoints (different DNS names)
- [ ] Separate ECR tags or separate repositories
- [ ] Deploying to one env does NOT affect another
```

---

## Step 4: Test Approval Gates

For environments that require manual approval (UAT, Production):

```bash
# Push to UAT branch → pipeline triggers
git checkout uat
git merge staging
git push origin uat

# Verify pipeline pauses at approval stage
aws codepipeline get-pipeline-state --name UAT_PIPELINE_NAME
# Expected: "Approval" stage in "InProgress" status

# Approve manually
aws codepipeline put-approval-result \
  --pipeline-name UAT_PIPELINE_NAME \
  --stage-name Approval \
  --action-name ManualApproval \
  --result summary="Approved for UAT",status=Approved \
  --token TOKEN

# Verify deployment continues after approval
aws codepipeline get-pipeline-state --name UAT_PIPELINE_NAME
# Expected: "Deploy" stage completes
```

---

## Step 5: Full E2E Per Environment

Run the full E2E suite against each environment independently:

```bash
# Staging (should already be passing from Stage 4)
bash scripts/test-e2e-remote.sh http://STAGING_ENDPOINT

# UAT
bash scripts/test-e2e-remote.sh http://UAT_ENDPOINT

# Production (if deployed)
bash scripts/test-e2e-remote.sh http://PROD_ENDPOINT
```

**Same test count required per environment**: If staging has 455 tests, UAT must also have 455 tests pass.

---

## Step 6: Verify Repeatability

Push a change through the full pipeline to verify the process is repeatable:

```bash
# Make a change on staging
git checkout staging
echo "// test change" >> src/app.ts
git commit -am "test: verify pipeline repeatability"
git push origin staging

# Verify: staging pipeline runs → deploys → E2E passes
# Then: merge to uat → approval gate → approve → deploys → E2E passes
```

---

## Step 7: Document Results

```markdown
# Stage 6: Multi-Environment — Results

## Environments Deployed
| Environment | Endpoint | Status | E2E Result |
|-------------|----------|--------|------------|
| Staging | [URL] | ✅ Healthy | [X]/[X] pass |
| UAT | [URL] | ✅ Healthy | [X]/[X] pass |
| Production | [URL] | [Status] | [X]/[X] pass |

## Branch→Environment Mapping
| Branch | Environment | Verified |
|--------|-------------|----------|
| staging | Staging | ✅ Push → auto-deploy |
| uat | UAT | ✅ Push → approval → deploy |
| main | Production | [✅/❌] |

## Approval Gates
| Environment | Gate Type | Tested | Result |
|-------------|-----------|--------|--------|
| UAT | Manual approval | ✅ | Pipeline pauses, resumes after approval |
| Production | Manual approval | [✅/❌] | [Result] |

## Isolation
- [ ] Each env has separate infrastructure
- [ ] Config values are env-scoped
- [ ] Deploying to staging does NOT affect UAT
- [ ] Deploying to UAT does NOT affect Production
- [ ] Test data is isolated per environment

## Repeatability
- Tested push-through-pipeline: [Yes/No]
- Same process works N times: [Yes/No]
```

---

## Step 8: Completion Message

```markdown
## Stage 6: Multi-Environment — COMPLETE ✅

**Evidence**:
- Environments deployed: [list]
- E2E per environment: [X] tests pass each
- Branch mapping: [branch→env verified]
- Approval gates: Working (pipeline pauses, resumes on approval)
- Isolation: Confirmed (separate infra, config, data)
- Repeatability: Push triggers consistent results

**Gate Status**: PASSED
- All environments independently pass E2E
- Branch→environment mapping works
- Approval gates function correctly
- Environment isolation confirmed

**Next Stage**: Stage 7: Operational Readiness [or Complete]

Choose:
1. **Request Changes** — specify what needs modification
2. **Continue to Stage 7** — proceed to operational readiness
3. **Complete** — operations phase done (if Stage 7 not needed)
```

---

## Key Rules for This Stage

1. **Independent E2E per environment** — each env passes the FULL test suite independently
2. **Same test count** — if staging has 455 tests, UAT has 455, Prod has 455
3. **Isolation verified** — deploying to one env cannot affect another
4. **Approval gates tested** — not just configured, actually tested (pause + approve + resume)
5. **Branch mapping proven** — push to branch X deploys to environment Y
6. **Repeatability** — process works the same way every time, not just the first time
