# claude-sdd-tools

A Claude Code plugin that implements two delivery workflows that share the same quality bar:

- **Spec-Driven Development (SDD)** — from problem definition through a complete specification, a full implementation plan, implementation, review, testing, and documentation. Best when a contract must be closed whole before use.
- **Outcome-Driven Development (ODD)** — the goal is framed once as an outcome map, then delivered as small vertical slices (outcomes), each planned just-in-time and closed completely (acceptance tests red → green, scoped review, docs) before the next one starts. Best when work can ship and be validated incrementally.

Both workflows share a [defensive programming baseline](references/defensive-programming.md), and an unattended [backlog loop](#backlog-loop) can triage each backlog item to SDD or ODD and deliver it as pull requests behind an independent quality gate.

## Overview

This plugin provides 5 interactive skills, 5 agentic (unattended) counterparts, an unattended backlog loop skill, and 8 agents. Both workflows ensure that every piece of code:

- comes from a well-defined specification (a full spec, or an outcome card);
- is implemented by following a concrete plan;
- is reviewed for correctness and security;
- is tested by exercising real code;
- is documented before delivery.

The SDD pipeline is shown below. The ODD workflow is described in [ODD Workflow](#odd-workflow-outcome-driven-development) and the loop in [Backlog Loop](#backlog-loop).

The toolkit is **polyglot**: skills and agents adapt to the project's language and tooling (Go, Java/Kotlin, TypeScript/JavaScript, Python, and others). The review checklist includes language-specific extensions for the bug classes idiomatic to each stack.

```
┌─────────────┐    ┌──────┐    ┌─────────────┐    ┌──────────────┐    ┌────────────┐    ┌────────────┐
│ spec-input  │───→│ spec │───→│ plan-agent  │───→│  implement   │───→│review-agent│───→│docs-agent  │
│   -check    │    │      │    │   (plan)    │    │(code + test) │    │  (review)  │    │   (docs)   │
│  (validate) │    │(spec)│    │             │    │              │    │            │    │            │
└─────────────┘    └──────┘    └─────────────┘    └──────────────┘    └────────────┘    └────────────┘
                                                       │
                                                       ▼
                                                 ┌────────────┐
                                                 │ test-agent │
                                                 │(per step)  │
                                                 └────────────┘
```

## Installation

Clone the repository and load it as a plugin:

```bash
git clone https://github.com/nachol/claude-sdd-tools.git
claude --plugin-dir /path/to/claude-sdd-tools
```

Or install it from a marketplace that lists this plugin:

```bash
claude plugin marketplace add <your-marketplace-source>
claude plugin install claude-sdd-tools@<your-marketplace>
```

## Components

### Skills

| Skill | Invocation | Purpose |
|-------|-----------|---------|
| [spec-input-check](#spec-input-check) | `/spec-input-check <path>` | Validate input documents before specification |
| [spec](#spec) | `/spec <idea>` | Transform ideas into implementable specifications |
| [implement](#implement) | `/implement <plan-folder>` | Execute implementation plans step by step |
| [agentic-spec-input-check](#agentic-skills) | Agents, or `/agentic-spec-input-check <path>` | Unattended input validation with a machine-readable verdict |
| [agentic-spec](#agentic-skills) | Agents, or `/agentic-spec <idea>` | Unattended specification — decisions reviewed by the advisor |
| [agentic-implement](#agentic-skills) | Agents, or `/agentic-implement <plan-folder>` | Unattended plan execution — blockers resolved with the advisor |
| [odd-map](#odd-map) | `/odd-map <idea \| doc \| map.md>` | Frame a goal and slice it into an ordered outcome backlog (or refine an existing map) |
| [odd-deliver](#odd-deliver) | `/odd-deliver <map.md> [O-id]` | Deliver one outcome end to end: card → red → implement → green → review → docs → close |
| [agentic-odd-map](#agentic-skills) | Agents, or `/agentic-odd-map <idea \| doc \| map.md> [--out <dir>] [--then none\|deliver-next\|deliver-all]` | Unattended outcome mapping (or refine) — decisions reviewed by the advisor |
| [agentic-odd-deliver](#agentic-skills) | Agents, or `/agentic-odd-deliver <map.md> [O-id \| --next \| --all]` | Unattended outcome delivery — gates resolved with the advisor |
| [agentic-backlog](#agentic-backlog) | `/loop /agentic-backlog [...]`, `/goal ...`, or once (`claude -p`) | Consume a backlog: triage SDD/ODD, deliver, quality-gate, open one PR per unit |

### Agents

| Agent | Triggered By | Purpose |
|-------|-------------|---------|
| [plan-agent](#plan-agent) | `/spec` (Phase 4), `agentic-spec` | Generate executable implementation plans |
| [implement-agent](#implement-agent) | `/implement` (per step), `/odd-deliver` (per card step), loop fix passes | Implement one plan or card step faithfully |
| [review-agent](#review-agent) | `/implement` (post-steps), `/odd-deliver` (per outcome) | Code review for correctness and security |
| [test-agent](#test-agent) | `/implement` (per step), `/odd-deliver` (red + green), loop fix passes | Create and validate tests; Acceptance-first mode for the red phase |
| [docs-agent](#docs-agent) | `/implement` (post-steps), `/odd-deliver` (per outcome) | Generate and update documentation |
| [outcome-agent](#outcome-agent) | `/odd-deliver` (per outcome) | Write the just-in-time outcome card for one outcome |
| [triage-agent](#triage-agent) | `agentic-backlog` (per item) | Route a backlog item to SDD, ODD, SPLIT or NEEDS_INPUT |
| [quality-gate-agent](#quality-gate-agent) | `agentic-backlog` (per unit) | Independent, read-only verification of a deliverable and its tests |

---

## Workflow in Detail

### Phase 1: Input Validation (Optional but Recommended)

#### spec-input-check

**What it does:** Evaluates a document that will be used as input for `/spec` and produces a structured quality assessment report. It identifies gaps, ambiguity, contradictions, and anti-patterns that would cause the specification process to stall or produce incomplete results.

**When to use it:** Before running `/spec`, when you have a product brief, requirements document, or any written description of what needs to be built.

**Invocation:**

```
/spec-input-check ./path/to/your-document.md
```

**Scope:**

- Evaluates 5 quality areas: Problem & Context, Functional Requirements, Non-Functional Requirements, Technical Context, and Document Structure
- Checks 22 individual quality criteria across those areas
- Detects 8 anti-patterns: vague qualifiers, mixed abstraction levels, implicit requirements, missing boundaries, aspirational language, undefined terms, contradictions, and open-ended lists
- Scores each area as Present, Partial, or Absent

**Output:** A structured report containing:

- Score summary table across all 5 areas
- Detailed findings per check with quoted evidence from the document
- Anti-patterns detected with "bad" quotes and "better" rewrites
- Contradictions detected (if any)
- Prioritized improvement suggestions (Critical / Recommended / Optional), each with a concrete example of what the improved text should look like
- Estimated `/spec` experience — what to expect if running the spec process with the document as-is

**Responsibilities:**

- Assesses and suggests — does NOT rewrite the document
- Does NOT run `/spec` — it is a pre-check only
- Every finding references specific text from the input document
- Respects the user's intent — a high-level product brief is valid input; the report notes what `/spec` will need to ask about rather than demanding a complete SRS

---

### Phase 2: Specification

#### spec

**What it does:** Takes a rough problem or solution idea and transforms it into a well-defined, implementable specification through rigorous collaborative analysis. Operates as a critical engineering peer — challenges assumptions, evaluates trade-offs, and demands evidence.

**When to use it:** When you have a problem to solve or a feature to build and need a complete, unambiguous specification before writing code.

**Invocation:**

```
/spec Build a rate limiter for our public API that supports per-tenant limits
```

**Scope:** Executes 4 mandatory phases in strict order:

**Phase 1 — Problem Definition:**
- Restates and confirms the problem
- Identifies and classifies every assumption (validated vs assumed)
- Scans for contradictions in the input
- Classifies every statement as: Functional Requirement, Non-Functional Requirement, Constraint, Pre-made Design Decision, or Deliverable
- Classifies constraints as Hard (non-negotiable) or Soft (negotiable)
- Extracts implicit constraints hidden inside requirements
- Defines measurable success criteria with verification methods
- Builds a glossary of ambiguous terms
- Establishes NFR thresholds (p99 latency, throughput, etc.)

**Phase 2 — Solution Exploration:**
- Identifies all key design decisions using a mandatory checklist: core architecture, public API shape, error model, configuration, behavioral contracts, edge cases, concurrency model, NFRs, internal mapping, non-goals, and documentation deliverables
- Evaluates the user's proposed solution (strengths, weaknesses, failure conditions)
- Proposes at least one alternative with concrete trade-off analysis
- Presents each decision one at a time, with options and trade-offs, and waits for the user to decide
- Produces a Decision Validation Table and a Requirements Traceability Table

**Phase 3 — Specification:**
- Produces a complete, self-contained specification document following RFC 2119 language precision (MUST/SHOULD/MAY)
- Applies NASA/INCOSE verifiability standards — every requirement has a concrete verification method
- Writes API contracts as actual code (types, signatures, doc comments), not prose
- Covers 9 sections: Problem Statement, Chosen Solution, Public API Contract, Behavioral Contract, NFRs, Internal Mapping, Explored and Discarded, Implementation Notes, and Spec Validation Checklist

**Phase 4 — Output Decision:**
- Offers three options: save as a document, generate an implementation plan (via `plan-agent`), or a custom output
- If the user chooses an implementation plan, launches `plan-agent` with the full spec and then offers to execute it via `/implement`

**Responsibilities:**

- Never makes unilateral decisions — every key decision requires explicit user confirmation
- Asks questions one at a time with choices to simplify responses
- Actively kills weak options rather than keeping them alive for balance
- Searches the web autonomously to validate claims, check library APIs, and verify defaults
- The output spec must be self-contained: another engineer or AI agent can produce a correct implementation plan from it without asking clarifying questions

---

### Phase 3: Planning

#### plan-agent

**What it does:** Transforms a specification into an executable, step-by-step implementation plan with precise code contracts (types, signatures, file paths) that can be mechanically followed by the `/implement` skill.

**When to use it:** Automatically invoked by `/spec` when the user chooses "Implementation Plan" in Phase 4. Can also be invoked directly by Claude when a spec is available.

**Scope:**

- Assesses implementation complexity (Simple vs Complex)
- For simple implementations: produces a single `00-overview.md` with all steps inline
- For complex implementations: produces `00-overview.md` (overview, execution order, dependency graph, conventions) plus separate phase files (`01-<name>.md` through `NN-<name>.md`)
- Supports sequential and parallel phases with explicit dependency graphs

**Step format — every step contains:**

| Section | Content |
|---------|---------|
| Goal | One sentence stating what the step achieves |
| File(s) | Exact file paths to create or modify |
| Types/Signatures | Full code block with struct definitions, method signatures, constants, doc comments — plus a [Defensive contract](#defensive-programming-baseline) (`Validates` / `Fails with` / `Invariants`) for every exposed symbol |
| Implementation Notes | Algorithmic details, edge cases, integration points, performance notes |
| Done-when | Mechanically verifiable completion criteria |

**Responsibilities:**

- Produces plans precise enough that an implementor fills in function bodies, not designs
- Never includes test specifications — `test-agent` handles tests at runtime
- Never includes documentation phases — `docs-agent` handles docs at runtime
- Every step must produce at least one file and have concrete completion criteria
- A step that exposes a symbol without its Defensive contract is incomplete; internal symbols that only receive validated values declare `Validates: none — internal` instead of re-validating
- If the spec is insufficient, lists specific gaps and asks for clarification before proceeding

---

### Phase 4: Implementation

#### implement

**What it does:** Executes an implementation plan generated by `plan-agent`, working through each step methodically with mandatory quality gates (testing after every step, review and documentation after all steps).

**When to use it:** After `plan-agent` has produced a plan, either automatically from `/spec` Phase 4 or manually.

**Invocation:**

```
/implement ./path/to/plan-folder
```

**Scope:** Executes the following pipeline per step:

```
Create Task → implement-agent → test-agent → Complete Task → [next step or post-pipeline]
```

After all steps complete, runs the post-steps pipeline:

```
review-agent → fix findings → docs-agent → Delivery Checklist → Final Summary
```

**Execution model:**

- Reads the full plan structure from `00-overview.md` and all phase files
- Tracks progress via tasks — can resume from a previous session
- For each step: delegates the implementation to `implement-agent`, verifies Done-when criteria, then invokes `test-agent`
- After all steps: invokes `review-agent` on all produced code, fixes any findings, then invokes `docs-agent`
- Runs the Delivery Checklist from the plan overview
- Presents a final summary with steps completed, findings fixed, tests created, and docs updated

**Responsibilities:**

- Never skips the per-step pipeline (implement → test → complete) or the post-steps pipeline (review → docs → completion)
- Never creates git commits — the user manages commits manually
- Never modifies files outside the current step's scope unless a review finding or test failure requires it
- If a step cannot be completed as specified, presents the blocker and asks the user how to proceed (fix plan, skip, or abort)
- If a review fix causes a test failure (or vice versa), stops after 2 iterations and asks for guidance — never enters an infinite fix loop

#### implement-agent

**What it does:** Implements ONE step or phase of an already-crystallized implementation plan, faithfully and without scope creep. Translates the step's exact types, signatures, files, and Done-when criteria into production-quality code.

**When to use it:** Automatically invoked by `/implement` for each plan step and by `/odd-deliver` for each outcome card step (outcome cards use the same Step Format). The backlog loop also uses it for code fix passes. Can also be invoked directly to implement a single, well-specified step.

**Scope:**

- Reads the plan step, overview conventions, and referenced spec sections in full before writing code
- Implements exactly what the step specifies — named files, exact types/signatures, implementation notes, Done-when
- May compile/build/lint to verify Done-when criteria
- Stops and reports a blocker when the plan is ambiguous, incomplete, or conflicts with the spec — never guesses

**Responsibilities:**

- Does NOT design, does NOT write tests (test-agent owns tests), does NOT commit
- No features, abstractions, or configuration beyond what the step calls for — smallest correct implementation
- Only touches files within the current step's scope; never implements later phases or refactors unrelated code
- Honors every convention in the plan overview (module path, encapsulation, error wrapping, naming) and the [defensive programming baseline](#defensive-programming-baseline); a contract that omits an obviously needed boundary validation is reported as a gap, not silently patched beyond the step's scope
- All code and comments in English; every exported symbol gets a doc comment

---

### Quality Gate: Code Review

#### review-agent

**What it does:** Performs a structured code review focused on correctness and security, modeled after real production bugs. Runs 6 parallel review passes and applies skeptical validation to eliminate false positives.

**When to use it:** Automatically invoked by `/implement` after all steps are complete (with the plan folder as the declared contract in `agentic-implement`), and by `/odd-deliver` for each outcome, scoped to that outcome's diff and checked against the card's Contract. Can also be used independently after writing or modifying code.

**Scope — 6 review passes executed in parallel:**

| Pass | Name | Focus |
|------|------|-------|
| 1 | The Consumer Test | Does it work for the most basic input? Return types match? Config fields actually used? String composition escaped? |
| 2 | The Symmetry Test | Similar functions handle same concerns the same way? Shared dependencies used consistently? Copy-paste artifacts? |
| 3 | The Robustness Test | Errors checked and propagated? Empty collections handled? Resources closed? Error wrapping preserves unwrapping? |
| 4 | The Concurrency Test | Package-level variables synchronized? Pointer arguments mutated safely? Locks don't deadlock via callbacks? |
| 5 | The Wiring Test | Middleware ordering correct? sync.Pool lifecycle valid? Defaults applied and overridable? Input normalization consistent? |
| 6 | The Security Test | Injection attacks? Sensitive data in logs? Hardcoded secrets? Unsafe deserialization? Input validation? Path traversal? SSRF? Internal types leaked? Weak crypto? |

**Language-specific extensions:** In addition to the 6 base passes, the checklist carries extension items per stack — JVM (JPA lazy loading/N+1, `@Transactional` proxy pitfalls, singleton bean state, coroutines, `BigDecimal`/time handling), Node/TypeScript (floating promises, event-loop blocking, async middleware error handling, type assertions masking runtime nulls), and Python (mutable default arguments, blocking calls in coroutines, bare `except`, late-binding closures). Each sub-agent receives the extension items for its pass that match the project's language(s). Go is covered by the base checklists.

**Defensive Programming Baseline Extension:** the Pass 3 sub-agent also receives DP2, DP3, DP4, DP5, DP7, DP8 and DP10, and the Pass 6 sub-agent receives DP1, DP6 and DP9, together with any Defensive contract declared in the plan step or outcome card. Where a DP rule overlaps an existing checklist item, the base item takes precedence. A declared validation, error or invariant that the code does not implement is a finding. There is no extra pass, and every finding still goes through Skeptical Validation.

**Severity levels:** CRITICAL (broken for normal use), HIGH (broken for realistic edge cases), MEDIUM (latent bug), LOW (minor inconsistency).

**Final Gate — Skeptical Validation:** After all 6 passes complete, every finding is re-examined with adversarial intent. Each finding must survive 4 questions:
1. Does the producer guarantee this can't happen?
2. Is this reachable in the actual call graph?
3. Am I confusing a type-level possibility with a runtime fact?
4. Can I construct a concrete, minimal reproduction?

Findings that fail this gate are removed entirely — not downgraded.

**Responsibilities:**

- Skips documentation files (README, CHANGELOG, etc.) and test code
- Every finding has a concrete trigger scenario — no theoretical concerns
- Does not suggest improvements, refactors, or style changes — correctness only
- All 6 passes always execute, even if critical bugs are found early
- Uses web search to verify runtime/library guarantees when uncertain

---

### Quality Gate: Testing

#### test-agent

**What it does:** Creates comprehensive tests for code changes, fixes failing tests, and improves existing test quality. Enforces strict constraints to ensure tests exercise real business logic with minimal mocking.

**When to use it:** Automatically invoked by `/implement` after each implementation step, and twice per outcome by `/odd-deliver` (red, then green). The backlog loop also uses it for test fix passes. Can also be used independently when tests need planning, fixing, or improvement.

**Scope — 3 operational workflows:**

| Workflow | When | Process |
|----------|------|---------|
| Test Planning | New code written | Analyze changes → identify coverage gaps → design tests exercising real code → plan edge cases |
| Test Fixing | Tests failing | Identify failures → analyze root cause → apply quality-preserving fixes → verify no regressions |
| Test Improvement | Existing tests need work | Assess quality → reduce mocks → improve coverage → optimize performance |
| Acceptance-first Mode | ODD red phase — only when the invoker explicitly requests "Acceptance-first mode" | Write acceptance tests from the card's criteria (path and name as specified) → no production code changes (only non-behavioral compilation stubs for contract signatures) → run them → classify each failure as right reason / wrong reason (fix) / unexpected pass → report |

**Universal quality constraints:**

- **Never modifies production code** unless a legitimate bug is found (demonstrated by a failing test)
- **Never simplifies tests** just to make them pass — maintains test value and intent
- **Minimizes mocks** to external world interactions only (APIs, file system, databases, network)
- **Replaces internal mocks** with real implementations whenever feasible
- Targets minimum 90% code coverage through real code exercise

**Responsibilities:**

- Tests must exercise real business logic, not test mock implementations
- Each test validates specific business behaviors with meaningful assertions
- Failed tests must reveal actual problems, not implementation details
- Continues until 100% test pass rate — never leaves test management partially done (except in Acceptance-first mode, where tests are expected to fail)

---

### Quality Gate: Documentation

#### docs-agent

**What it does:** Generates and updates project documentation — README, ARCHITECTURE, CONTRIBUTING, OpenAPI specs, PERSISTENCE docs, CHANGELOG entries, and code comments on exported symbols.

**When to use it:** Automatically invoked by `/implement` after review-agent completes, and by `/odd-deliver` once per outcome. Can also be used independently for initial documentation generation or to sync docs after code changes.

**Scope:**

| Document | Content |
|----------|---------|
| README.md | Project overview, installation, usage, API overview, configuration |
| ARCHITECTURE.md | System design, component relationships, data flows, design decisions |
| CONTRIBUTING.md | Dev setup, code style, testing requirements, PR process |
| OpenAPI spec | REST endpoint contracts (request/response schemas, status codes) |
| PERSISTENCE.md | Database schemas, DER diagrams, Redis structures, TTLs, entity descriptions |
| CHANGELOG | Entries under current version section — Added, Changed, Fixed, Removed |
| Code comments | Doc comments on all exported symbols (purpose, parameters, return values, behavior) |

**When updating after code changes (vs full generation):**

1. Runs `git diff` against the base branch to identify exactly what changed
2. Updates CHANGELOG under the current version section (never creates new version sections)
3. Updates README if any public API, endpoint, configuration, or behavior changed
4. Updates ARCHITECTURE.md if component relationships or design decisions changed
5. Updates OpenAPI spec if REST endpoints changed
6. Updates PERSISTENCE.md if persisted entities changed
7. Ensures every added/modified public symbol has a meaningful doc comment
8. Reports all files modified with a one-line summary per change

**Responsibilities:**

- Generates documentation in English regardless of request language
- Explains "why" rather than "what" in code comments
- Only creates files that add significant value
- Non-README documents go in the `docs/` folder
- Does not rewrite unaffected sections when updating

---

## ODD Workflow (Outcome-Driven Development)

SDD is waterfall by design: it plans everything up front and validates everything at the end. That preserves quality but makes the cycle long, and problems surface late. ODD keeps the same rigor but moves it to where it is cheap: the goal is framed **once**, and every outcome is planned **just-in-time** and closed **completely** before the next one starts.

```
┌──────────┐    ┌──────────────────────────── per outcome ───────────────────────────────────┐
│ odd-map  │───→│ outcome-agent → test-agent (red) → implement-agent → test-agent (green)     │
│ (framing,│    │      (card)      (acceptance)        (steps)           (unit + acceptance) │
│  runway, │    │   → review-agent (this outcome's diff) → docs-agent → close (update map)   │
│ slicing) │    └────────────────────────────────────────────┬───────────────────────────────┘
└──────────┘          ▲                                      │
     ▲                └──────────── next outcome ────────────┤
     └──────────────── refine (learnings change the map) ────┘
```

| SDD guarantee | Where it lives in ODD |
|---------------|-----------------------|
| Problem framing rigor (assumptions, contradictions, constraints, glossary, NFRs) | `odd-map` Phase 1, once per goal |
| Critical evaluation of design decisions | Architecture runway (expensive-to-reverse decisions, up front) + just-in-time decisions in each card |
| Verifiable success criteria | North-star outcome in the map; executable acceptance criteria per card, written and run red before code |
| Exact contracts for implementors | Card Contract (with its Defensive contract) + Steps in the plan-agent Step Format |
| Testing | `test-agent` red (Acceptance-first mode) + green per outcome |
| Review for correctness and security | `review-agent` per outcome, scoped to that outcome's diff (instead of once at the end) |
| Docs in sync | `docs-agent` per outcome |
| Defensive programming | Shared baseline enforced in contract, acceptance, implementation and review |

### When to use SDD vs ODD

- **SDD** — a contract that must be closed whole before use (library public API, protocol, wire/persisted format, schema migration), partial delivery has no value or is risky (security controls, money movement, irreversible data operations), or the input already is a closed spec.
- **ODD** — features that can ship as two or more independently observable slices, uncertainty about later parts, or early feedback is valuable.

The [triage-agent](#triage-agent) applies these criteria mechanically when the backlog loop picks an item. When neither clearly fits, it chooses ODD unless an SDD trigger fired.

### Artifacts

```
odd/<slug>/
  map.md                 # living outcome map
  outcomes/
    O-01-<name>.md       # outcome card, written just-in-time; Delivery Record appended at close
    O-02-<name>.md
```

**`map.md` sections:**

| Section | Content |
|---------|---------|
| 1. Goal | Problem, **north-star outcome** with measurable success criteria (`SC-n`), assumptions (`A-n`) |
| 2. Framing | Constraints (hard/soft), NFRs with thresholds (`NFR-n`), glossary, non-goals, conventions (including the defensive programming baseline) — written once, referenced by cards, never repeated in them |
| 3. Architecture Runway | Only the decisions that are expensive to reverse (`R-n`), with discarded alternatives, the deferred areas, and a Decision Validation Table |
| 4. Outcome Backlog | `ID \| Outcome (observable behavior) \| Acceptance (one line) \| Depends on \| Status \| Risk`, plus a Requirements Traceability table. Status is exactly `pending`, `in-progress`, `delivered` or `dropped` |
| 5. Learnings Log | `Date \| Outcome \| Learning \| Map change`, one row per delivered outcome or refinement |
| 6. Decision Log / 7. Open Questions for the Requester | Agentic maps only (created on demand by `agentic-odd-deliver` when it works on a map written interactively) |

**ID rules:** outcome IDs (`O-NN`) are never reused and rows are never deleted. A split outcome keeps its row with Status `dropped` and the note "(split into O-08, O-09)" in its Outcome cell; the sub-outcomes take the next free IDs, and every row that depended on the original is rewritten to depend on the sub-outcomes.

#### odd-map

**What it does:** Turns a goal into an outcome map. It frames the goal with the rigor of `/spec` Phase 1, decides only the architecture runway, and slices the goal into an ordered backlog of vertical outcomes. Given an existing `map.md`, it runs in **refine** mode instead.

**When to use it:** At the start of an ODD effort, and again whenever delivered outcomes produce learnings that change the plan (the map, a Gate 2, or `/odd-deliver` tells you when a refine is needed).

**Invocation:**

```
/odd-map Per-tenant rate limiting for the public API
/odd-map ./product-brief.md
/odd-map ./odd/per-tenant-rate-limiting/map.md     # refine mode
```

**Scope — new map mode** (4 mandatory phases, each ending in a checkpoint question):

- **Phase 1 — Framing:** restatement, assumptions (validated vs assumed), contradiction scan, input classification, hard/soft and implicit constraints, a measurable north-star outcome with verification methods, glossary, NFR thresholds, non-goals
- **Phase 2 — Architecture runway:** walks the `/spec` Decision Area Checklist and classifies each area as **Runway** (expensive to reverse: core architecture, API style, error model strategy, persistence, concurrency model, security model), **Deferred** (decided later in an outcome card) or **N/A**. Only runway decisions are explored, with at least two options and trade-offs each, and decided by the user
- **Phase 3 — Slicing:** every outcome is a vertical slice of consumer-observable behavior, independently verifiable by acceptance tests, deliverable in about a day or less, with at least one observable failure path and explicit dependencies. **O-01 is always a walking skeleton** — the thinnest end-to-end path that exercises every runway decision. The rest is ordered by risk first, then value. A traceability table maps every requirement to an outcome or a non-goal
- **Phase 4 — Output:** writes `map.md` (default `./odd/<slug>/`, never overwriting an existing map) and offers to deliver O-01 with `/odd-deliver`

**Scope — refine mode:** reads the map, every card and its Delivery Record; checks whether any learning invalidates a runway decision, the north-star, an NFR or a non-goal; proposes a change set (reprioritize, split, drop, add, reshape) with a rationale per change; re-runs the coverage check; applies only the approved changes and logs the refinement.

**Responsibilities:**

- Never makes unilateral decisions — questions one at a time, with choices
- Never plans beyond the map: no Given/When/Then criteria, contracts or steps (that is the outcome card's job)
- Never rewrites the history of delivered outcomes

#### odd-deliver

**What it does:** Delivers **one** outcome of a map end to end and closes it completely.

**When to use it:** After `odd-map`, once per outcome.

**Invocation:**

```
/odd-deliver ./odd/per-tenant-rate-limiting/map.md          # next eligible outcome
/odd-deliver ./odd/per-tenant-rate-limiting/map.md O-03     # a specific outcome
```

**Outcome selection:** an `in-progress` outcome is resumed first; otherwise the first `pending` outcome (in backlog order) whose dependencies are all `delivered`. If nothing is eligible, it reports the backlog state and suggests `/odd-map <map.md>` (refine).

**Scope:**

```
Card ─▶ Gate 1 ─▶ Red ─▶ Implement (per step) ─▶ Green ─▶ Scoped Review ─▶ Docs ─▶ Close ─▶ Gate 2
```

1. **Card** — `outcome-agent` writes the card. A split proposal, a runway conflict, or a decision with real alternatives is brought to you one at a time. A card self-check runs (1–7 criteria with at least one failure path, 1–5 steps, a Defensive contract per exposed symbol, no runway contradictions). **Gate 1:** you approve the card
2. **Red** — `test-agent` in **Acceptance-first mode** writes the acceptance tests; the skill runs them and verifies each one fails *for the right reason* (missing behavior, not compile or setup errors). Demo-verified criteria must not yet produce their expected output
3. **Implement** — `implement-agent` per card step, with the map's Conventions and the defensive baseline as mandatory conventions; Done-when verified after each step
4. **Green** — `test-agent` (normal mode) adds unit/integration tests (covering each declared failure mode) and drives the full suite and the acceptance tests green
5. **Scoped Review** — `review-agent` on the files changed since the outcome's baseline commit only, checking the code against the card's Contract and Defensive contract; every finding is fixed
6. **Docs** — `docs-agent` once for this outcome
7. **Close** — acceptance tests and full suite re-run, the card's **Delivery Record** written (red and green evidence per criterion, files, tests, findings, deviations, learnings), the outcome marked `delivered` and a Learnings Log row appended
8. **Gate 2** — if the learnings affect the backlog, the concrete changes are proposed and you approve them. Runway or north-star changes are left to `/odd-map` refine

**Responsibilities:**

- One outcome per run; never commits — it ends with a suggested commit message (one commit per outcome)
- Never implements before Gate 1 and before the acceptance tests were seen failing for the right reason
- Never changes the runway or the north-star
- Step failures offer fix the card / split / abort; circular review ↔ test fixes stop after 2 iterations and ask

#### outcome-agent

**What it does:** Writes exactly **one** outcome card — the just-in-time spec and plan for one vertical slice. It replaces plan-agent in the ODD flow.

**When to use it:** Invoked by `odd-deliver` / `agentic-odd-deliver` for each outcome. Re-invoked with a **change request** (the card path plus the exact edits) to apply decisions, gate feedback or deviations without rewriting the rest of the card.

**Card sections:**

| Section | Content |
|---------|---------|
| 1. Outcome | One sentence of observable behavior |
| 2. Acceptance Criteria | 1–7 Given/When/Then criteria, each verified by a named acceptance test (`path › name`) or, only when automation is impossible, a demo with an exact command and expected output. At least one covers a failure path |
| 3. Scope | In / out, and the runway decisions, NFRs and conventions that apply (by reference to the map) |
| 4. Just-in-time Decisions | Only the decision areas this outcome touches; a decision with real alternatives is marked `PENDING` for the invoking skill to resolve |
| 5. Contract | Types and signatures in code, plus a Defensive contract block per exposed symbol |
| 6. Steps | 1–5 steps in the plan-agent Step Format (Goal, File(s), Types/Signatures, Implementation Notes, Done-when) |
| 7. Delivery Record | Placeholder, filled at Close |

**Result statuses:** `CARD_WRITTEN` (possibly with surfaced `PENDING` decisions), `SPLIT_PROPOSED` (the outcome needs more than 5 steps or 7 criteria — no card is written, sub-outcomes are proposed instead), `RUNWAY_CONFLICT` (the outcome cannot be delivered within the runway), `BLOCKED`.

**Responsibilities:**

- Never implements, writes tests, or plans other outcomes
- Never contradicts runway decisions — it reports the conflict instead
- Never chooses silently between real alternatives

---

## Defensive Programming Baseline

[`references/defensive-programming.md`](references/defensive-programming.md) is the single, polyglot source of truth shared by SDD and ODD. Agents point to it; they do not copy it. Each rule comes with its rationale, what compliance looks like, a bad/good example and notes for Go, JVM, Node/TypeScript and Python.

| Rule | Summary | Reviewed in |
|------|---------|-------------|
| DP1 | Validate at trust boundaries (presence, type, range, length, format) | Pass 6 |
| DP2 | Fail fast and loud on programmer errors (violated preconditions/invariants) | Pass 3 |
| DP3 | Explicit error model: declared, distinguishable errors; causes preserved; no leaked dependency types | Pass 3 |
| DP4 | No swallowed errors | Pass 3 |
| DP5 | Bounded everything: timeouts, bounded retries, size and growth limits | Pass 3 |
| DP6 | Safe defaults; unknown values rejected | Pass 6 |
| DP7 | Resource safety: every acquired resource released on every path | Pass 3 |
| DP8 | Immutability and ownership at boundaries | Pass 3 |
| DP9 | No secrets or PII in errors and logs | Pass 6 |
| DP10 | Assertions are not validation | Pass 3 |

**Defensive contract.** Every exposed symbol in a plan step or outcome card declares:

```text
<SymbolName>
  Validates:  <each input → rule (presence, type, range, length, format)>; "none — internal, inputs validated by <caller>" when applicable
  Fails with: <each failure mode → declared error (type/sentinel/exception class)>; "never fails" only if true
  Invariants: <state/properties guaranteed before and after the call>; "none" when stateless
```

The reference also defines **anti-paranoia** rules: components that only receive already-validated values do not re-validate them.

**Enforcement points:**

- **Contract** — `plan-agent` (Types/Signatures) and `outcome-agent` (card Contract) declare the Defensive contract; a step that exposes a symbol without one is incomplete
- **Acceptance (ODD)** — every outcome card has at least one failure-path acceptance criterion, written and run red first
- **Implementation** — `implement-agent` honors the baseline as a convention and reports a contract that omits an obviously needed boundary validation instead of silently expanding scope
- **Review** — `review-agent` passes the DP items to its Robustness (Pass 3) and Security (Pass 6) sub-agents, together with the declared Defensive contracts; `quality-gate-agent` checks the declared contract is implemented at the changed boundaries

---

## Backlog Loop

The backlog loop consumes a backlog unattended. It routes each item to SDD or ODD, delivers it with the agentic skills, validates the result with an independent quality gate, and hands it over as small pull requests.

```
            ┌──────────── one tick = one unit of work ─────────────┐
backlog ──→ │ triage-agent ─→ SDD: agentic-spec --then plan         │──→ quality gate ──→ PR ──→ CI ──→ merge
 (file or   │   (route)          + agentic-implement               │   (Result block,              (only with
  GitHub)   │               ─→ ODD: agentic-odd-map (once)          │    fresh build/tests,          --auto-merge)
            │                   + agentic-odd-deliver --next       │    quality-gate-agent)
            └──────────────────────────────────────────────────────┘
```

### triage-agent

**What it does:** Routes one backlog item to a delivery method. It never specifies, plans or implements, and it is read-only.

**When to use it:** Invoked by `agentic-backlog` for each new item. Can also be invoked directly to decide how to approach a piece of work.

**Scope — criteria evaluated strictly in this order (the first that fires decides):**

1. **NEEDS_INPUT** — neither the problem nor one verifiable success criterion can be derived, even as a labeled assumption (for documents it may apply the `spec-input-check` checklist; Major Gaps ⇒ NEEDS_INPUT)
2. **SPLIT** — the item bundles independent goals that share no contract
3. **SDD** — any trigger fires: a closed contract (public API, protocol, wire/persisted format or schema, migration); partial delivery is worthless or risky; a closed spec is already provided
4. **ODD** — at least two independently observable slices, real uncertainty about later parts, or early feedback is valuable
5. **Tie-break** — ODD, unless an SDD trigger fired

**Output:**

```markdown
## Triage Result
- **Verdict:** SDD | ODD | SPLIT | NEEDS_INPUT
- **Confidence:** high | medium | low
- **Size:** S | M | L
- **Reasons:** <criteria that fired, quoting the item>
- **Split proposal:** <SPLIT only: child items, each with title + expected outcome>
- **Missing input:** <NEEDS_INPUT only: concrete questions>
```

**Responsibilities:**

- Applies the criteria mechanically, so two runs on the same item reach the same verdict; when in doubt it lowers its confidence instead of bending the criteria
- Every reason quotes the item or cites a file
- The loop consults the advisor whenever confidence is not `high`

### quality-gate-agent

**What it does:** The last independent check before code reaches a pull request. It verifies a committed unit against its governing artifacts using only evidence it gathers itself — it never sees the delivering skill's Result block or conclusions.

**When to use it:** Invoked by `agentic-backlog` as step 3 of its quality gate. Can also be invoked directly to audit a branch against a spec or an outcome card.

**Inputs:** the base and head refs (it diffs `<base>...<head>` itself; an empty diff is a FAIL), the governing artifacts (SDD: spec + plan folder; ODD: map + outcome card), and the defensive baseline.

**Scope — 4 checks, always all of them:**

| # | Check | What it verifies |
|---|-------|------------------|
| 1 | Traceability | Every success/acceptance criterion maps to a test that actually exercises it (file, name, assertion cited); demo criteria are run read-only and their output compared; every changed file is justified by the scope (lockfiles, generated code, docs and the unit's own governing artifacts excepted) |
| 2 | Test quality | Tests go through public entry points; no empty or tautological assertions; mocks only at external boundaries; failure paths covered; no skipped or disabled tests introduced |
| 3 | Defensive baseline | The declared Defensive contract is implemented at the changed code; baseline rules are honored at the changed boundaries |
| 4 | Contract fidelity | Exposed types, signatures, errors and config match the spec/card contract; no extra exposed symbols |

**Output:** a `## Quality Gate Result` block with `Overall: PASS | FAIL`, a table with one row per check or problem (evidence and fix hint), and a list of **failures for the fix pass**, each tagged `code` (for implement-agent) or `test` (for test-agent).

**Responsibilities:**

- Read-only — never edits code, tests or documents
- A FAIL needs file:line evidence and, for behavior, a concrete triggering input; an unverifiable check is a FAIL, never a default PASS

### agentic-backlog

**What it does:** Runs one **tick** of the loop: checks the preconditions, performs **at most one unit of work** (a whole SDD item, one ODD outcome, a map refine, a merge or a CI fix), persists state, returns to the default branch, schedules the next wakeup, and ends.

**When to use it:** Under `/loop` with no interval, so it paces itself; under `/goal` for a bounded batch (one tick per turn, ending on a Result `Status`); or once — interactively or with `claude -p` from a scheduler — to run a single tick. See [Use Cases → Automatic orchestration](#4-automatic-orchestration-with-loop-or-goal).

**Invocation:**

```
/loop /agentic-backlog                                      # backlog.md at repo root
/loop /agentic-backlog --source file:docs/backlog.md
/loop /agentic-backlog --source github:agent-ready          # GitHub Issues with a label
/loop /agentic-backlog --auto-merge --max-blocked 3
/goal Run /agentic-backlog once per turn until a tick reports Status IDLE, WAITING, STOPPED or ERROR, or stop after 30 turns
```

| Argument | Default | Meaning |
|----------|---------|---------|
| `--source file:<path>` \| `github:<label>` | `file:backlog.md` | Where items come from (`github:` defaults to the label `agent-ready`) |
| `--auto-merge` | off | Your explicit consent to merge a unit's PR when, and only when, its CI is green and its quality gate passed |
| `--max-blocked N` | `3` | Circuit breaker: stop after N consecutive units end `blocked` |

**Tick algorithm — the first applicable step is the tick's unit:**

1. **Service open PRs.** Detects PRs a human merged (bookkeeping, not a unit). With `--auto-merge` and green CI, merges using an explicit method allowed by the repository (`--merge`, else `--squash`, else `--rebase`). After any merge the default branch is fast-forwarded before the map is read. A red CI triggers a fix on the item's branch, re-gated before it is pushed to the same PR. Without `--auto-merge`, a green PR waits for a human
2. **Resume unblocked items** (Step 1b), according to the item's `block_kind`:
   - `code-test` → a fix pass
   - `refine` → a map refine, then the next outcome
   - `artifact` → re-runs the spec/map/card skill that blocked, with your `guidance` as extra input

   It never re-triages
3. **Continue an ODD item** whose previous outcome PR is merged (PRs are never stacked): applies a pending map refine first, then `agentic-odd-deliver <map> --next`. If no outcome is eligible, a map-only refine PR is opened, or the item goes to `needs-input`
4. **Start the next ready item:** `triage-agent`, then by verdict:
   - **SDD** → `agentic-spec --then plan`, then `agentic-implement` invoked directly. One PR per item
   - **ODD** → `agentic-odd-map --then none`, then `agentic-odd-deliver --next` invoked directly. Map plus walking skeleton are one unit, and there is one PR per outcome
   - **SPLIT** → child items `<parent>-1`, `<parent>-2`, …
   - **NEEDS_INPUT** → the questions are written to the report
5. **Nothing eligible:** waits for open PRs, or stops when nothing is left

**Branches:**

| Branch | Used for |
|--------|----------|
| `agent/<id>` | An SDD item |
| `agent/<id>-next-<n>` | ODD working branch, renamed before any push to `agent/<id>-O-NN` once the delivered outcome is known from the Result |
| `agent/<id>-refine-<n>` | A map-only refine PR |

`<n>` is the next integer free both locally and on the remote, so retried or refined units never collide.

**Quality gate (before any push or PR):**

1. **Result block validation** — rejects `BLOCKED`/`PARTIAL`, failing tests, red acceptance or an unverified red phase (ODD), open review findings, and completion claims without evidence. Assumptions, deviations and self-decided escalations are copied to the PR body under *Needs human attention*
2. **Fresh build + lint + full test suite** from the repository root, using the project's own commands. If they pass, the unit is **committed locally**
3. **`quality-gate-agent`** on the committed diff, without the delivery skill's conclusions
4. **PR CI** — after the branch is pushed and the PR opened, CI is the final gate, serviced in later ticks

The PR body lists the item, method, criteria covered (each with its verifying test), the delivery summary, the gate results and anything that needs human attention.

**Failure handling:**

- **One retry, as a fix pass:** `implement-agent` for code failures and `test-agent` for test failures, then a full re-gate. A retry never re-runs spec, plan or map creation. Blocks caused by the spec, plan, map or card itself are not retried
- **Second failure → `blocked`:** the advisor is consulted first; then the work in progress is committed (`wip(<id>)`), pushed as a **draft PR** with the failure reports, and the block kind is recorded
- **Circuit breaker:** after `--max-blocked` consecutive blocked units the loop stops. Any unit that reaches review resets the counter

**Pacing (`ScheduleWakeup`):**

| Situation | Next wakeup |
|-----------|-------------|
| A unit was processed | 60–120 s |
| Nothing eligible, PRs await CI or a human merge | ≥ 1200 s |
| Nothing eligible and nothing in review | stop |
| Circuit breaker, failed precondition, unreachable source or unreadable state | stop |

Every tick ends with an `## Agentic Backlog Tick Result` block. Fields:

- `Status` — `UNIT_DONE`, `WAITING`, `IDLE`, `STOPPED` or `ERROR`
- `Unit processed`, `Item`, `Method`, `Verdict` (with confidence)
- `Gate result`, `Unit outcome`, `PR`
- Backlog counts per status
- `Consecutive blocked`, `Next wakeup`, `Stop reason`

**Responsibilities:**

- One unit of work per tick; never waits for human input
- Never trusts a delivery skill's self-assessment
- Never writes the human-authored backlog file
- Never stashes, discards or commits changes it did not make; a dirty working tree stops the loop
- Every tick starts and ends on the default branch

### Backlog sources and state

**File source (default).** `backlog.md` is **human-authored only** — the loop never writes it. Each item is a level-2 section with a fenced `yaml` block (`id`, `title`, optional `links`, optional `status: ready | hold`), followed by the description; write the problem and at least one observable success criterion. Items are taken top to bottom. See [`templates/backlog.md`](templates/backlog.md).

Loop state lives in `.agentic-backlog/state.json` and reports in `.agentic-backlog/reports/<id>.md`. The directory is excluded through `.git/info/exclude` (never `.gitignore`), so the loop never dirties a tracked file. If the state is lost, it is reconstructed from the `agent/<id>*` branches and their PRs.

| `state.json` field (per item) | Meaning |
|-------------------------------|---------|
| `status` | `ready \| triaged \| in-progress \| in-review \| done \| blocked \| needs-input \| split` |
| `method`, `artifacts`, `outcome` | SDD/ODD, spec + plan or map path, current ODD outcome |
| `branch`, `pr`, `attempts`, `report` | Working branch, PR URL, retry count, report path |
| `triage` | The Triage Result block |
| `refine_pending` | ODD: refine recommendations to apply before the next outcome |
| `block_kind` | Set when blocked: `code-test`, `refine` or `artifact` |
| `guidance` | **Written by you** when unblocking: free text passed to the resumed unit |

**GitHub source.** Items are open issues with the source label, oldest first. Status is a single `agent:*` label:

- `agent:triaged`, `agent:in-progress`, `agent:in-review`, `agent:done`
- `agent:blocked`, `agent:needs-input`, `agent:split`, `agent:unblocked`

Other state lives in a hidden JSON block in the latest loop comment; triage results, gate reports and PR links are posted as comments. PRs say `Refs #<issue>`, and the issue is closed when the item is done. SPLIT creates child issues.

### Unblocking and resuming items

| Situation | File source | GitHub source |
|-----------|-------------|---------------|
| A `blocked` item | Set its `state.json` `status` to `ready` (keep its other fields) and optionally write `guidance`. The loop resumes it on its existing branch according to `block_kind` — never re-triaging it | Replace `agent:blocked` with `agent:unblocked`; add guidance as a comment |
| ODD item in `needs-input` (no eligible outcome) | Fix the map or write `guidance`, then set `status` to `ready`. The loop resumes it as an ODD item between outcomes, refining the map first | Replace `agent:needs-input` with `agent:triaged`; guidance in a comment |
| Triage `NEEDS_INPUT` (no map yet) | Update the item in `backlog.md`, then delete its `state.json` entry; it is re-triaged | Edit the issue and remove its `agent:*` label |
| Restart an item from scratch | Delete its `agent/<id>*` branches (local and remote) **and** its `state.json` entry | Delete the branches and remove the `agent:*` label |

> ⚠️ **Git policy exception.** Every other skill in this plugin never commits. `agentic-backlog` is the only one that does:
> - One branch per unit, and a local commit once the Result block and a fresh build/test run pass.
> - The branch is pushed and the PR opened only after `quality-gate-agent` passes. The exceptions are the `wip` draft PR of a blocked unit and a map-only refine PR.
> - It never pushes to the default branch, never force-pushes, and never rewrites pushed history.
> - It **merges only when you pass `--auto-merge` and CI is green**. Without `--auto-merge`, every PR waits for a human.
> - It never stashes, discards or commits changes it did not make.

### Requirements for the loop

- A git repository with a resolvable default branch and a **clean working tree** (ignoring `.agentic-backlog/`).
- A remote and an **authenticated `gh`** to push, open PRs and merge. Without them, commits still happen on local branches and units stop at `in-review`. With the GitHub source, an unauthenticated `gh` ends the tick with `ERROR`.
- **CI on pull requests** — it is the final gate and the condition for `--auto-merge`.
- The [advisor](#requirements) configured, for low-confidence triage and before blocking a unit.

---

## Agentic Skills

`spec`, `spec-input-check`, `odd-map` and `odd-deliver` are **user-invoked only** (`disable-model-invocation: true`): they depend on a human answering questions, so agents cannot trigger them. The `agentic-*` skills run the same process **unattended**, so an agent — a subagent, a workflow, or a headless `claude -p` session — can invoke them directly through the Skill tool. Each agentic variant reads its interactive skill as the single source of truth and applies only the documented differences.

The difference is **who answers the questions**. Where the interactive skill asks the user, the agentic skill writes a *Decision Brief* (or *Escalation Brief*) into the transcript — the question, the evidence, the options with trade-offs, and its recommendation — and consults the [advisor](https://code.claude.com/docs/en/advisor), a stronger reviewer model that reads the full transcript. Every decision is recorded in a log with its source:

| Source | Meaning |
|--------|---------|
| `from-input` | Stated explicitly in the input and not challenged by the analysis |
| `from-evidence` | Settled by the codebase, docs, or web research, with no real alternative |
| `advisor-reviewed` | Resolved with an advisor response |
| `self-decided` | The advisor was unavailable, declined, or not configured — the skill's own recommendation was used |

The skills **never block on the advisor**. If it is not configured or not reachable, they keep going with their own recommendations, mark those decisions `self-decided`, and report them so a human can review them afterwards.

| Interactive | Agentic | What changes |
|-------------|---------|--------------|
| `spec` | `agentic-spec` | Phase questions and gates are resolved by batched Decision Briefs reviewed by the advisor (one per phase gate, one per key design decision, one final spec review). Unknowns only the requester could answer become labeled assumptions listed under *Open Questions for the Requester*. Output location and follow-up (`--out`, `--then none\|plan\|implement`) come from arguments instead of questions. The spec includes the Decision Log as an appendix. |
| `implement` | `agentic-implement` | Starts without asking. Step failures, agent failures, and circular fixes are escalated to the advisor instead of the user, with a conservative default when the advisor is unavailable. Deviations from the plan are recorded. Never commits. |
| `odd-map` | `agentic-odd-map` | One Decision Brief for the framing gate, one per runway decision with real alternatives, one for the slicing gate (refine mode: one for the change set). Requester-only unknowns become conservative labeled assumptions under *Open Questions for the Requester*, and the map gains a *Decision Log*. `--out <dir>` (default `./odd/<slug>/`, never overwriting an existing map) and `--then none\|deliver-next\|deliver-all` (default `deliver-next`) chain into `agentic-odd-deliver`. On a block it writes `map.partial.md`. Never commits. |
| `odd-deliver` | `agentic-odd-deliver` | Selector `O-id \| --next \| --all` (default `--next`). Card decisions, Gate 1 (Card Brief), splits, runway conflicts, red-phase anomalies and Gate 2 (Backlog Change Brief) go through the advisor. Runway or north-star changes are never applied — they are reported in `Map refine recommended`. Blockers and circular fixes follow `agentic-implement`. `--all` delivers outcomes in order, each fully closed, and stops at the first `BLOCKED` or `PARTIAL` outcome. Never commits. |
| `spec-input-check` | `agentic-spec-input-check` | Reads the checklist from `spec-input-check` (single source of truth), has the advisor review the verdict, writes the report next to the document, and returns a recommended next action (`run-agentic-spec` or `fix-document-first`). |

Each agentic skill ends its response with a **Result block** that the invoking agent can parse:

| Skill | Result block | Statuses | Key fields |
|-------|--------------|----------|------------|
| `agentic-spec-input-check` | `## Agentic Spec Input Check Result` | `OK \| ERROR` | Overall readiness, scores, critical items, recommended next action |
| `agentic-spec` | `## Agentic Spec Result` | `COMPLETE \| COMPLETE_WITH_ASSUMPTIONS \| BLOCKED` | Spec and plan paths, decisions, open questions, implementation status |
| `agentic-implement` | `## Agentic Implement Result` | `COMPLETE \| COMPLETE_WITH_DEVIATIONS \| PARTIAL \| BLOCKED` | Steps, tests (`all passing`), review findings, delivery checklist, deviations, files changed |
| `agentic-odd-map` | `## Agentic ODD Map Result` | `COMPLETE \| COMPLETE_WITH_ASSUMPTIONS \| BLOCKED` | Mode (new/refine), map path, outcome counts and next eligible outcome, decisions, open questions, chained delivery status |
| `agentic-odd-deliver` | `## Agentic ODD Deliver Result` | `COMPLETE \| COMPLETE_WITH_DEVIATIONS \| PARTIAL \| BLOCKED` | Every field mandatory and evidence-bearing: outcomes delivered/remaining, acceptance green (with command and counts), red phase verified, tests, review findings, deviations, backlog changes, escalations, known issues, `Map refine recommended`, files changed, suggested commit message |
| `agentic-backlog` | `## Agentic Backlog Tick Result` | `UNIT_DONE \| WAITING \| IDLE \| STOPPED \| ERROR` | See [agentic-backlog](#agentic-backlog) |

`agentic-backlog` has no interactive counterpart: it orchestrates the agentic skills above (see [Backlog Loop](#backlog-loop)).

### Requirements

- **Advisor configured** for advisor-reviewed decisions: run `/advisor <model>`, set `advisorModel` in settings, or start with `claude --advisor <model>`. Subagents inherit the configured advisor.
- The advisor is a server-side tool of the Anthropic API. Through an LLM gateway (`ANTHROPIC_BASE_URL`) it works only if the gateway forwards it; otherwise every decision is `self-decided`.

### Example

```bash
# Headless: check the brief, then spec + plan + implement it without a human in the loop
claude -p --advisor opus "Use the claude-sdd-tools:agentic-spec-input-check skill on ./product-brief.md. \
If it recommends run-agentic-spec, use claude-sdd-tools:agentic-spec with ./product-brief.md --then implement."

# Headless: map a goal and deliver the walking skeleton (O-01)
claude -p --advisor opus "Use the claude-sdd-tools:agentic-odd-map skill with ./product-brief.md --then deliver-next."

# Headless: deliver every remaining eligible outcome of an existing map
claude -p --advisor opus "Use the claude-sdd-tools:agentic-odd-deliver skill with ./odd/per-tenant-rate-limiting/map.md --all."
```

---

## Use Cases

The plugin can be driven at four levels of autonomy. Pick the lowest one that fits: every level keeps the same quality gates, and higher levels only remove the human from more decisions.

| Level | Who decides | Commits | Typical use |
|-------|-------------|---------|-------------|
| [1. Manual (interactive)](#1-manual-interactive) | You, one question at a time | You | New or ambiguous work, design you want to own |
| [2. Agentic (unattended skill)](#2-agentic-unattended-skills) | The skill + the advisor | You | Well-understood work, overnight runs, headless sessions |
| [3. Orchestrated by your agents or skills](#3-orchestration-from-your-own-agents-or-skills) | Your orchestrator + the advisor | Your orchestrator | Custom pipelines, CI jobs, workflows |
| [4. Automatic with `/loop` or `/goal`](#4-automatic-orchestration-with-loop-or-goal) | `agentic-backlog` + triage + quality gate | `agentic-backlog` (branch + PR) | Draining a backlog unattended |

> **Skill names.** Plugin skills are namespaced: in the `/` menu they appear as `/claude-sdd-tools:<skill>` (for example `/claude-sdd-tools:agentic-backlog`). The examples below use the short form where it is unambiguous in your setup; use the namespaced form if it is not. The interactive skills (`spec`, `spec-input-check`, `implement`, `odd-map`, `odd-deliver`) can only be started by you, never by an agent, `/loop` or `/goal`.

### 1. Manual (interactive)

#### SDD — one closed contract, specified whole

```
# 1. (Optional) Validate your input document
/spec-input-check ./product-brief.md

# 2. Run the specification process
/spec Build a rate limiter for our public API with per-tenant limits
# ... Phase 1 (problem) and Phase 2 (decisions): one question at a time, you decide ...
# ... Phase 3 writes the spec; Phase 4: choose "Implementation Plan" → plan-agent writes it ...
# ... spec offers to execute → you say yes ...

# 3. Implementation runs:
#    - Each step: implement-agent → test-agent
#    - After all steps: review-agent → fixes → docs-agent → delivery checklist
#    - You commit when ready

# Or execute an existing plan at any point (optionally only some phases):
/implement ./specs/per-tenant-rate-limiter/plan
```

#### ODD — iterate in small, fully closed outcomes

```
# 1. Frame the goal and slice it into outcomes
/odd-map Per-tenant rate limiting for the public API
# ... framing, runway and slicing questions ...
# ... map written to ./odd/per-tenant-rate-limiting/map.md; O-01 is the walking skeleton ...

# 2. Deliver one outcome at a time
/odd-deliver ./odd/per-tenant-rate-limiting/map.md
#    - Card (outcome-agent) → you approve it (Gate 1)
#    - Acceptance tests written and seen failing (red)
#    - Steps implemented → everything green → scoped review → docs
#    - Delivery Record + map updated → you approve backlog changes (Gate 2)
#    - Commit the outcome (one commit per outcome — a commit message is suggested)

# 3. Repeat; pick a specific outcome, or refine the map when learnings change the plan
/odd-deliver ./odd/per-tenant-rate-limiting/map.md O-03
/odd-map ./odd/per-tenant-rate-limiting/map.md          # refine mode
```

#### Not sure which one?

Ask Claude to route it before you start — `triage-agent` is read-only and only returns a verdict:

```
Use the triage-agent agent on this item: "Add CSV export of invoices to the billing page".
```

It answers `SDD`, `ODD`, `SPLIT` (with child items) or `NEEDS_INPUT` (with the questions to answer first), with its reasons. Then run the matching interactive skill.

### 2. Agentic (unattended skills)

The agentic skills run the same processes without asking you anything; decisions go through the [advisor](#requirements) and are recorded with their source. They **never commit** — you review the working tree and commit. Run them from an interactive session (`/claude-sdd-tools:agentic-spec ...`) or headless with `claude -p`.

#### SDD

```bash
# Check the brief, then spec + plan + implement it
claude -p --advisor opus "Use the claude-sdd-tools:agentic-spec-input-check skill on ./product-brief.md. \
If it recommends run-agentic-spec, use claude-sdd-tools:agentic-spec with ./product-brief.md --then implement."

# Spec only (review it before planning), then plan + implement later
/claude-sdd-tools:agentic-spec ./product-brief.md --then none
/claude-sdd-tools:agentic-spec ./product-brief.md --out ./specs/rate-limiter --then plan
/claude-sdd-tools:agentic-implement ./specs/rate-limiter/plan --phases 1,2
```

Look at the `## Agentic Spec Result` / `## Agentic Implement Result` blocks: `COMPLETE_WITH_ASSUMPTIONS` lists the *Open Questions for the Requester*, and `self-decided` decisions are the ones to review first.

#### ODD

```bash
# Map a goal and deliver the walking skeleton (O-01)
claude -p --advisor opus "Use the claude-sdd-tools:agentic-odd-map skill with ./product-brief.md --then deliver-next."

# Map only, review the backlog, then deliver outcome by outcome
/claude-sdd-tools:agentic-odd-map ./product-brief.md --then none
/claude-sdd-tools:agentic-odd-deliver ./odd/per-tenant-rate-limiting/map.md --next
/claude-sdd-tools:agentic-odd-deliver ./odd/per-tenant-rate-limiting/map.md O-04

# Deliver every remaining eligible outcome (each fully closed; stops at the first BLOCKED/PARTIAL)
claude -p --advisor opus "Use the claude-sdd-tools:agentic-odd-deliver skill with ./odd/per-tenant-rate-limiting/map.md --all."

# Refine the map after learnings (e.g. when a Result reports "Map refine recommended")
/claude-sdd-tools:agentic-odd-map ./odd/per-tenant-rate-limiting/map.md --then none
```

A good rhythm for ODD without the backlog loop: run `--next`, read the `## Agentic ODD Deliver Result` (and its `Suggested commit message`), commit, repeat.

#### Unattended permissions

Unattended runs need tool calls to proceed without prompts. Use auto mode in an interactive session, or for `claude -p` pass an explicit allowlist (`--allowedTools`) or a permission mode (`--permission-mode`). Prefer an allowlist scoped to your build/test tools and `git`/`gh` over broad bypass modes.

### 3. Orchestration from your own agents or skills

Every agentic skill ends with a machine-readable Result block, and every agent ends with a fixed result block (`## Triage Result`, `## Outcome Agent Result`, `## Quality Gate Result`). That makes them building blocks for your own orchestrator — a custom skill, a subagent, a Claude Code workflow, or a CI job running `claude -p`.

**Route, then deliver** — what `agentic-backlog` does, in your own skill:

```markdown
1. Launch the `claude-sdd-tools:triage-agent` agent with the work item. Parse `## Triage Result`.
2. SDD → invoke the `claude-sdd-tools:agentic-spec` skill with `--then plan`, then
   `claude-sdd-tools:agentic-implement` with the plan folder from its Result.
   ODD → invoke `claude-sdd-tools:agentic-odd-map` with `--then none`, then
   `claude-sdd-tools:agentic-odd-deliver <map> --next`.
3. Fail the run if the delivery Result is BLOCKED or PARTIAL, has open review findings,
   or its tests/acceptance are not green.
```

**Independent verification of any branch** — `quality-gate-agent` is read-only and does not trust the author's conclusions:

```
Commit the work locally, then use the quality-gate-agent agent with base main, head feature/invoices-export,
the spec at ./specs/invoices-export/spec.md and the plan folder ./specs/invoices-export/plan.
```

It traces every criterion to a test that exercises it, audits test quality (failure paths, no tautologies, mocks only at external boundaries), checks the [defensive baseline](#defensive-programming-baseline) and contract fidelity, and returns `PASS`/`FAIL` per check with file:line evidence. Feed the FAIL items to `implement-agent` / `test-agent` and re-run it.

**CI / cron, one tick at a time** — run the backlog loop's tick from a scheduler instead of a session (each run does at most one unit and exits):

```bash
claude -p --advisor opus --allowedTools "<your allowlist>" "/claude-sdd-tools:agentic-backlog --source github:agent-ready"
```

### 4. Automatic orchestration with `/loop` or `/goal`

`agentic-backlog` consumes a backlog end to end: it triages each item, runs SDD or ODD, applies the quality gate, and opens one PR per unit (see [Backlog Loop](#backlog-loop) for the full algorithm and the git policy). Each invocation is one **tick** that does at most one unit of work, so something has to keep invoking it — Claude Code's [`/loop`](https://code.claude.com/docs/en/scheduled-tasks) or [`/goal`](https://code.claude.com/docs/en/goal).

#### `/loop` — long-running, paced by the skill (recommended for backlogs)

```
# 1. Write backlog.md at the repo root (see templates/backlog.md)

# 2. Start the loop with no interval: the skill paces itself
/loop /claude-sdd-tools:agentic-backlog
#    Each tick: one unit → triage → SDD or ODD delivery → quality gate → branch + PR
#    After a unit it wakes up again in 1–2 minutes; while only PRs are waiting it sleeps ≥ 20 minutes
#    You review and merge PRs; the next tick detects the merge and advances the item
#    (an ODD item's next outcome starts only after its previous outcome's PR is merged)
#    It stops by itself when nothing is left, after --max-blocked consecutive blocked units,
#    or on a failed precondition (e.g. dirty working tree). Press Esc to stop it yourself.

# GitHub Issues as the backlog, merge automatically when CI is green, stop after 2 blocked in a row
/loop /claude-sdd-tools:agentic-backlog --source github:agent-ready --auto-merge --max-blocked 2
```

Use the dynamic form (no interval). A fixed interval (`/loop 30m ...`) also works but wastes ticks while PRs wait and cannot be stopped with `Esc`.

#### `/goal` — run until a condition is met

`/goal` keeps starting turns until a small evaluator model judges the condition met (or impossible). It suits a bounded run that should end on a clear, visible result. Under `/goal`, `agentic-backlog` runs one tick per turn and prints its Result every turn, so the condition can read its `Status`:

```
# Drain the ready items in one sitting; end when only PRs awaiting review remain
/goal Run /claude-sdd-tools:agentic-backlog once per turn until a tick's "Agentic Backlog Tick Result" shows Status IDLE, WAITING, STOPPED or ERROR, or stop after 30 turns

# Deliver the rest of an ODD map outcome by outcome (you commit afterwards)
/goal Run /claude-sdd-tools:agentic-odd-deliver ./odd/per-tenant-rate-limiting/map.md --next once per turn until its Result shows "Outcomes remaining: none" or Status BLOCKED or PARTIAL, or stop after 15 turns

# Fix a branch until the independent quality gate passes
/goal Use the quality-gate-agent agent on base main and head HEAD with the spec at ./specs/invoices-export/spec.md; fix every FAIL with implement-agent or test-agent and commit the fixes locally; repeat until the Quality Gate Result shows "Overall: PASS", or stop after 6 turns

# Headless: same as the first example, to completion in one invocation
claude -p --advisor opus "/goal Run /claude-sdd-tools:agentic-backlog once per turn until a tick reports Status IDLE, WAITING, STOPPED or ERROR, or stop after 30 turns"
```

Write conditions the evaluator can check from the conversation alone — it does not run commands or read files, so name a field of a Result block the skill prints. Always include a turn bound. `/goal` (no arguments) shows progress; `/goal clear` stops it.

#### `/loop` or `/goal`?

| | `/loop /claude-sdd-tools:agentic-backlog` | `/goal ...agentic-backlog...` |
|---|---|---|
| Ends when | The skill stops it (backlog empty, circuit breaker, precondition) or you press `Esc` | The condition is judged met/impossible, the turn bound is hit, or `/goal clear` |
| Waiting on CI / human merges | Sleeps (≥ 20 min) and resumes on its own — fits `--auto-merge` | Cannot wait; end the goal on `WAITING` and start it again later |
| Best for | Continuous, long-running backlog consumption across many PRs | A bounded batch: "process what is ready now", "finish this map", "make this branch pass" |
| Headless (`claude -p`) | Self-paced loops are waited for until they end | Runs to completion in one invocation |

#### Unblocking and resuming

Blocked and needs-input items never restart from scratch. For the file source, edit the item's entry in `.agentic-backlog/state.json` (set `status: "ready"`, optionally add `"guidance"`); for GitHub, comment your guidance and set the `agent:unblocked` label. The next tick resumes the item on its existing branch. See [Unblocking and resuming items](#unblocking-and-resuming-items).

---

## Plugin Structure

```
claude-sdd-tools/
├── .claude-plugin/
│   └── plugin.json            # Plugin manifest
├── skills/
│   ├── spec/
│   │   └── SKILL.md           # Specification skill
│   ├── spec-input-check/
│   │   └── SKILL.md           # Input validation skill
│   ├── spec-implement/
│   │   └── SKILL.md           # Implementation skill
│   ├── agentic-spec/
│   │   └── SKILL.md           # Unattended specification skill
│   ├── agentic-spec-input-check/
│   │   └── SKILL.md           # Unattended input validation skill
│   ├── agentic-implement/
│   │   └── SKILL.md           # Unattended implementation skill
│   ├── odd-map/
│   │   └── SKILL.md           # Outcome mapping skill
│   ├── odd-deliver/
│   │   └── SKILL.md           # Outcome delivery skill
│   ├── agentic-odd-map/
│   │   └── SKILL.md           # Unattended outcome mapping skill
│   ├── agentic-odd-deliver/
│   │   └── SKILL.md           # Unattended outcome delivery skill
│   └── agentic-backlog/
│       └── SKILL.md           # Unattended backlog loop skill
├── agents/
│   ├── plan-agent.md          # Planning agent
│   ├── implement-agent.md     # Step implementation agent
│   ├── review-agent.md        # Code review agent
│   ├── test-agent.md          # Testing agent
│   ├── docs-agent.md          # Documentation agent
│   ├── outcome-agent.md       # Outcome card agent
│   ├── triage-agent.md        # SDD/ODD routing agent
│   └── quality-gate-agent.md  # Independent quality verification agent
├── references/
│   └── defensive-programming.md  # Shared defensive programming baseline
├── templates/
│   └── backlog.md             # Example backlog file
├── LICENSE
└── README.md
```

## License

[MIT](LICENSE)
