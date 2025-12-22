# Plan: Verbatim Capture Protocol

## Problem Statement

Current agent behavior causes these issues:
1. **Rephrases user input** - Changes words even when user is already high-level
2. **Multiple correction rounds** - 6-7 iterations to fix agent's interpretations
3. **Invents information** - Makes up relationships, details user never mentioned
4. **Asks irrelevant questions** - Goes off-script with tangential questions
5. **Time consuming** - 2+ hours for a single session

**Root cause**: Agent tries to "help" by interpreting, summarizing, and improving. More instructions don't fix this - it's fundamental LLM behavior.

## Solution: Verbatim Capture Protocol

### Core Principles

1. **Capture, Don't Interpret**
   - Store user's exact words in quote blocks
   - Zero rephrasing, zero summarization
   - User reviews their own words, not agent's interpretation

2. **Fixed Interview Protocol**
   - Predefined question list per category
   - No improvisation, no tangents
   - Agent cannot invent new questions

3. **Quick Verify**
   - After each answer: "Captured. Correct?" (yes/no)
   - No read-back of summaries
   - No interpretation to dispute

4. **Session Transcript**
   - Simple Q&A format
   - Verbatim quotes only
   - PRD generated from transcript

---

## How Each Problem Is Solved

### Problem 1: Rephrases things
**Before**: Agent writes "provisions cluster resources" when user said "creates the pods"
**After**: Agent captures `> "creates the pods"` - exact words, no change

### Problem 2: 6-7 correction rounds
**Before**: User corrects agent's interpretation, agent re-interprets wrong again
**After**: Nothing to correct - it's the user's own words

### Problem 3: Invents things
**Before**: Agent deduces CR relationships user never mentioned
**After**: Agent only asks fixed questions, only captures what user says

### Problem 4: Irrelevant questions
**Before**: "What targets the operator to configure ingress?" (never mentioned)
**After**: Agent cannot ask questions outside the fixed list

### Problem 5: Time consuming
**Before**: 2+ hours with corrections and tangents
**After**: 30-60 minutes (complex operators still need time, but no wasted time)

---

## New Agent Structure

### File Size
- **Before**: ~850 lines
- **After**: ~150 lines

### Agent Behavior

```
1. Create session-transcript.md
2. For each category:
   a. Ask question from fixed list
   b. Capture answer verbatim: > "user's exact words"
   c. Quick verify: "Captured. Correct?"
   d. Next question (no improvisation)
3. Generate PRD from transcript
```

### Fixed Question Protocol

```markdown
## Category 1: Operator Identity
Q1: "What is the name of this operator?"

## Category 2: Problem & Purpose
Q1: "What problem does this operator solve?"
Q2: "What manual processes will it automate?"
Q3: "Who are the users of this operator?"

## Category 3: Custom Resources
Q1: "What Custom Resource(s) will users create?"
Q2: "For each CR, what does it represent?"
Q3: "What are the main lifecycle states for each CR?"

## Category 4: Operator Responsibilities
Q1: "What does the operator do when a CR is created?"
Q2: "What does the operator do when a CR is updated?"
Q3: "What does the operator do when a CR is deleted?"
Q4: "What external systems does the operator interact with?"

## Category 5: Success & Deployment
Q1: "How will users know the operator is working correctly?"
Q2: "How will the operator be deployed?"
```

### Capture Format

```markdown
## Session Transcript

### Category 1: Operator Identity

**Q1: What is the name of this operator?**
> "config-sync-operator"
✓ Verified

### Category 2: Problem & Purpose

**Q1: What problem does this operator solve?**
> "ConfigMaps need to be synced across namespaces. Right now developers manually copy them and forget to update when source changes."
✓ Verified

**Q2: What manual processes will it automate?**
> "Copying ConfigMaps from source namespace to targets, and keeping them in sync when source updates"
✓ Verified
```

---

## Terminology (Distinct Naming)

| Concept | Name | Description |
|---------|------|-------------|
| Core protocol | **Verbatim Capture** | Capture exact user words |
| Question structure | **Fixed Interview Protocol** | No improvisation |
| Answer format | **Quote Blocks** | `> "exact words"` |
| Confirmation | **Quick Verify** | Simple yes/no, no read-back |
| Storage file | **Session Transcript** | Q&A record |
| Context refresh | **Transcript Sync** | Read transcript at checkpoints |

---

## Implementation Steps

### Phase 1: Create New Agent
- [ ] Create `operator-requirements-engineer.md` (~150 lines)
- [ ] Define Fixed Interview Protocol (all questions)
- [ ] Define Verbatim Capture format
- [ ] Define Quick Verify behavior

### Phase 2: Update Supporting Files
- [ ] Update `.gitignore` for `session-transcript.md`
- [ ] Remove old complex agent file

### Phase 3: Test
- [ ] Run through complete interview
- [ ] Verify no rephrasing occurs
- [ ] Verify no invented questions
- [ ] Measure time vs old approach

### Phase 4: Update Other Agents
- [ ] Apply similar principles to spec-designer
- [ ] Apply similar principles to implementer
- [ ] Apply similar principles to test-engineer

---

## Expected Outcomes

| Metric | Before | After |
|--------|--------|-------|
| Agent file size | 850 lines | 150 lines |
| Correction rounds | 6-7 | 0-1 |
| Invented information | Frequent | None |
| Irrelevant questions | Frequent | None |
| Session time | 2+ hours | 30-60 min |
| User frustration | High | Low |

---

## Risks & Mitigations

### Risk: Too rigid for complex operators
**Mitigation**: Add optional "clarifying question" slot after each answer (but agent cannot invent topics)

### Risk: User gives incomplete answers
**Mitigation**: Agent can ask "Anything else about [topic]?" but cannot assume or invent

### Risk: PRD quality depends on user input quality
**Mitigation**: This is actually correct - PRD should reflect what user said, not what agent imagined

---

## Success Criteria

1. Zero rephrasing in captured answers
2. Zero invented information in PRD
3. All questions from fixed list only
4. Session time under 60 minutes
5. User corrections reduced to 0-1 per category
