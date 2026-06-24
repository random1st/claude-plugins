# groundwork

Personal Claude Code plugin marketplace — `random1st/groundwork`.

Five plugins that turn a single AI session into a calibrated one — independent multi-model audit, a deeper two-round debate audit, cross-CLI delegation, a reasoning baseline, and an evidence-before-claim verification gate. Plus a project-agnostic `AGENTS.md` template that documents the discipline behind them.

No coupling to any personal runtime. Each plugin uses raw CLIs (`claude`, `codex`, `gemini`, `grok`) or standard Claude Code conventions.

## Install

```text
/plugin marketplace add random1st/groundwork
/plugin install <plugin-name>@groundwork
```

Update later:

```text
/plugin marketplace update groundwork
```

## Plugins

### [tribunal](plugins/tribunal/)

Independent code review by two other AI CLIs. Whichever CLI you are running stays as the arbiter; the two others (any combo of Claude, Codex, Gemini) audit the code in parallel and you read both verdicts before deciding. Use for pre-merge reviews, security-critical code, and changes you don't want to ship alone.

### [conclave](plugins/conclave/)

The deep sibling of `tribunal` — two rounds instead of one. Every other CLI (Claude, Codex, Gemini, Grok) audits independently at max effort, then each one re-judges after reading every round-1 verdict; the arbiter synthesizes all of it. A closing GPT-5.5 historian scores each model on the run and appends one JSON line to a history log, so over time you can see which models actually audit best. For the hardest calls — auth, payments, crypto, irreversible migrations, pre-prod deploys — where a single pass isn't enough.

### [delegate](plugins/delegate/)

Routing matrix for sending work to another AI CLI. Tells you which model and which reasoning effort to use for each task — code review, architecture analysis, whole-project scans, quick documentation. Uses raw `codex exec` and `gemini` binaries; no wrapper required.

### [fpf](plugins/fpf/)

Reasoning baseline plus a local builder for the full FPF corpus. The `SKILL.md` ships the always-loaded operational subset — ADI cycle, calibration tags, I/D/S discipline, weakest-link heuristic. A bundled `refresh.sh` script pulls the upstream [`ailev/FPF`](https://github.com/ailev/FPF) spec on first use and splits it into navigable modules, cards, and a relation graph; re-runs are a silent no-op when upstream hasn't moved (caches the commit SHA). No FPF text is bundled — every user builds their own derivative locally against the latest upstream. Requires `python3` (stdlib only) and `curl`.

### [evidence-gate](plugins/evidence-gate/)

Refuses to claim completion without proof. Identifies the command that would prove the claim, runs it fresh, reads the actual output, and reports PASS or FAIL with the evidence. Companion to the tag discipline in `fpf` — the gate that makes `[OBSERVED]` tags trustworthy.

## Template: AGENTS.md

[`templates/AGENTS.md`](templates/AGENTS.md) is one canonical instructions file usable by any agentic CLI. Same content, just copy to wherever your CLI looks:

- Codex → `~/.codex/AGENTS.md` or `<repo>/AGENTS.md`
- Claude Code → `~/.claude/CLAUDE.md` or `<repo>/CLAUDE.md`
- Gemini → `~/.gemini/GEMINI.md` or `<repo>/GEMINI.md`

It includes the universal core (Constitution, Calibration, FPF, Protocol) plus a compact CLI Quick Reference covering native mechanics of all three CLIs.

## Prerequisites

`tribunal`, `conclave`, and `delegate` shell out to the other providers' CLIs. Install and authenticate the ones you use before invoking:

- [`codex`](https://github.com/openai/codex)
- [`gemini`](https://github.com/google-gemini/gemini-cli)
- `grok` (the grok.com CLI; `grok models` confirms login)

`conclave` additionally runs `gpt-5.5` through `codex` for its scoring historian, and reads back the log with `jq`. `evidence-gate` is pure guidance — no external tooling needed. `fpf` ships a builder for its corpus but no external tooling beyond `python3` and `curl`.

## Layout

```text
.
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   ├── tribunal/
│   ├── conclave/
│   ├── delegate/
│   ├── fpf/
│   └── evidence-gate/
└── templates/
    └── AGENTS.md
```

Each plugin is a single `SKILL.md` plus `.claude-plugin/plugin.json`.

## Related

- **[`ailev/FPF`](https://github.com/ailev/FPF)** — Anatoly Levenchuk's original First Principles Framework spec, which the `fpf` plugin distils into an operational subset.
- **[`m0n0x41d/claude-code-fpf`](https://github.com/m0n0x41d/claude-code-fpf)** — a separate public Claude Code skill that ships the full FPF corpus as an embedded SQLite/FTS5 index with a `fpf-rag` search binary. Install alongside this marketplace's `fpf` plugin if you want deep on-demand search over the spec text.

## License

MIT.

