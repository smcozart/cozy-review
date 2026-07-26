---
name: cozyreview
description: Reviews code changes so the architect can explain and own them — comprehension and system impact, not code quality. Use when the user wants to review what changed, catch up on what an agent built, or be briefed on a component before iterating on it.
argument-hint: "[scope | drill <component>]"
---

# CozyReview

## Purpose

Let an architect **own** a codebase they are shipping faster than they can read. Every run walks one **trace**, in a fixed order, so the whole chain is scannable in seconds:

> **the code that changed** → **which component** → **what it means for the system** → **what it means for users**

**CozyReview shows the code.** Not the whole diff — the *decisive* lines, before and after, annotated where they sit. A review that only points at `file.ts:42` is homework, and homework does not get done on a ship day. The job is to find the 10% of a diff that carries the meaning and put it in front of the user already explained.

The IDE is the **escape hatch, not the main path**. Opening an editor, navigating, and rebuilding context is the slow part — the thing to avoid when shipping. When it is genuinely needed, hand off with a real runnable command (`§ IDE handoff`), never a file reference.

## Variables

USER_PROMPT: $ARGUMENTS
REVIEW_DIR: `.cozyreview/` at repo root
STATE: `REVIEW_DIR/state.json` — machine-owned. `{ "last_reviewed_sha": "<sha>", "last_reviewed_at": "<iso8601>", "reviews": <int> }`
BRIEFINGS: `REVIEW_DIR/reviews/<YYYY-MM-DD>-<kebab-slug>.md`
SPECS_DIR: `specs/` — cozyplan plans, when present
IDE: `code`
VAULT_LOG: optional path to a personal knowledge log (e.g. an Obsidian `log.md`). Append one line per review when configured; skip silently when not

## The Prime Directive — confidence is labeled, never assumed

A confidently wrong explanation is **worse than no explanation**, because it sends the architect into a meeting believing something false. Every claim in every output carries a marker:

| Marker | Meaning |
| --- | --- |
| `✓` | **Verified** — read directly out of the diff, the code, or a plan's own words |
| `~` | **Inferred** — a reasonable read of naming, structure, or convention. Probably right. Not checked |
| `?` | **Unknown** — no source says this. Go look, or accept not knowing |

`?` is a **feature**. Never upgrade a `?` to prose that sounds settled. Never invent user impact, business rationale, or intent that no plan, ADR, comment, or commit message actually states. "The plan does not say who this is for" is a correct, useful answer.

**But uncertainty must be earned.** `?` and `~` are for what is genuinely unavailable or expensive to check — never for what is merely unchecked. If a marker is one `ls`, one `grep`, or one file away from being `✓`, resolve it before reporting (Review § step 7). Handing the user an uncertainty you could have settled in five seconds is unfinished work wearing humility as a costume.

## Scope — what cozyreview is and is not

CozyReview is the **comprehension layer**. It exists so the user can explain their own system.

- It is **not** a linter, a test runner, or a full code-quality review. It flags risk that bears on *correctness of understanding* and *completeness of change*, not style.
- It is **not** a plan tool. `cozyplan` owns plans; cozyreview reads them.
- It does **not** write `SYSTEM.md`, `STACK.md`, or `CONTEXT.md` — the `discuss` skill owns those. CozyReview *proposes* rows and lets the user apply them.
- It does **not** store a wiring/edge graph. Edges drift on every commit and a stale graph is a lie. Blast radius is computed live, scoped to what changed.

## Prompts and agent configs are code

Anything that changes runtime behavior is in scope, whatever its extension. A modified system prompt, tool description, agent definition, JSON schema, or eval fixture has blast radius exactly like a function signature. **Never dismiss a changed `.md`, `.yaml`, `.json`, or `.txt` as "just docs" without checking whether something loads it at runtime.** Genuine documentation (READMEs, changelogs) is noted in one line and not traced.

## Context Layer — read what the repo already knows

Tier the run by what exists. Never require any of it.

| Artifact | Location | Read it for |
| --- | --- | --- |
| Plans | `specs/*.html` | The stated intent of a change — the ground truth for "why" |
| Event logs | `specs/*.log.ndjson` | Which commits/agents/sessions touched a plan |
| `SYSTEM.md` | repo root | Component map: what each component is FOR, and who owns it |
| `CONTEXT.md` | repo root | Project vocabulary — use its terms, don't invent synonyms |
| `STACK.md` | repo root | Technology defaults, so deviations are visible as deviations |
| `docs/adr/` | `docs/adr/NNNN-*.md` | The "why" behind standing decisions |
| `CLAUDE.md` | repo root / nested | House rules a change may be violating |

**Full fidelity** when plans exist: intent is `✓` verified and drift detection works.
**Degraded** on a bare repo: intent falls back to commit messages and code inference, and is marked `~` or `?`. Still useful — say so, and note which artifact would most improve future runs.

## Artifacts

Only two, both under `REVIEW_DIR`:

- **`state.json`** — the watermark. Personal; add to `.gitignore`.
- **`reviews/*.md`** — the briefings. **Commit these.** They are the durable record: grep them by component name and you have the wiring history and the comprehension history, with no second artifact that can drift out of sync with the code.

There is deliberately no `wiring.md` and no comprehension ledger. The briefings *are* the ledger. If grep over briefings ever proves insufficient, that is the moment to reconsider — not before.

## IDE handoff

`IDE` is on PATH and takes these — use them verbatim, never a bare file reference:

| Need | Command |
| --- | --- |
| Real before/after of a committed change | `git show <sha>^:<path> > <tmp>` then `code --diff <tmp> <path>` |
| Before/after of uncommitted work | `git show HEAD:<path> > <tmp>` then `code --diff <tmp> <path>` |
| Jump to an exact line | `code --goto <path>:<line>` |
| Open the whole change set in one window | `code -r <path1> <path2> …` |

Write `<tmp>` under the OS temp dir, never in the repo. On Windows/PowerShell use `$env:TEMP`; elsewhere `/tmp`. Emit commands in the **user's shell dialect**, ready to paste — a command they have to translate is a command they won't run.

For any review with more than two groups, also write `REVIEW_DIR/reviews/<slug>.open.ps1` (or `.sh`): one labeled block per group that materializes the before-file and opens the diff. One command replays the entire review in the editor.

## Output contract

Non-negotiable, because a briefing that is too long to read on a ship day does not get read.

- **5 change groups maximum.** More → cover the 5 highest-blast-radius and **state explicitly** how many were not covered and how to see them. Never truncate silently.
- Each group: the decisive hunk (**≤20 lines of code**, trimmed per Review § step 5), its annotations, and the trace. Prose stays under the code, not above it.
- **Never a rubber stamp.** Every review names its weakest link — the claim it is least sure of. If everything genuinely checks out, say so and still name the shakiest inference.

## Workflow

Select the single best match and read its file before acting.

| Workflow | When to call it | File to read |
| --- | --- | --- |
| Review | Default. Review changes — since last review, a named range, a branch, or the working tree | `workflows/review.md` |
| Drill | The user cannot explain something, or is about to iterate on a component and wants to be briefed first | `workflows/drill.md` |

If `USER_PROMPT` is empty, run **Review** with default scope.

## Scope resolution (Review)

Resolve in this order and **state the resolved scope in one line** before reviewing:

1. **Explicit** in `USER_PROMPT` — a SHA range, branch, tag, `--staged`, a path, or "the last N commits". Honor it.
2. **Watermark** — `state.json.last_reviewed_sha`..`HEAD`, plus uncommitted working-tree changes. This is the default.
3. **First run in a repo** (no `state.json`) — do not review all history. Offer: last 10 commits · current branch vs. its base · working tree only. Ask, then create `state.json`.

Uncommitted changes are **always** folded in when present, and labeled as uncommitted — this is where fresh agent output lives.

Advance the watermark to `HEAD` only after a briefing is written. Uncommitted work never advances it.
