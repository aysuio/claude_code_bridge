---
name: review
description: Dual review with Claude and Codex for quality verification. Use for step or task review.
---

# Review

Deterministic dual review via `ccb-review` script. Claude and cross-reviewer (default: codex) assess independently, results are merged by code.

## Execution (MANDATORY)

1. **Claude assesses first** (you do this yourself): Evaluate the code/changes against the criteria. Output your assessment as JSON:
   ```json
   {"verdict": "PASS or FIX", "reason": "...", "fixItems": [{"file": "...", "issue": "...", "severity": "high|medium|low"}]}
   ```

2. **Run `ccb-review` for cross-review + merge**: Pipe your verdict JSON via stdin using a heredoc (safe for quotes/newlines). The script sends to Codex, waits for response, and merges both verdicts deterministically.
   ```
   Bash(ccb-review "$ARGUMENTS" <<'VERDICT'
   <your JSON here>
   VERDICT)
   ```
   With scope:
   ```
   Bash(ccb-review --scope "lib/foo.py" "$ARGUMENTS" <<'VERDICT'
   <your JSON here>
   VERDICT)
   ```

3. **Report the result**: The script outputs a merged verdict JSON. Present it to the user.

## Rules

- When `/review` is triggered, you MUST execute the full 3-step flow above. No exceptions — regardless of whether the content is code changes, data analysis, architecture decisions, or any other topic.
- NEVER skip `ccb-review` or do the review purely by yourself. The cross-review is mandatory.
- ALWAYS use `ccb-review` for the cross-review step. NEVER call providers yourself.
- The script handles: sending to reviewer, waiting for response, merging verdicts.
- You only do your own assessment, then hand off to the script.
