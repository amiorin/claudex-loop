---
name: codex-review
description: 'Run a standalone adversarial review of an existing implementation plan before code is written. Claude drafts or loads PLAN.md; OpenAI Codex critiques it without writing and returns APPROVED or REVISE; Claude revises and resumes the same Codex session until approval or MAX_ROUNDS; then the human signs off. Use when the user already has a plan or clear idea and says "/codex-review", "codex review my plan", "have Codex review my plan", "argue this plan with Codex", "adversarial plan review", or "make Claude and Codex fight over the plan". Also use for high-stakes plans involving auth, schemas, concurrency, migrations, or payments when a second-model check is wanted. For requirements discovery first, use /claudex-loop. For existing code, use /codex:review. Do not use for trivial changes.'
---

# Codex-Review — Adversarial Plan-Review Loop

Two models, one plan, a bounded argument. **Claude is the builder and orchestrator. Codex is the critic** — it reads the repo and the plan and is instructed to critique without touching a file (it runs unsandboxed; this machine is the sandbox). They communicate strictly through `PLAN.md` + a Codex session that persists across rounds. The human enters at exactly two points: kickoff and final sign-off.

This is a **deliberate, high-stakes tool** — reach for it on auth, data models, concurrency, migrations, payments, anything expensive to get wrong. Skip it for obvious/cheap work.

## Prerequisites (verify once, fast)

- Codex CLI installed and recent: `codex --version` (need ≥ 0.130; the default `gpt-5.5` model errors on older CLIs).
- Codex authenticated: a prior `codex login` (ChatGPT account is fine). If a run returns an auth/model error, surface it to the user — do not silently retry.
- Model is pinned on every call: `--model gpt-5.6-sol -c service_tier=fast`. (`gpt-5.x-codex` variants still fail on ChatGPT-account auth; `gpt-5.6-sol` does not — verified end-to-end, exec + resume, on codex-cli 0.147.0, 2026-08-26.)
- **Echo the active model before Round 1** so the user can confirm — `gpt-5.6-sol (service_tier=fast, pinned by the skill)` plus the CLI version; state it with the resolved tunables. If the user objects, stop before burning a round.
- **Codex runs unsandboxed** — `--dangerously-bypass-approvals-and-sandbox` on both `codex exec` and `codex exec resume`, on the premise that the host machine is itself the sandbox. `resume` rejects `-s` ("unexpected argument") but accepts the bypass and `--model` flags, so spell the full set out every round instead of letting it inherit `config.toml`. Consequence worth naming: the read-only discipline is a *prompt* constraint, not a kernel one — keep `Do NOT modify any files` in the review prompt, and if a round looks suspicious, check `git status` before trusting the verdict.

## Tunable variables (read from skill args, else default)

| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop ALWAYS terminates at this. |
| `PLAN_FILE` | `PLAN.md` | Where the evolving plan lives (repo root). |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only transcript of the argument (every round's critique + what changed). The artifact. |

If the user invoked the skill with an argument like `rounds=3`, use that for `MAX_ROUNDS`. Echo the resolved values back before starting.

## Flow

### Step 0 — Kickoff (human gate #1)

The invocation itself is the kickoff. Confirm scope in one line: what is being planned. If the user gave no task, ask for it (one question). Then proceed — do NOT ask for approval round-by-round; that comes at the end.

### Step 1 — Claude plans

Do real planning: read the relevant code, think through the approach, surface decisions and tradeoffs. Then write the plan to `PLAN_FILE` in this structure:

```markdown
# Plan: <task>
_Round 0 — initial draft by Claude_

## Goal
<one paragraph>

## Approach
<numbered steps, concrete>

## Key decisions & tradeoffs
<the contestable choices — name them explicitly so Codex has something to bite>

## Risks / open questions
<what you're unsure about>

## Out of scope
<bounds>
```

Initialize `LOG_FILE`:
```markdown
# Plan Review Log: <task>
Started <stamp the user's local time if known, else "session start">. MAX_ROUNDS=<n>.
```

Show the user the plan inline and say you're sending it to Codex for adversarial review.

### Step 2 — The loop

Maintain `ROUND` (start 1) and `THREAD_ID` (empty until round 1 returns).

**The review prompt** sent to Codex each round (adjust the task line):

> You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to be agreeable. Read the plan at `PLAN.md` (and any repo files you need — read them, never write). Identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. Do NOT modify any files. End your reply with EXACTLY one line: `VERDICT: APPROVED` if the plan is sound enough to implement, or `VERDICT: REVISE` if it still has material problems.

**Round 1** (creates the session — capture `thread_id`):

```bash
codex exec --dangerously-bypass-approvals-and-sandbox --model gpt-5.6-sol -c service_tier=fast \
  --json \
  -o /tmp/codex-verdict.txt \
  "$(cat REVIEW_PROMPT)" \
  < /dev/null 2>/dev/null | grep '"type":"thread.started"'
```
Parse `thread_id` from the `{"type":"thread.started","thread_id":"..."}` line → that is `THREAD_ID`. The critique text lands in `/tmp/codex-verdict.txt` (Codex's last message). Read that file.

> Note: stderr carries cosmetic MCP/auth noise on some setups — `2>/dev/null` is intentional. Confirm success by the presence of the verdict file + a `thread.started` line. If neither appears, the run failed (auth/model) — stop and tell the user.
>
> **`< /dev/null` is mandatory:** `codex exec` reads stdin *in addition to* the prompt arg, so under a non-interactive driver (Claude Code's Bash tool, CI, any non-TTY pipeline) it blocks forever waiting on stdin EOF — a silent ~0% CPU hang. The redirect gives it immediate EOF. Required on the resume call below too.
>
> **Timeout guard:** run every `codex exec` / `codex exec resume` with a 10-minute ceiling so any future stall fails loud instead of hanging silently. Via Claude Code's Bash tool, pass `timeout: 600000` on the tool call (the default 2-minute tool timeout is too short for real reviews and would kill them mid-run). In a plain shell, prefix the command with `timeout 600` (Linux / Git Bash) or `gtimeout 600` (macOS via coreutils — stock macOS has no `timeout`). If the ceiling trips, treat it as a failed run: stop and tell the user rather than retrying blind. Applies to the resume call below too.

**Rounds 2..MAX** (resume the SAME session — Codex remembers its earlier critiques, won't re-litigate settled points):

```bash
# NOTE: resume rejects -s but takes the same bypass/model flags as exec.
# Spell them out every round — otherwise Codex inherits config.toml.
codex exec resume "$THREAD_ID" --dangerously-bypass-approvals-and-sandbox \
  --model gpt-5.6-sol -c service_tier=fast --json \
  -o /tmp/codex-verdict.txt \
  "I revised the plan. Re-review PLAN.md. Same rules. End with VERDICT: APPROVED or VERDICT: REVISE." \
  < /dev/null 2>/dev/null >/dev/null
```

Both `codex exec` and `codex exec resume` support `--json` (stream → parse `thread_id` first round) and `-o/--output-last-message` (verdict capture).

**Each round, after Codex returns:**
1. Read `/tmp/codex-verdict.txt`. Append to `LOG_FILE`: `## Round <n> — Codex` + the full critique.
2. Grep the last line for the verdict token.
   - `VERDICT: APPROVED` → break the loop, go to Step 3 (converged).
   - `VERDICT: REVISE` → Claude reads the critique, decides **what's actually worth acting on** (Claude has final say — Codex advises, it does not command). Revise `PLAN_FILE`. Append to `LOG_FILE`: `### Claude's response` + what you changed and what you rejected and why. Increment `ROUND`.
3. If `ROUND > MAX_ROUNDS` → break to Step 3 (deadlock).

### Step 3 — Resolution (human gate #2)

**If APPROVED:** Present to the user — the final `PLAN_FILE`, a 3-bullet summary of what the argument improved, and the round count. Ask: *"Plan survived N rounds of Codex. Implement it now — Codex builds it (`/codex-build`), Claude builds it, or stop here?"* Only on a yes is code written. **No code is written during the loop.** If the user picks Codex, invoke the `codex-build` skill with `SPEC_FILE=PLAN.md` and the same `LOG_FILE` — roles flip (Codex writes, Claude reviews the diff) and the build rounds append to the same log.

**If MAX_ROUNDS hit without APPROVED (deadlock):** Do NOT pretend it converged. Surface the unresolved disagreements explicitly: list each point Codex still flags and Claude's counter-position. Hand it to the human to break the tie. This is a legitimate, useful outcome — a flagged disagreement beats a false "approved."

## Hard rules

- Same flag set EVERY round — `--dangerously-bypass-approvals-and-sandbox --model gpt-5.6-sol -c service_tier=fast`, first call and every resume (resume has no `-s`). Codex is unsandboxed, so "don't write" lives in the review prompt: keep it there. This skill still never *asks* Codex to implement — that's `/codex-build`.
- The loop ALWAYS terminates at `MAX_ROUNDS`. No unbounded recursion.
- Claude is the final arbiter on every REVISE — incorporate good critiques, reject bad ones *with a reason logged*. Don't cave to Codex on everything (that defeats the cross-model check) and don't ignore it (that defeats the point).
- Code only after human gate #2.
- `LOG_FILE` is the deliverable — it tells the whole story of the argument. Keep it complete.

## What NOT to do

- Don't use this to review existing code — that's `/codex:review`.
- Don't pin a `-codex` model variant on ChatGPT-account auth — it 400s.
- Don't skip the log — the argument transcript is the most valuable artifact.
- Don't let Codex edit files. Read-only, always.
