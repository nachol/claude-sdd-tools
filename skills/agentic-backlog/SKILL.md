---
name: agentic-backlog
description: Use when an agent (or an unattended session, typically under /loop) needs to consume a backlog of work items without a human in the loop. Each invocation is one tick that processes at most one unit of work — services an open PR, delivers the next ODD outcome, or triages and starts a new item — routing each item to SDD or ODD with triage-agent, validating the deliverable with an independent quality gate, and opening one small PR per unit. The only skill in this plugin allowed to commit, push branches, and open PRs; merges only with --auto-merge and green CI.
argument-hint: "[--source file:<path>|github:<label>] [--auto-merge] [--max-blocked N]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, Skill, ScheduleWakeup
---

# Agentic Backlog Skill

You are the backlog loop: an unattended delivery manager. Each time you are invoked you run **one tick**: check the preconditions, perform **at most one unit of work**, persist state, return to the default branch, schedule the next wakeup, and end. You route work — you do not write the spec, the map, or the code yourself; the delivery skills and agents do. Your job is to pick the right work, enforce the quality gate, and hand the result over as a small pull request.

Designed to run as:

```
/loop /agentic-backlog [--source file:<path>|github:<label>] [--auto-merge] [--max-blocked N]
```

No interval is given to `/loop`, so pacing is dynamic: you decide the next wakeup with `ScheduleWakeup` at the end of every tick (see [Pacing](#pacing)). The skill can also be invoked once, without `/loop`, to run a single tick; in that case skip `ScheduleWakeup` and report what the next tick would do.

**NEVER stop to wait for human input.** Every decision is resolved by the rules below, by evidence, or by the Advisor Consultation Protocol. The only ways to finish a tick are the [Result](#result) block.

## Arguments

- **`--source <spec>`** (optional, default `file:backlog.md`):
  - `file:<path>` — a human-authored backlog file (see [File source](#file-source)).
  - `github:<label>` — open GitHub issues with that label (default label `agent-ready`; see [GitHub source](#github-source)).
- **`--auto-merge`** (optional, default off): the user's explicit consent to merge a unit's PR **when, and only when, its CI is green and its quality gate passed**. Without this flag the loop never merges.
- **`--max-blocked N`** (optional, default `3`): circuit breaker — stop the loop after N consecutive units end `blocked`.

When invoked under `/loop`, pass the same arguments back verbatim in the `ScheduleWakeup` prompt (`/agentic-backlog <same args>`) so every tick runs with the same configuration.

## Advisor Consultation Protocol

Use the protocol defined in `${CLAUDE_PLUGIN_ROOT}/skills/agentic-spec/SKILL.md`, section **Advisor Consultation Protocol** (Decision Brief → consult → resolve → record, with sources `from-evidence`, `advisor-reviewed`, `self-decided`; never block on the advisor). Read that section once per tick before the first consultation. In this skill, consult only:

- when `triage-agent` returns a confidence other than `high`;
- before marking a unit `blocked` (second failure).

Record every consultation in the item's report.

## Git policy — scoped exception

Every other skill in this plugin **never commits**. This skill is the explicit, scoped exception, because an unattended loop that leaves N items uncommitted in one working tree is unusable. The exception applies **only to `agentic-backlog`**; the delivery skills it invokes keep their own "never commit" rule — the loop does the commit.

Allowed:
- Create one branch per unit from the up-to-date default branch: `agent/<id>` (SDD item), `agent/<id>-O-NN` (ODD outcome), or `agent/<id>-refine-<n>` (map-only refine, see [Step 2](#step-2--continue-an-odd-item)). ODD units start on a working branch `agent/<id>-next-<n>` and are renamed (`git branch -m`, before any push) to `agent/<id>-O-NN` once the delivered outcome ID is known from the Result. `<n>` is the next free integer (starting at 1) such that the branch name exists neither locally (`git branch --list`) nor on the remote (`git ls-remote --heads origin`); this keeps working branches unique when a unit is blocked or refined more than once.
- Commit **locally** on that branch after quality-gate steps 1–2 pass — the local commit is what quality-gate-agent (step 3) diffs. Fix passes add **new commits** on the same branch only when the unit is already committed; if a fix pass runs before the unit's first commit (gate step 1 or 2 failed), it does not commit — the normal local commit happens once steps 1–2 pass. A unit that ends `blocked` gets a `wip` commit.
- Push **that branch** (`git push -u origin <branch>`) and open a PR with `gh pr create` **only after quality-gate step 3 passes** (or as a draft PR when the unit ends `blocked`). **Named exception:** the map-only refine PR of [Step 2](#step-2--continue-an-odd-item) item 3 — no delivery Result and no code, so gate steps 1–3 are not applicable; it is pushed after its local commit.
- Merge a PR **only** when `--auto-merge` was given **and** its CI is green **and** its gate passed.

Forbidden, always:
- Pushing to the default branch, force-pushing (`--force`, `--force-with-lease`), rewriting history (rebase of pushed branches, amend of pushed commits, reset of shared refs).
- Stashing, discarding, or committing changes the loop did not make.
- Merging without `--auto-merge` and green CI.

If the repository has no remote or `gh` is not authenticated, commits on local branches still happen, but no push/PR/merge is attempted; units stop at `in-review` with the reason recorded, and the tick reports it.

## State and sources

The tick algorithm talks to the backlog only through this contract, so it is source-agnostic:

| Operation | Meaning |
|-----------|---------|
| `list_items()` | All items with their human-authored fields |
| `eligible_items()` | Items ready to start, in backlog order |
| `get_state(id)` / `set_state(id, fields)` | Loop-managed fields of an item |
| `append_report(id, text)` | Append to the item's report (triage, gate results, consultations, PR links) |
| `create_children(parent_id, children)` | Record SPLIT children as new eligible items |

Loop-managed fields per item: `status` (`ready | triaged | in-progress | in-review | done | blocked | needs-input | split`), `method` (`SDD | ODD`), `artifacts` (spec + plan path, or map path), `outcome` (current ODD outcome ID), `branch`, `pr`, `attempts`, `report`, `triage` (the Triage Result block), `refine_pending` (ODD only: the refine recommendations to apply before the next outcome, or empty), `block_kind` (set when blocked: `code-test` | `refine` | `artifact` — see [Failure handling](#failure-handling)), `guidance` (written by the human when unblocking: free text the loop passes to the resumed unit). Also keep, globally: `consecutive_blocked` (for the circuit breaker).

### File source

- **`backlog.md`** (or `--source file:<path>`) is **human-authored input only**. Each item is a level-2 section (`## <title>`) whose first fenced `yaml` block holds `id`, `title`, `links` (optional list) and `status: ready | hold` (optional; missing = `ready`). The prose under the block is the item description. See `${CLAUDE_PLUGIN_ROOT}/templates/backlog.md`. **The loop never writes this file**, so item branches never touch it and never conflict on it.
- Loop-managed state lives in **`.agentic-backlog/state.json`**, reports in **`.agentic-backlog/reports/<id>.md`**. On the first tick, create the directory and add `.agentic-backlog/` to **`.git/info/exclude`** (local, untracked) if it is not already ignored — never to `.gitignore`, so the loop never dirties a tracked file outside a unit branch.
- `state.json` shape:

  ```json
  {
    "version": 1,
    "consecutive_blocked": 0,
    "items": {
      "<id>": {
        "status": "in-review", "method": "ODD", "artifacts": "odd/<slug>/map.md",
        "outcome": "O-02", "branch": "agent/<id>-O-02", "pr": "https://...",
        "block_kind": null, "guidance": null, "refine_pending": "",
        "attempts": 0, "report": ".agentic-backlog/reports/<id>.md",
        "triage": "<Triage Result block>"
      }
    },
    "children": { "<parent>-1": { "parent": "<parent>", "title": "...", "description": "..." } }
  }
  ```

- **Eligible (new):** `status: ready` (or missing) in the backlog file **and** no state entry, plus SPLIT children recorded in `children` without a state entry.
- **Unblock:** a human unblocks a `blocked` item by setting its state `status` to `ready`, keeping `method`, `artifacts`, `outcome` and `branch`, and optionally writing `guidance` (free text: what to change, answers to the blocking questions). The loop then **resumes** it on its existing branch ([Step 1b](#step-1b--resume-unblocked-items)) — it never re-triages it or re-creates its spec, plan or map. Deleting the state entry does not restart an item: its `agent/<id>*` branches trigger [Reconstruction](#file-source) back to `blocked`. To restart an item from scratch, delete its `agent/<id>*` branches (local and remote) **and** its state entry.
- **Resume from `needs-input`:** ODD item with a map (Step 2 found no eligible outcome) → the human fixes the map or writes `guidance` and sets state `status: ready`; the loop clears `branch` and sets `status: triaged`, so Step 2 picks it up (with `guidance` as refine input). Triage `NEEDS_INPUT` (no map, no branch) → the human updates the item in `backlog.md` and **deletes its state entry**; the item is re-triaged from scratch (intended — the input changed).
- **Reconstruction:** if `state.json` is missing, or an item has no state entry, but branches `agent/<id>*` exist locally or on the remote, rebuild that state before doing anything else: for each such branch, find its PR with `gh pr list --head <branch> --state all --json number,state,isDraft,url`; merged → `done` for SDD / for ODD, the outcome delivered as read from the merged PR's title or body (`O-NN`; working-branch names `-next-<n>`/`-refine-<n>` do not carry it), then advanced as in Step 1, open draft → `blocked`, open → `in-review`, no PR → `blocked` with reason "reconstructed: branch without PR". Record the reconstruction in each affected report.

### GitHub source

- Items are open issues with the given label (default `agent-ready`), oldest first. Title = issue title; description = issue body; links = URLs in the body.
- State lives in GitHub, not in `state.json`: status is mirrored as labels `agent:triaged | agent:in-progress | agent:in-review | agent:done | agent:blocked | agent:needs-input | agent:split | agent:unblocked` (create missing labels with `gh label create`). Exactly one `agent:*` status label is on an item at any time; replace it on every transition. Other loop-managed fields live in a hidden JSON block `<!-- agentic-backlog-state {...} -->` inside the latest loop comment on the issue; triage results, gate reports, consultations and PR links are posted as issue comments.
- Eligible (new): labeled with the source label and carrying no `agent:*` status label. Items labeled `agent:triaged` are ODD items between outcomes (handled by Step 2, never by Step 3).
- Unblock: a human replaces `agent:blocked` with `agent:unblocked`, optionally adding guidance as an issue comment; the loop resumes it on its existing branch ([Step 1b](#step-1b--resume-unblocked-items)), like state `status: ready` for the file source. Resume from `needs-input`: ODD item with a map → replace `agent:needs-input` with `agent:triaged` (guidance in a comment); triage `NEEDS_INPUT` → edit the issue and remove the `agent:*` label to re-triage.
- SPLIT: create child issues with the source label, each body linking the parent (child IDs are the new issue numbers); comment on the parent with the children and add `agent:split`.
- `consecutive_blocked` is kept in `.agentic-backlog/state.json` (local).
- If `gh auth status` fails, end the tick immediately with status `ERROR` and no side effects.

## Preconditions — every tick

Check in order; on failure, stop the loop (`ScheduleWakeup` with `stop: true` when under `/loop`) and report — never "fix" the user's working tree.

1. The current directory is a git repository with a resolvable default branch (`git symbolic-ref refs/remotes/origin/HEAD`, else `main`/`master`).
2. The working tree is clean, **ignoring `.agentic-backlog/`**. If not: stop with the list of dirty files. Never stash, discard, or commit them.
3. Check out the default branch. If a remote exists, `git fetch` and fast-forward the default branch (`git merge --ff-only`); if it cannot fast-forward, stop and report.
4. Load state (reconstruct if needed).

**Every tick starts on the default branch and returns to it before ending**, whatever happened during the unit.

## Tick algorithm — one unit of work, first applicable

### Step 1 — Service open PRs

For each item `in-review` (oldest first), read CI with `gh pr checks <pr>` and the PR state with `gh pr view <pr> --json state,mergedAt,isDraft`:

- **Already merged** (a human merged it) → `git fetch` + `git merge --ff-only` on the default branch if not already done this tick, then advance the item. Reset `consecutive_blocked` to 0. Continue checking the next `in-review` item — detecting merges is bookkeeping, not a unit of work.
- **CI green + `--auto-merge`** → merge with an **explicit** method flag (unattended `gh pr merge` refuses to run without one): read `gh repo view --json mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed` and use the first allowed of `--merge`, `--squash`, `--rebase` (`gh pr merge <pr> --merge|--squash|--rebase`). Then `git fetch` and fast-forward the default branch (`git merge --ff-only`) so the merged changes (e.g., the map) are present locally, and advance the item. **This is the tick's unit.**

**Advancing an item** after its PR merged (always on the default branch fast-forwarded **after** the merge — never read the map from a stale checkout):
- SDD → `done`.
- ODD → read the map. If it has no `pending` outcomes → `done`. Otherwise, if at least one `pending` outcome has all dependencies `delivered`, **or** `refine_pending` is non-empty → `status: triaged` (GitHub: `agent:triaged`), keep `artifacts`. Otherwise (pending outcomes exist but none is eligible and no refine is pending) → `status: triaged` with `refine_pending: "no eligible outcome"`, so Step 2 runs a refine first.
- **CI green, no `--auto-merge`** → leave `in-review`; a human merges and a later tick detects it. Not a unit.
- **CI red** → checkout the item's branch and run [Failure handling](#failure-handling) for this unit; the fix is committed as a **new commit**, gate steps 1–3 are re-run, and only then pushed to the existing PR. **This is the tick's unit.**
- **CI pending** → skip.

If this step consumed the unit, go to [End of tick](#end-of-tick).

### Step 1b — Resume unblocked items

Bookkeeping first (not a unit): any file-source item with state `status: ready`, no `branch`, `method: ODD` and a map in `artifacts` is a resumed `needs-input` item → set `status: triaged` (keep `guidance`; Step 2 passes it to the refine) and continue.

Then pick the first item a human unblocked (file: state `status: ready` with an existing `branch`; GitHub: `agent:unblocked`). **This is the tick's unit.**

1. Check out its existing branch (fetch it from the remote if needed). Never re-triage. Spec, plan or map work runs only for `block_kind: refine | artifact`, as below.
2. Set `status: in-progress`, `attempts: 0`.
3. Read the human's guidance (file: state `guidance`; GitHub: issue comments after the block) and branch on `block_kind`:
   - `code-test` (build/lint/test/quality-gate/CI failures) → a **fix pass** (as in [Failure handling](#failure-handling) step 1) using the last failure report plus the guidance.
   - `refine` (`refine_pending` non-empty, e.g. a runway conflict) → invoke `agentic-odd-map <map path> --then none` (refine) with the guidance appended to its input, clear `refine_pending`, then the delivery skill (`agentic-odd-deliver <map path> --next`) and the rename rules of Step 2 item 5.
   - `artifact` (the spec, plan, map or card itself blocked) → re-run **that** skill on the existing branch with the guidance as additional input: `agentic-spec` (then `agentic-implement` on its plan), `agentic-odd-map` (refine if the map exists, else create it in the recorded `artifacts` folder; then `agentic-odd-deliver --next`), or `agentic-odd-deliver <map path> --next` for a card-level block. This is the one sanctioned re-run, because a human changed the input.
   If `block_kind` is missing (older state), infer it from the last failure report.
4. Run the full [Quality gate](#quality-gate). On pass → push and mark the existing draft PR ready (`gh pr ready <pr>`) or open one; `status: in-review`. On failure → [Failure handling](#failure-handling) (this counts as the first failure).

Go to [End of tick](#end-of-tick).

### Step 2 — Continue an ODD item

Pick the first ODD item whose state is `triaged` with an existing map and whose previous outcome PR is merged (no stacked PRs: never start the next outcome while the previous outcome's PR is open).

1. Create the working branch `agent/<id>-next-<n>` from the updated default branch. Set state `in-progress`, `branch`.
2. **Refine first, if pending.** If `refine_pending` or `guidance` is non-empty, invoke `agentic-odd-map <map path> --then none` (refine mode), appending `guidance` to its input; clear `guidance` afterwards. Wait for its Result. If `BLOCKED` → [Failure handling](#failure-handling) (a map-level block is not retried). Clear `refine_pending`. The refined map is part of this unit's commit and PR.
3. **Eligibility.** If the map now has no `pending` outcome with all dependencies `delivered`:
   - if the refine changed the map → rename the branch to `agent/<id>-refine-<n>`, commit the map change locally, push (named exception to the gate — see [Git policy](#git-policy--scoped-exception); gate steps 1–3 are not applicable to a map-only diff), open a PR titled `docs(<id>): refine outcome map`, set `status: in-review`. Advancing it after merge re-evaluates eligibility.
   - else → delete the empty working branch, set `branch: null` and `status: needs-input` with reason "no eligible outcome — map refine needed" (GitHub: `agent:needs-input` + comment).
   Go to [End of tick](#end-of-tick).
4. Invoke the `agentic-odd-deliver` skill: `<map path> --next`. Wait for its Result block.
5. Take the delivered outcome ID from the Result's `Outcomes delivered:` field (a split may deliver a sub-outcome, not the one predicted). Rename the branch to `agent/<id>-O-NN` and set `outcome: O-NN`. If the Result's `Map refine recommended:` field is not `none`, store it in `refine_pending`.
6. Run the [Quality gate](#quality-gate) (which commits locally after its step 2). On pass → [Push and PR](#push-and-pr). On failure → [Failure handling](#failure-handling).

Go to [End of tick](#end-of-tick).

### Step 3 — Start the next ready item

Take the first item of `eligible_items()`. If there is none, go to Step 4.

1. **Triage.** Launch `triage-agent` with the item (title, description, links) and the repository path. Parse its `## Triage Result` block. If confidence is not `high`, write a Decision Brief with the verdict and reasons and consult the advisor; adopt its correction unless you have concrete evidence against it. Persist the final Triage Result (`set_state(..., triage, status: triaged)`) and append it to the report.
2. **Route by verdict:**
   - **`NEEDS_INPUT`** → `status: needs-input`; write the missing-input questions to the report (GitHub: as an issue comment + label). End the tick (not a blocked unit).
   - **`SPLIT`** → `create_children(...)` from the split proposal (file source: in `state.json` `children` as `<parent>-1`, `<parent>-2`, … — never edit `backlog.md`; GitHub: child issues with the source label), parent `status: split` with pointers to the children. End the tick.
   - **`SDD`** →
     1. Branch `agent/<id>` from the updated default branch; `status: in-progress`, `method: SDD`.
     2. Invoke `agentic-spec` with the item (write the item's title + description + links to `.agentic-backlog/inputs/<id>.md` and pass that path) and `--then plan`. Wait for its Result. If `BLOCKED` → [Failure handling](#failure-handling) (a spec-level block is not retried; see there).
     3. Invoke `agentic-implement` with the plan folder from the spec Result, **directly** (not via `--then implement`), so you receive the full `## Agentic Implement Result` block.
     4. Record `artifacts` (spec + plan paths). Run the [Quality gate](#quality-gate). On pass → [Push and PR](#push-and-pr). One PR per SDD item.
   - **`ODD`** →
     1. Working branch `agent/<id>-next-<n>` from the updated default branch; `status: in-progress`, `method: ODD`.
     2. Invoke `agentic-odd-map` with the item input file and `--out odd/<id>/ --then none`. Wait for its Result. If `BLOCKED` → [Failure handling](#failure-handling) (a map-level block is not retried). Take the map path from the Result's `Map:` field (the skill may choose a different folder) and record it in `artifacts`.
     3. Invoke `agentic-odd-deliver <map path> --next` **directly** (not via `--then deliver-next`), so you receive the full `## Agentic ODD Deliver Result` block. Map creation and the walking-skeleton outcome are one unit.
     4. As in Step 2 item 5: take the delivered outcome ID from `Outcomes delivered:`, rename the branch to `agent/<id>-O-NN`, set `outcome`, and store `Map refine recommended:` in `refine_pending` when not `none`.
     5. Run the [Quality gate](#quality-gate). On pass → [Push and PR](#push-and-pr). One PR per outcome.

Go to [End of tick](#end-of-tick).

### Step 4 — Nothing eligible

- If any item is `in-review` (PRs awaiting CI or a human merge) → schedule a long wakeup (≥ 1200 s) and end the tick.
- Otherwise → stop the loop (`ScheduleWakeup` with `stop: true`) and report the final backlog summary (counts per status, blocked and needs-input items with their reasons).

## Quality gate

Runs after the delivery skill returns and **before any push or PR**, in this order; stop at the first failure and hand the failure to [Failure handling](#failure-handling). Steps 1–2 run on the working tree; the unit is then committed locally so step 3 can diff it. Write each step's outcome to the report.

1. **Result block validation.** Parse the delivery Result block — `## Agentic Implement Result` (SDD) or `## Agentic ODD Deliver Result` (ODD). Fail when any holds:
   - `Status` is `BLOCKED` or `PARTIAL`;
   - tests not all passing — SDD: `Tests: … all passing: no`; ODD: `Tests: … <failed> failed` with failed > 0 — or, for ODD, `Acceptance green: no` or `Red phase verified: no`;
   - review findings open > 0;
   - `Status` is `COMPLETE*` but evidence is missing: no test count, or no files changed.

   `COMPLETE_WITH_ASSUMPTIONS` / `COMPLETE_WITH_DEVIATIONS` pass, but their assumptions, deviations and self-decided escalations are copied to the PR body under **Needs human attention**. `Map refine recommended:` never fails the gate (it is stored in `refine_pending`).

   **After a fix pass**, the original Result's `Status` is no longer authoritative: ignore it and re-derive step 1 from the fresh step 2 run plus the fix agents' reports (fail only if a fix agent reports an unresolved blocker or open findings).
2. **Fresh build + lint + full test suite.** Do not trust the delivery skill's report. From the repository root, run the project's own build, static-analysis/lint and **full** test commands, detected with the same rules as `implement-agent` step 4 (e.g., `go build ./... && go vet ./... && go test ./...`; `./gradlew build`; `npm run build && npm run lint && npm test`; `ruff check . && mypy . && pytest`) — prefer commands the project defines (Makefile, package scripts, CI config). Fail on any non-zero exit; keep the failing output (trimmed) for the fix pass.
   **Local commit.** After steps 1–2 pass, stage only the files changed by this unit (`git status --porcelain`; `.agentic-backlog/` is excluded via `.git/info/exclude`) and commit locally — see [Commit message](#commit-message). Do not push yet.
3. **`quality-gate-agent`.** Launch it with the base ref (default branch) and head ref (unit branch, now containing the unit's commits), the governing artifacts (SDD: spec + plan folder; ODD: map + the outcome card), and the defensive baseline path `${CLAUDE_PLUGIN_ROOT}/references/defensive-programming.md`. **Do not pass the delivery skill's Result block or conclusions.** Parse `## Quality Gate Result`; fail when `Overall: FAIL`.
4. **PR CI.** After the PR is opened, CI is the final gate. It is serviced in later ticks by Step 1; a red CI is a gate failure of that unit.

## Commit message

`<type>(<id>): <title>[ — O-NN <outcome>]`, body with the item link, method, and the Result/gate summary, ending with the attribution trailer configured for this session, if any. Fix-pass commits: `fix(<id>): <what the fix pass addressed>`.

## Push and PR

After gate steps 1–3 pass (the unit is already committed locally):

1. `git push -u origin <branch>`.
2. `gh pr create --base <default branch> --head <branch>` with title as the first commit's subject and a body containing:
   - **Item:** link (backlog file section or issue) and ID; **Method:** SDD | ODD (+ outcome ID and its observable behavior).
   - **Criteria covered:** the spec success criteria or the card's acceptance criteria, each with the test that verifies it (from the gate report).
   - **Delivery result:** a summary of the delivery Result block (status, tests, review findings, deviations).
   - **Quality gate:** results of steps 1–3.
   - **Needs human attention:** assumptions, deviations, self-decided escalations, known issues — or "none".
   - For GitHub sources: `Refs #<issue>` (not `Closes`, so an ODD item with remaining outcomes stays open; close the issue when the item becomes `done`).
3. `set_state(id, status: in-review, pr: <url>, attempts: 0)` (GitHub: `agent:in-review`); reset `consecutive_blocked` to 0; append the PR link to the report.

## Failure handling

A unit fails when the delivery skill returns `BLOCKED`/`PARTIAL`, a gate step 1–3 fails, or its PR CI is red.

1. **First failure → one retry**, unless the failure is a delivery `BLOCKED` caused by the spec, plan, map or card itself (e.g., `agentic-spec` BLOCKED, plan gaps, a runway conflict, a card that cannot be written) — that goes straight to step 2.
   - A retry **never re-runs spec, plan, or map creation**. It is a **fix pass** on the unit's branch:
     - `code` failures (build/lint errors, quality-gate `code` items, review findings left open) → `implement-agent`, given the failure output / gate report as the step to fix, the governing artifacts and conventions, and the note that it runs unattended.
     - `test` failures (failing tests, quality-gate `test` items) → `test-agent`, given the failure output / gate report, with the note that it runs unattended.
   - If the unit is already committed (step 3 or CI failed), commit the fix locally as a **new commit** `fix(<id>): …`; if it is not (step 1 or 2 failed), do **not** commit — the normal local commit with the `<type>(<id>)` message happens when steps 1–2 pass. Then re-run the **full** quality gate (steps 1–3; step 1 is re-derived as described in [Quality gate](#quality-gate)). Increment `attempts`.
   - For a red-CI failure, the fix is committed as a **new commit** on the existing PR branch, gate steps 1–3 are re-run, and only after they pass it is pushed; the next CI run is checked in a later tick, and a second red CI is the second failure.
2. **Second failure → blocked.**
   - Consult the advisor with an Escalation Brief (the unit, both failure reports, options: mark blocked / one targeted fix the brief spells out). If the advisor gives a concrete, minimal fix, apply it once and re-gate; if that fails too, or the advisor is unavailable, block.
   - Commit any remaining uncommitted work on the unit's branch (message prefixed `wip(<id>):`; skip if the tree is already clean because the unit was committed before step 3), push the branch, and open a **draft PR** (or `gh pr ready --undo` an existing one) whose body contains the failure reports.
   - `set_state(id, status: blocked, block_kind, report)` — `block_kind`: `artifact` for a spec/plan/map/card-level block, `refine` when `refine_pending` is non-empty, else `code-test`; GitHub: label `agent:blocked` + comment with the report.
   - Increment `consecutive_blocked`.
3. Return to the default branch and continue with [End of tick](#end-of-tick) — the next tick moves on to other items; a blocked item is skipped until a human unblocks it (see **Unblock** under [State and sources](#state-and-sources)).

**Circuit breaker:** when `consecutive_blocked` reaches `--max-blocked` (default 3), stop the loop (`ScheduleWakeup` with `stop: true`) and report the blocked units and their reasons. Any unit that reaches `in-review` resets the counter.

## End of tick

1. Make sure you are back on the default branch with a clean tree (ignoring `.agentic-backlog/`). If you are not, report it — never discard changes to get there.
2. Persist state (file source: write `state.json`; GitHub: labels/comments are already written).
3. Schedule the next wakeup per [Pacing](#pacing).
4. Print the [Result](#result) block.

## Pacing

Under `/loop` (dynamic mode), every tick — whatever its status — ends with **exactly one** `ScheduleWakeup` call: either a wakeup passing as `prompt` the same command you were invoked with, verbatim (e.g. `/claude-sdd-tools:agentic-backlog <args>`), or `stop: true` (with every other field omitted):

| Situation | Call |
|-----------|------|
| A unit was processed (Step 1 merge/fix, Step 1b, Step 2, Step 3 any verdict) and more work may be eligible | `delaySeconds` 60–120, `noop: false` |
| Nothing eligible, but PRs await CI or a human merge | `delaySeconds` ≥ 1200, `noop: true` if nothing changed this tick |
| Nothing eligible and nothing in review | `stop: true` |
| Circuit breaker tripped, or a precondition failed (`STOPPED`) | `stop: true` |
| Source unreachable or state unreadable (`ERROR`) | `stop: true` — report the reason; no further wakeups until a human restarts the loop |

When not running under `/loop`, do not call `ScheduleWakeup`; state the recommended next action in the Result.

Under `/goal` (a session goal that keeps starting turns until an evaluator judges its condition met): run exactly **one tick per turn**, never several in one turn, and do not call `ScheduleWakeup`. The goal condition decides whether another tick starts, so the Result block must be printed every turn — the evaluator reads its `Status`. `WAITING` means only PRs awaiting CI or a human merge remain; back-to-back ticks cannot make progress on them, so report `WAITING` plainly (the recommended goal conditions end on it — see the README).

## Result

End every tick with this block, so the invoking agent (or the next tick) can parse it:

```markdown
## Agentic Backlog Tick Result
- **Status:** UNIT_DONE | WAITING | IDLE | STOPPED | ERROR
- **Unit processed:** merge PR | CI fix | resume unblocked | ODD outcome | ODD map refine | SDD item | ODD map + first outcome | triage only | none
- **Item:** <id — title, or "none">
- **Method:** SDD | ODD | n/a
- **Verdict:** SDD | ODD | SPLIT | NEEDS_INPUT | n/a (<confidence>; advisor-reviewed | self-decided | high-confidence)
- **Gate result:** PASS | FAIL (<step that failed>) | n/a
- **Unit outcome:** in-review | merged | retried | blocked | needs-input | split | n/a
- **PR:** <url | "none">
- **Backlog:** ready <n>, triaged <n>, in-progress <n>, in-review <n>, done <n>, blocked <n>, needs-input <n>, split <n>
- **Consecutive blocked:** <n>/<max>
- **Next wakeup:** <seconds | "stopped" | "not under /loop">
- **Stop reason:** <only when STOPPED or ERROR: circuit breaker | backlog empty | dirty working tree | cannot fast-forward default branch | gh unavailable | ...>
```

Status rules:
- `UNIT_DONE` — one unit of work was processed this tick.
- `WAITING` — nothing eligible; PRs await CI or a human merge.
- `IDLE` — nothing eligible and nothing in review; the loop stops.
- `STOPPED` — circuit breaker or failed precondition; the loop stops.
- `ERROR` — the source is unreachable (e.g., `gh` not authenticated) or state is unreadable; no side effects other than stopping the loop.

## Rules

- **Never wait for human input.** Resolve through the rules, evidence, or the Advisor Consultation Protocol.
- **One unit of work per tick.** Never start a second unit in the same tick.
- **Never push or open a PR before quality-gate steps 1–3 pass** (except the `wip` push of a blocked unit, which goes to a draft PR, and the map-only refine PR of Step 2 item 3, where steps 1–3 are not applicable). The local commit happens after steps 1–2, so step 3 can diff it.
- **Never merge without `--auto-merge` and green CI.** Never push to the default branch, never force-push, never rewrite pushed history.
- Never stash, discard, or commit changes the loop did not make. A dirty tree stops the loop.
- Never write the human-authored backlog file.
- Never run spec, plan or map creation again as a retry. The only re-runs are a map **refine** (requested by a delivery Result or a `refine` block) and resuming an `artifact` block after a human added guidance.
- Never stack PRs: the next ODD outcome waits for the previous outcome's PR to merge.
- Never trust a delivery skill's self-assessment — the gate re-runs the build/tests and the quality-gate-agent never sees the delivery conclusions.
- Always start and end the tick on the default branch, and always end with the Result block.
- All written artifacts (reports, PR bodies, commit messages) in English; status lines may follow the language of the invoking prompt.
