---
name: workflow-status
description: Use when a human or an agent needs the current status of a project that uses this plugin's workflows — backlog loop items (agentic-backlog), ODD outcome maps and cards, and SDD specs and plans — including what needs human attention, open PRs and CI, blocked or needs-input items with their reasons, and inconsistencies between state sources. Strictly read-only; ends with a machine-readable result block.
argument-hint: "[--source file:<path>|github:<label>] [--item <id>] [--out <file>]"
allowed-tools: Read, Glob, Grep, Bash, Write
---

# Workflow Status Skill

You produce a **status dashboard** for the project in the current working directory: what the agentic and interactive workflows of this plugin have produced, where each piece of work stands, and — above all — **what needs a human now**.

You are **strictly read-only**. You never change state, never fix what you find, never create commits, branches or PRs, never run the delivery skills. Your only possible write is the report file when `--out` is given. Everything you report must come from an artifact you read or a command you ran; when you infer, say so.

## Input

- **`--source <spec>`** (optional): the backlog source, same format as `agentic-backlog`: `file:<path>` or `github:<label>` (label default `agent-ready`). Default: `file:backlog.md` if that file exists; otherwise GitHub only if open issues carry an `agent:*` label; otherwise no backlog section.
- **`--item <id>`** (optional): report only this backlog item, in full detail (state fields, triage result, report excerpt, branch, PR, CI, artifacts, and the status of its map or spec).
- **`--out <file>`** (optional): also write the report (without the Result block) to this file. Never overwrite a file outside the working directory; refuse and report instead.

## What you read

Read the artifact formats from their source of truth when a detail matters: `${CLAUDE_PLUGIN_ROOT}/skills/agentic-backlog/SKILL.md` (state fields, statuses, labels, branch names), `${CLAUDE_PLUGIN_ROOT}/skills/odd-map/SKILL.md` (map.md template), `${CLAUDE_PLUGIN_ROOT}/agents/outcome-agent.md` (card template), `${CLAUDE_PLUGIN_ROOT}/skills/agentic-spec/SKILL.md` (spec layout).

1. **Backlog loop**
   - File source: the backlog file (`## <title>` sections with a fenced `yaml` block: `id`, `title`, `links`, `status: ready | hold`), `.agentic-backlog/state.json` (`items.<id>`: `status`, `method`, `artifacts`, `outcome`, `branch`, `pr`, `attempts`, `report`, `triage`, `refine_pending`, `block_kind`, `guidance`; global `consecutive_blocked`; `children`), and `.agentic-backlog/reports/<id>.md` (read only the latest sections you need).
   - GitHub source: `gh issue list --label <label> --state all --json number,title,labels,state,url` and, for items needing detail, the hidden `<!-- agentic-backlog-state {...} -->` block in the latest loop comment (`gh issue view <n> --json comments`).
   - An item with no state entry and `status: ready` (or none) is **ready (not started)**; `hold` is **on hold**.
2. **ODD maps** — find maps with Glob (`**/map.md`, `**/map.partial.md`), excluding dependency and build folders (`node_modules`, `vendor`, `.git`, `dist`, `build`, `target`, `.venv`). Keep only files whose first heading starts with `# Outcome Map —`. For each: north-star (§1.2), Outcome Backlog table (§4: `ID | Outcome | Acceptance | Depends on | Status | Risk`), the last rows of the Learnings Log (§5), and — if present — Open Questions for the Requester (§7) and Decision Log (§6, count `self-decided`). For each outcome card in the sibling `outcomes/` folder, note whether its Delivery Record (`## 7. Delivery Record`) is filled.
3. **SDD specs** — find `**/spec.md` and `**/spec.partial.md` (same exclusions) that have the spec layout (a `## Appendix A. Decision Log` or the `/spec` section structure), and a sibling `plan/00-overview.md` if any. For each: title, Open Questions for the Requester (Appendix B), count of `self-decided` decisions (Appendix A), plan present, number of phases/steps.
4. **Git** — `git status --porcelain` (ignoring `.agentic-backlog/`), current and default branch (`git symbolic-ref refs/remotes/origin/HEAD`, fallback `main`), ahead/behind the remote using the existing remote-tracking refs (`git rev-list --left-right --count <default>...origin/<default>`; do not fetch — say the result is as fresh as the last fetch), and `git branch -a --list '*agent/*'`.
5. **GitHub PRs** (only if `gh auth status` succeeds; otherwise mark every PR/CI field "not checked" and continue): `gh pr list --state all --search "head:agent/" --json number,title,headRefName,state,isDraft,url,mergedAt,statusCheckRollup,reviewDecision --limit 100`.

Use only read-only commands. Allowed: `git status`, `git log`, `git branch`, `git rev-list`, `git symbolic-ref`, `git show`, `gh auth status`, `gh pr list/view`, `gh issue list/view`, `gh repo view`. Never run `git fetch`, `git checkout`, `git merge`, `git push`, `gh pr merge/create/edit`, `gh issue edit/comment`, `gh label create`, or anything that writes.

## Implementation status of SDD work

Outside the backlog loop, the implementation progress of an SDD plan is **not persisted on disk** (`agentic-implement` and `/implement` track it in session tasks). Determine it in this order and always label the source:
1. **loop** — the item's state (`in-progress | in-review | done | blocked`) and PR.
2. **git** — a merged or open PR / branch whose title, body or diff references the spec folder.
3. **inferred** — the share of files listed in the plan's steps (`**File(s):**`) that exist in the working tree. Report it as "inferred: N/M plan files present"; never as "done".

## Inconsistency checks

Report — never fix — every mismatch you find, each with both sides cited:
- State says `in-review` but the PR is merged or closed; state says `done` but the PR is open; state has a `pr` that does not exist.
- An `agent/*` branch (local or remote) with no state entry or issue that owns it (orphan), or a state `branch` that no longer exists.
- A map outcome `in-progress` with no item/branch working on it, or `delivered` without a filled Delivery Record.
- A map row whose `Depends on` lists a `dropped` outcome; a map where outcomes are `pending` but none is eligible and no refine is pending.
- `consecutive_blocked` ≥ the loop's `--max-blocked` default (3) — the circuit breaker has tripped or will on the next block.
- A dirty working tree or a default branch behind its remote (the loop's preconditions would fail).
- GitHub source: an issue with more than one `agent:*` status label.

## Report

Write the report in the language the user wrote in (default English). Keep tables compact; link every path and PR. Sections, in this order:

### 1. Summary
One line per area: backlog counts per status (`ready, on hold, triaged, in-progress, in-review, done, blocked, needs-input, split`), loop health (`consecutive_blocked`/3, working tree clean yes/no, default branch in sync yes/no/unknown), ODD (`<maps>` maps — `<delivered>/<total>` outcomes delivered), SDD (`<specs>` specs — implementation status counts by source).

### 2. Needs human attention
The most important section — list every item, most urgent first. If none, say "Nothing needs a human right now."
- **Blocked** items: id, title, `block_kind`, the blocking reason (from the latest report section), the branch/draft PR, and the exact unblock action (from the agentic-backlog rules: set state `status: ready` with optional `guidance`, or label `agent:unblocked`).
- **Needs-input** items: the concrete questions (from the Triage Result `Missing input` or the report), and the resume action (ODD with map vs triage `NEEDS_INPUT`).
- **PRs awaiting a human**: open, non-draft `agent/*` PRs — CI status (green/red/pending/none) and review decision. Red CI first.
- **Open Questions for the Requester** in maps and specs (count + the questions), and the number of `self-decided` decisions to review.
- **Inconsistencies** from the checks above.

### 3. Backlog
Table: `ID | Title | Status | Method | Outcome | Branch | PR | CI`. Include ready-but-not-started and on-hold items. With `--item`, replace the table by the full detail of that item.

### 4. ODD maps
Per map: path, north-star (one line), `delivered / in-progress / pending / dropped` counts, next eligible outcome (first `pending` whose dependencies are all `delivered`) or "none — refine needed", refine pending (from loop state) or `map refine recommended` (latest learnings), last learning (one line).

### 5. SDD specs
Table: `Spec | Plan | Phases/steps | Implementation (source) | Open questions | Self-decided`.

### 6. Recommended next actions
At most five, concrete and ordered: e.g., "Review and merge PR #12 (green CI)", "Answer the 2 questions on BL-004, then delete its state entry", "Run `/claude-sdd-tools:agentic-odd-map odd/bl-003/map.md --then none` (refine: no eligible outcome)", "Start the loop: `/loop /claude-sdd-tools:agentic-backlog`" when ready items exist and nothing is running.

## Result

End your response with this block (always, even when sections are empty), so an invoking agent can parse it:

```markdown
## Workflow Status Result
- **Status:** OK | PARTIAL | ERROR
- **Backlog:** source <file:<path> | github:<label> | none> — ready <n>, on-hold <n>, triaged <n>, in-progress <n>, in-review <n>, done <n>, blocked <n>, needs-input <n>, split <n>
- **Loop health:** consecutive blocked <n>/3 — working tree clean: yes | no — default branch in sync: yes | no | unknown
- **ODD:** <n> maps — outcomes delivered <n>/<total>, in-progress <n>, eligible next: <O-ids or "none">
- **SDD:** <n> specs — implemented (loop) <n>, (git) <n>, inferred-partial <n>, not started <n>
- **PRs:** open <n> (red CI <n>, green <n>, pending <n>), draft <n> — or "not checked: <reason>"
- **Needs human attention:** <n> — <one line each: kind — id — action, or "none">
- **Inconsistencies:** <n> — <one line each, or "none">
- **Next action:** <the first recommended next action>
- **Report file:** <path | "not written">
- **Error:** <only when ERROR or PARTIAL: what could not be read>
```

Status rules: `OK` — every source that exists was read; `PARTIAL` — some source could not be read (e.g., `gh` unavailable, unreadable state file) and the report says which; `ERROR` — not a git repository or nothing at all could be read.

## Rules

- **Read-only.** Never modify state, artifacts, branches, labels, issues or PRs; never fix an inconsistency — report it. The only write allowed is `--out`.
- Never run delivery skills or agents; this skill only reads and reports.
- Every claim cites its source (file path, state field, PR number, command). Mark inferred values as inferred.
- Never wait for human input: when something is missing, report it as missing and continue.
- Do not dump raw files or full reports into the response; quote only the lines needed.
