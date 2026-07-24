# Stage 3: Staging Simulation Locally

## Purpose

Simulate the real deployment architecture locally, matching the actual target environment. This is the critical "catch issues before cloud" stage. Every script that will run remotely is tested here first on a matching environment.

---

## Core Principle: Match the Target

The simulation MUST match the actual deployment target. The purpose is to catch environment-specific issues BEFORE paying the cost of cloud deployment failures.

| Deployment Target | Stage 3 Simulation | Why |
|-------------------|-------------------|-----|
| ECS (container) | Docker container locally | Same container runtime |
| EC2 on-prem (bare metal) | Run on matching local machine (same OS) | Same systemd, packages, users |
| RDS SQL Server | SQL Server container locally | Same SQL dialect, constraints |
| Lambda | Local invoke (SAM/serverless-offline) | Same handler interface |
| Amplify | Run amplify.yml build commands locally | Same build process |
| S3 static hosting | Local web server serving built files | Same file structure |

**CRITICAL**: Docker is a POOR simulation for bare-metal targets. Docker lacks systemd, has minimal packages, and behaves differently from a real OS. For on-prem targets, use a machine with the SAME OS as the target.

---

## Step 1: Set Up Simulation Per Path

### For Container Path (ECS/Kubernetes)

```bash
# Build the image (same as Stage 2)
bash scripts/docker-build.sh

# Run with staging-like configuration
docker run -d --name manserv-staging \
  -p 8085:8080 \
  -e ASPNETCORE_ENVIRONMENT=Staging \
  -e ConnectionStrings__Default="..." \
  IMAGE_NAME:TAG

# Verify internal health (simulates ECS health check)
docker exec manserv-staging curl -f http://localhost:8080/health

# Run FULL E2E against the container
bash scripts/test-e2e-remote.sh http://localhost:8085
```

### For On-Prem Path (EC2/Bare Metal)

Run deploy scripts ON THIS MACHINE (if it matches the target OS):

```bash
# Run lifecycle scripts in order (same as CodeDeploy would)
sudo bash scripts/deploy/before_install.sh
sudo bash scripts/deploy/after_install.sh
sudo bash scripts/deploy/start.sh
sudo bash scripts/deploy/validate.sh

# Run FULL E2E against the local service
bash scripts/test-e2e-remote.sh http://localhost:8080
```

**If this machine doesn't match target**: Use a VM or machine that matches (e.g., Amazon Linux 2023 for EC2 targets).

### For Database (Real Engine)

```bash
# Start real DB engine in container
docker run -d --name sqlserver-test \
  -e 'ACCEPT_EULA=Y' \
  -e 'SA_PASSWORD=TestPassword123!' \
  -p 1433:1433 \
  mcr.microsoft.com/mssql/server:2022-latest

# Run seed/migration against real engine
bash scripts/test-setup.sh  # Creates isolated test DB, seeds data

# Run tests against real engine (not SQLite)
API_BASE_URL=http://localhost:PORT bash scripts/test-e2e-remote.sh http://localhost:PORT

# Cleanup
bash scripts/test-teardown.sh  # Drops test DB
docker stop sqlserver-test && docker rm sqlserver-test
```

---

## Known Pitfalls at Stage 3 (Check BEFORE running simulation)

These issues are commonly discovered at Stage 3. Check for them PROACTIVELY before running `docker compose up` or deploying to simulation:

### 1. Port Conflicts
- macOS uses port 5000 for AirPlay Receiver — remap to 5050 or disable AirPlay
- Check ports: `lsof -i :PORT` before starting containers
- Common conflicts: 80 (nginx/apache), 5000 (AirPlay), 3000 (other node apps), 1433 (local SQL Server)

### 2. Container Binding (0.0.0.0 vs localhost)
- App inside container MUST bind to `0.0.0.0` (all interfaces), NOT `localhost`
- `localhost` inside a container = container's own loopback, unreachable from host
- Fix: Use `ASPNETCORE_URLS=http://+:PORT` in Dockerfile/env, NOT `app.Urls.Add("http://localhost:PORT")`

### 3. ORM Dual-Provider Migration Incompatibility
- **EF Core migrations generated on SQLite have SQLite types** (TEXT, INTEGER)
- **These migrations CANNOT execute on SQL Server** (expects nvarchar, int, datetime2)
- Solutions:
  - (a) Use `Database.EnsureCreated()` for non-dev environments (skips migrations, creates from model)
  - (b) Maintain separate migration assemblies per provider
  - (c) Generate migrations against SQL Server provider (not SQLite)
- This MUST be addressed at Code Generation time — if discovered here, go back and fix the design

### 4. Config Leaking Between Environments
- Base `appsettings.json` should have production-safe defaults
- Dev-only settings (e.g., `UseSqlite: true`) belong in `appsettings.Development.json` ONLY
- Container env vars override app config — verify ALL required vars are set in docker-compose

### 5. Client→API URL Mismatch in Frontend Apps
- Blazor WASM / React / Vue apps compiled with a hardcoded API base URL won't work in Docker
- Fix: Use relative URLs + nginx reverse proxy, OR inject API URL via environment at build time
- E2E tests must validate the CLIENT→PROXY→API path, not just API-direct

### 6. Seed Data Approach
- `HasData()` / static seed in migrations may fail on provider switch (type serialization differences)
- Prefer programmatic seeding: check if table empty → insert seed data in `Program.cs` / startup
- Programmatic seeding works identically on SQLite AND SQL Server

---

## Step 2: Validate First-Try Success

**CRITICAL RULE**: Every script must pass on FIRST attempt.

If a script fails:
1. STOP — do not retry immediately
2. Identify root cause from error output
3. Fix the script
4. Re-run from the BEGINNING (all scripts in order)
5. Must pass the ENTIRE sequence on a single run without manual intervention

This ensures that when these scripts run on cloud (Stage 4), they will succeed first-try.

---

## Step 3: Full E2E Validation

The same full E2E test suite that passed at Stage 2 must pass here:

```bash
# Same test script, different target URL
bash scripts/test-e2e-remote.sh http://SIMULATION_ENDPOINT

# Expected: SAME number of tests pass as Stage 2
# If fewer pass: there's an environment-specific issue to fix HERE, not on cloud
```

**Test count must match**: If Stage 2 had 455 tests pass, Stage 3 must also have 455 pass. Any discrepancy indicates an environment-specific bug.

---

## Step 4: Validate Seed Data on Real Engine

If the deployment uses a different DB engine than Stage 2:

```markdown
## Seed Data Validation
- [ ] Real DB engine container starts
- [ ] test-setup.sh creates isolated test database
- [ ] Seed data loads without errors on real engine
- [ ] E2E tests pass with real engine (not SQLite)
- [ ] test-teardown.sh cleans up
```

---

## Step 5: Document Results

Create `aidlc-docs/operations/stage3-simulation-results.md`:

```markdown
# Stage 3: Staging Simulation — Results

## Path: [ECS / On-Prem / etc.]

### Simulation Setup
- Target: [what was simulated]
- Machine: [where simulation ran]
- OS Match: [Yes/No — must be Yes for on-prem]

### Script Execution (in order)
- [ ] [script 1] — exit 0, [evidence]
- [ ] [script 2] — exit 0, [evidence]
- [ ] [script N] — exit 0, [evidence]

### First-Try Success
- All scripts passed on first attempt: [Yes/No]
- If No: [what failed, what was fixed, re-run passed]

### E2E Results
- Tests run: [X]
- Tests pass: [X] (must match Stage 2 count)
- Target URL: [endpoint tested]
- Execution time: [X]s

### Health Check
- Endpoint: [URL]
- Response: [status code + body]
```

---

## Step 6: Per-Path Gate

Each deployment path is gated independently:

```markdown
### ECS Path Gate
- [ ] Docker image builds
- [ ] Container starts and responds to health check
- [ ] Internal health check passes (docker exec curl)
- [ ] Full E2E passes against container ([X] tests)
- [ ] First-try success (no retries needed)

### On-Prem Path Gate
- [ ] before_install.sh passes on matching machine
- [ ] after_install.sh passes (runtime installed, binaries copied)
- [ ] start.sh passes (service starts, health responds)
- [ ] validate.sh passes (health check OK)
- [ ] Full E2E passes against local service ([X] tests)
- [ ] First-try success (no retries needed)
```

---

## Step 7: Completion Message

```markdown
## Stage 3: Staging Simulation — COMPLETE ✅

**Evidence**:
- [Path 1]: All scripts pass first-try, E2E [X] tests pass
- [Path 2]: All scripts pass first-try, E2E [X] tests pass
- Seed data: Works on real engine ([engine name])

**Gate Status**: PASSED (all paths)
- First-try success on matching environments
- Full E2E validates business logic on real architecture
- Ready for cloud deployment

**Next Stage**: Stage 4: Staging Cloud Deployment

Choose:
1. **Request Changes** — specify what needs modification
2. **Continue to Stage 4** — proceed to cloud deployment
```

---

## Key Rules for This Stage

1. **Match the target** — Docker for containers, matching OS for bare metal, real DB engine for database
2. **First-try success required** — if scripts need fixes, fix and re-run the ENTIRE sequence
3. **Full E2E, same count** — not a subset, not fewer tests than Stage 2
4. **Independent paths** — each target has its own simulation and gate
5. **Fix HERE, not on cloud** — any issue caught here saves 10x debugging time on cloud
6. **Document everything** — exact commands, exact outputs, exact test counts
7. **DO NOT use Docker to simulate bare metal** — it will miss systemd issues, package issues, user context issues

---

## MANDATORY: Stage 3 Must Be Meaningfully Different From Stage 2

**CRITICAL**: Stage 3 (Staging Simulation) MUST test something that Stage 2 (Local Scripts) did NOT test. If Stage 3 is identical to Stage 2 — same server, same commands, same environment — it provides zero additional validation and is a waste.

**The Differentiation Rule**:
| Deployment Target | Stage 2 Tests | Stage 3 MUST Additionally Test |
|-------------------|---------------|-------------------------------|
| S3 + CloudFront (static) | Local file serves via `python -m http.server` | HTTPS behavior, cache headers, CORS, MIME types, CDN simulation (e.g., nginx with caching proxy) |
| ECS (container) | Docker image builds | Container runs with production-like env vars, internal health check passes, E2E against container port |
| EC2 on-prem | Script syntax valid | Full deploy lifecycle on matching machine (systemd, users, permissions, nginx) |
| RDS (database) | SQLite/in-memory tests pass | Real DB engine container, migrations run, seed data loads, E2E against real engine |
| Lambda/Serverless | Code compiles | Local invoke with SAM/serverless-offline, actual handler execution |

**If Stage 3 cannot be meaningfully differentiated from Stage 2** (e.g., truly static file with no cloud target), then the deployment plan from Stage 1 is likely wrong — revisit Stage 1 and define what cloud target actually makes the simulation meaningful.

**Anti-pattern detected in testing**: Running `python3 -m http.server` for Stage 3 when Stage 2 already tested `python3 -m http.server` is NOT a valid simulation. Stage 3 must simulate the ACTUAL deployment target's behavior (S3 serving behavior, CloudFront caching, HTTPS termination, etc.).
