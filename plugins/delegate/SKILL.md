---
name: delegate
description: "Unified external AI delegation — route tasks to Codex, Gemini, or Grok based on cost/quality tradeoffs. Use when user says 'delegate', 'codex review', 'ask gemini', 'ask grok', 'second opinion', or needs external model."
---

# Delegate — Unified External AI Delegation

Route tasks to the best external AI tool: **Codex CLI**, **Gemini CLI**, or **Grok CLI** via their native binaries.

Prerequisites: `codex`, `gemini`, and `grok` already installed and authenticated locally.

---

## Routing Matrix

| Task | Tool | Command | Effort |
|------|------|---------|--------|
| Code review (uncommitted) | Codex | `codex exec` on `git diff` | medium |
| Code review (vs branch) | Codex | `codex exec` on `git diff <branch>` | medium |
| Deep code review / tech debt | Codex | `codex exec` | xhigh |
| Architecture / design review | Codex | `codex exec` | xhigh |
| Refactoring strategy | Codex | `codex exec` | xhigh |
| Tech debt analysis | Codex | `codex exec` | xhigh |
| Security audit | Codex | `codex exec` | high |
| Second opinion on my code | Codex | `codex exec` | medium |
| Plan validation | Codex | `codex exec` | xhigh |
| Bug investigation | Codex | `codex exec` | xhigh |
| Large codebase analysis | Gemini | `gemini --all-files` | — |
| Full project scan / patterns | Gemini | `gemini --all-files` | — |
| Deep architecture analysis | Gemini | `gemini --all-files -m gemini-3.1-pro-preview` | — |
| File review (>200 lines) | Gemini | `cat FILE \| gemini -p` | — |
| Documentation / summary | Gemini | `gemini -p` | — |
| Rust code review | Codex | `codex exec` | high |
| Quick second opinion (free) | Grok | `grok -p` | — |
| Web-aware code question | Grok | `grok -p` | — |
| Third independent opinion | Grok | `grok -p` | — |

**When multiple tools fit:** prefer cheapest → Gemini (free tier) / Grok (free, grok.com) > Codex (paid).
**When quality matters most:** GPT-5.4 (xhigh) > Codex 5.3 (xhigh) > Gemini 3.1 Pro.
**Rust-specific:** Codex is best for ownership, lifetimes, concurrency reasoning.
**Grok niche:** free via grok.com, agentic, **built-in web search** (Codex has none) — good for web-aware questions and a cheap third opinion.

---

## 1. Codex CLI

### Non-interactive execution

```bash
# Read-only task
codex exec \
  -c model_reasoning_effort="EFFORT" \
  --sandbox read-only \
  --full-auto \
  --skip-git-repo-check \
  "PROMPT"

# Write-capable task (Codex modifies files)
codex exec \
  -c model_reasoning_effort="EFFORT" \
  --sandbox workspace-write \
  --full-auto \
  "PROMPT"
```

**Flags reference:**
- `-c model_reasoning_effort="..."` — `minimal` | `low` | `medium` | `high` | `xhigh`
- `--sandbox read-only|workspace-write` — control file modification scope
- `--full-auto` — auto-accept tool calls (non-interactive)
- `--skip-git-repo-check` — skip the safety check when running outside a git repo

**Reasoning effort auto-select:**

| Task complexity | Effort |
|----------------|--------|
| Trivial (format, explain) | low |
| Standard (review, check) | medium |
| Complex (debug, investigate) | high |
| Critical (architecture, security) | xhigh |

### Code review against branch

```bash
# Pipe diff into Codex
git diff main..HEAD | codex exec \
  -c model_reasoning_effort="medium" \
  --sandbox read-only \
  --full-auto \
  --skip-git-repo-check \
  "Review this diff for correctness, security, and maintainability issues. Cite file:line."
```

### Codex prompt template

Structure prompts for Codex with **mandatory goal articulation** before execution:

```
TASK: [One sentence, atomic goal]
EXPECTED OUTCOME: [What the output should look like]
CONTEXT: [Relevant code, state, background — paste snippets]
CONSTRAINTS: [Technical limits, patterns to follow]
MUST DO: [Non-negotiable requirements]
MUST NOT DO: [Anti-patterns to avoid]
OUTPUT FORMAT: [How to structure the response]

Before executing: restate the goal in your own words. Identify implicit constraints and assumptions. Only then proceed.
```

For simple tasks, just TASK + CONTEXT is enough. Scale up sections with complexity.
The "Before executing" line is **always included** — even in minimal prompts.

---

## 2. Gemini CLI

### Headless mode

```bash
# Simple prompt
gemini -p "PROMPT" -m gemini-3-flash-preview -e none -o text

# Full project analysis (1M context window)
gemini --all-files -p "PROMPT" -m gemini-3-flash-preview -e none

# Pipe file content
cat src/auth.py | gemini -p "Review for security issues" -m gemini-3-flash-preview -e none
```

**Flags reference:**
- `-p "PROMPT"` — **required for headless** (non-interactive mode)
- `-m MODEL` — model selection
- `-e none` — **always use** — disables MCP extensions, saves 2-3s startup
- `-o text` — output format: `text` (default), `json`, `stream-json`
- `--all-files` or `-a` — include ALL project files in context (uses 1M window)
- `--include-directories DIR1,DIR2` — scope to specific dirs
- `-y` / `--yolo` — auto-accept all actions
- `--approval-mode plan` — read-only mode (safest)

**Model selection:**

| Model | Use for | Context | Speed |
|-------|---------|---------|-------|
| `gemini-3-flash-preview` | **Default** — most tasks, fast | 1M | Fast |
| `gemini-3.1-pro-preview` | Deep analysis, complex reasoning | 1M | Slower |
| `gemini-3.1-flash-lite-preview` | Budget tasks, high volume | 1M | Fastest |

**Gemini prompt style:** Keep prompts **direct and specific**. Gemini works best with a clear task statement, specific things to look for, and desired output structure. Avoid verbose 7-section format — Gemini prefers concise instructions.

**Goal articulation for Gemini:** Append to every prompt: `First, restate the goal and key constraints in one sentence, then proceed.` This is lightweight enough for Gemini's style while forcing structured reasoning.

---

## 3. Grok CLI

Agentic coding CLI (xAI), free via grok.com login. Full tool use and **built-in web search** — the only CLI here with native web access. Best as a fast free third opinion or for web-aware questions.

### Headless mode

```bash
# Read-only (review / second opinion — no shell, edit, or web)
grok -p "PROMPT" --tools "read_file,grep,list_dir" --output-format plain

# Web-aware task (web_search / web_fetch on by default)
grok -p "PROMPT" --output-format plain

# JSON output for parsing
grok -p "PROMPT" --output-format json
```

**Flags reference:**
- `-p "PROMPT"` — **required for headless** (alias `--single`)
- `-m MODEL` — `grok-build` (default) or `grok-composer-2.5-fast`
- `--tools "read_file,grep,list_dir"` — allowlist; keeps the run read-only
- `--disallowed-tools "..."` — denylist (e.g. `run_terminal_cmd,search_replace,web_search,web_fetch`)
- `--allow RULE` / `--deny RULE` — `ToolPrefix(glob)` permission gates (e.g. `--deny "Bash(rm*)"`)
- `--output-format plain|json|streaming-json` — `plain` default
- `--always-approve` — auto-approve tool executions (write tasks)
- `-s ID` / `-r ID` / `-c` — named session / resume / continue

**⛔ No `--effort`** — `grok-build` rejects `reasoningEffort` with HTTP 400; the model has no effort knob.

**Models:**

| Model | Use for |
|-------|---------|
| `grok-build` | **Default** — agentic coding, review, second opinion |
| `grok-composer-2.5-fast` | Lighter / faster tasks |

**Grok prompt style:** accepts structured prompts like Codex. Keep a `Before executing: restate the goal…` line. For review / second-opinion always pass `--tools "read_file,grep,list_dir"` to prevent edits, shell, and web. Single grok.com account — `grok models` checks login; `grok login` is interactive.

---

## 4. Model Selection Guide

GPT-5.3 Codex and Opus 4.6 have converged in capability. GPT-5.4 stands apart for deep work.

| Task Type | Model | Effort | Why |
|-----------|-------|--------|-----|
| Planning & architecture | GPT-5.4 (Codex) | xhigh | Most thorough, raises corner cases, prevents arch drift |
| Refactoring strategy | GPT-5.4 (Codex) | xhigh | Best at minimizing tech debt |
| Bug investigation (complex) | GPT-5.4 (Codex) | xhigh | Superior context gathering, finds non-trivial root causes |
| Verification & review | GPT-5.4 (Codex) | xhigh | Catches what other models miss, "pedantic" but correct |
| Implementation of plans | GPT-5.3 Codex | high-xhigh | Fast execution, good at following instructions |
| Quick code review | GPT-5.3 Codex | medium | Fast turnaround for standard reviews |
| Interactive fast work | GPT-5.3 Codex | medium | Best speed/quality for conversational coding |
| Documentation / summary | Gemini Flash | — | Cheap, fast, 1M context |
| Whole-project pattern scan | Gemini Flash (`--all-files`) | — | 1M window beats everything for breadth |

**GPT-5.4 traits to expect:**
- Slow start (10-20 min context gathering) — this is a feature, not a bug
- Will flag corner cases and impossibilities — feels "pedantic" but catches real issues
- More agentic — can autonomously drive multi-step tasks to completion
- Best at preventing architectural drift and reducing tech debt

**When NOT to use GPT-5.4:**
- Quick questions or simple tasks (use 5.3 Codex low)
- Time-sensitive interactive work (use 5.3 Codex)
- Tasks that don't benefit from deep reasoning (use Gemini flash)

---

## 5. Error Handling

- Codex timeout (>60s) — retry with lower `-c model_reasoning_effort=...`
- Codex auth issues — run `codex login` (or your install's documented login flow)
- Gemini fails — try without `-e none`, or different model
- Gemini opens browser OAuth in headless — stop and ask user to authenticate first
- Grok `400 … does not support parameter reasoningEffort` — remove `--effort` (grok-build has no effort knob)
- Grok asks to log in / hangs on OAuth — run `grok login`, then retry
