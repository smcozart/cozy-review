# Workflow — Drill

Depth on demand, for one thing at a time. Called when the user cannot explain something, failed a gate question, or is about to iterate on a component and wants to be briefed **before** they start guessing.

Drill is **not** scoped to a diff. It answers "how does this actually work and why does it exist" about code that may not have changed in months.

---

## 1. Establish the target and the gap

The target is a component, a function, a call path, a prompt, or a specific question ("why does the retry live here and not in the client?").

Before reading anything, establish **what the user already believes**. A drill that re-explains what they know is wasted; the value is in the delta. If it's not obvious from context, ask one short question: what's your current mental model of this?

If a recent briefing in `REVIEW_DIR/reviews/` covers the target, read it first. Do not re-derive what is already recorded — build on it, and note if it now looks stale against the code.

---

## 2. Read for real

This is the one place cozyreview spends depth. Read the actual implementation, not just signatures.

- The component's entry points and its seams — what calls in, what it calls out to.
- The **data** moving through it: shape in, shape out, where it's transformed.
- Error, empty, and failure paths. These carry most of the design intent and are where an architect's understanding is usually thinnest.
- State, caching, and lifecycle — what persists between calls, what doesn't.
- Its plan (`specs/`), its ADRs, and its `SYSTEM.md` row, for the recorded why.

Keep the same `✓` / `~` / `?` discipline. Reading more code converts `~` into `✓` — that is the entire point of drilling. Say explicitly which markers moved.

---

## 3. Explain in mechanism order — with the code on the page

Not file order. Not diff order. **Follow the data.**

Walk one concrete path end to end: a real input arrives here → transformed there → handed to this → produces that. A single traced example teaches more than an abstract description of every branch.

**Show the source at each stop.** A drill that describes a function the user cannot see has failed at its only job — they came here *because* prose wasn't enough. Paste the actual block, elide its uninteresting interior with `…`, and annotate the load-bearing lines in place:

```js
function runTarget(scriptPath, dir, reportName) {
  const r = spawnSync(process.execPath, [scriptPath], { cwd: dir, … });
  //        ① returns { status: null, error: Error } on spawn failure — it does NOT throw
  …
  return { exit: r.status, … };
  //              ② r.error is never read, so a spawn failure leaves here as exit: null
}
```

Blocks stay small — one function, one branch, one call path at a time. Verbatim from the file, never paraphrased. A drill may run longer than a review, but it earns the length with code, not with commentary.

Then, and only then, cover what varies: the other branches, the failure modes, the edge cases.

**Answer "why this shape."** This is what makes the user an owner rather than a narrator. Where a design decision is recorded in a plan or ADR, cite it `✓`. Where it isn't, offer the most likely engineering reason marked `~` and say plainly that it is unrecorded — and that an ADR would fix that permanently.

---

## 4. Close the gap that started the drill

Return to the specific question or failed gate question and answer it directly, in one or two sentences. Do not make the user reassemble the answer from the walkthrough.

Then a **new** gate question — harder than the one that failed, testing the mechanism just explained, deliberately not answered above.

---

## 5. Record it

Append the drill to the most recent relevant briefing under a `## Drill — <target>` heading, or write a new briefing if none fits.

This matters: the briefings are the comprehension ledger. A drill that lives only in chat history is a lesson learned twice. A drill written down is greppable the next time this component comes up.

If the drill surfaced something durable — an unrecorded design decision, a component missing from `SYSTEM.md`, a plan that no longer matches the code — say so and name the fix:

- Unrecorded decision → propose an ADR (`docs/adr/NNNN-title.md`), one paragraph.
- Missing component → propose a `SYSTEM.md` row.
- Plan/code divergence → recommend `PLAN_TOOL amend` so the plan stops being a lie.

Drill does **not** advance the watermark. It reviewed understanding, not changes.
