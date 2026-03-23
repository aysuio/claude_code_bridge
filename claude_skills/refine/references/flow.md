# Refine

Iterative quality-convergence loop for a scoped code target. Claude fixes, dual review checks, repeat until PASS or termination.

**Relationship to /tr**: `/tr` handles linear step progression. `/refine` handles iterative quality convergence within a single scope. A `/tr` step can invoke `/refine` internally.

---

## Input

User provides a natural-language description. Claude extracts:

| Field | How to resolve | Default |
|-------|---------------|---------|
| `goal` | From user message | (required — ask if truly unclear) |
| `scope` | From user message, or infer from goal (mentioned files/functions/modules) | Current conversation context |
| `acceptance_criteria` | Extract from goal (e.g. "all tests pass"), or infer obvious ones | Goal restated as checkable condition |
| `max_rounds` | Explicit `rounds=N` in message, or default | 2 |

**Only ask the user if `goal` is genuinely ambiguous.** Do not ask for structured fields — extract them.

Examples:
- `/refine 把 worker.py 的竞态条件修掉，所有测试通过` → goal, scope, criteria all clear
- `/refine 优化 lib/ 下的性能` → goal clear, scope clear, criteria vague but reasonable ("measurable perf improvement") — proceed
- `/refine 改一下代码` → goal unclear — ask "改什么？目标是什么？"

---

## State

Track internally (not persisted to disk). Reset on each `/refine` invocation.

```
round:            current round number (1-based)
status:           running | passed | stalled | max_rounds
fix_history:      array of { round, fix_list, verdict }
stall_counter:    consecutive rounds with no substantive changes
```

---

## Execution Flow

### 0. Parse Input

1. Extract `goal`, `scope`, `acceptance_criteria`, `max_rounds` from user message.
2. Only ask the user if `goal` is genuinely unclear. Otherwise, start immediately.
3. Brief echo before Round 1 (no confirmation needed):
   ```
   Refine: [goal]
   Scope: [scope]
   Criteria: [criteria]
   Max rounds: N
   ```

### 1. Execute Fix (Claude)

**Round 1**: Read all files in `scope`. Evaluate current state against `acceptance_criteria`. Make changes to address gaps.

**Round 2+**: Apply fixes from the normalized fix list produced in Step 3 of the previous round. Focus only on the fix items — do not re-examine areas already marked PASS.

After making changes, produce a brief change summary:
```
Round N changes:
- [file:line] description of change
- [file:line] description of change
```

### 2. Dual Review

#### 2.1 Resolve Reviewer

Default: `codex`. The reviewer can be overridden via `--reviewer` flag when calling `ccb-review`.

#### 2.2 Claude Assessment

Evaluate all files in `scope` against each item in `acceptance_criteria`:
- Which criteria are now met?
- Which criteria are still unmet? Why?
- Any new issues introduced by this round's changes?

Output:
```json
{
  "verdict": "PASS | FIX",
  "met": ["criteria that passed"],
  "unmet": ["criteria still failing"],
  "newIssues": ["issues introduced this round"],
  "fixItems": [
    { "file": "...", "issue": "...", "severity": "high|medium|low" }
  ]
}
```

#### 2.3 Cross-Review via ccb-review

**MANDATORY**: Use the `ccb-review` script for cross-review. Pipe your Claude assessment JSON via stdin. The script handles sending to the reviewer, waiting for response, and merging verdicts.

```
Bash(ccb-review "Refine round [N]: [goal]. Scope: [scope]. Criteria: [acceptance_criteria]." <<'VERDICT'
<your Claude assessment JSON from step 2.2>
VERDICT)
```

The script outputs merged verdict JSON to stdout. Parse it and proceed to Step 3 (Normalize Fix List).

### 3. Normalize Fix List

**This step is critical to prevent oscillation.**

If either review returned FIX, merge both fix lists:

1. **Deduplicate**: Same file + same issue = one entry. Keep higher severity.
2. **Resolve conflicts**: If Claude says PASS on an item but cross-reviewer says FIX (or vice versa), Claude makes the final call with a one-line reason logged.
3. **Cap**: Max 5 items per round. If more, keep only `high` and `medium` severity; drop `low`.
4. **Diff against previous round**: If a fix item appeared in the previous round's fix list and was supposedly addressed but reappears, flag it as `recurring` and escalate severity to `high`.

Output — the **normalized fix list**:
```json
{
  "round": N,
  "verdict": "PASS | FIX",
  "fixList": [
    { "file": "...", "issue": "...", "severity": "high|medium|low", "recurring": false }
  ],
  "conflictsResolved": [
    { "item": "...", "claudeVerdict": "...", "crossVerdict": "...", "resolution": "..." }
  ],
  "droppedLowSeverity": ["..."]
}
```

### 4. Termination Check

Evaluate termination conditions in order:

| Condition | Action |
|-----------|--------|
| Both verdicts PASS | Set `status = passed`. Go to Step 5. |
| `round >= max_rounds` | Set `status = max_rounds`. Go to Step 5. |
| `stall_counter >= 2` (no substantive changes for 2 consecutive rounds) | Set `status = stalled`. Go to Step 5. |
| Otherwise | Increment `round`. Go to Step 1 with normalized fix list. |

**Stall detection**: A round has "no substantive changes" if:
- The fix list is identical to the previous round's fix list (same files, same issues), OR
- Claude made zero code changes in Step 1

### 5. Final Report

Output a structured summary:

```
## Refine Result

**Status**: PASSED | STALLED | MAX_ROUNDS
**Rounds**: N / max_rounds
**Goal**: [goal]
**Scope**: [scope]

### Acceptance Criteria
- [x] criteria 1
- [x] criteria 2
- [ ] criteria 3 (unmet: reason)

### Round History
| Round | Verdict | Fix Items | Key Changes |
|-------|---------|-----------|-------------|
| 1     | FIX     | 3         | ...         |
| 2     | FIX     | 1         | ...         |
| 3     | PASS    | 0         | ...         |

### Remaining Issues (if not PASSED)
- [ ] issue description (severity)

### Conflicts Resolved
- Round N: [item] — Claude: X, Cross: Y → Resolution: Z
```

If `status != passed`, clearly state what remains unresolved so the user can decide next steps.

---

## Integration with /tr

A `/tr` step can use `/refine` by specifying it in the step description:

```
Step 3: "Refine lib/worker.py — eliminate race conditions"
  → /refine goal="..." scope="lib/worker.py" acceptance_criteria=[...] max_rounds=3
```

When invoked from `/tr`:
- PASS → `/tr` continues to finalize
- STALLED or MAX_ROUNDS → `/tr` treats as FIX with remaining issues as fix items

---

## Principles

1. **Explicit input**: Never infer goal or acceptance criteria — require them
2. **Normalized fixes**: Always merge, deduplicate, resolve conflicts before next round
3. **Bounded**: Hard cap via max_rounds + stall detection, no infinite loops
4. **Traceable**: Every round's verdict and fix list is logged for the final report
5. **No oscillation**: Recurring items get escalated; stall detection catches loops
6. **Claude leads**: Claude has final call on all conflict resolutions
