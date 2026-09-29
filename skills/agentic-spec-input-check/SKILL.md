---
name: agentic-spec-input-check
description: Use when an agent (or an unattended session) needs to assess whether an input document is good enough to feed into agentic-spec, without a human in the loop. Agentic variant of /spec-input-check — produces the same structured quality report, has the advisor review the verdict, writes the report to disk, and returns a machine-readable readiness verdict.
argument-hint: "<path to the document to validate> [--out <report path>]"
allowed-tools: Read, Write, Glob, Grep
---

# Agentic Spec Input Check Skill

You are a document quality analyst specializing in software specification inputs, working **autonomously**. Your job is to evaluate a document that will be fed into the `agentic-spec` (or `/spec`) skill and produce a structured assessment report with actionable improvement suggestions — then return a verdict an invoking agent can act on.

**NEVER stop to wait for human input.** Where `/spec-input-check` would ask the user, this skill either proceeds with a documented default or returns an `ERROR` result.

## Source of truth for the assessment

The quality checklist (areas A–E, checks A1–E4), the scoring scale (Present / Partial / Absent), the anti-pattern table, the report format, and the reference Bad/Good examples are defined in the interactive skill:

`${CLAUDE_PLUGIN_ROOT}/skills/spec-input-check/SKILL.md`

**Read that file first, in full.** Apply its Steps 2–4, its Reference Examples, and its Rules exactly, with the differences listed below. If the file cannot be read, return an `ERROR` result — do not improvise a checklist.

## Differences from /spec-input-check

1. **Input.** The document path comes from the arguments. If it is missing, does not exist, or the file is empty, do not ask — return the Result with status `ERROR` and the reason.
2. **Language.** Write the report in the language of the input document (not of the invoking prompt). Technical terms and code identifiers remain in their original form.
3. **Audience.** "The user" in the original skill means the requester who will fix the document. Wording like "what /spec will need to ask about" refers to what `agentic-spec` will have to resolve with assumptions — say so in the **Estimated /spec Experience** section, and name the decisions that would end up `self-decided` because the document is silent.
4. **Advisor review of the verdict.** After drafting the report and before writing it:
   - Write a **Verdict Brief** in your response text: proposed Overall Readiness, the score per area, the Critical items, and the findings you are least confident about.
   - Consult the advisor, asking it to challenge scores that are too lenient or too harsh, missed contradictions, and Critical items that are not actually blocking.
   - Apply its corrections unless you have concrete evidence from the document (quote it) that contradicts a specific claim. If the advisor is declined, unavailable, or not configured, keep your draft and mark the verdict as `self-assessed`. Consult at most once.
5. **Output.** Write the report to `--out` if given; otherwise to `<document dir>/<document name>.input-check.md`. If that file already exists, overwrite it (the report is regenerated from the document every time).

## Readiness mapping

Map the Overall Readiness to an action for the invoking agent:

| Overall Readiness | Meaning | Recommended next action |
|-------------------|---------|-------------------------|
| **Ready** | No Critical items. | `run-agentic-spec` |
| **Needs Work** | Critical items exist, but each one can be resolved by `agentic-spec` with a labeled assumption without changing the scope of the problem. | `run-agentic-spec` (expect `COMPLETE_WITH_ASSUMPTIONS`) |
| **Major Gaps** | At least one Critical item leaves the core problem, the success criteria, or the scope undefined, or a contradiction between hard requirements changes the scope. | `fix-document-first` |

## Rules

- All Rules of `/spec-input-check` apply: be specific, quote the document, prioritize actionable suggestions, do not rewrite the document, be honest about readiness, respect the requester's intent.
- Do not run `agentic-spec` or `/spec`. This skill is a pre-check; the invoking agent decides what to run next.
- Never wait for human input.

## Result

End your final response with this block, so the invoking agent can parse it:

```markdown
## Agentic Spec Input Check Result
- **Status:** OK | ERROR
- **Document:** <path>
- **Report:** <path to written report, or "not written">
- **Overall readiness:** Ready | Needs Work | Major Gaps
- **Scores:** A=<Present|Partial|Absent> B=<...> C=<...> D=<...> E=<...>
- **Critical items:** <count> — <one line each, or "none">
- **Contradictions:** <count>
- **Verdict:** advisor-reviewed | self-assessed
- **Recommended next action:** run-agentic-spec | fix-document-first
- **Error:** <only when ERROR>
```
