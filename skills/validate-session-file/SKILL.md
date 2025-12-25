---
name: validate-session-file
description: Automatically detects active session file, determines its type, validates structure and completeness, and reports readiness for document generation
---

# Validate Session File

**Purpose**: Auto-detect active session file, determine type, validate structure/completeness, and report whether it's ready for final document generation.

## Execution (Fully Automatic)

### Step 1: Auto-Detect Session File

Search for session files in priority order:
1. `requirements-session.md` → Requirements session
2. `design-session.md` → Design session
3. `implementation-tracker.md` → Implementation tracker
4. `validation-session.md` → Validation session

Use the **first file found**.

**If no session file found:**
```
⚠️  No session file found
Searched for: requirements-session.md, design-session.md, implementation-tracker.md, validation-session.md
Cannot validate - no active session detected.
```

### Step 2: Determine Session Type

Based on file found:
- `requirements-session.md` → Type: "Requirements"
- `design-session.md` → Type: "Design"
- `implementation-tracker.md` → Type: "Implementation"
- `validation-session.md` → Type: "Validation"

### Step 3: Run Type-Specific Validation

Execute validation checks based on detected type:

---

#### For Requirements Session (`requirements-session.md`)

**Required Structure:**
- Category 1: Operator Name
- Category 2: Problem Statement & Purpose
- Category 3: Custom Resources Definition
- Category 4: Operator Responsibilities (High-Level)
- Category 5: Success Criteria & Deployment

**Validation Checks:**
- [ ] All 5 categories present
- [ ] Each category has "## Status: ✓ VERIFIED by user" marker
- [ ] Category 1 has operator name
- [ ] Category 3 has at least one CR with lifecycle stages
- [ ] No DRAFT sections remaining (all must be VERIFIED)
- [ ] No placeholder text ([TODO], [TBD], [Fill in])

---

#### For Design Session (`design-session.md`)

**First, determine design path:**
- Look for "# Design Type:" marker
- Check for PATH A (CR) indicators: "API Structure", "Spec Fields", "Status Fields"
- Check for PATH B (Feature) indicators: "Reconciliation Logic", "RBAC Permissions", "Testing Strategy"

**For PATH A (CR Definition):**
- [ ] Design Type specified
- [ ] API Structure section (Kind, Group, Domain, Version)
- [ ] Spec Fields section (at least one field)
- [ ] Status Fields section (at least one field)
- [ ] Examples section

**For PATH B (Feature Implementation):**
- [ ] Design Type specified
- [ ] Feature Overview section
- [ ] CR Fields Used section
- [ ] Reconciliation Logic section
- [ ] Error Handling section
- [ ] RBAC Permissions section
- [ ] Testing Strategy section

---

#### For Implementation Tracker (`implementation-tracker.md`)

**Required Structure:**
- ## Implementation Progress
- ### Completed
- ### In Progress
- ### Pending

**Validation Checks:**
- [ ] Has "Completed", "In Progress", and "Pending" sections
- [ ] Tasks use proper markdown checkbox syntax (- [ ] or - [x])
- [ ] At least one task listed
- [ ] Only one task marked "In Progress" (or zero)

---

#### For Validation Session (`validation-session.md`)

**Required Structure:**
- ## Validation Session: [Date]
- ### Spec Under Review
- ### Implementation Files
- ### Compliance Status
- ### Recommendations

**Validation Checks:**
- [ ] Has "Spec Under Review" with spec file path
- [ ] Has "Implementation Files" section
- [ ] Has "Compliance Status" table
- [ ] Has recommendations or issues documented

---

### Step 4: Generate Validation Report

Output format:

```
═══════════════════════════════════════
Session File Validation Report
═══════════════════════════════════════

File: [filename]
Type: [session type]
Date: [current date]

Structure Validation
────────────────────
✅ File exists and readable
✅ Required sections present
✅ Proper markdown formatting
⚠️  Warning: [warning message]
❌ Error: [error message]

Content Validation
──────────────────
✅ All categories verified (requirements only)
✅ API structure complete (design CR only)
✅ Reconciliation logic documented (design feature only)
✅ All tasks properly formatted (implementation only)
❌ Missing: [what's missing]

Completeness Check
──────────────────
✅ No DRAFT sections remaining
✅ No placeholder text
✅ All required fields populated
⚠️  Incomplete: [what's incomplete]

Overall Status
──────────────
[PASS ✓ / FAIL ✗] - [Ready for document generation / Issues must be resolved]

Issues to Fix:
1. [Issue 1]
2. [Issue 2]
```

### Step 5: Return Pass/Fail Status

**PASS** if:
- All required sections present
- All validation checks ✓
- No DRAFT sections (for requirements/design)
- No critical errors

**FAIL** if:
- Missing required sections
- Has DRAFT sections not VERIFIED
- Has placeholder text
- Critical validation errors

## Validation Examples

### Valid Requirements Session
```
✅ Session File Validation: PASS

File: requirements-session.md
Type: Requirements

All 5 categories present and VERIFIED
No DRAFT sections remaining
Operator name: postgres-ha-operator
Custom Resources: 2 defined

Ready for PRD generation (use generate-prd skill)
```

### Invalid Design Session
```
❌ Session File Validation: FAIL

File: design-session.md
Type: Design (Feature - PATH B)

Issues Found:
1. Missing section: Error Handling
2. Missing section: Testing Strategy
3. RBAC Permissions section is empty

Cannot generate feature spec until these are completed.
```

### Valid Implementation Tracker
```
✅ Session File Validation: PASS

File: implementation-tracker.md
Type: Implementation

Structure: Complete
Tasks: 5 completed, 1 in progress, 3 pending
Format: All tasks properly formatted

Tracker is well-maintained and up-to-date.
```

## Error Handling

**No session file found:**
```
⚠️  No active session file found
Cannot validate - no session in progress
```

**File empty:**
```
❌ Validation Error: [filename] is empty
Cannot validate empty session file
```

**Unrecognized format:**
```
⚠️  Warning: Cannot determine session type from content
File: [filename]
Manual review recommended
```

**Corrupt markdown:**
```
❌ Validation Error: Malformed markdown detected
Line [N]: [issue description]
```

## When This Skill is Invoked

Agents invoke this skill:
1. **Before final document generation** - Ensure session is complete
2. **After completing all questions** - Validate nothing was missed
3. **When user requests validation** - On-demand session check
4. **After resuming from interruption** - Ensure session integrity

## Benefits

1. **Prevents Incomplete Generation**: Catches missing sections before document creation
2. **Ensures Quality**: Validates all required information captured
3. **Clear Feedback**: Specific error messages about what's missing
4. **Saves Time**: Avoids generating incomplete specs that need rework
5. **Auto-Detection**: No parameters needed - skill finds and validates the right file
