# Verbatim Capture Protocol - Implementation Plan

## Overview

This document describes a fundamental redesign of the operator requirements agent to solve persistent issues with LLM-based interviewing.

---

## Part 1: Problem Analysis

### 1.1 Observed Problems

After extensive testing of the requirements engineer agent, the following problems consistently occur regardless of instruction changes:

#### Problem 1: Rephrasing User Input
**Description**: The agent changes the user's words when capturing answers, even when the user's language is already appropriate and high-level.

**Example**:
- User says: "creates the pods and sets up networking"
- Agent writes: "provisions container workloads and establishes network connectivity"

**Impact**: User must correct the agent's interpretation, which often introduces new errors.

#### Problem 2: Multiple Correction Rounds
**Description**: Users require 6-7 iterations of corrections per category to get the captured information accurate.

**Example flow**:
1. User provides answer
2. Agent summarizes incorrectly
3. User corrects
4. Agent re-summarizes, introduces new error
5. User corrects again
6. Agent partially fixes, breaks something else
7. (Repeat 3-4 more times)

**Impact**: Frustrating experience, wastes time, erodes trust in the tool.

#### Problem 3: Inventing Information
**Description**: The agent creates information that was never discussed, including relationships, technical details, and requirements.

**Examples**:
- Agent writes CR relationships user never mentioned
- Agent adds technical implementation details user didn't specify
- Agent assumes features based on operator name

**Impact**: PRD contains inaccurate information that must be caught and removed.

#### Problem 4: Asking Irrelevant Questions
**Description**: The agent asks questions about topics the user never mentioned, going off on tangents.

**Examples**:
- "What targets the operator to configure ingress?" (user never mentioned ingress)
- "What targets the operator to manage DHCP?" (user never mentioned DHCP)
- "How will the operator handle multi-cluster scenarios?" (user never mentioned multi-cluster)

**Impact**: Wastes time, confuses the user, adds noise to the session.

#### Problem 5: Excessive Time Consumption
**Description**: A single requirements session takes 2+ hours with no breaks.

**Breakdown**:
- Actual information gathering: ~30 minutes
- Correcting agent interpretations: ~60 minutes
- Handling tangent questions: ~20 minutes
- Re-reading and verifying: ~30 minutes

**Impact**: Exhausting for users, reduces adoption, defeats purpose of automation.

### 1.2 Root Cause Analysis

**Why instructions don't fix these problems:**

| Problem | Why Instructions Fail |
|---------|----------------------|
| Rephrasing | LLM's training incentivizes "improving" text - can't be instructed away |
| Corrections | Agent keeps re-interpreting during corrections |
| Inventing | LLM fills gaps with plausible-sounding content |
| Tangents | Too much freestyle allowed in current design |
| Time | Downstream effect of all above problems |

**Core insight**: The current agent tries to be "helpful" by interpreting user input. This helpfulness IS the problem.

### 1.3 What We've Tried (And Why It Failed)

| Attempt | Description | Result |
|---------|-------------|--------|
| Detailed instructions | 850 lines of rules | Agent ignores or misapplies |
| "Don't rephrase" rules | Explicit prohibition | Agent still rephrases |
| Spec Sync Protocol | Read files before responding | Helps memory, not interpretation |
| Checkpoint Confirmation | Prove file was read | Helps memory, not interpretation |
| Verification loops | User confirms each section | Enables corrections but doesn't prevent errors |

**Conclusion**: We need a fundamentally different approach, not more instructions.

---

## Part 2: Solution Design

### 2.1 Core Concept: Verbatim Capture

**Principle**: The agent captures the user's EXACT words and does not interpret, summarize, or rephrase anything.

**How it works**:
```
User says: "The operator watches ConfigMaps and copies them to other namespaces"

Agent captures:
> "The operator watches ConfigMaps and copies them to other namespaces"

Agent responds: "Captured. Correct?"
User: "Yes"
Agent: "Next question..."
```

**Why this works**:
- Nothing to correct (it's the user's own words)
- Nothing invented (agent only stores what user said)
- No interpretation (quote blocks preserve exact input)

### 2.2 Core Concept: Fixed Interview Protocol

**Principle**: The agent asks ONLY predefined questions in a fixed order. No improvisation allowed.

**How it works**:
```
Category 2 Questions (fixed list):
1. "What problem does this operator solve?"
2. "What manual processes will it automate?"
3. "Who are the users of this operator?"

Agent cannot add: "What about ingress?" (not on list)
Agent cannot add: "How does it handle DHCP?" (not on list)
```

**Why this works**:
- No tangent questions (not on the list = can't ask)
- Predictable session (user knows what's coming)
- Faster sessions (no wasted time on irrelevant topics)

### 2.3 Core Concept: Quick Verify

**Principle**: After capturing verbatim, agent asks simple "Correct?" - no read-back, no interpretation.

**Before (current)**:
```
Agent: "Let me summarize what you said. You mentioned that the operator
       provisions resources and manages lifecycle... [200 words of interpretation]
       Is this accurate?"
User: "No, I said 'creates pods' not 'provisions resources'"
Agent: "I've updated it to say 'creates container workloads'..."
User: "No! Just say 'creates pods'!"
[5 more rounds]
```

**After (proposed)**:
```
Agent: Captured:
       > "creates pods and sets up networking"
       Correct?
User: "Yes"
Agent: "Next question..."
```

### 2.4 Agent Structure Comparison

| Aspect | Current Agent | New Agent |
|--------|---------------|-----------|
| File size | ~850 lines | ~150 lines |
| Question source | Agent improvises | Fixed list only |
| Answer capture | Agent summarizes | Verbatim quotes |
| Verification | Multi-round read-back | Single "Correct?" |
| Interpretation | Agent "improves" text | Zero interpretation |
| Session time | 2+ hours | 30-60 minutes |

---

## Part 3: Detailed Specification

### 3.1 Session Transcript Format

The agent creates and maintains a `session-transcript.md` file:

```markdown
# Operator Requirements Session

**Started**: 2024-01-15 10:30
**Operator**: [to be captured]

---

## Category 1: Operator Identity

**Q: What is the name of this operator?**
> "config-sync-operator"
✓ Verified

---

## Category 2: Problem & Purpose

**Q: What problem does this operator solve?**
> "ConfigMaps need to be synced across namespaces. Developers manually copy them and forget to update when source changes."
✓ Verified

**Q: What manual processes will it automate?**
> "Copying ConfigMaps from source namespace to targets, keeping them in sync automatically"
✓ Verified

**Q: Who are the users of this operator?**
> "Platform engineers who manage shared configurations across namespaces"
✓ Verified

---

## Category 3: Custom Resources
[continues...]
```

### 3.2 Fixed Question List

```markdown
## Category 1: Operator Identity (1 question)
1. What is the name of this operator?

## Category 2: Problem & Purpose (3 questions)
1. What problem does this operator solve?
2. What manual processes will it automate?
3. Who are the users of this operator?

## Category 3: Custom Resources (4 questions)
1. What Custom Resource(s) will users create to use this operator?
2. For each CR: What does it represent? (ask per CR)
3. For each CR: What are the main states/phases? (ask per CR)
4. Are there any relationships between the CRs? (only if multiple CRs)

## Category 4: Operator Responsibilities (5 questions)
1. What happens when a user creates a CR?
2. What happens when a user updates a CR?
3. What happens when a user deletes a CR?
4. What external systems or APIs does the operator interact with?
5. What other Kubernetes resources does the operator create or manage?

## Category 5: Success & Deployment (3 questions)
1. How will users know the operator is working correctly?
2. How will the operator be deployed? (Helm, OLM, manifests, etc.)
3. Any specific requirements for where it runs? (namespace, permissions, etc.)
```

**Total: 16 fixed questions** (some conditional based on answers)

### 3.3 Agent Behavior Rules

```markdown
## Capture Rules
1. ALWAYS use quote blocks: > "user's exact words"
2. NEVER paraphrase, summarize, or "improve" the user's words
3. NEVER add information the user didn't say
4. If user's answer is unclear, ask: "Could you clarify what you mean by [X]?"
5. If user gives partial answer, ask: "Anything else about [topic]?"

## Question Rules
1. ONLY ask questions from the Fixed Question List
2. NEVER invent new questions based on user's answers
3. NEVER assume topics (ingress, DHCP, multi-cluster, etc.)
4. If something seems relevant but isn't on the list, DON'T ask about it
5. Skip conditional questions if they don't apply

## Verification Rules
1. After each capture, ask: "Captured. Correct?"
2. If user says "no", ask: "What should I capture instead?"
3. Replace with new verbatim capture
4. Do NOT interpret the correction

## Session Rules
1. Create session-transcript.md before greeting
2. Append each Q&A immediately after capture
3. Mark verified answers with ✓
4. At category end, show count: "Category 2 complete (3/3 questions)"
5. No summaries, no read-backs, no interpretations
```

### 3.4 PRD Generation

After all 5 categories are complete, the agent generates `operator-prd.md` by:

1. Reading the session transcript
2. Organizing verbatim quotes into PRD sections
3. NOT interpreting or expanding the quotes

```markdown
# Operator PRD: config-sync-operator

## Problem Statement
> "ConfigMaps need to be synced across namespaces. Developers manually copy them and forget to update when source changes."

## Automated Processes
> "Copying ConfigMaps from source namespace to targets, keeping them in sync automatically"

## Users
> "Platform engineers who manage shared configurations across namespaces"

## Custom Resources

### ConfigSync
**Represents**: > "A sync rule that defines source ConfigMap and target namespaces"
**States**: > "Pending, Syncing, Synced, Error"

[etc - all verbatim quotes organized into sections]
```

---

## Part 4: Terminology

To maintain distinction from other frameworks:

| Concept | Name | Description |
|---------|------|-------------|
| Core approach | **Verbatim Capture** | Store exact user words |
| Question structure | **Fixed Interview Protocol** | Predefined questions only |
| Answer format | **Quote Blocks** | `> "exact words"` format |
| Confirmation | **Quick Verify** | Simple yes/no after capture |
| Storage file | **Session Transcript** | The Q&A record file |
| Context refresh | **Transcript Sync** | Read transcript at checkpoints |
| Output generation | **Quote Assembly** | Organize quotes into PRD |

---

## Part 5: Test Plan

### Test 1: No Rephrasing
**Objective**: Verify agent captures exact user words without modification.

**Steps**:
1. Start requirements session
2. When asked for operator name, say: "my-weird-NAME-with-CAPS"
3. Check session-transcript.md

**Expected**: File contains `> "my-weird-NAME-with-CAPS"` exactly
**Fail if**: Any variation like "my-weird-name-with-caps" or "MyWeirdNameWithCaps"

### Test 2: No Invented Questions
**Objective**: Verify agent only asks questions from the fixed list.

**Steps**:
1. Start requirements session
2. For problem statement, say: "Syncs ConfigMaps between namespaces"
3. Note all questions agent asks in Category 2 and 3
4. Compare against Fixed Question List

**Expected**: All questions match the fixed list exactly
**Fail if**: Agent asks about ingress, DHCP, multi-cluster, or anything not on list

### Test 3: No Invented Information
**Objective**: Verify agent doesn't add information user didn't provide.

**Steps**:
1. Complete full session with minimal answers
2. Review generated PRD
3. Search for any content not in session transcript

**Expected**: Every statement in PRD traces back to a verbatim quote
**Fail if**: PRD contains relationships, details, or features not in transcript

### Test 4: Quick Verify Works
**Objective**: Verify corrections don't trigger re-interpretation.

**Steps**:
1. When asked for operator name, say: "test-operator"
2. When agent asks "Captured. Correct?", say: "No, it should be test-operator-v2"
3. Check how agent handles correction

**Expected**: Agent captures `> "test-operator-v2"` and moves on
**Fail if**: Agent interprets, summarizes, or asks clarifying questions about the correction

### Test 5: Session Time
**Objective**: Verify session completes in reasonable time.

**Steps**:
1. Start timer
2. Complete full 5-category session
3. Use moderately complex operator (3 CRs, 4-5 responsibilities)
4. Stop timer when PRD is generated

**Expected**: Under 45 minutes
**Fail if**: Over 60 minutes

### Test 6: Zero Corrections Needed
**Objective**: Verify user doesn't need to correct agent's interpretation.

**Steps**:
1. Complete full session
2. Count how many times user had to correct the agent's capture

**Expected**: 0-1 corrections total (only if user misspoke)
**Fail if**: More than 2 corrections needed

### Test 7: Transcript Sync at Checkpoints
**Objective**: Verify agent reads transcript before new categories.

**Steps**:
1. Complete Category 1 and 2
2. Before Category 3 starts, check for Transcript Sync output
3. Verify it shows correct operator name and completed categories

**Expected**:
```
📋 Transcript Sync (Category 3)
- Operator: [correct name from file]
- Completed: Categories 1-2
```
**Fail if**: Wrong name, missing categories, or no sync block

### Test 8: PRD Accuracy
**Objective**: Verify final PRD matches session transcript exactly.

**Steps**:
1. Complete full session
2. Compare every statement in PRD against session transcript
3. Check for additions, omissions, or modifications

**Expected**: 100% match between transcript quotes and PRD content
**Fail if**: Any PRD content not traceable to transcript

---

## Part 6: Implementation Checklist

### Phase 1: Create New Agent
- [ ] Create new `operator-requirements-engineer.md` (~150 lines)
- [ ] Define Verbatim Capture rules
- [ ] Define Fixed Question List (16 questions)
- [ ] Define Quick Verify behavior
- [ ] Define Transcript Sync checkpoints
- [ ] Define PRD generation (Quote Assembly)

### Phase 2: Update Files
- [ ] Update `.gitignore` for `session-transcript.md`
- [ ] Archive old 850-line agent file
- [ ] Update any cross-references in other agents

### Phase 3: Testing
- [ ] Run Test 1: No Rephrasing
- [ ] Run Test 2: No Invented Questions
- [ ] Run Test 3: No Invented Information
- [ ] Run Test 4: Quick Verify Works
- [ ] Run Test 5: Session Time
- [ ] Run Test 6: Zero Corrections Needed
- [ ] Run Test 7: Transcript Sync
- [ ] Run Test 8: PRD Accuracy

### Phase 4: Iterate
- [ ] Fix any test failures
- [ ] Adjust Fixed Question List if needed
- [ ] Re-run failed tests
- [ ] Document any edge cases

### Phase 5: Apply to Other Agents
- [ ] Apply Verbatim Capture to spec-designer
- [ ] Apply Verbatim Capture to implementer
- [ ] Apply Verbatim Capture to test-engineer

---

## Part 7: Success Metrics

| Metric | Current | Target |
|--------|---------|--------|
| Agent file size | 850 lines | <200 lines |
| Correction rounds per category | 6-7 | 0-1 |
| Invented information instances | Multiple | 0 |
| Irrelevant questions | Multiple | 0 |
| Session time (medium complexity) | 2+ hours | <45 min |
| User satisfaction | Low | High |
| PRD accuracy | Requires heavy review | Matches transcript |

---

## Appendix A: Example Session Flow

```
Agent: [Creates session-transcript.md]

Agent: "Starting requirements session for your new operator.
        I'll ask 16 questions across 5 categories.
        I'll capture your exact words - no interpretation.

        Category 1: Operator Identity (1 question)

        Q1: What is the name of this operator?"

User: "config-sync-operator"

Agent: [Writes to transcript]
       "Captured:
        > "config-sync-operator"
        Correct?"

User: "Yes"

Agent: "✓ Category 1 complete (1/1)

        📋 Transcript Sync (Category 2)
        - Operator: config-sync-operator
        - Completed: Category 1

        Category 2: Problem & Purpose (3 questions)

        Q1: What problem does this operator solve?"

User: "We have ConfigMaps that teams share across namespaces. Right now
       someone has to manually copy them and remember to update all copies
       when the source changes. People forget and configs get out of sync."

Agent: [Writes to transcript]
       "Captured:
        > "We have ConfigMaps that teams share across namespaces. Right now
        > someone has to manually copy them and remember to update all copies
        > when the source changes. People forget and configs get out of sync."
        Correct?"

User: "Yes"

Agent: "Q2: What manual processes will it automate?"

[continues with fixed questions only...]
```

---

## Appendix B: What This Approach Does NOT Do

To set expectations correctly:

1. **Does not make the operator simpler** - Complex operators still need thorough requirements
2. **Does not reduce question count** - 16 questions is the minimum for complete requirements
3. **Does not prevent user error** - If user gives wrong info, PRD will have wrong info
4. **Does not add AI "insights"** - Agent won't suggest improvements or catch gaps
5. **Does not validate technical feasibility** - That's the spec-designer's job

**This approach trades "helpful AI" for "accurate capture"** - which is the right trade-off for requirements gathering.
