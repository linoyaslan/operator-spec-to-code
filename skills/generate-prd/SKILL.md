---
name: generate-prd
description: Automatically finds requirements-session.md, generates operator-prd.md following the standard template, and cleans up the session file
---

# Generate Product Requirements Document

**Purpose**: Auto-detect and read requirements session file, apply PRD template, generate final document, and clean up working files.

## Execution (Fully Automatic)

### Step 1: Locate Session File

Search for `requirements-session.md` in the current directory.

**If not found:**
```
❌ Cannot generate PRD: requirements-session.md not found
This skill requires a completed requirements session from the requirements-engineer agent.
```

### Step 2: Validate Session Completeness

Read `requirements-session.md` and verify all 5 categories are VERIFIED:

Required categories:
1. ✓ Category 1: Operator Name - must have "✓ VERIFIED by user"
2. ✓ Category 2: Problem Statement & Purpose - must have "✓ VERIFIED by user"
3. ✓ Category 3: Custom Resources Definition - must have "✓ VERIFIED by user"
4. ✓ Category 4: Operator Responsibilities (High-Level) - must have "✓ VERIFIED by user"
5. ✓ Category 5: Success Criteria & Deployment - must have "✓ VERIFIED by user"

**If any category missing or not VERIFIED:**
```
⚠️  Cannot generate PRD: Session file incomplete
Missing or unverified categories:
- Category [N]: [Name] - [Status]

Please complete the requirements-engineer agent session first.
```

### Step 3: Extract Data from Session File

Parse each VERIFIED category:

**Category 1:**
- Extract: Operator name

**Category 2:**
- Extract: Problem description
- Extract: Manual processes being automated
- Extract: Purpose/automation value

**Category 3:**
- For each CR:
  - Extract: CR Kind name
  - Extract: What it represents
  - Extract: Lifecycle stages (Created, Ready, Updated, Deleted)
- Extract: CR relationships (if multiple CRs)

**Category 4:**
- For each responsibility:
  - Extract: Responsibility name/description
  - Extract: Trigger
  - Extract: Outcome

**Category 5:**
- Extract: Success indicators
- Extract: User workflows
- Extract: Deployment method

**Also check for:**
- "## Addition" sections (merge into main content)
- "## Correction" sections (merge into main content)
- Target users (OPTIONAL - only if mentioned)

### Step 4: Generate operator-prd.md

Apply the PRD template structure with extracted data:

```markdown
# Operator Product Requirements Document

## Metadata
- **Operator Name**: [from Category 1]
- **Created**: [current date YYYY-MM-DD]
- **Last Updated**: [current date YYYY-MM-DD]

## Problem Statement

[2-3 paragraphs synthesized from Category 2 - combine problem description and manual processes into narrative form]

## Target Users

[ONLY include if user mentioned target users during interview - otherwise OMIT this entire section]

## Operator Overview

### Purpose
[High-level description from Category 2]

## Custom Resources

### [CR Kind 1 from Category 3]

#### Description
[What one instance represents]

#### Lifecycle
1. **Created**: [What happens when created]
2. **Ready**: [What ready means]
3. **Updated**: [What happens on update]
4. **Deleted**: [What cleanup happens]

[Repeat for each CR]

## CR Relationships

[Describe relationships from Category 3, or "N/A - single CR" if only one]

## Operator Responsibilities (High-Level)

### Responsibility 1: [Name from Category 4]

**Trigger**: [Trigger from Category 4]

**Outcome**: [Outcome from Category 4]

[Brief description expanding on the outcome]

[Repeat for each responsibility]

## Success Criteria

**How users know the operator works correctly**:
- [Indicator 1 from Category 5]
- [Indicator 2 from Category 5]
- [Indicator 3 from Category 5]

## User Workflows

### Workflow 1: [Workflow name from Category 5]

[Description with steps]:

1. [Step 1]
2. [Step 2]
3. Expected outcome: [outcome]

[Repeat for each workflow]

## Deployment

**Deployment Method**: [Method from Category 5]

## Open Questions

[List any unresolved questions from session, or "No open questions identified during the interview."]
```

Write this content to `operator-prd.md`.

### Step 5: Clean Up Session File

Delete `requirements-session.md` (it was temporary working memory).

### Step 6: Report Success

Output:
```
✅ PRD Generated Successfully

Created: operator-prd.md

Summary:
- Operator: [name]
- Custom Resources: [count] ([list CR names])
- Responsibilities: [count]
- User Workflows: [count]
- Deployment: [method]

Session file (requirements-session.md) has been cleaned up.

Next Steps:
- Review the PRD for accuracy
- Ready to move to spec-designer agent for technical specifications
```

## Error Handling

**Session file not found:**
```
❌ Error: requirements-session.md not found
Cannot generate PRD without a requirements session.
Run the requirements-engineer agent first.
```

**Incomplete session (missing categories):**
```
⚠️  Warning: Session incomplete
Found [X/5] verified categories. Missing:
- Category 3: Custom Resources
- Category 5: Success Criteria

Cannot generate PRD until all categories are verified.
```

**Malformed session file:**
```
⚠️  Error: Cannot parse requirements-session.md
File appears malformed or doesn't match expected structure.
Manual review needed.
```

**Write permission error:**
```
❌ Error: Cannot write operator-prd.md
Check file permissions in current directory.
```

## Data Extraction Rules

1. **Use VERIFIED content only**: Ignore DRAFT sections
2. **Include additions/corrections**: Merge "## Addition" and "## Correction" sections into main content
3. **Preserve user language**: Use exact terminology from session file
4. **No hallucination**: Only include information present in session file
5. **Synthesize narratives**: Convert Q&A format into cohesive paragraphs for Problem Statement

## When This Skill is Invoked

The requirements-engineer agent invokes this skill after:
1. All 5 categories are completed and VERIFIED
2. User confirms readiness to generate PRD
3. Optional: After validate-session-file confirms completeness
