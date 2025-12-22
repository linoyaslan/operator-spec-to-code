# Edge Case Test for Verbatim Capture Protocol

## Purpose
Test if the new agent actually solves the original problems:
1. **Rephrasing user input** (user says X, agent writes Y)
2. **Inventing information** not discussed by user
3. **Asking irrelevant questions** outside fixed 16-question list
4. **Long sessions** (2+ hours with 6-7 correction rounds)

## Test Scenario: "Log Sync Operator"
User wants operator to sync log files between pods.

## Critical Test Inputs (Designed to Trigger Old Problems)

### Test 1: Vague Answer (Rephrasing Risk)
**Q1**: "What is the name of this operator?"
**User**: "something for logs"
**OLD BEHAVIOR**: Agent would rephrase to "log-management-operator" or expand
**EXPECTED NEW**: Capture exactly `> "something for logs"` and ask clarification

### Test 2: Technical Term (Language "Improvement" Risk)
**Q2**: "What problem does this operator solve?"
**User**: "pods create logs everywhere and devs can't find them"
**OLD BEHAVIOR**: "improves" to "container workloads generate distributed logs that developers cannot easily locate"
**EXPECTED NEW**: Capture exactly `> "pods create logs everywhere and devs can't find them"`

### Test 3: Minimal Answer (Info Addition Risk)
**Q3**: "What manual processes will it automate?"
**User**: "copying files"
**OLD BEHAVIOR**: Agent adds details "copying log files from source pods to centralized storage"
**EXPECTED NEW**: Capture exactly `> "copying files"` - no additions

### Test 4: Ambiguous Terms (Clarification Question Risk)
**Q4**: "What Custom Resource(s) will users create?"
**User**: "LogSync thingy"
**OLD BEHAVIOR**: Ask follow-up about "what kind of LogSync", "what properties", etc.
**EXPECTED NEW**: Capture exactly `> "LogSync thingy"` and continue to next fixed question

### Test 5: Unclear Answer (Question Invention Risk)
**Q5**: "What does one LogSync instance represent?"
**User**: "handles the syncing stuff"
**OLD BEHAVIOR**: Ask "what specific syncing operations?", "between which components?"
**EXPECTED NEW**: Capture exactly `> "handles the syncing stuff"` and continue

## Pass/Fail Criteria

### ✅ PASS = ALL of these:
- Agent captures ALL 5 test inputs EXACTLY as written (zero rephrasing)
- Agent asks ZERO questions outside the fixed 16-question list
- Session completes in under 30 minutes
- Agent never adds information user didn't provide
- User needs 0-1 corrections maximum

### ❌ FAIL = ANY of these:
- Agent rephrases even one answer (e.g., "pods" → "container workloads")
- Agent asks any clarifying questions beyond the fixed list
- Agent adds details not mentioned by user
- Session takes over 45 minutes
- Agent requires multiple correction rounds

## Expected Session Flow

```
Agent: Starting requirements session. I'll ask 16 questions across 5 categories.

Q1: What is the name of this operator?
User: something for logs
Agent:
Captured:
> "something for logs"
Correct?
User: yes
Agent: Q2: What problem does this operator solve?
User: pods create logs everywhere and devs can't find them
Agent:
Captured:
> "pods create logs everywhere and devs can't find them"
Correct?
User: yes
Agent: Q3: What manual processes will it automate?
...continues through fixed 16 questions only...
```

## Success Indicators
- Zero "Let me clarify..." responses
- Zero "I want to make sure I understand..." responses
- Zero expansion of user's exact words
- Zero follow-up questions beyond the fixed list
- Quick "Captured. Correct?" format consistently used
- Complete session in 20-30 minutes vs 2+ hours

## Test Execution
Run operator-requirements-engineer agent with the 5 test inputs above and measure results.