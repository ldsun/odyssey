# Addendum: Ver4 Refinement - `/declutter`

## Why this addendum exists

This addendum refines Ver4 to close the trust gap between:

- what is in mind (live threads, concrete obligations, exact wording), and
- what the system shows (often too synthesized, too detached from raw input).

The goal is not a "better summary." The goal is a lower-friction external mirror that the brain can trust.

---

## Core change

Replace separate `/briefing` and `/process` with a single command:

- **`/declutter`** - one pass that both clarifies inbox input and sets focus.

Cadence:

- **Default:** once in the morning.
- **Also allowed:** on-demand anytime cognitive load rises, context gets fragmented, or priorities shift.

---

## Design principles for `/declutter`

1. **Thread-first, not document-first**
   - Organize around active workstreams/threads, not around file categories.
   - The system should answer: "what threads are live, and what is my next move in each?"

2. **Minimum transformation**
   - Preserve original phrasing when moving items from `inbox.md` into thread notes/tasks.
   - Rewrite only to make action ownership/time explicit.
   - No combining distinct items unless explicitly requested.

3. **Action before synthesis**
   - Output next actions and follow-ups first.
   - Keep narrative summaries short and optional.

4. **Cognitive offload, not cognitive replacement**
   - `/declutter` mirrors and stabilizes mental context.
   - It does not force a different model when the current mental model is working.

5. **Notes stay good**
   - Keep the existing capture habit in `inbox.md`.
   - Improve how captured content is reflected back, not how capture is done.

---

## New operating flow

```
inbox.md (raw fragments)
   + existing thread state (dashboard/tasks/notes/references)
   ↓
/declutter
   1) Thread Reality Check (what is active now)
   2) GTD Clarify each new fragment (minimal rewrite)
   3) Update thread pages + tasks + references + inbox state
   4) Output Focus Strip (today/now)
```

### Step 1: Thread Reality Check

Before classification, AI identifies current active threads and asks:

- Is this still active, paused, blocked, or done?
- Is anything missing that is active in your head but not in the system?

This prevents stale system state from drifting away from real cognitive state.

### Step 2: Clarify new inbox items (GTD)

For each new fragment:

- Actionable?
  - No -> Trash / `someday.md` / `references.md`
  - Yes -> next action + owner + timing
- < 2 min?
  - Surface as "do now"
- > 2 min?
  - Delegate or defer into `tasks.md`

Rule: keep wording close to original input unless precision requires edits.

### Step 3: Update thread artifacts

- `dashboard.md`: active threads and status (minimal prose)
- `tasks.md`: concrete next actions and follow-ups
- `notes/<thread>.md`: append thread-relevant context with light structure
- `references.md`: key links only (especially meeting-critical links)
- `inbox.md` (full pass): clear or mark processed lines so the file reflects remaining unprocessed capture

### Step 4: Focus Strip output

`/declutter` ends with a compact "what matters now" section:

- Top 1-3 threads for today
- Exact next actions (verb-first)
- Required follow-ups/delegation checks
- Urgent reference links if needed in meetings

No broad daily narrative unless explicitly asked.

---

## Behavioral rules (anti-perfunctory guardrails)

1. **Do not over-synthesize**
   - Avoid compressing multiple discrete fragments into one generalized sentence.
   - Keep atomic items atomic.

2. **Do not invent closure**
   - If status is unknown, mark unknown.
   - Do not silently mark "done" or "resolved."

3. **Do not bury actions**
   - Any actionable item must appear as a visible next action or follow-up.

4. **Do not hide source language**
   - When uncertain, preserve the original line and annotate rather than rewriting it.

5. **Do not force full processing**
   - On-demand `/declutter` can run in "light pass" mode:
     - update focus + urgent actions first,
     - defer inbox rewrite/cleanup.

---

## File-level refinements (without changing baseline structure)

Keep existing Ver4 files, but apply these conventions:

- **`dashboard.md`**
  - One section per active thread.
  - Each thread shows: `status | why active now | next visible move`.

- **`tasks.md`**
  - Group by thread first, then by `Do / Delegate / Follow-up`.
  - Keep entries as close as possible to entered wording.

- **`notes/<thread>.md`**
  - Append "raw-to-structured" blocks:
    - Raw fragment
    - Clarified interpretation (1-2 lines max)
    - Linked action (if any)

- **`inbox.md`**
  - No extra capture burden added. Continue current capture style.
  - `full pass` rewrites inbox state to reflect processed items.
  - `light pass` does not rewrite inbox.

---

## `/declutter` prompt contract (draft)

Use this instruction frame whenever `/declutter` runs:

1. Start from active threads; do a quick reality check against current artifacts.
2. Process only new/unprocessed inbox items.
3. Preserve original wording unless ownership/date/action verb must be clarified.
4. Keep separate items separate; do not merge without explicit request.
5. Surface < 2 minute actions first.
6. Update `dashboard.md`, `tasks.md`, `references.md`, relevant `notes/` files, and (in full pass) `inbox.md`.
7. End with a Focus Strip:
   - Top threads now
   - Next actions today
   - Delegation follow-ups due
   - Critical links needed soon
8. Keep output concise and operational; avoid high-level summaries.

---

## Morning cadence and on-demand usage

### Default morning run

- Run `/declutter` once after initial Slack/Outlook scan.
- Outcome: a realistic "today state" anchored in current threads and next moves.

### On-demand run triggers

Run `/declutter` again when:

- context switching spikes,
- too many open loops are being held mentally,
- an urgent thread appears unexpectedly,
- post-meeting obligations are unclear,
- trust drops ("system view doesn't match what I know").

---

## Trust calibration loop (weekly, 5 minutes)

Once per week, quickly review:

1. Which active threads in mind were missing from `dashboard.md`?
2. Which task entries felt too rewritten to trust?
3. Which references were still hard to retrieve in real time?

Then tune `/declutter` behavior:

- reduce transformation further,
- tighten thread naming,
- promote recurring urgent links into `references.md`.

Success metric: lower need to "hold everything in head" during the day.

---

## Expected improvement over baseline Ver4

- Fewer ritualistic steps (`/briefing` + `/process` -> `/declutter`)
- Lower mind-system mismatch (thread-first + wording preservation)
- Better reliance under real constraints (morning default + on-demand recovery)
- Better retrieval in execution moments (actions and links surfaced by thread)

