# Stage 0: Prerequisites & Test Development

## Purpose

Ensure the application has a working automated test suite, organized project structure, and defined seed data strategy before any deployment work begins. This stage uses **auto-discovery** — the AI scans the project first and presents findings for validation, rather than asking questions about what can be detected.

---

## Step 1: Auto-Discovery (AI scans project)

The AI scans the workspace and reports findings. Detection targets:

| What to Detect | How to Detect | Example Finding |
|----------------|---------------|-----------------|
| Test projects | Scan for `*.Tests.csproj`, `*test*` dirs, `jest.config`, `pytest.ini`, `pom.xml` with test scope | "Found: App.Tests (xUnit, 438 tests)" |
| Test types | Analyze test file names, namespaces, attributes | "Unit (243), Integration (141), UI (49), Property-based (5)" |
| DB type | Connection strings, ORM config, `.db` files, docker-compose | "SQLite (dev), SQL Server provider registered" |
| Dockerfile | Check root and subdirs | "Found: Dockerfile (multi-stage, .NET 8)" |
| CI/CD artifacts | buildspec.yml, Jenkinsfile, .github/workflows, .gitlab-ci.yml | "Found: buildspec.yml, appspec-onprem.yml" |
| Project structure | Directory layout analysis | "API + Frontend + Tests at root, scripts/ exists" |
| Seed data | .db files, seed scripts, migration folders, Seed() methods | "Found: manserv.db, sql files/ directory" |
| Build system | Makefile, package.json scripts, build.sh, .sln, pom.xml | ".NET solution, 3 projects" |
| Dependencies | Package manifests (*.csproj, package.json, requirements.txt) | "17 NuGet packages, 2 npm packages" |

---

## Step 2: Present Findings for Validation

Present auto-discovery results in question format:

```markdown
## Stage 0: Project Discovery Validation

Based on scanning your project, I've detected the following. Please validate:

### Test Suite
Detected: [description of test projects and counts]
- [breakdown by type]

Is this the complete test suite?
A) Complete — these cover all critical paths
B) Needs expansion — additional test types needed (specify which)
C) Other (please describe)

[Answer]:

### Project Structure
Detected layout:
- [current structure description]

Recommended: [restructuring suggestion if applicable]
A) Reorganize now (before proceeding to Stage 1)
B) Keep current structure (document as-is, reorganize later)
C) Other (please describe)

[Answer]:

### Database Strategy
Detected: [DB findings]

A) Correct — [detected strategy description]
B) Different plan (please describe)

[Answer]:

### Seed Data
Detected: [seed data findings]

A) This is the seed data strategy — use file-based seeding
B) Need migration-based seeding instead
C) Other (please describe)

[Answer]:

### Containerization
Detected: [Dockerfile/container findings]

A) Correct — ready for containerized deployment
B) Needs modification (please describe)
C) No containerization needed

[Answer]:
```

---

## Step 3: Handle Gaps

If auto-discovery finds critical gaps, present options:

### No Tests Found
```markdown
### Test Suite
Detected: NO automated test projects found.

This is a PREREQUISITE for the Operations phase. Options:
A) Develop comprehensive test suite now (unit + integration + E2E)
B) Develop minimal test suite (smoke tests + critical path E2E only)
C) Tests exist elsewhere (please point me to them)
D) Skip tests (NOT RECOMMENDED — weakens all subsequent stages)

[Answer]:
```

### No Build System
```markdown
### Build System
Detected: No build scripts or CI configuration found.

A) Create build scripts now (build.sh, test.sh, ci-local.sh)
B) Use IDE/manual build only (NOT RECOMMENDED for CI/CD)
C) Build system exists elsewhere (please describe)

[Answer]:
```

---

## Step 4: Validate Test Suite Passes

After validation answers are collected:

1. Run the detected test suite: `dotnet test` / `npm test` / `pytest` / etc.
2. Verify 100% pass rate
3. If tests fail: fix them BEFORE proceeding (this is Stage 0 — entry gate)
4. Document test count and pass rate

**Gate Criteria**:
- [ ] Automated test suite exists
- [ ] All tests pass (100% pass rate)
- [ ] Project structure documented
- [ ] Seed data strategy defined
- [ ] Build system works (compile/build succeeds)

---

## Step 5: Document Findings

Create `aidlc-docs/operations/prerequisites.md`:

```markdown
# Stage 0: Prerequisites — Validated

## Test Suite
- Framework: [xUnit/Jest/pytest/etc.]
- Total tests: [count]
- Breakdown: Unit ([n]), Integration ([n]), E2E ([n]), Property-based ([n])
- Pass rate: 100% (verified [date])

## Project Structure
- Layout: [description]
- Decision: [keep as-is / reorganize]

## Database Strategy
- Dev: [SQLite/H2/etc.]
- Staging/Prod: [SQL Server/PostgreSQL/etc.]
- Migration: [EF Core/Flyway/manual SQL/etc.]

## Seed Data
- Strategy: [file-based/migration-based/programmatic]
- Location: [path to seed data]
- Coverage: [what data is seeded]

## Build System
- Tool: [dotnet/npm/maven/etc.]
- Entry point: [build.sh/Makefile/package.json]
- Verified: [date]
```

---

## Step 6: Completion Message

```markdown
## Stage 0: Prerequisites — COMPLETE ✅

**Auto-Discovery Results**:
- Test suite: [X] tests pass (100%)
- Project structure: [documented/reorganized]
- Seed data: [strategy defined]
- Build system: [verified working]

**Gate Status**: PASSED
- All prerequisites met for operations progression

**Next Stage**: Stage 1: Environment Strategy

Choose:
1. **Request Changes** — specify what needs modification
2. **Continue to Stage 1** — proceed to environment strategy
```

---

## Key Rules for This Stage

1. **Auto-discover FIRST, ask SECOND** — don't ask what the AI can detect
2. **100% test pass rate required** — fix failures before proceeding
3. **Structure reorganization is itself a change** — if reorganizing, verify tests still pass after
4. **This stage gates ALL subsequent stages** — no progression without passing Stage 0
5. **Document everything** — findings, decisions, and validation results
