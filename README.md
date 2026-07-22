# AI-DLC Workflow Template

An adaptive AI-guided software development lifecycle (AIDLC) workflow for Kiro IDE. This template provides structured steering rules that guide AI through three phases: Inception, Construction, and Operations.

## What is AI-DLC?

AI-DLC is a structured workflow that helps AI assistants guide software development projects through planning, implementation, and deployment. It enforces:

- Progressive validation (no skipping stages)
- Explicit approval gates between phases
- Complete audit trails of all decisions
- Adaptive depth based on project complexity

## The Three Phases

### 🔵 INCEPTION — What to build and why
- Workspace Detection (always)
- Reverse Engineering (brownfield only)
- Requirements Analysis (always, adaptive depth)
- User Stories (conditional)
- Workflow Planning (always)
- Application Design (conditional)
- Units Generation (conditional)

### 🟢 CONSTRUCTION — How to build it
- Functional Design (conditional, per-unit)
- NFR Requirements (conditional, per-unit)
- NFR Design (conditional, per-unit)
- Infrastructure Design (conditional, per-unit)
- Code Generation (always, per-unit)
- Build and Test (always)

### �� OPERATIONS — How to deploy and run it
- Stage 0: Prerequisites & Test Development (always)
- Stage 1: Environment Strategy (always)
- Stage 2: Local Scripts & Validation (always)
- Stage 3: Staging Simulation Locally (always)
- Stage 4: Staging Cloud Deployment (always)
- Stage 5: CI/CD Pipeline (conditional)
- Stage 6: Multi-Environment (conditional)
- Stage 7: Operational Readiness (conditional)

## Usage

1. Clone or use this template for a new project
2. Open the project in Kiro IDE
3. Start your development request — the AI will follow the AIDLC workflow automatically
4. The steering file at `.kiro/steering/aws-aidlc-rules/core-workflow.md` activates the workflow

## Directory Structure

```
.kiro/
├── steering/
│   └── aws-aidlc-rules/
│       └── core-workflow.md          # Main steering rule (entry point)
└── aws-aidlc-rule-details/           # Detailed stage instructions
    ├── common/                       # Shared rules (validation, questions, session)
    ├── inception/                    # Planning & architecture stages
    ├── construction/                 # Design & implementation stages
    ├── operations/                   # Deployment & operations stages
    └── extensions/                   # Optional extensions (security, testing)
```

## Operations Phase: Key Principles

The Operations phase enforces a progression framework with hard gates:

1. **No cloud deployment without local validation**
2. **Never skip progression stages**
3. **Match the target environment** (Docker for containers, matching OS for bare metal)
4. **First-try success required** — scripts must work on first attempt
5. **Exhaust automated steps before manual** — don't send user to Console prematurely
6. **Same artifact at every stage** — image identity, no rebuilds between stages
7. **Full E2E at every stage** — not spot checks, the entire test suite

## Extensions

Optional extensions can be enabled during Requirements Analysis:
- **Security Baseline**: Enforces security best practices throughout
- **Property-Based Testing**: Adds property-based test requirements

## License

MIT
