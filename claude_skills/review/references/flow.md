# Review

Dual review: Claude assesses first, then `ccb-review` script handles cross-review and verdict merging deterministically.

## Execution Flow

### 1. Claude Initial Assessment

Evaluate against the user's criteria:
- What was accomplished?
- Are all conditions met?
- Any issues or risks?

Output as JSON:
```json
{"verdict": "PASS|FIX|BLOCKED", "reason": "...", "fixItems": [{"file": "...", "issue": "...", "severity": "high|medium|low"}]}
```

### 2. Cross-Review via ccb-review

Pass Claude's verdict to `ccb-review` via stdin. The script:
- Sends cross-review request to reviewer (default: codex, override with `--reviewer`)
- Waits for response
- Merges both verdicts deterministically

```
Bash(ccb-review "review message" <<'VERDICT'
{"verdict":"...","reason":"...","fixItems":[...]}
VERDICT)
```

With scope:
```
Bash(ccb-review --scope "lib/foo.py" "review message" <<'VERDICT'
{"verdict":"...","reason":"...","fixItems":[...]}
VERDICT)
```

### 3. Report Result

The script outputs merged verdict JSON to stdout:

```json
{
  "verdict": "PASS|FIX|BLOCKED",
  "reason": "...",
  "claudeVerdict": "PASS|FIX|BLOCKED",
  "crossVerdict": "PASS|FIX|BLOCKED",
  "fixItems": [{"file": "...", "issue": "...", "severity": "high|medium|low"}]
}
```

Present the result to the user.

## Verdict Merge Rules

| Claude | Cross-review | Result |
|--------|-------------|--------|
| BLOCKED | * | → BLOCKED |
| * | BLOCKED | → BLOCKED |
| PASS | PASS | → PASS |
| PASS | FIX | → FIX |
| FIX | PASS | → FIX |
| FIX | FIX | → FIX |

## Principles

1. **Deterministic**: Verdict merging is code, not LLM interpretation
2. **Traceable**: Both verdicts preserved in output
3. **Simple**: Claude assesses, script does the rest
