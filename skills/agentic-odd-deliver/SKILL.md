---
name: agentic-odd-deliver
description: Use when an agent (or an unattended session, such as agentic-backlog or agentic-odd-map --then deliver-*) needs to deliver one or more outcomes of an ODD outcome map without a human in the loop. Agentic variant of /odd-deliver — card, red acceptance tests, implementation, green, scoped review and docs per outcome, with every gate resolved through the advisor instead of the user. Never commits.
argument-hint: "<path to map.md> [O-id | --next | --all]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, Skill, TaskCreate, TaskGet, TaskList, TaskUpdate
---

# Agentic ODD Deliver Skill

You deliver outcomes of an Outcome-Driven Development (ODD) map **autonomously**. The flow is exactly the one of the interactive `/odd-deliver` skill — Card → Gate 1 → Red → Implement → Green → Scoped Review → Docs → Close → Gate 2 — but every point where `/odd-deliver` would ask the user is resolved with the **advisor** instead: a stronger reviewer model that reads the full transcript when consulted.

**NEVER stop to wait for human input.** Do not ask questions in your response text and then end your turn. The only way to finish is the [Result](#result) block.

## Source of truth

The flow, stages, outcome selection rule, card self-check, Delivery Record template, map updates and rules are defined in the interactive skill:

`${CLAUDE_PLUGIN_ROOT}/skills/odd-deliver/SKILL.md`

**Read that file first, in full.** Execute its Startup, Flow (stages 1–8) and Rules exactly, with the differences listed below. If the file cannot be read, return `BLOCKED` — do not improvise the flow.

The **Advisor Consultation Protocol** (Decision Brief → consult → resolve → record; sources `from-input | from-evidence | advisor-reviewed | self-decided`; batching; never block on the advisor) is defined in:

`${CLAUDE_PLUGIN_ROOT}/skills/agentic-spec/SKILL.md` — section "Advisor Consultation Protocol"

**Read that section** and apply it wherever this skill says **"consult"**. For blockers and failures, use the **Escalation Brief** format and the **Safe-default rule** of `${CLAUDE_PLUGIN_ROOT}/skills/agentic-implement/SKILL.md` (section "Advisor Consultation Protocol").

## Input

- **Map** (required): path to `map.md`. If missing or not found, return `BLOCKED` with reason "no map".
- **Selector** (optional, default `--next`):
  - `O-id` — deliver exactly that outcome (it must be eligible per the selection rule; otherwise `BLOCKED`).
  - `--next` — deliver the next eligible outcome (an `in-progress` outcome is resumed first).
  - `--all` — deliver eligible outcomes one after another, in backlog order, **each fully closed (Close stage done) before the next starts**. Stop at the first outcome that ends `BLOCKED` or `PARTIAL`, and do not start further outcomes on top of it.

If no outcome is eligible, finish with status `COMPLETE` and `Outcomes delivered: none`, listing the remaining outcomes and why they are not eligible.

## Differences from /odd-deliver

1. **Startup confirmation.** Do not ask "Deliver O-NN now?". Print one status line (selected outcome, dependencies, baseline sha) and start. Pre-existing uncommitted changes are recorded (not fixed) and excluded from the scoped review and from `Files changed`.
2. **Unattended agents.** Tell every agent you launch (outcome-agent, test-agent, implement-agent, review-agent, docs-agent) that it runs unattended: it must return a blocker report instead of waiting for clarification; docs-agent chooses the option most consistent with existing documentation and lists its choices.
3. **Surfaced decisions (Card stage).** For outcome-agent's `PENDING` decisions, write **one Decision Brief** for the card with one numbered sub-item per decision (options, trade-offs, outcome-agent's recommendation, your recommendation), consult, resolve, log each in the map's **Decision Log**, and re-invoke outcome-agent with the resolutions. If the map has no `## 6. Decision Log` / `## 7. Open Questions for the Requester` sections (maps written by the interactive `/odd-map`), create them with the format of `${CLAUDE_PLUGIN_ROOT}/skills/agentic-odd-map/SKILL.md`; in the `Phase` column write `deliver/O-NN`. Decisions settled by the map or the codebase are logged `from-input` / `from-evidence` without consultation.
4. **Gate 1 (card approval).** Run the card self-check, then write a **Card Brief** (outcome, acceptance criteria with the failure-path ones marked, steps, defensive contract summary, scope vs. runway, your concerns) and consult, asking the advisor to challenge: criteria that are not observable, missing failure paths, steps that are horizontal layers, runway contradictions, scope creep. Apply corrections through outcome-agent as a **change request** (only the listed edits). Record the gate in the Decision Log (`Phase` = `deliver/O-NN`).
5. **Splits.** On `SPLIT_PROPOSED`, consult with the proposed sub-outcomes. After advisor review (or `self-decided` if unavailable), apply the split to the backlog exactly as `/odd-deliver` describes, log it, and continue with the first eligible sub-outcome — it counts as the same delivery unit for `--next`.
6. **Runway conflicts.** On `RUNWAY_CONFLICT`, consult. Runway changes are never made here: if the resolution is "change the runway", end the outcome as `BLOCKED` with reason "runway change required — run agentic-odd-map refine", leaving it `pending`, and list the required change under `Map refine recommended`. Reshaping the outcome within the runway is allowed and logged.
7. **Red-phase anomalies.** Unexpected passes or tests that cannot be made to fail for the right reason are escalated (Escalation Brief): strengthen the test, adjust the card (deviation), or mark the criterion as already satisfied with evidence.
8. **Gate 2 (backlog changes).** Write one **Backlog Change Brief** with every proposed change and its rationale from the learnings, consult, and apply only the changes that survive review. Changes to the runway or the north-star are never applied here; list them in the Result field `Map refine recommended` (not under Known issues — a refine recommendation does not affect the outcome status). The same applies when, after Close, no `pending` outcome is eligible.
9. **Final summary.** Replace the human-facing summary and commit reminder with the [Result](#result) block. The suggested commit message is still produced, inside the Result.

## Error Handling

Use the model of `agentic-implement`:

### Step Failure

If a card step cannot be completed as specified, write an Escalation Brief with these options and consult:

- **Adjust** — a concrete, minimal adjustment consistent with the card's Contract and the runway (spell it out). Apply via implement-agent (and outcome-agent, as a change request, to update the card), record as a deviation, continue.
- **Skip** — skip the step and every step that depends on it; the outcome can then only end `PARTIAL`. Record them.
- **Abort** — stop this outcome; it ends `BLOCKED`.

When `self-decided`, apply the Safe-default rule. Choose **Abort** only when continuing would build on a broken foundation that the remaining steps depend on. A split discovered mid-implementation (the outcome is bigger than the card) is handled as Abort with reason "split required" and the split proposal recorded in the Learnings Log; the outcome's map status follows the `BLOCKED` rule in [Outcome status](#outcome-status).

### Agent Failure

If test-agent, review-agent or docs-agent reports an issue it cannot resolve: consult with the issue, the agent's report and the options (fix it yourself concretely, accept as known issue, re-run the agent with more context). Apply and record. Known issues make the outcome `PARTIAL`.

### Circular Fixes

After **2 iterations** of fix → review → test without converging, consult with the full history, apply the resolution once. If it still does not converge, revert the files touched by this loop to the last state where the full suite and the acceptance tests passed, record the unresolved finding as a known issue (open finding), and continue. The outcome ends `PARTIAL`.

### Outcome status

Each delivered outcome gets one of:
- `COMPLETE` — Close done, acceptance and full suite green, 0 open findings, no deviations, no self-decided decisions.
- `COMPLETE_WITH_DEVIATIONS` — Close done, all green, 0 open findings, but at least one deviation or `self-decided` decision.
- `PARTIAL` — steps skipped, open findings, or known issues; acceptance may not be fully green. The map status stays `in-progress` and the Delivery Record states what is missing.
- `BLOCKED` — aborted before Close. Map status is `pending` (if nothing usable was produced and changes were reverted) or `in-progress` (otherwise), stated explicitly.

The overall status is the worst outcome status in the run (`BLOCKED` > `PARTIAL` > `COMPLETE_WITH_DEVIATIONS` > `COMPLETE`).

## Result

End your final response with this block. **Every field is mandatory** — write `none` or `0` explicitly instead of omitting a field. Every claim must be backed by the evidence named in the field (commands and their counts), because an independent quality gate parses this block and rejects completion claims without evidence.

```markdown
## Agentic ODD Deliver Result
- **Status:** COMPLETE | COMPLETE_WITH_DEVIATIONS | PARTIAL | BLOCKED
- **Map:** <path to map.md>
- **Baseline:** <commit sha at start>
- **Outcomes delivered:** <O-id — status — card path, one per outcome, or "none">
- **Outcomes remaining:** <O-id — status in map (pending|in-progress) — eligible: yes|no (unmet deps), one per outcome, or "none">
- **Acceptance green:** yes | no — command: `<exact command>` — <passed>/<total> acceptance tests (<failing test names, or "none">)
- **Red phase verified:** yes | no — <per outcome: <n> criteria failed for the right reason before implementation>
- **Tests:** <added> added, <updated> updated — full suite command: `<exact command>` — <passed> passed, <failed> failed, <skipped> skipped
- **Review findings:** <found> found, <fixed> fixed, <open> open — <one line per open finding: severity, file:line, summary, or "none">
- **Docs updated:** <files, or "none">
- **Deviations from card:** <one line each: O-id — what changed — why, or "none">
- **Backlog changes:** <applied changes (splits, new/dropped/reordered outcomes), or "none">
- **Escalations:** <total> (advisor-reviewed: <n>, self-decided: <n>) — <one line each: ID — situation — resolution — source, or "none">
- **Known issues:** <list, or "none">
- **Map refine recommended:** <changes to the runway/north-star, or "no eligible outcome", that need `agentic-odd-map` refine — one line each; or "none". Never affects Status>
- **Files changed:** <path list> (uncommitted — pre-existing uncommitted changes excluded: <list or "none">)
- **Suggested commit message:** <one per delivered outcome, or "none">
- **Blocking reason:** <BLOCKED only: reason and the stage where it stopped; otherwise "none">
```

## Rules

- **Never wait for human input.** Every question is resolved through evidence or the Advisor Consultation Protocol.
- All Rules of `/odd-deliver` apply, except those about asking the user (replaced by consultations), the commit reminder (replaced by the Result), and the one-outcome-per-run rule when `--all` is given.
- Never skip a stage. An outcome is not delivered until its Close stage has run; with `--all`, the next outcome never starts before the previous one is closed.
- Never implement before Gate 1 has been resolved and the acceptance tests have been observed failing for the right reason.
- Never change the map's Architecture Runway or north-star.
- **NEVER create git commits, push, or change branches.** The invoking agent or the user manages git.
- Print a one-line status at each stage transition ("O-NN: Red verified — 4/4 failing for the right reason").
- All code, comments, documentation, cards and map content in English.
- When resuming, check `TaskList` and the map/card state first, and continue from the first incomplete stage.
