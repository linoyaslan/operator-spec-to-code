---
name: generate-feature-spec
description: Automatically finds design-session.md for feature design (PATH B), generates feature-[name].md specification, and cleans up session file
---

# Generate Feature Specification

**Purpose**: Auto-detect design session file, verify it's a feature/workflow design (PATH B), extract interview data, apply feature spec template, and clean up.

## Execution (Fully Automatic)

### Step 1: Locate Session File

Search for `design-session.md` in the current directory.

**If not found:**
```
❌ Cannot generate feature spec: design-session.md not found
This skill requires a completed design session from the spec-designer agent.
```

### Step 2: Verify Design Type is PATH B (Feature/Workflow)

Read `design-session.md` and check for design type indicator.

Look for markers indicating Feature/Workflow design:
- Section titled "# Design Type:" with value "Feature Implementation" or "PATH B"
- Presence of "Reconciliation Logic" section
- Presence of "Error Handling" section
- Presence of "RBAC Permissions" section
- Presence of "Testing Strategy" section

**If this is PATH A (CR Definition):**
```
⚠️  Wrong skill: This session designed a CR, not a feature
Use the generate-cr-spec skill instead.
```

### Step 3: Extract Feature Data from Session

Parse the design session file:

**Overview:**
- Extract: Feature name
- Extract: Purpose (2-3 sentences)
- Extract: Scope (In Scope / Out of Scope items)
- Extract: Trigger (what starts this feature)
- Extract: Prerequisites

**Target CR:**
- Extract: CR Kind
- Extract: API Group
- Extract: Domain
- Extract: Version
- Compute: Full API Group and Full API

**CR Fields Used:**
- Extract: Spec fields read by this feature
- Extract: Purpose of each field in this feature

**Status Fields Added/Modified:**
- Extract: New status fields (name, type, purpose, when populated, example)
- Extract: Modified existing fields
- Extract: New conditions (type, purpose, reason codes)
- Extract: Modified existing conditions

**Reconciliation Logic:**
- Extract: Desired state source
- Extract: Actual state source
- Extract: Reconciliation flow (step-by-step)
- Extract: Detailed steps (action, idempotency, error handling, requeue)
- Extract: Finalizer handling (name, deletion flow)
- Extract: Requeue strategy

**External Integrations** (for each):
- Extract: System name
- Extract: Connection details (endpoint, protocol, API version, authentication)
- Extract: Client library
- Extract: API operations (purpose, method, endpoint, request/response, errors, retry, timeout)
- Extract: Error handling
- Extract: Example client usage

**Error Handling:**
- Extract: Error categories (validation, transient, permanent)
- Extract: Edge cases (scenario, detection, handling)

**RBAC Permissions:**
- Extract: ClusterRole rules
- Extract: Permission justifications
- Extract: ServiceAccount

**Testing Strategy:**
- Extract: Unit test cases
- Extract: Integration test cases
- Extract: E2E test cases
- Extract: Mock requirements

**Performance & Scale:**
- Extract: Expected scale
- Extract: Rate limiting
- Extract: Timeout configuration
- Extract: Parallelism strategy
- Extract: Caching

**Observability:**
- Extract: Logging (Info/Debug/Error levels)
- Extract: Metrics
- Extract: Events
- Extract: Distributed tracing

**Dependencies:**
- Extract: Go dependencies
- Extract: Dependency matrix
- Extract: Compatibility validation

### Step 4: Generate feature-[name].md

Determine output filename:
- Extract feature name, convert to kebab-case
- Example: "Database Provisioning" → `database-provisioning`
- Filename: `feature-{name-kebab-case}.md`

Apply the Feature specification template with extracted data.

[Template structure follows - same as before but populated with auto-extracted data]

Write this content to `feature-[name].md`.

### Step 5: Clean Up Session File

Delete `design-session.md` (it was temporary working memory).

### Step 6: Report Success

Output:
```
✅ Feature Specification Generated Successfully

Created: feature-[name].md

Summary:
- Feature: [name]
- Target CR: [CR Kind]
- External Integrations: [count]
- Reconciliation Steps: [count]
- Test Cases: [count]
- RBAC Permissions: [resource count]

Session file (design-session.md) has been cleaned up.

Next Steps:
- Review the feature specification for accuracy
- Ready for implementer agent to write the code
```

## Error Handling

**Session file not found:**
```
❌ Error: design-session.md not found
Cannot generate feature spec without a design session.
Run the spec-designer agent first with PATH B (Feature/Workflow).
```

**Wrong design type (CR, not Feature):**
```
⚠️  Error: Wrong design type
This session designed a CR (PATH A), not a Feature.
Use generate-cr-spec skill instead.
```

**Missing required sections:**
```
⚠️  Error: Incomplete feature design session
Missing required sections:
- Reconciliation Logic
- RBAC Permissions
- Testing Strategy

Cannot generate spec until design is complete.
```

**Cannot determine feature name:**
```
❌ Error: Cannot determine feature name
Overview section missing or malformed.
Manual review of design-session.md needed.
```

**Write permission error:**
```
❌ Error: Cannot write feature-[name].md
Check file permissions in current directory.
```

## Data Extraction Rules

1. **Auto-detect structure**: Parse session file to find sections
2. **Preserve code examples**: Include all code snippets from interview
3. **Maintain technical details**: Keep exact API versions, timeouts, etc.
4. **Build complete specs**: Fill all template sections with interview data
5. **No hallucination**: Only include data from session file

## When This Skill is Invoked

The spec-designer agent invokes this skill after:
1. PATH B (Feature/Workflow) interview is complete
2. All required sections captured
3. User confirms readiness to generate spec
4. Optional: After validate-session-file confirms completeness
