# TDD Multi-Agent Workflow

This directory contains AI agents that implement a **test-driven development (TDD) workflow** for feature development. The system uses three specialized agents that collaborate via a shared scratchpad file to deliver high-quality, well-tested code.

## System Overview

The TDD workflow consists of three sequential agents, each with a distinct role:

```
┌─────────────────────────────────────────────────────────────────┐
│                     User Requests Feature                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  TDD-Architect: Analyze & Plan                                  │
│  • Read codebase (analysis only)                                │
│  • Identify affected files and dependencies                     │
│  • Extract patterns and conventions                             │
│  • Write findings to scratchpad                                 │
│  ✅ Output: SCRATCHPAD-[ISSUE_ID].md (Architectural Context)    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                    [User Approves]
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  TDD-Planner: Design Tests                                      │
│  • Analyze requirements and acceptance criteria                 │
│  • Design test cases (before implementation)                    │
│  • Plan test batches                                            │
│  • Map criteria to tests                                        │
│  ✅ Output: SCRATCHPAD-[ISSUE_ID].md (Test Design)              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                    [User Approves]
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  TDD-Implementer: Write Tests & Code                            │
│  • Write tests FIRST (using test plan)                          │
│  • Implement minimum code to pass tests (Red-Green)             │
│  • Refactor for clarity and patterns (Blue/Refactor)            │
│  • Achieve coverage targets                                     │
│  ✅ Output: Code changes + PR ready for review                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                    [Tests Pass ✓]
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│      Code Review & Merge (Human Review)                         │
└─────────────────────────────────────────────────────────────────┘
```

## Agent Roles & Responsibilities

### 🏗️ TDD-Architect: Analysis & Planning

**Purpose**: Understand the codebase and plan architectural changes without modifying code.

**Responsibilities**:
- Extract issue ID from git branch name (e.g., `feature/CSS-21342_stripe-spp` → `CSS-21342`)
- Initialize versioned scratchpad (`SCRATCHPAD-CSS-21342.md`)
- Search codebase to identify all affected files
- Map file dependencies and ripple effects
- Extract existing patterns (table-driven tests, fixture builders, mock patterns)
- Document global invariants (e.g., "use `decimal.Decimal` for all currency")
- Generate a context map showing:
  - **Primary Files**: Files that will be modified
  - **Secondary Files**: Related files that may need updates
  - **Patterns**: Existing conventions to follow
  - **Dependency Graph**: How files interact

**Scope**: READ-ONLY codebase analysis + scratchpad writing

**Handoff**: Writes to `Architectural Context` section of scratchpad → Planner reads it

**When to invoke**:
- Complex changes affecting multiple files
- New integrations with external services
- Multi-package features
- When understanding existing patterns is critical

---

### 📋 TDD-Planner: Test Design

**Purpose**: Transform requirements into a comprehensive test plan **before implementation**.

**Responsibilities**:
- Extract issue description and acceptance criteria (from Jira or user input)
- Reference Architect's analysis (if available) or perform focused pattern search
- Break down acceptance criteria into testable behaviors
- Design test cases across unit, integration, and contract testing levels
- Create mapping of acceptance criteria → test names
- Identify test data needs and fixture requirements
- Plan test batches (Red-Green-Refactor cycles)
- Get explicit user approval before handoff

**Scope**: Test planning, design, and documentation only (no implementation)

**Handoff**: Writes to `Test Design` section of scratchpad → Implementer reads it

**Can work standalone**: Yes, for bug fixes and simple changes (≤3 files)

**When to invoke**:
- Before any code is written
- For feature implementation
- Bug fixes requiring test coverage
- Domain logic changes (payment processing, contracts, etc.)

---

### ⚙️ TDD-Implementer: Red-Green-Refactor

**Purpose**: Implement features and code changes following test-first methodology.

**Responsibilities**:
- Read test plan from scratchpad
- Write test cases FIRST (Red phase)
- Write minimum code to make tests pass (Green phase)
- Refactor for clarity and patterns (Blue/Refactor phase)
- Validate all tests pass
- Measure and achieve coverage targets
- Run quality checks (no race conditions, parallel-safe tests)
- Prepare PR for human review

**Scope**: Code changes (tests and implementation)

**Handoff**: Receives test plan from scratchpad → Writes code changes

**When to invoke**:
- After Planner has approved test design
- Or directly if Planner already executed

---

## Scratchpad: Agent Communication Protocol

All agents communicate via a shared **versioned scratchpad file** that acts as the "source of truth":

- **File naming**: `SCRATCHPAD-[ISSUE_ID].md` (e.g., `SCRATCHPAD-CSS-21342.md`)
- **Fallback** (no issue ID): `SCRATCHPAD-[YYYYMMDD-HHMMSS].md` (timestamp-based)
- **Location**: `.github/skills/scratchpad/`
- **Not committed**: Scratchpad files are in `.gitignore` (session-specific)

### Scratchpad Sections

```markdown
# SCRATCHPAD: CSS-XXXX Issue Title

## Architectural Context
(Written by: Architect)
- Global Invariants
- Primary Files
- Secondary Files
- Patterns
- Dependency Graph

## Test Design
(Written by: Planner)
- Acceptance Criteria & Test Mapping
- Unit Tests
- Integration Tests
- Error/Edge Cases
- Test Data Requirements

## Implementation Progress
(Written by: Implementer)
- Batch Status (RED/GREEN/BLUE)
- Coverage Metrics
- Completed Tests
- Known Issues

## Approval Log
(Written by: All agents on approval)
- Timestamp: [ISO 8601]
- Agent: [Architect|Planner|Implementer]
- Status: [Approved|Revisions Requested|Blocked]

## Roadblocks & Decisions
(Written by: Any agent needing help)
- Issue: [Description]
- Impact: [What blocks progress]
- Resolution: [How resolved or pending user input]
```

### How Scratchpad Is Used

1. **Architect** initializes → writes Architectural Context
2. **User approves** → approves Architect's analysis
3. **Planner** reads Architect section → writes Test Design
4. **User approves** → approves test plan
5. **Implementer** reads Test Design → writes Implementation Progress
6. **Handoff happens via scratchpad** (agents don't see chat messages from other agents)

---

## Workflow Example: CSS-21342 (Stripe Payment Processing)

### Starting Point
- **Jira Issue**: CSS-21342 "Add Stripe SCA Payment Processing"
- **User**: "Implement feature CSS-21342"
- **Current Branch**: `feature/CSS-21342_stripe-spp`

### Step 1: Architect Analysis (5-10 min)
```
User: "Architect, analyze issue CSS-21342"

Architect:
1. Extracts issue ID: CSS-21342
2. Initializes: SCRATCHPAD-CSS-21342.md
3. Searches codebase:
   - Finds: internal/payment/gateway.go, internal/price/calculator.go
   - Identifies patterns: Table-driven tests, decimal usage
   - Notes: External service (Stripe) needs mocking
4. Writes to Architectural Context section
5. Presents context map to user for approval

User approves ✅
```

### Step 2: Test Planning (15-30 min)
```
Planner:
1. Reads Architectural Context from scratchpad
2. Extracts acceptance criteria from CSS-21342:
   - AC1: Accept valid SCA payments
   - AC2: Reject invalid/expired cards
   - AC3: Audit trail for all payments
3. Designs tests:
   - Unit: Payment amount validation
   - Unit: Decimal precision (currency)
   - Integration: Stripe API mocking
   - Integration: Database transaction rollback
   - Error: Network timeout handling
4. Maps AC → tests
5. Proposes 3 test batches
6. Writes to Test Design section
7. Presents plan to user for approval

User approves ✅
```

### Step 3: Implementation (1-2 hours)
```
Implementer:
1. Reads Test Design from scratchpad
2. Batch #1 (Core Logic):
   - RED: Write TestValidatePayment, TestRejectExpiredCard
   - GREEN: Implement minimum payment validation
   - BLUE: Refactor into validators package
3. Batch #2 (Integration):
   - RED: Write TestProcessPaymentWithStripe
   - GREEN: Mock Stripe client, implement basic flow
   - BLUE: Add retry logic
4. Batch #3 (Error Handling):
   - RED: Write TestPaymentTimeout, TestAuditTrail
   - GREEN: Implement error cases
5. Verifies: all tests pass, coverage ≥90%
6. Runs: go test -race (no race conditions)
7. Prepares code for PR review

Ready for merge ✅
```

---

## How to Use the Agents

### Invoking Architect
```
User: "Architect, analyze CSS-21342 and identify all affected files"
```
- Architect reads Jira issue
- Searches codebase for affected files
- Identifies patterns and dependencies
- Writes findings to scratchpad
- Presents context map for approval

### Invoking Planner
```
User: "Planner, design tests for CSS-21342"
```
- Planner reads scratchpad (Architect's analysis)
- OR independently analyzes if no Architect ran
- Designs test cases and test batches
- Writes test plan to scratchpad
- Presents plan for approval

### Invoking Implementer
```
User: "Implementer, implement CSS-21342 using the test plan"
```
- Implementer reads test plan from scratchpad
- Writes tests first (RED phase)
- Implements code to pass tests (GREEN phase)
- Refactors for clarity (BLUE phase)
- Verifies all tests pass and coverage meets target

---

## Key Interaction Patterns

### ✅ Correct Workflow

1. **Sequential**: Architect → Planner → Implementer (one at a time)
2. **Scratchpad-based**: Each agent reads what previous agent wrote
3. **User approval gates**: User approves Architect, then Planner (gates Implementer start)
4. **No code changes until Planner approves**: Architecture analysis → test design → implementation
5. **Error recovery**: If Implementer finds issue, escalates to Planner (documented in scratchpad)

### ❌ Anti-Patterns to Avoid

- ❌ Skipping Planner: Don't go directly from Architect to Implementer without test plan
- ❌ Modifying code in Architect phase: Architect is read-only analysis
- ❌ Chat-based handoff: Don't expect agents to see each other's chat messages—they only read the scratchpad
- ❌ Multiple agents on same issue simultaneously: Can cause scratchpad conflicts for one issue (but parallel agents on different issues are safe via versioning)
- ❌ No approval gates: Always get user approval before proceeding to next agent

---

## Parallel Sessions & Versioning

Each issue gets its own scratchpad file, enabling multiple concurrent development efforts:

```
Session 1: CSS-21342 → SCRATCHPAD-CSS-21342.md
Session 2: CSS-21343 → SCRATCHPAD-CSS-21343.md
Session 3: CSS-21340 → SCRATCHPAD-CSS-21340.md
```

**How versioning works**:
1. Architect extracts issue ID from branch name: `feature/CSS-21342_stripe-spp` → `CSS-21342`
2. Initializes versioned scratchpad: `SCRATCHPAD-CSS-21342.md`
3. Planner and Implementer read/write to same versioned file
4. When switching issues, agents auto-detect new issue ID and use correct scratchpad
5. No conflicts—each issue has isolated session file

**If issue ID not detectable**: Uses timestamp-based fallback `SCRATCHPAD-20260304-103000.md`

---

## Configuration & Setup

### Agent Definitions
- [TDD-Architect.agent.md](./TDD-Architect.agent.md) — Analysis and planning
- [TDD-Planner.agent.md](./TDD-Planner.agent.md) — Test design
- [TDD-Implementer.agent.md](./TDD-Implementer.agent.md) — Code implementation

### Supporting Files
- [SKILLS.md](../skills/scratchpad/SKILLS.md) — Scratchpad read/write operations
- [SCRATCHPAD_SCHEMA.md](../skills/scratchpad/SCRATCHPAD_SCHEMA.md) — Required scratchpad structure
- [SCRATCHPAD_EXAMPLE.md](../skills/scratchpad/SCRATCHPAD_EXAMPLE.md) — Real-world example

### Git Configuration
- Scratchpad files are in `.gitignore` (ephemeral session files)
- Do NOT commit `SCRATCHPAD-*.md` files

---

## Best Practices

### For Users

1. **Provide clear context**: Include Jira issue number or detailed description
2. **Approve at each gate**: Don't skip user approvals—they exist to catch issues early
3. **Provide feedback**: If Architect/Planner outputs need revision, give specific feedback
4. **Don't modify midstream**: Don't manually edit code while agents are working
5. **Let agents complete phases**: Don't interrupt between RED/GREEN/BLUE cycles

### For Agents

1. **Communication is via scratchpad only**: Don't assume other agents saw chat messages
2. **Respect role boundaries**: Architect doesn't modify code, Planner doesn't implement
3. **Document decisions**: When making choices, explain in scratchpad.Roadblocks
4. **Update approval timestamps**: Record when user approved your work
5. **Escalate blockers**: Use scratchpad.Roadblocks for issues needing user input

### For Integration with CI/CD

1. **Test before PR**: Implementer runs `make test` and verifies coverage
2. **Quality gates**: Use `go test -race` to catch concurrency issues
3. **Flakiness checks**: Run tests multiple times (`go test -count=10`)
4. **Coverage validation**: Ensure coverage meets thresholds per module type

---

## Error Recovery & Escalation

If any agent encounters a blocker:

1. **Document in scratchpad**: Add to `Roadblocks & Decisions` section
2. **Describe the issue**: What blocked progress?
3. **Propose options**: What are the alternatives?
4. **Wait for user input**: Agent pauses and asks user for direction

Examples of blockers:
- Jira inaccessible (Planner)
- Code pattern doesn't match assumptions (Implementer)
- Requirements too vague to test (Planner)
- Testability issues require refactoring first (Implementer)

---

## Definitions

| Term | Definition |
|------|-----------|
| **Scratchpad** | Shared text file (`SCRATCHPAD-[ISSUE_ID].md`) for agent-to-agent communication |
| **Versioning** | Each issue gets its own scratchpad file (e.g., CSS-21342 → separate from CSS-21343) |
| **Handoff** | Agent completes work, writes to scratchpad, next agent reads it |
| **Approval Gate** | User reviews and approves agent output before next agent runs |
| **RED phase** | Write failing test (nothing implemented yet) |
| **GREEN phase** | Write minimum code to make test pass |
| **BLUE/Refactor phase** | Improve code quality, extract patterns, reduce duplication |
| **Test Batch** | Group of tests executed together (e.g., Batch #1 unit tests) |
| **Invariant** | Project constraint (e.g., "all currency must use `decimal.Decimal`") |

---

## Questions & Support

- **How do I invoke an agent?** See "How to Use the Agents" section above
- **What if Jira is inaccessible?** Planner will ask you to provide requirements as text
- **Can agents run in parallel?** No—one issue per sequential workflow (prevents scratchpad conflicts)
- **What if I need to change the test plan?** Request revision during Planner's approval gate
- **How do I know when an agent is done?** They'll present output and ask for approval
- **Can I skip the Planner?** Only for trivial bug fixes; complex features need test design first

---

Last updated: March 4, 2026
