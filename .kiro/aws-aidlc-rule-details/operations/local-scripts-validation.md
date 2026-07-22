# Stage 2: Local Scripts & Validation

## Purpose

Create or validate all build/test/run/deploy scripts and confirm everything works locally with the simplest possible infrastructure (e.g., SQLite, localhost). This is Framework Stage 1 — the foundation all other stages build upon.

**Key Principle**: Auto-discover existing scripts first. If build.sh, test.sh, deploy scripts, Dockerfile, etc. already exist, present them for validation ("I found these scripts, are they the ones to use?") — don't ask "do you have build scripts?" when the AI can scan the project directory.

---

## Step 1: Auto-Discover Existing Scripts

Before creating anything, scan the project for existing operational scripts:

| What to Scan | Where to Look | Example Finding |
|--------------|---------------|-----------------|
| Build scripts | `scripts/build.sh`, `Makefile`, `build.sh`, `package.json` scripts | "Found: scripts/build.sh (compiles API + Frontend)" |
| Test scripts | `scripts/test.sh`, `test.sh`, `npm test` in package.json | "Found: scripts/test.sh (runs dotnet test)" |
| Docker scripts | `scripts/docker-build.sh`, `docker-compose.yml`, `Dockerfile` | "Found: scripts/docker-build.sh, Dockerfile" |
| Deploy scripts | `scripts/deploy/*.sh`, `deploy/`, lifecycle hooks | "Found: scripts/deploy/ (before_install, after_install, start, validate)" |
| CI orchestrator | `scripts/ci-local.sh`, `ci.sh`, `Makefile all` | "Found: scripts/ci-local.sh (orchestrates full pipeline)" |
| E2E scripts | `scripts/e2e-test.sh`, `scripts/test-e2e-remote.sh`, `e2e-*.sh` | "Found: scripts/test-e2e-remote.sh (accepts URL param)" |
| Clean scripts | `scripts/clean.sh`, `clean.sh` | "Found: scripts/clean.sh" |
| Pipeline artifacts | `buildspec.yml`, `appspec*.yml`, `taskdef*.json`, `amplify.yml` | "Found: buildspec.yml, appspec-onprem.yml, taskdef-template.json" |

**Present findings for validation**:

```markdown
## Stage 2: Existing Scripts Discovery

I scanned your project and found the following operational scripts:

### Build & Test
- `scripts/build.sh` — compiles all projects
- `scripts/test.sh` — runs full test suite
- `scripts/clean.sh` — removes build artifacts

### Container
- `Dockerfile` — multi-stage .NET 8 build
- `scripts/docker-build.sh` — builds Docker image
- `scripts/docker-run.sh` — runs container locally

### Deploy (On-Prem lifecycle)
- `scripts/deploy/before_install.sh` — pre-deployment cleanup
- `scripts/deploy/after_install.sh` — install runtime + binaries
- `scripts/deploy/start.sh` — start systemd service
- `scripts/deploy/validate.sh` — health check

### Orchestrator & E2E
- `scripts/ci-local.sh` — full local CI pipeline
- `scripts/test-e2e-remote.sh` — E2E against any URL

### Pipeline Artifacts
- `buildspec.yml`, `appspec-onprem.yml`, `appspec-ecs.yaml`
- `taskdef-template.json`, `amplify.yml`

Are these the correct scripts to use for the operations progression?
A) Yes — validate and use these as-is
B) Yes but some need updates (specify which)
C) Need additional scripts (specify what's missing)
D) Start fresh (ignore existing, create new)

[Answer]:
```

**If NO scripts found**:
```markdown
## Stage 2: Existing Scripts Discovery

No operational scripts found in the project. I'll create the required scripts based on the deployment plan from Stage 1.

Proceeding to create scripts...
```

---

## Step 2: Determine Required Scripts (gap analysis)

Based on the deployment plan (Stage 1) AND discovered scripts, identify what's needed vs. what exists:

### Core Scripts (ALWAYS required)
| Script | Purpose | Validates |
|--------|---------|-----------|
| `build.sh` | Compile/build all projects | Code compiles without errors |
| `test.sh` | Run full test suite | All tests pass |
| `clean.sh` | Remove build artifacts | Clean slate for rebuilds |

### Container Scripts (if containerized targets exist)
| Script | Purpose | Validates |
|--------|---------|-----------|
| `docker-build.sh` | Build container image | Dockerfile correct, image builds |
| `docker-run.sh` | Run container locally | Container starts, health responds |

### Deploy Scripts (if on-prem or server targets exist)
| Script | Purpose | Validates |
|--------|---------|-----------|
| `deploy/before_install.sh` | Pre-deployment cleanup | Old artifacts removed safely |
| `deploy/after_install.sh` | Install runtime + copy binaries | App installed correctly |
| `deploy/start.sh` | Start application service | Service runs, health responds |
| `deploy/validate.sh` | Verify deployment succeeded | Health check passes |

### Orchestrator Scripts (ALWAYS required)
| Script | Purpose | Validates |
|--------|---------|-----------|
| `ci-local.sh` | Run full local CI pipeline | Everything works end-to-end |

### E2E Test Scripts (if E2E tests exist)
| Script | Purpose | Validates |
|--------|---------|-----------|
| `e2e-test.sh` | Run E2E tests against local endpoint | Full business logic verified |
| `test-e2e-remote.sh` | Run E2E tests against any endpoint (URL param) | Reusable for all stages |

---

## Step 2: Create or Validate Scripts

For each required script:

1. Check if it already exists (from Stage 0 detection)
2. If exists: validate it runs successfully
3. If missing: create it with minimal implementation
4. All scripts must:
   - Have `#!/bin/bash` shebang
   - Use `set -euo pipefail` (fail-fast)
   - Print clear progress messages
   - Exit 0 on success, non-zero on failure
   - Be idempotent (safe to run multiple times)

---

## Step 3: Validate Each Script Locally

Run each script and verify:

```bash
# Syntax check ALL scripts first
for script in scripts/*.sh scripts/deploy/*.sh; do
  bash -n "$script" || echo "SYNTAX ERROR: $script"
done

# Then execute in order:
bash scripts/clean.sh          # Clean state
bash scripts/build.sh          # Must exit 0, 0 errors
bash scripts/test.sh           # Must exit 0, all tests pass
bash scripts/docker-build.sh   # Must exit 0 (if applicable)
bash scripts/docker-run.sh     # Must exit 0, container starts (if applicable)
# Health check
curl -sf http://localhost:PORT/health  # Must return 200
# E2E
bash scripts/e2e-test.sh       # Must exit 0, all tests pass
# Full orchestrator
bash scripts/ci-local.sh       # Must exit 0 (runs everything above)
```

---

## Step 4: Validate Deploy Scripts (Syntax Only at This Stage)

Deploy scripts are validated for SYNTAX at Stage 2. They are EXECUTED at Stage 3 (on matching machine).

```bash
# Syntax validation only
bash -n scripts/deploy/before_install.sh
bash -n scripts/deploy/after_install.sh
bash -n scripts/deploy/start.sh
bash -n scripts/deploy/validate.sh
```

---

## Step 5: Document Results

Create deployment plan checkboxes in `aidlc-docs/operations/stage2-local-validation.md`:

```markdown
# Stage 2: Local Scripts & Validation — Results

## Script Validation
- [ ] clean.sh — exit 0
- [ ] build.sh — exit 0, 0 errors
- [ ] test.sh — exit 0, [X]/[X] tests pass
- [ ] docker-build.sh — exit 0, image built (if applicable)
- [ ] docker-run.sh — exit 0, container healthy (if applicable)
- [ ] e2e-test.sh — exit 0, [X] tests pass
- [ ] ci-local.sh — exit 0 (full pipeline)
- [ ] deploy/*.sh — syntax valid (bash -n)

## Test Results
- Total tests: [X]
- Pass rate: 100%
- Execution time: [X]s

## Evidence
- Build output: [summary]
- Test output: [summary]
- Health check: [endpoint] → [status code]
```

---

## Step 6: Completion Message

```markdown
## Stage 2: Local Scripts & Validation — COMPLETE ✅

**Evidence**:
- Scripts created/validated: [count]
- Build: 0 errors
- Tests: [X]/[X] pass (100%)
- Docker: Image builds, container healthy (if applicable)
- E2E: [X] tests pass against localhost
- ci-local.sh: Full pipeline passes

**Gate Status**: PASSED
- All scripts work locally
- Full test suite passes
- Health check responds

**Next Stage**: Stage 3: Staging Simulation Locally

Choose:
1. **Request Changes** — specify what needs modification
2. **Continue to Stage 3** — proceed to staging simulation
```

---

## Key Rules for This Stage

1. **Every script must pass** — not "most" scripts, ALL scripts
2. **Full test suite** — run ALL tests, not a subset
3. **Idempotent scripts** — safe to run multiple times without side effects
4. **Deploy scripts: syntax only** — execution happens at Stage 3 on matching machine
5. **ci-local.sh is the final gate** — if this passes, Stage 2 is complete
6. **Document evidence** — exact commands, exact outputs, exact counts
