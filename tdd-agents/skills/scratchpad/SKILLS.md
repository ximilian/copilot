---
name: manage_scratchpad
description: Accesses and updates the SCRATCHPAD.md file to maintain a "source of truth" between the Architect, Planner, and Implementer.
---

# Skill Instructions

The scratchpad is the shared communication mechanism between all three agents. This skill provides reliable read/write operations to maintain consistency across parallel sessions.

## Required Schema

All agents **MUST** follow the schema defined in [SCRATCHPAD_SCHEMA.md](./SCRATCHPAD_SCHEMA.md). Before using this skill, familiarize yourself with:
- Required sections: Architectural Context, Test Design, Implementation Progress, Approval Log
- Field validation rules
- Handoff requirements between agents

## Versioning & Parallel Sessions

Each issue gets its own scratchpad file, enabling multiple parallel sessions without conflicts.

**File naming**: `SCRATCHPAD-[ISSUE_ID].md` or `SCRATCHPAD-[TIMESTAMP].md`
- Example: CSS-21342 session uses `SCRATCHPAD-CSS-21342.md`
- Fallback: If no issue ID detected, use timestamp: `SCRATCHPAD-20260304-103000.md` (prevents conflicts in parallel sessions)

**Issue ID detection** (in order):
1. Explicitly provided `issueId` parameter (recommended for parallel work)
2. Parsed from git branch name (e.g., `feature/CSS-21342_stripe-spp` → `CSS-21342`)
3. Extracted from issue context or environment
4. Generate timestamped filename `SCRATCHPAD-[YYYYMMDD-HHMMSS].md` if detection fails

## Parameters

**issueId** (String, optional):
- Issue identifier (e.g., `CSS-21342`)
- Creates versioned scratchpad: `SCRATCHPAD-[issueId].md`
- If omitted, auto-detected from git branch
- **Recommended**: Always provide explicitly in parallel sessions to avoid ambiguity

**action** (String): 
- `initialize`: Create SCRATCHPAD.md with template based on schema
- `read`: Retrieve full scratchpad content
- `append_section`: Add data to a specific section (creates section if missing)
- `validate`: Check scratchpad structure against schema and report errors

**content** (String): 
- Text or data to record (used with `append_section` and `initialize`)

**section** (String): 
- Header target matching schema sections:
  - "Architectural Context"
  - "Test Design"
  - "Implementation Progress"
  - "Approval Log"
  - "Roadblocks & Decisions"

## Validation Rules

Before and after each operation, ensure:
1. Versioned scratchpad file exists (e.g., `SCRATCHPAD-CSS-21342.md`)
2. All required fields for current stage are present (see SCRATCHPAD_SCHEMA.md "Validation Checklist")
3. Timestamps are ISO 8601 format (e.g., 2025-03-04T10:30:00Z)
4. Status fields use allowed values (not-started|in-progress|complete|PASS|FAIL)

## Error Handling

If scratchpad operations fail:
1. **Issue ID ambiguous**: Alert if auto-detection fails or multiple possible IDs found
2. **Missing schema sections**: Automatically create missing sections with placeholder content
3. **Validation failures**: Report specific errors and required fixes before proceeding
4. **Handoff validation**: If previous agent's sections are missing, return error with specific guidance
5. **File not found**: If scratchpad doesn't exist and not in `initialize` action, return error with filename

## File Location & Permissions

- **Location**: `.github/skills/scratchpad/` directory
- **Naming**: `SCRATCHPAD-[ISSUE_ID].md` (issue-based) or `SCRATCHPAD-[TIMESTAMP].md` (session fallback when ID unavailable)
- **Git tracking**: Scratchpad files should be in `.gitignore` (session-specific, not committed)
- **Concurrency**: Multiple scratchpads can be read/written simultaneously (separate files, no conflicts)