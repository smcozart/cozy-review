# cozyreview

Review code changes for **architectural ownership** — not code quality.

Built for the case where you are shipping faster than you can read: agents write the code, it works, it merges, and three days later you cannot explain the component you are about to iterate on. This skill closes that gap without slowing the ship rate.

A skill for [Claude Code](https://claude.com/claude-code). Every run walks one **trace**:

> **the code that changed** → **which component** → **what it means for the system** → **what it means for users**

## What it actually does

- **Groups a diff by intent**, not by file — "added retry to the ingestion path (3 files)", not `client.ts +40 −3`.
- **Sources the "why" from what the repo already records** — plans, decision logs, defect registers, ADRs. It discovers these by shape rather than expecting fixed paths, because most repos keep an excellent why-layer under names nobody standardized.
- **Detects drift** — where the implementation quietly diverged from what the plan said.
- **Detects incomplete changes** — a contract changed but one call site wasn't updated, a schema changed with no migration, a new branch with no error path. The most common agent failure mode.
- **Detects mirror divergence** — the same artifact living in two trees where only one of them moved, so a fix lands in a tree that nothing ships.
- **Compounds.** Each review reads the ones before it, so it carries forward what's established, reopens unresolved questions, and flags prior claims the new changes contradict. Drift across reviews is the only drift a single diff cannot show.
- **Shows the decisive code** — leads with the one or two lines that *are* the change, before and after, annotated in place. Not the whole diff; the 10% that carries the meaning.
- **Gates on explanation** — ends with questions it deliberately didn't answer. Reading an AI explanation feels like understanding; these check whether it was.

The editor is the **escape hatch, not the main path.** Opening VS Code and rebuilding context is the slow part, so the decisive hunk arrives already annotated — and when you do need full context, you get a runnable `code --diff` command, never a file reference to go chase.

## The prime directive

Every claim is marked `✓` verified · `~` inferred · `?` no source.

A confidently wrong explanation is worse than no explanation — it sends you into a meeting believing something false. `?` is a feature. If no plan, ADR, or commit message states user impact, the review says so instead of inventing a plausible sentence.

## Usage

```
/cozyreview                      # everything since your last review, plus uncommitted work
/cozyreview --staged             # what's about to be committed
/cozyreview HEAD~5..HEAD         # explicit range
/cozyreview drill <component>    # depth on demand — not scoped to a diff
```

First run in a repo asks how far back to go, then sets a watermark.

## Artifacts

```
.cozyreview/
  state.json        # watermark — personal, gitignore this
  reviews/*.md      # briefings — commit these
```

Two files, deliberately. **The briefings are the ledger** — grep them by component name and you have the wiring history and your own comprehension history, with no separate artifact that can drift out of sync with the code.

There is no stored wiring graph. Edges drift on every commit and a stale graph is a lie; blast radius is computed live, scoped to what changed.

## Works with, not on top of

Tiered by what the repo already has. Nothing is required.

| Present | Effect |
| --- | --- |
| Plans (`specs/*.html`, from [cozyplan](https://github.com/smcozart/cozyplan)) | Intent is `✓` verified. Drift detection works. |
| `SYSTEM.md` — a component map | Component mapping and system impact are `✓` rather than `~`. |
| A decision log or defect register, under any name | Reviews cite real rationale and can flag genuine convention breaks. |
| *(bare repo)* | Falls back to git history and code inference, marked `~`/`?`. Still useful — and it names which artifact would most improve the next run. |

Nothing is required, and the discovery step means you rarely have to rename anything to get the benefit — `architecture_decisions.md`, `docs/decisions/`, `FINDINGS.md`, and `docs/adr/` all count.

cozyreview never writes `SYSTEM.md` — it *proposes* rows and leaves the decision to you, since one line written once permanently upgrades every future review in that repo.

## Scope

This is a comprehension layer, not a linter, test runner, or full quality review. It flags risk that bears on **correctness of understanding** and **completeness of change**. Style is somebody else's job.

Prompts, tool schemas, and agent definitions count as code — a changed system prompt has blast radius exactly like a function signature does.

## Triggering

The skill description handles the obvious asks. For the moments that matter most — where you'd rather not depend on the model deciding — add a house rule to the project's `CLAUDE.md`:

```markdown
## Code review — cozyreview

Run `cozyreview` when:
- A cozyplan Build finishes, or an agent lands a change I didn't write
- I'm about to iterate on a component I haven't reviewed (`cozyreview drill <component>`)
- I ask "what changed", "catch me up", or "what did you just build"
- Unreviewed commits have piled up since `.cozyreview/state.json`

Do NOT run it mid-implementation, after individual edits, or when I've asked a
question. It is a checkpoint, not a monitor — firing it continuously turns it
into noise I'll learn to skip.
```

Match the file's existing voice; the rule only works if it reads like the rest of your house rules rather than a bolted-on block.

### If that isn't enough

**Hooks cannot invoke a skill** — they run shell commands. What they can do is inject a line into context so the model reaches for it. Two are worth having, in this order:

1. **`PreToolUse` on `git push`** — the last checkpoint before code leaves the machine. Compare `state.json.last_reviewed_sha` against `HEAD`; if commits are unreviewed, say so. This is the one that earns its keep.
2. **`SessionStart`** — reports drift when you open a repo cold, which is exactly the catch-up case.

Verify matcher syntax against current Claude Code hook docs when you install, and register hooks explicitly — never silently, since they execute commands.

## Install

Bare skill — no plugin, no hooks:

```
npx skills add ./skills/cozyreview
```

Or drop `skills/cozyreview/` into `.claude/skills/` (project) or `~/.claude/skills/` (global).

Packaging as a plugin only buys one thing: a pre-push hook that warns when unreviewed commits pile up. Worth doing once the skill has proved itself in daily use — not before.
