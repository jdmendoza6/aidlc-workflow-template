# Stage 7: Operational Readiness

## Purpose

Ensure the deployed application is production-ready with monitoring, alerting, runbooks, rollback procedures, and incident response documentation. This stage transitions ownership from "development" to "operations."

---

## When to Execute

**Execute IF**:
- Application is going to production
- Team needs operational documentation
- SLA/uptime requirements exist
- On-call rotation or incident response needed

**Skip IF**:
- Staging/UAT only (not going to production yet)
- Operational readiness deferred explicitly
- Single developer, no ops team

---

## Step 1: Monitoring Setup

### Health Checks
```markdown
- [ ] Application health endpoint exists and returns meaningful status
- [ ] Health check configured in load balancer/container orchestrator
- [ ] Synthetic monitoring (external checks) configured
- [ ] Health includes dependency checks (DB connectivity, external services)
```

### Metrics
```markdown
- [ ] Request latency tracked (p50, p95, p99)
- [ ] Error rate tracked (4xx, 5xx)
- [ ] Throughput tracked (requests/second)
- [ ] Resource utilization tracked (CPU, memory, disk)
- [ ] Custom business metrics (if applicable)
```

### Logging
```markdown
- [ ] Structured logging configured (JSON format preferred)
- [ ] Log aggregation in place (CloudWatch Logs, ELK, etc.)
- [ ] Log retention policy defined
- [ ] Sensitive data excluded from logs (PII, secrets)
- [ ] Request correlation IDs for tracing
```

---

## Step 2: Alerting Configuration

Define alerts based on SLOs:

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| Health check failure | 3 consecutive failures | Critical | Page on-call |
| High error rate | 5xx > 5% for 5 min | High | Page on-call |
| High latency | p99 > 5s for 10 min | Medium | Notify team |
| CPU saturation | CPU > 80% for 15 min | Medium | Auto-scale or notify |
| Disk space | < 20% remaining | High | Notify team |
| Deployment failure | Pipeline fails | Medium | Notify deployer |

```markdown
- [ ] Critical alerts → immediate notification (PagerDuty/phone)
- [ ] High alerts → team channel notification
- [ ] Medium alerts → team channel notification (business hours)
- [ ] Alert routing configured (correct recipients)
- [ ] Alert de-duplication configured (no alert storms)
```

---

## Step 3: Runbooks

Create runbooks for common operational tasks:

### Runbook Template
```markdown
# Runbook: [Task Name]

## When to Use
[Describe the scenario that triggers this runbook]

## Prerequisites
- [ ] [Access required]
- [ ] [Tools needed]

## Steps
1. [Step with exact command]
2. [Step with exact command]
3. [Verification step]

## Rollback
[If this goes wrong, how to undo]

## Escalation
[Who to contact if steps don't resolve the issue]
```

### Required Runbooks
```markdown
- [ ] Runbook: Application restart
- [ ] Runbook: Rollback deployment
- [ ] Runbook: Scale up/down
- [ ] Runbook: Database connection issues
- [ ] Runbook: High memory/CPU troubleshooting
- [ ] Runbook: Log investigation
- [ ] Runbook: Secret rotation
- [ ] Runbook: Certificate renewal (if applicable)
```

---

## Step 4: Rollback Procedures

### Automated Rollback
```markdown
- [ ] ECS: Previous task definition revision can be activated
- [ ] CodeDeploy: Rollback to last successful deployment configured
- [ ] Pipeline: Rollback action available in pipeline
- [ ] Database: Migration rollback scripts exist (if applicable)
```

### Manual Rollback
```markdown
## Rollback Steps
1. Identify the last known good deployment
2. Revert to previous version:
   - ECS: `aws ecs update-service --task-definition PREV_REVISION`
   - CodeDeploy: `aws deploy create-deployment --revision PREV_REVISION`
   - Frontend: Redeploy previous branch commit
3. Verify health check passes
4. Run smoke tests against rolled-back version
5. Notify team of rollback
6. Investigate root cause
```

### Rollback Testing
```markdown
- [ ] Rollback tested in staging (deploy v2, rollback to v1, verify v1 works)
- [ ] Rollback time measured (how long from decision to healthy rollback)
- [ ] Data compatibility verified (v1 can work with v2's data changes, if any)
```

---

## Step 5: Incident Response

### Incident Response Plan
```markdown
## Severity Levels
| Level | Definition | Response Time | Example |
|-------|-----------|---------------|---------|
| SEV-1 | Service down, all users affected | 15 min | Health check failing |
| SEV-2 | Major degradation, many users affected | 30 min | 50% error rate |
| SEV-3 | Minor issue, some users affected | 2 hours | Slow responses |
| SEV-4 | Cosmetic/minor, no user impact | Next business day | Log warnings |

## Response Process
1. Detect (monitoring/alert fires)
2. Acknowledge (on-call acknowledges within SLA)
3. Triage (determine severity, gather context)
4. Mitigate (apply fix or rollback)
5. Communicate (update stakeholders)
6. Resolve (permanent fix deployed)
7. Post-mortem (document root cause + prevention)
```

---

## Step 6: Documentation

Create ops documentation:

```markdown
# Operations Documentation

## Architecture
- [Link to architecture diagram]
- [Link to infrastructure documentation]

## Endpoints
| Environment | URL | Purpose |
|-------------|-----|---------|
| Staging | [URL] | Integration testing |
| UAT | [URL] | User acceptance |
| Production | [URL] | Live traffic |

## Access
- AWS Console: [account info]
- Logs: [CloudWatch log group links]
- Metrics: [Dashboard links]
- Alerts: [Alert configuration links]

## Contacts
- Primary on-call: [team/person]
- Escalation: [team/person]
- Business owner: [team/person]

## Key Decisions
- [Decision 1]: [rationale]
- [Decision 2]: [rationale]
```

---

## Step 7: Completion Message

```markdown
## Stage 7: Operational Readiness — COMPLETE ✅

**Evidence**:
- Monitoring: [health checks, metrics, logging configured]
- Alerting: [X] alerts configured with routing
- Runbooks: [X] runbooks created
- Rollback: Tested in staging (time: [X]s)
- Incident response: Plan documented, severity levels defined

**Gate Status**: PASSED
- Application is operationally ready
- Team can respond to incidents
- Rollback is tested and documented

**Operations Phase — COMPLETE** 🎉

The application has progressed through all stages:
- Stage 0: Prerequisites ✅
- Stage 1: Environment Strategy ✅
- Stage 2: Local Scripts ✅
- Stage 3: Staging Simulation ✅
- Stage 4: Cloud Deployment ✅
- Stage 5: CI/CD Pipeline ✅
- Stage 6: Multi-Environment ✅
- Stage 7: Operational Readiness ✅
```

---

## Key Rules for This Stage

1. **Rollback MUST be tested** — not just documented, actually executed in staging
2. **Alerts must fire** — trigger test alerts to verify routing works
3. **Runbooks are executable** — commands must work, not just be theoretical
4. **Monitoring before traffic** — observability in place before production receives traffic
5. **Incident response is a team agreement** — not just documentation, team must acknowledge
