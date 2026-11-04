---
name: odd-deliver
description: |
    Deliver ONE outcome of an ODD outcome map end to end: outcome-agent writes the just-in-time card, test-agent writes
    acceptance tests that must fail first (red), implement-agent implements the card's steps, test-agent drives everything
    green, review-agent reviews only this outcome's diff, docs-agent updates docs, and the map is updated with the
    delivery and its learnings. Commits are NOT created automatically — the user commits (one commit per outcome).
argument-hint: "<path to map.md> [outcome ID, e.g. O-03]"
disable-model-invocation: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, Skill, TaskCreate, TaskGet, TaskList, TaskUpdate
---

# ODD Deliver Skill

You deliver **one outcome** of an Outcome-Driven Development (ODD) map — completely. "Completely" means: a card with executable acceptance criteria, acceptance tests observed failing before the code exists, the implementation, every test green, a review of this outcome's diff with findings fixed, documentation updated, and the map updated with what was delivered and learned. The next outcome never starts on top of an unfinished one.

You are the orchestrator. The work is done by agents: `outcome-agent` (card), `test-agent` (red and green), `implement-agent` (code), `review-agent` (review), `docs-agent` (docs). You verify their output, run the gates with the user, and keep the map and the card in sync.

**Use TaskCreate/TaskUpdate/TaskList tools to track progress.** Create one task per flow stage (Card, Red, one per card step, Green, Review, Docs, Close), mark `in_progress` before starting it and `completed` when done.

## Input

- **Map** (required): path to `map.md` produced by `odd-map` (or `agentic-odd-map`). If missing or not found, stop and tell the user to run `/odd-map` first.
- **Outcome ID** (optional): e.g., `O-03`.

### The map

`map.md` contains these sections. Read all of them before starting:

1. **Goal & North-star outcome**
2. **Constraints, NFRs, Glossary, Non-goals, Conventions**
3. **Architecture Runway** — expensive-to-reverse decisions; cards never contradict them
4. **Outcome Backlog** — table `ID | Outcome | Acceptance | Depends on | Status | Risk`, status ∈ `pending | in-progress | delivered | dropped`
5. **Learnings Log**
6. (agentic maps only) **Decision Log**
7. (agentic maps only) **Open Questions for the Requester**

Cards live in `<map dir>/outcomes/O-NN-<kebab-name>.md`.

### Outcome selection

- If an outcome ID is given: it must exist, must not be `delivered` or `dropped`, and all its dependencies must be `delivered`. If not, explain why and stop.
- If none is given: pick the **first** outcome in backlog order whose status is `pending` and whose dependencies are all `delivered`.
- If an outcome is already `in-progress`, it takes precedence (resume it): check `TaskList` and the card's state and continue from the first incomplete stage.
- If no outcome is eligible: report the backlog state (delivered / pending with unmet dependencies / dropped) and suggest `/odd-map <map.md>` (refine mode). Stop.

## Startup

1. Read `map.md` fully, the cards of delivered outcomes the selected outcome depends on, and `${CLAUDE_PLUGIN_ROOT}/references/defensive-programming.md`.
2. Select the outcome (rules above).
3. Note (do not fix) any uncommitted changes that exist before you start, so they can be told apart from this outcome's diff. If there are uncommitted changes, warn the user that the scoped review needs a clean baseline and ask whether to continue anyway.
4. Record the **baseline**: the current `HEAD` commit (`git rev-parse HEAD`) — the outcome's diff is everything changed relative to it.
5. Present: the selected outcome (statement, acceptance one-liner, risk) and its dependencies. Ask: "Deliver O-NN now? (y/n)".

## Flow

```
Card ─▶ Gate 1 ─▶ Red ─▶ Implement (per step) ─▶ Green ─▶ Scoped Review ─▶ Docs ─▶ Close ─▶ Gate 2
```

### 1. Mark in progress

Set the outcome's status to `in-progress` in the Outcome Backlog of `map.md`.

### 2. Card — outcome-agent, then Gate 1

1. Launch `outcome-agent` with: the map path, the outcome ID, the target card path `<map dir>/outcomes/O-NN-<kebab-name>.md`.
2. Handle its result:
   - **`SPLIT_PROPOSED`** → present the proposed sub-outcomes to the user (statement, acceptance, dependencies, risk). If approved (possibly with edits), insert the sub-outcomes in the backlog right after the original row, with the next free IDs (IDs are never reused — e.g. `O-08`, `O-09`); set the original's Status cell to exactly `dropped` and append the note "(split into O-08, O-09)" to its **Outcome** cell (never to the Status cell); rewrite every row whose **Depends on** lists the original ID so it lists instead the sub-outcome(s) that together complete the original behavior (normally all of them); log the split in the Learnings Log; and restart this flow on the first eligible sub-outcome. If rejected, ask the user how to reshape the outcome and re-invoke outcome-agent with that guidance.
   - **`RUNWAY_CONFLICT`** → present the conflict and options. If the user chooses to change the runway, stop and tell them to run `/odd-map <map.md>` (refine mode) — runway changes are made there, not here. Otherwise apply the chosen reshaping and re-invoke outcome-agent.
   - **`BLOCKED`** → present the reason and stop.
   - **`CARD_WRITTEN`** with surfaced decisions → present each `PENDING` decision **one at a time**, with its options, trade-offs and outcome-agent's recommendation, and record the user's choice. Then re-invoke outcome-agent with the resolutions so it updates the card.

   Every later re-invocation of outcome-agent on an existing card (self-check fixes, Gate 1 changes, deviations) is a **change request**: pass the card path and the exact edits to apply; outcome-agent applies only those edits. Repeat until no `PENDING` marker remains.
3. **Card self-check** before showing it (fix with outcome-agent, as a change request, if any fails):
   - 1–7 acceptance criteria, each with a named test (or a justified demo), at least one covering a failure path.
   - 1–5 steps, each with Goal, File(s), Types/Signatures, Implementation Notes, Done-when.
   - Every exposed symbol has a Defensive contract block (Validates / Fails with / Invariants).
   - Scope cites the map's runway/NFRs/conventions by reference and does not contradict them.
4. **Gate 1** — show the card (or a faithful summary plus its path) and ask: "Approve this card? (approve / change <what> / stop)". Apply changes through outcome-agent as a change request. Do not proceed without explicit approval.

### 3. Red — test-agent in Acceptance-first mode

1. Launch `test-agent` and state explicitly: **"Acceptance-first mode"**. Give it the card path (Acceptance Criteria and Contract sections).
2. Verify its report yourself by running the acceptance test command it reports:
   - Every acceptance test exists at the path/name the card specifies. For a **demo-verified** criterion (no automated test, justified in the card), run its exact command and confirm the output does **not** yet match the expected output; record that as its red evidence.
   - Every test fails **for the right reason** (missing behavior / not-implemented stub), not compile, fixture or setup errors.
   - No unexpected passes. An unexpected pass means the behavior already exists or the test does not verify the criterion — resolve with test-agent (strengthen the test) or, if the behavior truly exists, tell the user and ask whether to adjust the card.
3. Record the red evidence (command + summarized output per criterion) for the Delivery Record.

### 4. Implement — implement-agent per step

For each step of the card, in order:

1. Launch `implement-agent` with: the card path and step number, `map.md` (its Conventions are the plan conventions), and `${CLAUDE_PLUGIN_ROOT}/references/defensive-programming.md` as a mandatory convention. Tell it the card's Contract and Defensive contract are the normative spec, and that stubs created in the red phase are to be replaced by real implementations.
2. Verify the step's Done-when yourself (run the build/lint/check commands it names).
3. If implement-agent reports a blocker or Done-when fails after one retry with the failure output → [Step Failure](#step-failure).
4. Print: "Step N done — <brief description>".

Do not run the full test cycle per step here; acceptance tests are the target and they are driven green in the next stage.

### 5. Green — test-agent (normal mode)

1. Launch `test-agent` (normal mode) for the code produced by this outcome (diff relative to the baseline): add unit/integration tests for the new code, covering the defensive contract (each declared failure mode), and make **all** tests pass — the acceptance tests included, and the pre-existing suite unchanged in intent.
2. Verify yourself: run the full test suite and the acceptance tests. Every acceptance test must pass. If test-agent cannot get them green → [Agent Failure](#agent-failure).

### 6. Scoped Review — review-agent on this outcome's diff only

1. Launch `review-agent` restricted to the files changed relative to the baseline (`git diff --name-only <baseline>` plus untracked files created by this outcome, excluding pre-existing uncommitted changes noted at startup). Tell it the scope explicitly: review only these files and this outcome's changes; do not review the rest of the codebase. Include `${CLAUDE_PLUGIN_ROOT}/references/defensive-programming.md` as its defensive baseline extension, and the **card path** (section `## 5. Contract`, including its Defensive contract blocks) as the declared contract to check the code against.
2. Fix every finding (code and, if needed, tests) via implement-agent or test-agent as appropriate. Re-run the full test suite and the acceptance tests after fixes.
3. If a fix requires changing the card's Contract (type, signature, public behavior), ask the user first and record the change as a **deviation**.

### 7. Docs — docs-agent

Launch `docs-agent` once for this outcome's changes: doc comments on exposed symbols, README if any public API, configuration or behavior changed, CHANGELOG entries under the current version section.

### 8. Close — Delivery Record, map update, Gate 2

1. Re-run the acceptance tests and the full suite one last time. Both must be green; if not, go back to Green.
2. Replace the card's `## 7. Delivery Record` placeholder with:

   ```markdown
   ## 7. Delivery Record

   **Delivered:** <YYYY-MM-DD>   **Baseline:** `<commit sha>`

   | Criterion | Test (path › name) | Red evidence | Green evidence |
   |-----------|--------------------|--------------|----------------|
   | AC-1 | `<path>` › `<name>` | failed: <reason> | passed |

   **Files changed:** <path — one-line purpose, per file>
   **Tests:** <added/updated count> — command: `<exact command>` — result: <N passed, 0 failed>
   **Review findings:** <found> found, <fixed> fixed, <open> open — <one line each, or "none">
   **Docs updated:** <files, or "none">
   **Deviations from card:** <list, or "none">
   **Learnings:** <what reality taught that the map did not know — new risks, wrong assumptions, follow-up outcomes, or "none">
   ```

3. Update `map.md`:
   - Outcome status → `delivered`.
   - Append a row to the **Learnings Log** table (`| Date | Outcome | Learning | Map change |`): date, `O-NN`, the learnings, and what it implies for the map (or `none`).
4. **Gate 2** — if the learnings affect the backlog (an outcome becomes unnecessary, a new outcome is needed, an order or dependency changes, a later outcome is now too big), propose the concrete backlog changes with a rationale for each and ask the user to approve. Apply only the approved changes. If a learning invalidates a runway decision or the north-star, do not change it here — recommend `/odd-map <map.md>` (refine mode).
5. **Final summary**:
   - Outcome delivered, acceptance criteria and their tests
   - Steps completed (with task IDs)
   - Tests created/updated
   - Review findings found and fixed
   - Documentation updated
   - Deviations from the card
   - Backlog changes applied, and the next eligible outcome (or "none — run /odd-map refine")
   - **Suggested commit message** (one commit per outcome), e.g.:

     ```
     feat(<area>): <outcome statement, imperative>

     Delivers O-NN of <map name>. Acceptance: AC-1..AC-n green.
     ```

6. Remind the user to commit when ready. **Do NOT commit.**

## Error Handling

### Step Failure

If a card step cannot be completed as specified:
1. Identify the blocker (missing dependency, ambiguous card, conflict with the runway or a delivered contract).
2. Present it to the user with context.
3. Ask how to proceed:
   - **Fix the card**: the user gives guidance; update the card through outcome-agent (change request) and record the change as a deviation.
   - **Split**: the outcome is bigger than it looked — stop, mark the outcome `pending` again, re-invoke outcome-agent asking for a split proposal (`SPLIT_PROPOSED`), then apply it as in the Card stage.
   - **Abort**: stop; leave the outcome `in-progress`, and summarize what was done.

### Agent Failure

If test-agent, review-agent or docs-agent reports an issue it cannot resolve: present it to the user and ask for guidance before continuing.

### Circular Fixes

If a review fix causes a test to fail (or vice versa):
1. Do NOT enter an infinite fix loop.
2. After **2 iterations** of fix → review → test, stop and present the full history of the conflict to the user.
3. Ask for guidance on how to resolve it.

## Rules

- **One outcome per run.** Never start the next outcome in the same run — the user decides when (typically after committing).
- Never skip a stage: Card → Gate 1 → Red → Implement → Green → Scoped Review → Docs → Close. The outcome is NOT delivered until Close has run.
- Never implement before Gate 1 approval, and never implement before the acceptance tests have been observed failing for the right reason.
- The review is scoped to this outcome's diff; do not review, refactor or "fix" unrelated code.
- Never modify files outside this outcome's scope unless a review finding or test failure requires it.
- Never contradict the map's Architecture Runway; runway changes happen only in `/odd-map` refine mode.
- Ask questions one at a time, with choices where possible.
- **NEVER create git commits.** The user commits — one commit per outcome is recommended.
- Keep the map and the card consistent: the map's status and Learnings Log, and the card's Delivery Record, are updated before the final summary.
- All code, comments, documentation, cards and map content in English.
- When resuming, check `TaskList` and the map/card state first, and continue from the first incomplete stage.
