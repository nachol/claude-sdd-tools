# claude-sdd-tools

A Claude Code plugin that implements a complete **Spec-Driven Development (SDD)** workflow — from problem definition through specification, planning, implementation, review, testing, and documentation.

## Overview

This plugin provides 3 interactive skills, 3 agentic (unattended) counterparts, and 5 agents that work together as a structured pipeline. The workflow ensures that every piece of code originates from a well-defined specification, is implemented following a concrete plan, reviewed for correctness and security, tested with real code exercise, and documented before delivery.

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

### Agents

| Agent | Triggered By | Purpose |
|-------|-------------|---------|
| [plan-agent](#plan-agent) | `/spec` (Phase 4) | Generate executable implementation plans |
| [implement-agent](#implement-agent) | `/implement` (per step) | Implement one plan step faithfully |
| [review-agent](#review-agent) | `/implement` (post-steps) | Code review for correctness and security |
| [test-agent](#test-agent) | `/implement` (per step) | Create and validate tests |
| [docs-agent](#docs-agent) | `/implement` (post-steps) | Generate and update documentation |

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
| Types/Signatures | Full code block with struct definitions, method signatures, constants, doc comments |
| Implementation Notes | Algorithmic details, edge cases, integration points, performance notes |
| Done-when | Mechanically verifiable completion criteria |

**Responsibilities:**

- Produces plans precise enough that an implementor fills in function bodies, not designs
- Never includes test specifications — `test-agent` handles tests at runtime
- Never includes documentation phases — `docs-agent` handles docs at runtime
- Every step must produce at least one file and have concrete completion criteria
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

**When to use it:** Automatically invoked by `/implement` for each plan step. Can also be invoked directly to implement a single, well-specified step.

**Scope:**

- Reads the plan step, overview conventions, and referenced spec sections in full before writing code
- Implements exactly what the step specifies — named files, exact types/signatures, implementation notes, Done-when
- May compile/build/lint to verify Done-when criteria
- Stops and reports a blocker when the plan is ambiguous, incomplete, or conflicts with the spec — never guesses

**Responsibilities:**

- Does NOT design, does NOT write tests (test-agent owns tests), does NOT commit
- No features, abstractions, or configuration beyond what the step calls for — smallest correct implementation
- Only touches files within the current step's scope; never implements later phases or refactors unrelated code
- Honors every convention in the plan overview (module path, encapsulation, error wrapping, naming)
- All code and comments in English; every exported symbol gets a doc comment

---

### Quality Gate: Code Review

#### review-agent

**What it does:** Performs a structured code review focused on correctness and security, modeled after real production bugs. Runs 6 parallel review passes and applies skeptical validation to eliminate false positives.

**When to use it:** Automatically invoked by `/implement` after all steps are complete. Can also be used independently after writing or modifying code.

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

**When to use it:** Automatically invoked by `/implement` after each implementation step. Can also be used independently when tests need planning, fixing, or improvement.

**Scope — 3 operational workflows:**

| Workflow | When | Process |
|----------|------|---------|
| Test Planning | New code written | Analyze changes → identify coverage gaps → design tests exercising real code → plan edge cases |
| Test Fixing | Tests failing | Identify failures → analyze root cause → apply quality-preserving fixes → verify no regressions |
| Test Improvement | Existing tests need work | Assess quality → reduce mocks → improve coverage → optimize performance |

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
- Continues until 100% test pass rate — never leaves test management partially done

---

### Quality Gate: Documentation

#### docs-agent

**What it does:** Generates and updates project documentation — README, ARCHITECTURE, CONTRIBUTING, OpenAPI specs, PERSISTENCE docs, CHANGELOG entries, and code comments on exported symbols.

**When to use it:** Automatically invoked by `/implement` after review-agent completes. Can also be used independently for initial documentation generation or to sync docs after code changes.

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

## Agentic Skills

`spec` and `spec-input-check` are **user-invoked only** (`disable-model-invocation: true`): they depend on a human answering questions, so agents cannot trigger them. The `agentic-*` skills run the same process **unattended**, so an agent — a subagent, a workflow, or a headless `claude -p` session — can invoke them directly through the Skill tool.

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
| `spec-input-check` | `agentic-spec-input-check` | Reads the checklist from `spec-input-check` (single source of truth), has the advisor review the verdict, writes the report next to the document, and returns a recommended next action (`run-agentic-spec` or `fix-document-first`). |

Each agentic skill ends its response with a **Result block** (`Status: COMPLETE | COMPLETE_WITH_ASSUMPTIONS | ... | BLOCKED`, paths, counts) that the invoking agent can parse.

### Requirements

- **Advisor configured** for advisor-reviewed decisions: run `/advisor <model>`, set `advisorModel` in settings, or start with `claude --advisor <model>`. Subagents inherit the configured advisor.
- The advisor is a server-side tool of the Anthropic API. Through an LLM gateway (`ANTHROPIC_BASE_URL`) it works only if the gateway forwards it; otherwise every decision is `self-decided`.

### Example

```bash
# Headless: check the brief, then spec + plan + implement it without a human in the loop
claude -p --advisor opus "Use the claude-sdd-tools:agentic-spec-input-check skill on ./product-brief.md. \
If it recommends run-agentic-spec, use claude-sdd-tools:agentic-spec with ./product-brief.md --then implement."
```

---

## End-to-End Example

```
# 1. (Optional) Validate your input document
/spec-input-check ./product-brief.md

# 2. Run the specification process
/spec Build a rate limiter for our public API with per-tenant limits

# ... interactive Q&A through Phases 1-4 ...
# ... spec chooses "Implementation Plan" → plan-agent generates the plan ...
# ... spec offers to execute → user says yes ...

# 3. Implementation runs automatically:
#    - Each step: code → test-agent
#    - After all steps: review-agent → docs-agent → delivery checklist
#    - Final summary presented
#    - User commits when ready

# Or manually at any point:
/implement ./implementation-plan
```

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
│   └── agentic-implement/
│       └── SKILL.md           # Unattended implementation skill
├── agents/
│   ├── plan-agent.md          # Planning agent
│   ├── implement-agent.md     # Step implementation agent
│   ├── review-agent.md        # Code review agent
│   ├── test-agent.md          # Testing agent
│   └── docs-agent.md          # Documentation agent
├── LICENSE
└── README.md
```

## License

[MIT](LICENSE)
