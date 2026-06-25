---
name: conclave
description: "Two-round debate audit. The current agent arbitrates while the other AI CLIs (from Claude, Codex, Gemini, Grok) audit in parallel — round 1 independently at max effort, round 2 each re-judges seeing all round-1 verdicts. The arbiter synthesizes every output, then a GPT-5.5 historian scores each model and appends the run to a history log. Heavier than tribunal — for the hardest security/architecture/pre-prod calls. Triggers: /conclave, 'deep tribunal', 'two-round audit', 'debate audit'."
---

# Conclave

Two-round debate audit with a final synthesis and a model-scoring historian. The deep sibling of `tribunal`: where tribunal runs one round, conclave runs two — independent audit, then cross-examination where each auditor re-judges after seeing the others.

- **Round 1** — every non-runtime provider audits the same input independently, at max effort.
- **Round 2** — each provider re-judges after reading all round-1 verdicts (agree, overturn, or extend).
- **Synthesis** — the current agent (arbiter) reads all outputs and renders the verdict.
- **Historian** — GPT-5.5 scores each model on the run and appends one JSON line to a history log, so you can track which models audit best over time.

The point is independence. **Never use the current runtime as one of its own auditors** — it is the arbiter only. Conclave is always max effort; there is no standard mode.

## Prerequisites

The other providers' CLIs (everything except the one you are running as host):

| Provider | CLI binary | Auth check |
|----------|-----------|------------|
| Claude | `claude` | `claude --version` |
| Codex | `codex` | `codex --version` |
| Gemini | `gemini` | `gemini --version` |
| Grok | `grok` | `grok models` |

Each must already be logged in. Conclave does not start interactive auth. GPT-5.5 for the historian runs through `codex`, so a Codex login is required for the history step regardless of host.

## Protocol

### 1. Input

```bash
/conclave path/to/file.py   # file review
/conclave                    # uncommitted changes (git diff)
```

File mode prepends the file content; diff mode prepends `git diff`. All providers receive identical input; the arbiter adds no opinion to any auditor prompt.

### 2. Select auditors

Detect the current runtime, then audit with **all other providers** (Grok joins here — unlike tribunal, conclave is a deep pass and uses everyone).

| Current runtime | Auditors |
|-----------------|----------|
| Claude Code | Codex + Gemini + Grok |
| Codex | Claude + Gemini + Grok |
| Gemini | Claude + Codex + Grok |
| Grok | Claude + Codex + Gemini |

If a CLI is unavailable, proceed with the rest and note the degraded panel in the verdict. Do not substitute the arbiter for a missing auditor.

### 3. Auth preflight

```bash
which claude codex gemini grok
claude --version && codex --version && gemini --version && grok models
```

If a CLI is missing, asks for login, opens OAuth, or hangs — stop and report the exact blocker. Do not silently downgrade.

### 4. Round 1 — independent audit (parallel, max effort)

Send every auditor the same `R1_PROMPT`. Capture each output separately (`/tmp/conclave-r1-<provider>.txt` or captured stdout). Read none until all finish.

```text
You are conducting an independent code audit. Review for:
- Security (injection, auth bypass, data leaks, insecure crypto)
- Correctness (logic errors, edge cases, off-by-one, null handling)
- Performance (N+1 queries, inefficient algorithms, memory leaks)
- Maintainability (complexity, coupling, unclear contracts)

VERDICT: [APPROVE | CONCERNS | REJECT]
SEVERITY: [CRITICAL | HIGH | MEDIUM | LOW | NONE]
FINDINGS: [specific issues with file:line]
REASONING: [why this verdict, what patterns led to it]

CODE TO REVIEW:
[content]
```

Launch each auditor through `ask` — it writes the prompt to a file (immune to ARG_MAX and shell metacharacters when the input embeds code), retries once on an empty reply, and degrades the panel cleanly instead of handing the arbiter a blank verdict. Run in parallel when the host supports it.

```bash
PF=$(mktemp); printf '%s' "R1_PROMPT" > "$PF"

ask() {  # ask LABEL -- cmd...   (prompt is the file $PF)
  local label="$1"; shift 2; local out
  for _ in 1 2; do
    out="$("$@" 2>/dev/null)"
    [ -n "${out//[$' \t\n']/}" ] && { printf '%s\n' "$out"; return 0; }   # retry once on empty
  done
  printf 'DEGRADED: %s returned no output\n' "$label"; return 1            # never hand back a blank verdict
}

R1_CLAUDE=$(ask claude -- claude -p --model opus --effort high --permission-mode plan --tools "" --no-session-persistence < "$PF")
R1_CODEX=$(ask codex   -- codex exec -c model_reasoning_effort="xhigh" --sandbox read-only --full-auto --skip-git-repo-check "$(cat "$PF")" < /dev/null)
R1_GEMINI=$(ask gemini -- gemini -p "$(cat "$PF")" -m gemini-3.1-pro-preview -e none -o text < /dev/null)
R1_GROK=$(ask grok     -- grok -p "$(cat "$PF")" --tools "read_file,grep,list_dir" --output-format plain < /dev/null)
```

`claude` takes the prompt on stdin (`< "$PF"`); the others get it as an argument via `$(cat "$PF")` and read `< /dev/null` so none can block on an open stdin. Quirks: if Opus is unavailable, use `--model sonnet --effort high` and disclose it. Never pass `--effort` to Grok (`grok-build` rejects `reasoningEffort` with HTTP 400); keep its `--tools` read-only.

### 5. Round 2 — cross-examination (parallel, max effort)

After round 1 completes, send each auditor the **same packet**: the original code plus all round-1 verdicts, labeled `AUDITOR 1..N` — never as "the strong one" (that biases capitulation). Rebuild `$PF` with `R2_PROMPT` and relaunch each auditor through `ask` (same commands as round 1). Store as `R2_<PROVIDER>`.

```text
Several independent auditors reviewed the code below. Read all of their verdicts, then render your own final judgment — agree, overturn, or extend.

VERDICT: [APPROVE | CONCERNS | REJECT]
SEVERITY: [CRITICAL | HIGH | MEDIUM | LOW | NONE]
CONFIRMED: [findings you uphold, with file:line]
REJECTED: [findings you discard, with why]
MISSED: [blockers none of them caught]
REASONING: [...]

CODE:
[content]

AUDITOR 1:
[R1 output A]
AUDITOR 2:
[R1 output B]
AUDITOR 3:
[R1 output C]
```

### 6. Arbiter synthesis

Read all round-1 and round-2 outputs. Weight round 2 (informed) over round 1, but use round 1 to catch findings a provider dropped after seeing the others (groupthink / capitulation). Any blocker confirmed by ≥1 round-2 verdict **and** independently verified by the arbiter against the actual code path blocks. A lone REJECT is not auto-blocking — verify its line references before discounting it. The arbiter may add its own verified findings, labeled separately.

```text
CONCLAVE VERDICT

Runtime: [Claude | Codex | Gemini | Grok] | Panel: [N/N | degraded M/N]
Round 1 — [A]:[V](S)  [B]:[V](S)  [C]:[V](S)
Round 2 — [A]:[V](S)  [B]:[V](S)  [C]:[V](S)
SHIFTS: [who changed verdict R1→R2 and why — capitulation vs genuine update]

ARBITER DECISION: [APPROVE | APPROVE WITH CONDITIONS | REJECT | BLOCKED]
REASONING: [...]
KEY ISSUES: [with line references]
REQUIRED ACTIONS: [what must be fixed]
BLOCKERS: [auth/tooling blockers; note degraded panel if a provider was down]
```

### 7. Historian — GPT-5.5 scores the run (final step, always)

Give GPT-5.5 the target, every auditor's round-1 and round-2 output, and the arbiter's decision. It scores each model and emits **one JSON line**, which you append to the history log. This tracks model quality over time — never skip it.

```bash
HIST="${CONCLAVE_HISTORY:-$HOME/.conclave/history.jsonl}"
mkdir -p "$(dirname "$HIST")"
codex exec -c model="gpt-5.5" -c model_reasoning_effort="high" \
  --sandbox read-only --full-auto --skip-git-repo-check "HIST_PROMPT" \
  | tail -1 >> "$HIST"
```

`HIST_PROMPT` (fill the bracketed data; demand a single-line JSON object, no prose):

```text
You are the historian for a multi-model code audit. Using the arbiter's verified decision as ground truth, score each model on this run. Output ONLY one minified JSON line, no markdown, matching:
{"ts":"<ISO8601>","skill":"conclave","target":"<path|git diff>","arbiter_runtime":"<runtime>","arbiter_decision":"<decision>","panel":["<model>",...],"scores":{"<model>":{"accuracy":0-10,"blockers_found":N,"false_positives":N,"false_negatives":N,"calibration":0-10,"r1_to_r2":"held|capitulated|sharpened","notes":"<short>"}},"best_model":"<name>","worst_model":"<name>","summary":"<one line>"}
TARGET: [target]
ARBITER DECISION: [decision + key verified blockers]
MODEL OUTPUTS (round 1 then round 2 per model):
[A r1]/[A r2] ... [C r1]/[C r2]
```

If the account has no `gpt-5.5`, fall back to the strongest available Codex model at `model_reasoning_effort="high"` and note the substitution in the record's `summary`. Confirm the appended line is valid JSON; if GPT-5.5 emitted prose, re-run demanding JSON only. Report the history path and `best_model` in the closing summary.

Read the accumulated ranking any time:

```bash
HIST="${CONCLAVE_HISTORY:-$HOME/.conclave/history.jsonl}"
jq -rs '
  . as $all | ([ $all[].scores | keys[] ] | unique) as $m
  | [ $m[] as $k | [ $all[] | select(.scores[$k]) | .scores[$k] ] as $r
      | {model:$k, runs:($r|length),
         acc:(($r|map(.accuracy)|add)/($r|length)*100|round/100),
         calib:(($r|map(.calibration)|add)/($r|length)*100|round/100),
         blockers:($r|map(.blockers_found)|add), fp:($r|map(.false_positives)|add)} ]
  | sort_by(-.acc)
  | (["MODEL","runs","acc","calib","blockers","FP"]|@tsv), (.[]|[.model,.runs,.acc,.calib,.blockers,.fp]|@tsv)
' "$HIST" | column -t
```

### 8. Independence guarantee

Auditors must not see each other's round-1 output until round 2 (where it is given to all equally). Run each round's auditors in parallel when the host can. Use separate captured outputs or temp files. The arbiter reads outputs only after each round completes. Do not reuse the current agent as an external auditor.

## When to use

The hardest calls where one pass is insufficient: production auth/payment/crypto, irreversible migrations, privilege-escalation paths, wide-blast-radius architecture decisions, pre-production deploy of security-sensitive code. Conclave costs ~6 audit calls + synthesis + the historian — for routine pre-merge review, use `tribunal` instead. Not for formatting, typos, docs, or config.

## Anti-patterns

Don't: let the arbiter submit a round (it synthesizes only); read round-1 outputs before all auditors finish; attribute round-1 verdicts as "the strong one" in the round-2 packet; skip round 2; auto-approve on unanimous round-1 APPROVE (round 2 exists to surface what consensus missed); treat round-2 capitulation as agreement without rechecking the dropped finding; pass `--effort` to Grok; skip the historian step.
