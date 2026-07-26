# Review — {{DATE}} · {{SCOPE_LABEL}}

<!-- Written by cozyreview. Commit this file: the briefings are the durable comprehension
     record for this repo. Grep them by component name to recover wiring and reasoning
     history without maintaining a separate artifact that drifts out of sync with code. -->

**Scope** {{RANGE}} · {{N}} commits{{UNCOMMITTED_NOTE}}
**Context** {{TIER}} <!-- e.g. "plans + SYSTEM.md" / "commit messages only — no specs/ in this repo" -->

Confidence: `✓` verified · `~` inferred · `?` no source — go look

---

## {{GROUP_N}}. {{GROUP_TITLE}} {{PLAN_REF_OR_UNPLANNED}}

<!-- repeat this block per change group · max 5 -->

**The line that is the change** — `{{file}}:{{line}}`

```diff
{{THE DECISIVE 1–3 LINES. Verbatim. This alone should tell the reader what happened.}}
```

**What had to change around it**

```diff
{{SUPPORTING HUNK — ≤20 lines, trimmed hard, verbatim from git.
  Annotate load-bearing lines in place with trailing ① ② markers.
  Elide interiors with `…`. Summarize churn in one line instead of showing it.}}
```

① {{mechanism — why this line matters, not a restatement of the diff}}
② {{what it enables or breaks}}

- **Component** {{✓|~}} {{WHICH COMPONENT, and how that was determined}}
- **System** {{✓|~|?}} {{WHAT THE COMPONENT IS FOR + what this does to that job. Blast radius: named callers/dependencies.}}
- **Users** {{✓|?}} {{FROM A STATED SOURCE ONLY — or: "the plan does not state user impact"}}

{{FLAGS — omit the line entirely if none}}
- ⚠ **{{drift | incomplete | convention}}** — {{concrete finding}} → `{{file:line}}`

**See it** `{{paste-ready IDE command in the user's shell dialect}}`

---

## Flags

<!-- Consolidated. Omit this section if there are genuinely none. -->

| # | Type | Finding | Where |
| --- | --- | --- | --- |
| 1 | {{drift/incomplete/convention}} | {{what to check and why it matters}} | `{{file:line}}` |

---

## Read this

Only what could **not** be shown inline — whole-file structure, a large unfamiliar module, a config whose meaning is its totality. If the hunks above already covered it, this section is empty, and that is the good outcome.

Ordered along the causal chain.

1. `{{file.ts:42}}` — {{what to look for}} · *{{Changed|Component|System|Users}}*

**Replay the whole review in the editor** `./{{slug}}.open.ps1`

---

## Explain it back

{{3–5 sentences the architect could say out loud. Repo's own vocabulary. No hedging,
no marketing. If a trace step was `?`, say so plainly rather than papering over it.}}

**Can you answer these?** <!-- deliberately NOT answered above -->

1. {{mechanism question}}
2. {{failure-mode question}}
3. {{design-rationale question}}

Any you can't answer is a drill target.

---

## Weakest link

{{MANDATORY. The claim in this review least supported by evidence. Never "all good" —
if everything checked out, name the shakiest inference and what would confirm it.}}

{{NOT_COVERED — if the 5-group cap dropped anything, say what and how to see it.}}

{{PROPOSED SYSTEM.md ROWS — if any component here is unrecorded. Present, never write.}}
