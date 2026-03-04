---
name: SCRATCHPAD_SCHEMA
description: 'Definitive schema for the shared SCRATCHPAD.md file. All agents must follow this structure to ensure compatible data exchange.'
version: '1.0'
---

# Scratchpad Schema

This document defines the required structure, sections, and fields for `SCRATCHPAD.md`. All three agents (Architect, Planner, Implementer) must use this schema to ensure reliable communication.

**For concrete examples showing how SCRATCHPAD.md looks when filled out, see [SCRATCHPAD_EXAMPLE.md](./SCRATCHPAD_EXAMPLE.md).**

## Versioning & Parallel Sessions

Each session gets its own scratchpad file:
- **Issue-based**: `SCRATCHPAD-[ISSUE_ID].md` when issue ID is available (e.g., `SCRATCHPAD-CSS-21342.md`)
- **Timestamp-based**: `SCRATCHPAD-[YYYYMMDD-HHMMSS].md` as fallback when ID cannot be detected

This enables multiple parallel development sessions without conflicts. See [SKILLS.md](./SKILLS.md) for how versioning works.

## File Structure Overview

```
# SCRATCHPAD: [CSS-XXXX] [Issue Title]

## Architectural Context
## Test Design
## Implementation Progress
## Approval Log
## Roadblocks & Decisions
```

---

## Section 1: Architectural Context

**Purpose**: Records Architect's analysis. Initialized by Architect, read by Planner.

**Status Field** (required):
- `not-started` | `in-progress` | `complete`

**Global Invariants** (required):
- Bulleted list of project-wide constraints (e.g., "Use `decimal` for all currency", "No shared state in tests")
- Format: `- [Invariant Name]: [Description]`

**Primary Files** (required):
- Format: `- [file/path.go]: [Why this file needs changes]`
- List all files directly modified for this issue

**Secondary Files** (required):
- Format: `- [file/path.go]: [Relationship to primary files]`
- Files that may need updates due to dependency changes

**Patterns & References** (required):
- Format: `- [Pattern Name]: See [file/path.go] — [What pattern to match]`
- Existing code patterns the issue should follow
- **Important**: Reference file paths only, not line numbers (line numbers become stale as code changes)

**Dependency Graph** (optional):
- Text or ASCII representation of file dependencies
- Useful for large issues with complex interdependencies

---

## Section 2: Test Design

**Purpose**: Records Planner's test strategy. Initialized by Planner, read by Implementer.

**Issue Reference** (required):
- Format: `CSS-XXXX`

**Acceptance Criteria & Test Mapping** (required):
- Format: Table with columns: `Criterion | Test Name | Type | Scenario`
- All acceptance criteria must map to at least one test
- Example format: Single row would be `| AC1: ... | TestXxx | unit | ... |`

**Batch Structure** (required):
- Format: `### Batch #N: [Test Category] ([X test cases])`
- List batches in order: `Batch #1: Unit Tests`, `Batch #2: Integration Tests`, etc.
- Each batch should contain 3-5 test cases
- Each batch lists test names with brief descriptions

**Test Data Requirements** (required):
- Fixtures: Names and descriptions of test data structures needed
- Mocks: External services to mock (DocuSign, Stripe API, etc.) and their expected behaviors
- Database state: Pre-conditions or seed data needed before tests run

**Coverage Target** (required):
- Format: `[Percentage]% line coverage, [Percentage]% branch coverage`
- Example: `85% line coverage, 80% branch coverage`

**Technical Context** (required):
- Affected Packages: Which Go packages contain logic to test
- External Dependencies: Service versions, frameworks, libraries
- Database: Schema details, data types, transaction behavior
- Existing Patterns: Reference to proven patterns in the codebase

---

## Section 3: Implementation Progress

**Purpose**: Tracks Implementer's progress through Red-Green-Refactor cycles.

**Batch Status** (required for each batch):
- Format: `### Batch #N: [Category] — [PENDING|IN-PROGRESS|PASS|FAIL]`
- Update status after each Red-Green-Refactor cycle
- List individual test results with checklist (`[x]` for pass, `[ ]` for pending)
- Include test execution time and any refactoring notes

**Coverage Reports** (optional but recommended):
- Snapshot of coverage percentage after each batch
- Format: `Coverage: 92% line, 88% branch (target: 85% line, 80% branch)`

**Refactoring Notes** (optional):
- Record extracted helpers, interfaces, or patterns discovered during this batch
- Useful for understanding what changed during refactor phase

---

## Section 4: Approval Log

**Purpose**: Records decision points and approvals at each stage.

**Architect Approval** (required after Architect completes):
- Format: `**Architect Approval**: [Timestamp ISO 8601] — [Status: Approved|Approved with notes|Pending user input]`
- If pending, include specific questions or blockers for Planner

**Planner Approval** (required after Planner completes):
- Format: `**Planner Approval**: [Timestamp ISO 8601] — [Status: Approved|Requires revision]`
- Include brief note on what was approved (test count, coverage target, etc.)

**User Confirmations** (required at each Implementer checkpoint):
- Format: `**User Checkpoint #[N]**: [Timestamp ISO 8601] — [Batch #X] — [Approved|Returned for revision]`
- Record specific reason if revision requested
- Include user comments or feedback

---

## Section 5: Roadblocks & Decisions

**Purpose**: Tracks architectural conflicts, design decisions, and unblocked items.

**Active Roadblocks** (only present if exists):
- Format: Describe each blocking issue with test name, technical conflict, attempted solutions, and what's needed to unblock
- Record as soon as roadblock is discovered
- Include: What needs to change in the plan / design / architecture

**Resolved Decisions** (append as issues are resolved):
- Format: `- [Decision]: [Rationale]`
- Document all major architectural decisions made during planning or implementation
- Include: Why this approach was chosen, what alternatives were considered

---

## Concrete Examples

**See [SCRATCHPAD_EXAMPLE.md](./SCRATCHPAD_EXAMPLE.md) for a complete end-to-end example** showing:
- ✅ Architect phase output (Architectural Context)
- ✅ Planner phase output (Test Design)
- ✅ Implementer progress (Implementation Progress across multiple batches)
- ✅ Approval Log filled out with timestamps
- ✅ Roadblocks & Decisions resolved

---

## Field Validation Rules

| Field | Required? | Validation | Error Recovery |
|-------|-----------|-----------|-----------------|
| Global Invariants | Yes | Must be non-empty list | Fail with error: "Invariants not specified" |
| Primary Files | Yes | Must reference real files | Agent must search codebase or ask user |
| Acceptance Criteria | Yes | Must map to test names | Fail with error: "Test mapping incomplete" |
| Coverage Target | Yes | Must be number + % | Default to 80% if missing, confirm with user |
| Batch Status | Yes | Must be PENDING\|IN-PROGRESS\|PASS\|FAIL | Never proceed without status update |
| Approval Log | No | Timestamps recommended for audit trail | Can be empty if no approvals yet |

---

## Validation Checklist

Before handing off between agents:

**Architect → Planner**:
- [ ] Architectural Context.Status = `complete`
- [ ] Architectural Context has all 4 required fields (Invariants, Primary, Secondary, Patterns)
- [ ] Timestamp recorded in Approval Log

**Planner → Implementer**:
- [ ] Test Design has issue reference
- [ ] All acceptance criteria mapped to tests
- [ ] At least 2 batches defined
- [ ] Coverage target specified
- [ ] Timestamp recorded in Approval Log

**Implementer Progress**:
- [ ] After each batch: Coverage reported, status updated
- [ ] Batch Status never left as "IN-PROGRESS" at checkpoint
- [ ] Patterns extracted documented
