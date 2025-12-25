---
name: generate-cr-spec
description: Automatically finds design-session.md for CR design (PATH A), generates cr-[Kind].md specification, and cleans up session file
---

# Generate Custom Resource Specification

**Purpose**: Auto-detect design session file, verify it's a CR definition (PATH A), extract interview data, apply CR spec template, and clean up.

## Execution (Fully Automatic)

### Step 1: Locate Session File

Search for `design-session.md` in the current directory.

**If not found:**
```
❌ Cannot generate CR spec: design-session.md not found
This skill requires a completed design session from the spec-designer agent.
```

### Step 2: Verify Design Type is PATH A (CR Definition)

Read `design-session.md` and check for design type indicator.

Look for markers indicating CR Definition design:
- Section titled "# Design Type:" with value "CR Definition" or "PATH A"
- Presence of "API Structure" section
- Presence of "Spec Fields" section
- Presence of "Status Fields" section

**If this is PATH B (Feature/Workflow):**
```
⚠️  Wrong skill: This session designed a feature, not a CR
Use the generate-feature-spec skill instead.
```

### Step 3: Extract CR Data from Session

Parse the design session file:

**API Structure:**
- Extract: Kind (e.g., "DPFHCPBridge", "Database")
- Extract: Group (e.g., "dpf")
- Extract: Domain (e.g., "hcp.bridge.com")
- Extract: Version (e.g., "v1alpha1")
- Compute: Full API Group = `{group}.{domain}`
- Compute: Full API = `{group}.{domain}/{version}`
- Compute: CRD Name = `{kind-plural}.{group}.{domain}`

**Overview:**
- Extract: Purpose (2-3 sentences)
- Extract: Scope (Namespaced / Cluster-scoped)
- Extract: Relationships (to other CRs)

**Spec Fields** (for each field):
- Extract: Field name
- Extract: Go type
- Extract: Required/Optional
- Extract: Description
- Extract: Validation constraints
- Extract: Default value (if any)
- Extract: Immutable flag
- Extract: Example value

**Status Fields** (for each field):
- Extract: Field name
- Extract: Go type
- Extract: Description
- Extract: Populated by (which feature)
- Extract: Example value

**Status Phases:**
- Extract: Phase names and meanings
- Extract: Phase transitions

**Status Conditions:**
- Extract: Condition types
- Extract: Meanings (True/False/Unknown)
- Extract: Reason codes
- Extract: Which feature sets them

**Validation Rules:**
- Extract: Admission validation
- Extract: Reconciliation validation
- Extract: Cross-field validation

**Examples:**
- Extract: YAML examples provided

### Step 4: Generate cr-[Kind].md

Determine output filename:
- Convert Kind to lowercase: `DPFHCPBridge` → `dpfhcpbridge`
- Filename: `cr-{kind-lowercase}.md`

Apply the CR specification template with extracted data.

[Template structure follows - same as before but populated with auto-extracted data]

Write this content to `cr-[kind].md`.

### Step 5: Clean Up Session File

Delete `design-session.md` (it was temporary working memory).

### Step 6: Report Success

Output:
```
✅ CR Specification Generated Successfully

Created: cr-[kind].md

Summary:
- CR Kind: [Kind]
- API: [group].[domain]/[version]
- CRD Name: [kind-plural].[group].[domain]
- Spec Fields: [count]
- Status Fields: [count]
- Conditions: [count]
- Validation Rules: [count]

Session file (design-session.md) has been cleaned up.

Next Steps:
- Review the CR specification for accuracy
- Ready for implementer agent to create the API types
```

## Error Handling

**Session file not found:**
```
❌ Error: design-session.md not found
Cannot generate CR spec without a design session.
Run the spec-designer agent first with PATH A (CR Definition).
```

**Wrong design type (Feature, not CR):**
```
⚠️  Error: Wrong design type
This session designed a Feature/Workflow (PATH B), not a CR.
Use generate-feature-spec skill instead.
```

**Missing required sections:**
```
⚠️  Error: Incomplete CR design session
Missing required sections:
- API Structure
- Spec Fields
- Status Fields

Cannot generate spec until design is complete.
```

**Cannot determine Kind:**
```
❌ Error: Cannot determine CR Kind name
API Structure section missing or malformed.
Manual review of design-session.md needed.
```

**Write permission error:**
```
❌ Error: Cannot write cr-[kind].md
Check file permissions in current directory.
```

## Data Extraction Rules

1. **Auto-detect structure**: Parse session file to find sections
2. **Preserve exact types**: Use Go types exactly as specified in session
3. **Maintain examples**: Include all YAML examples from interview
4. **Build complete structs**: Generate full Go type definitions with tags
5. **No hallucination**: Only include data from session file

## When This Skill is Invoked

The spec-designer agent invokes this skill after:
1. PATH A (CR Definition) interview is complete
2. All required sections captured
3. User confirms readiness to generate spec
4. Optional: After validate-session-file confirms completeness
