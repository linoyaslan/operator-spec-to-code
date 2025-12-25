---
name: checkpoint-sync
description: Automatically detects active session file, reads current state, and outputs synchronization confirmation to prevent context drift
---

# Checkpoint Synchronization

**Purpose**: Auto-detect the active session file, read current state, and output a sync confirmation proving the agent has refreshed from the authoritative source.

## Execution (Fully Automatic)

### Step 1: Auto-Detect Session File

Search for session files in priority order:
1. `requirements-session.md` → Requirements Engineer
2. `design-session.md` → Spec Designer
3. `implementation-tracker.md` → Implementer
4. `validation-session.md` → Test Engineer

Use the **first file found**.

### Step 2: Determine Agent Type

Based on file found:
- `requirements-session.md` → Agent Type: "Requirements"
- `design-session.md` → Agent Type: "Design"
- `implementation-tracker.md` → Agent Type: "Implementation"
- `validation-session.md` → Agent Type: "Validation"

### Step 3: Extract Current State

**For requirements-session.md:**
- Extract: Operator name (from Category 1)
- Extract: Completed categories (count categories with "✓ VERIFIED")
- Extract: Current category (first category without "✓ VERIFIED", or "PRD Generation" if all verified)

**For design-session.md:**
- Extract: Feature/CR name (from header or overview section)
- Extract: Completed sections (list major sections present)
- Extract: Current section (determine from interview progress or "Spec Generation" if complete)

**For implementation-tracker.md:**
- Extract: Spec file being implemented (from "## Implementation Progress" or context)
- Extract: Completed tasks (count tasks marked `[x]`)
- Extract: In Progress task (task in "### In Progress" section)
- Calculate: Progress (X/Y steps complete)

**For validation-session.md:**
- Extract: Spec file under review (from "### Spec Under Review")
- Extract: Implementation files (from "### Implementation Files")
- Extract: Issues found (count FAIL entries in Compliance Status table)

### Step 4: Determine Current Phase

If specific context not clear from file, use:
- Requirements: Current category number or "PRD Generation"
- Design: Current interview section or "Spec Generation"
- Implementation: Current task description or component name
- Validation: Current validation activity (e.g., "Test Coverage Analysis")

### Step 5: Output Sync Confirmation

Output in this exact format:

```
📋 **[Agent Type] Sync** ([Current Phase])
- [Primary Context Key]: [value]
- Completed: [completed items]
- Current: [current item]
[Optional: Additional context line if relevant]
```

**Examples:**

```
📋 **Requirements Sync** (Category 4)
- Operator: postgres-ha-operator
- Completed: Categories 1-3 (all VERIFIED)
- Current: Category 4: Operator Responsibilities
```

```
📋 **Design Sync** (Reconciliation Logic)
- Feature: database-provisioning
- Completed: Overview, CR Fields, External Integrations
- Current: Reconciliation Logic
```

```
📋 **Implementation Sync** (Controller Implementation)
- Spec: cr-DPFHCPBridge.md
- Completed: API types (5/5 spec fields), CRD generation
- Current: Controller reconciliation logic
- Progress: 3/7 steps complete
```

```
📋 **Validation Sync** (Test Coverage Analysis)
- Spec: feature-cluster-provisioning.md
- Validated: API compliance (PASS), RBAC (PASS)
- Current: Test Coverage Analysis
- Issues Found: 3 (2 HIGH, 1 MEDIUM)
```

## Error Handling

**No session file found:**
```
⚠️  Checkpoint Sync: No active session file found
Expected one of: requirements-session.md, design-session.md, implementation-tracker.md, validation-session.md
```

**Session file empty or malformed:**
```
⚠️  Checkpoint Sync: Session file [filename] found but appears empty or malformed
Cannot extract state for synchronization
```

**Cannot determine current phase:**
```
📋 **[Agent Type] Sync** (Unknown Phase)
- Session file: [filename]
- Status: File found but current phase unclear
- Action: Agent should update session file with clearer state markers
```

## When This Skill is Invoked

Agents invoke this skill at key checkpoints:
1. Before starting a new major section/category
2. Before final document generation
3. After user returns from a pause
4. After completing a major milestone

## Benefits

1. **Prevents Context Drift**: Forces agent to re-read authoritative session file
2. **Proves Synchronization**: Visible output confirms agent synced with file
3. **Automatic Detection**: No parameters needed - skill figures out context
4. **Consistent Format**: Standardized sync confirmation across all agents
