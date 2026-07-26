# Workflow — Review

Produce one briefing covering everything in scope. Work through the steps in order. Keep gathering cheap: the point is attention direction, not exhaustive analysis.

---

## 1. Resolve scope

Apply the ladder in `SKILL.md § Scope resolution`. Then gather, cheaply:

```
git log --oneline <range>
git diff --stat <range>
git status --short                 # uncommitted work
git diff <range> -- <file>         # only for files you actually need to reason about
```

**Do not read every changed file.** Read `--stat` first, decide what matters, then read selectively. A 40-file diff usually has 4 files that carry the meaning.

State the resolved scope in one line, including the commit count and whether uncommitted work is included.

### Nothing to review

Say so plainly and stop — do not manufacture a review out of an empty range. The cases:

- **No commits at all** (`git log` fails with "does not have any commits yet") — if files exist, everything is untracked; offer to review the working tree as the initial state. If not, there is nothing here.
- **Clean tree, watermark at `HEAD`** — nothing new since last review. Name the last briefing and its date.
- **Range is only merge commits** — review the merged content or say the range is empty of authored change.
- **Range is entirely generated/vendored files** — say what they are and skip. Do not trace `package-lock.json`.
- **`git rev-parse --show-toplevel` is above where the user pointed** — the repo root is a parent. Say so before reviewing, because the user's mental model of the repo boundary is wrong and every scope after this inherits the error.

### Pick up the thread

Before grouping, check `REVIEW_DIR/reviews/` for what is already established. The briefings are the ledger — a review that starts cold every time is the exact failure this design was meant to avoid.

Bound it, because this runs on every review and the folder only grows: **the 3 most recent briefings whose components or paths overlap the current scope.** Not the whole folder, not briefings about untouched components.

From them, carry forward:

- **What is already explained** — do not re-derive it. Reference it and move on.
- **Open questions and weakest links** — if a prior review flagged something unresolved and this change touches it, that is the first thing to check now.
- **Contradictions.** If a prior briefing asserted something these changes make false — a component's responsibility moved, a claimed invariant broke, an owner changed — **say so explicitly**, citing the briefing and its date. Architectural drift shows up across reviews, never inside one. This is the only place cozyreview can catch it.

---

## 2. Group by intent, not by file

The unit of a briefing is a **change group**: a set of edits that exist for one reason. `client.ts +40 −3` is not a group. "Added retry to the ingestion path (3 files)" is.

Build groups in this order of preference:

1. **By owning plan.** For each cozyplan plan in `SPECS_DIR`, find the commits that implement it:
   - Grep the plan HTML for `data-meta="commits"` and `data-meta="id"` — the `commits` field lists implementing SHAs.
   - Read `specs/<plan>.log.ndjson` if present for commit/session events.
   - **Legacy plans** (no `data-*` anchors — hand-authored HTML) have neither. Fall back to: the plan's kebab slug or id appearing in commit messages, or a strong overlap between changed paths and the plan's "Relevant Files" section. Mark this association `~`.
   - If `PLAN_TOOL brief <plan>` resolves easily, use it for a compact intent extract. Otherwise read the plan's Purpose / Problem / Solution sections directly. **Do not** make the run depend on the CLI being installed.
2. **By commit message convention** — conventional-commit scope (`bench:`, `docs:`) or a shared ticket id.
3. **By coherent code locality** — same component, same call path, obviously one intent.

Commits belonging to no plan are their own group, labeled **unplanned**. This is not an accusation; hotfixes and follow-ups are normal. It matters because their "why" has a weaker source.

---

## 3. Run the trace

**Changed → Component → System → Users.** Every group gets all four steps, each with its own `✓` / `~` / `?` marker. A step with no source gets `?` and a short note on what would answer it — **never** a plausible-sounding guess.

*Completion criterion: four marked steps for every group, no step silently omitted.*

**Changed.** One line naming the edit in the repo's own vocabulary (use `CONTEXT.md` terms when it exists). Keep it short — **the code itself carries this step** and gets shown in step 5. Do not describe in prose what you are about to display. Always `✓`.

**Component.** Map changed paths to components using `SYSTEM.md` rows first, then package/directory boundaries, then naming convention. `✓` if `SYSTEM.md` names it, `~` if inferred from layout.

**System.** What the component is FOR and what this change does to that job. Sources, in order: `SYSTEM.md`'s Responsibility column, the plan's Problem/Solution, relevant ADRs. Include **blast radius** — computed live, scoped to the change:

```
grep -rn "<changed symbol>" --include=<relevant globs>    # callers
```

Read the changed function's own imports/calls for what it depends on. One or two hops, not a transitive graph. Name the specific callers found, not "several consumers."

**Users.** Only from a stated source: the plan's Purpose, an ADR, a commit message that actually says it. **If no source states user impact, the answer is `?` with "the plan does not state user impact."** Do not derive it from the code. Code does not know who it is for. This step will legitimately be `?` often — that is honest, and it is useful pressure toward writing user intent into plans.

---

## 4. Flag risk — three distinct checks

Scoped to comprehension and completeness. Not a style review.

**Drift — stated intent vs. actual diff.** Only possible when the group has a plan. Compare what the plan said it would do against what the code did:
- Code does something the plan never mentioned → **unplanned scope**. Name the file and line.
- Plan specifies work the diff does not contain → **incomplete against plan**.
- Code takes a different approach than the plan specified → **approach deviation**. Not necessarily wrong; often the plan was stale. Flag it so the plan gets amended rather than silently diverging.

This check is the highest-value thing cozyreview does. Nothing else in the toolchain catches it.

**Incomplete change — what should have moved and didn't.** The most common agent failure mode. Check specifically:
- A signature, schema, or contract changed → were **all** call sites updated? Grep and count.
- A data shape or migration changed → is there a corresponding migration/version bump?
- A new config key or env var → is there a default, and is it documented where the others are?
- A branch of logic added → is there an error/empty/failure path for it?
- A prompt or tool schema changed → do the evals, fixtures, or parsers that depend on its shape still match?

**Mirror divergence — the twin that didn't move.** Repos carry the same artifact in more than one tree all the time: authoring vs. published, vendored copies, skill mirrors, monorepo duplication. When one copy changes and its twin doesn't, the copies silently diverge, and a fix can land in a tree that nothing ships.

Fire this **only** when all three hold, or it becomes noise you learn to ignore:

1. A changed file has a same-basename file elsewhere in the repo, **outside** ignored/generated paths.
2. The two are **substantially the same artifact** — compare content, not just the name. Two unrelated `index.ts` or `README.md` files are not twins.
3. **Only one of them changed** in this range.

Then say which copy moved, which didn't, and — the part that matters — **which one ships**. If you cannot tell which is authoritative, that is itself the finding: name both and say what would settle it.

Skip entirely for files whose duplication is structural and expected: `package.json`, lockfiles, per-package configs, test fixtures.

**Convention break.** Contradicts `STACK.md`, an ADR, `CLAUDE.md`, or the dominant pattern in surrounding code. Say which rule and where it lives.

Every flag is concrete — `file:line` and what to check. No vague unease.

---

## 5. Show the decisive code

**This is the step that makes the review worth reading.** Everything before it is analysis; this is where the user actually comes to own the change. Describing code they cannot see is the single fastest way to make this skill useless.

### Find the decisive line

Almost every change group has **one or two lines that ARE the change** — the exit code that flipped, the condition that inverted, the call that moved. Everything else in the diff exists to support them. Find them and lead with them:

```diff
- process.exit(0);
+ process.exit(fails.length + errors.length > 0 ? 1 : 0);
```

One line, and the reader already knows what happened. *Then* show what had to change around it. Reading a diff top to bottom is slow because the decisive line is usually buried in the middle; leading with it is most of the speed.

### Show the supporting hunk, annotated in place

A real `diff` block, **≤20 lines**, trimmed hard. Attach numbered markers as trailing comments on the exact lines they explain, then expand them underneath:

```diff
-      let run;
+      let run, verdict, notes;
       try {
         run = runTarget(scriptPath, dir, suite.reportName);
-      } finally {
-      }
+        ({ verdict, notes } = evaluate(kase, run));      // ① moved INSIDE the try
+      } catch (e) {
+        verdict = "ERROR";                                // ② third verdict, new
+      }
```

① why this line matters — the mechanism, not a restatement of the diff
② what it enables or breaks

**Trim like it costs money.** Cut reformatting, renamed locals, moved imports, and mechanical churn — one summary line covers all of it (*"plus ~40 lines of import reordering"*). Elide unchanged interior with `…`. If the point needs more than 20 lines, it is two groups or it is a drill.

*Completion criterion: every group either shows its decisive code, or states in one line why it has none (pure deletion, generated file, binary).*

**Never invent or paraphrase code.** Every shown line comes verbatim from `git show` / `git diff` / the file. If a line is elided or reformatted for width, mark it.

### Where there is no diff

For a group that is new code rather than modified code, show the **shape** — signature, the core branch, the return — not the whole file. For a deletion, show what was removed and name what now serves that purpose, or flag that nothing does.

### Then hand off to the IDE

Per group, one paste-ready command in the user's shell dialect (`SKILL.md § IDE handoff`) that opens the real before/after. This is the escape hatch for full context — the inline hunk should already have answered the question 90% of the time.

Keep a short **reading list** only for what genuinely cannot be shown inline: whole-file structure, a large unfamiliar module, a config whose meaning is its totality. Order it along the causal chain. If a file only confirms something already marked `✓`, leave it out.

---

## 6. Explain-it-back

Two parts.

**The paragraph.** 3–5 sentences the user could say out loud in a standup, in their own system's vocabulary. Concrete, no hedging, no marketing. If a trace step was `?`, the paragraph says so plainly rather than papering over it.

**The gate questions.** 2–3 questions that the briefing **deliberately did not answer**. Recall questions are worthless — reading an explanation feels like understanding and isn't. Ask what would only be answerable by having actually grasped the mechanism: what breaks if this is called twice, what happens when this dependency is down, why this seam and not the other one.

Then state plainly: *any question you can't answer is a Drill target.*

---

## 7. Resolve what's cheap to resolve

Before writing anything, sweep every `~` and `?` marker and every flag, and ask one question of each: **is this one command away?**

If it is, run the command. A claim you could have settled in five seconds but left marked uncertain is not humility — it is unfinished work handed to the user, and it costs them more to resolve than it would have cost you. The `?` marker exists for what is genuinely unavailable, not for what is merely unchecked.

Cheap almost always means: one `ls -l`, one `grep`, one `git show`, reading one more file. Typical wins:

- "appears to be generated / a symlink / derived" → `ls -l` it
- "nothing seems to call this" → widen the grep past the file types you first searched, and check `.git/hooks` and script entry points before asserting it
- "this looks unrelated to the plan" → open the plan and confirm
- "probably follows the existing pattern" → open the neighbouring file

Not cheap, and correctly left marked: reading a large implementation end to end, anything needing the code to be run, and **anything only a human knows** — intent, priority, who asked for it. Never resolve those by inference.

Mark what you settled. In the briefing's weakest-link section, note which claims started uncertain and were verified during the run — it shows the markers are load-bearing rather than decorative, and it keeps this step honest.

Then re-check: the weakest link you report in step 9 must be something that **survived** this sweep. If your stated weakest link is one command away, you skipped this step.

---

## 8. Write, propose, advance

**Write the briefing** to `BRIEFINGS` using `templates/briefing.md`. Create `REVIEW_DIR/reviews/` if needed. Slug it from the dominant change group.

**Propose `SYSTEM.md` rows.** If the review touched a component absent from `SYSTEM.md` — or `SYSTEM.md` doesn't exist — output the proposed row(s) in the exact table format:

```
| Component | Responsibility (what it's FOR) | Owner | Why (ADRs / plans) |
```

Present them for the user to apply, or to hand to the `discuss` skill. Say why it's worth doing: one line, written once, permanently upgrades Component and System for every future review in this repo.

**Advance the watermark.** Update `state.json` to `HEAD` — only if the reviewed range included commits, and only after the briefing is written. A working-tree-only review does not advance it.

**Log to the vault.** If `VAULT_LOG` is configured, append one line:
```
## [YYYY-MM-DD] verification | cozyreview: <repo> — <N> groups, <M> flags
<briefing path> — <one-line takeaway>
```
Skip silently if not configured.

---

## 9. Close

Print the fast pass to the terminal within the output budget (`SKILL.md § Output contract`), ending with:

- The **weakest link** — the claim in this review you are least sure of. Mandatory. Never "all good."
- Anything **not covered** because of the 5-group cap, and how to see it.
- The offer: *drill into any component, or answer the gate questions and I'll check you.*
