# PRD Builder Skill

> **Purpose:** Build comprehensive, AI-agent-executable PRDs through a multi-draft, multi-reviewer methodology. The methodology is universal; the output adapts to the project (size, tech stack, scope, type).

---

## When to Use This Skill

Trigger on any of:
- User asks to write a PRD, product spec, technical spec, or build plan
- User wants to scope a non-trivial software project or feature
- User says "I want an AI agent to build this" or "I want to give this to a developer"
- User describes a system/feature/product and expects you to formalize it

**Do NOT use for:** trivial bug fixes, single-line changes, or anything where a few sentences in chat suffice.

---

## Core Principles

1. **No ambiguity, ever.** Every contradiction in the PRD becomes a bug in the implementation. The reviewer rounds exist to find and kill these.
2. **Canonical sources.** Each concern (schema, API, security, naming, etc.) has ONE canonical section. All other sections reference it; they never redefine it.
3. **Parallel where possible, sequential where dependent.** Drafting and reviewing parallelize naturally. Phases are sequential.
4. **At least three drafts.** Draft 1 (write) → Reviews → Draft 2 (fixes) → Final Review → Draft 3 (final fixes). Skipping rounds costs more downstream than it saves now.
5. **Two reviewer perspectives.** A self-review (architect's eye) and a different-perspective review (security/QA/UX/etc.) catch different categories of issues.
6. **Defaults are decisions.** Every customization question must have a working default so the executing agent never blocks.
7. **The PRD is also a runbook.** The final section instructs the executor — the PRD is not just *what to build*, it's *how to build it*.
8. **Right-size everything.** A small feature PRD should not pretend to be a platform spec. The methodology is fixed; the output's scope, depth, and length adapt to the request.

---

## Project Types — Adapt Before Drafting

Before applying the canonical structure, classify the request. Different types need different emphasis.

| Project Type | What changes |
|---|---|
| **New full-stack product** | Full canonical structure applies. All 10+ sections relevant. |
| **New backend service / API** | Frontend sections are minimal or skipped. API and schema sections expand. |
| **New frontend / UI app** | Schema may be omitted. Component architecture, state management, design system expand. |
| **Feature added to existing product** | **MUST analyze the existing codebase first.** PRD references existing patterns, conventions, schema, and adds delta. Many sections become "follow existing conventions in `<file>`" rather than restating them. |
| **Refactor / migration** | Schema diffs, migration plan, rollback plan, testing matrix become primary. |
| **Internal tool / CLI** | UI sections drop; CLI ergonomics, configuration, distribution take their place. |
| **Library / SDK** | Public API surface, semver policy, examples, docs expand. Server/UI sections drop. |
| **Integration / connector** | External service contract, error handling, retries, rate limits become primary. |

If the request is to extend an existing codebase, **Phase 1 expands** to include a codebase analysis pass before drafting begins. Do not write a PRD that contradicts the code that's already shipped.

---

## The Seven Phases

```
Phase 1: Requirement Discovery        (parse intent, codebase analysis if applicable, fill gaps)
Phase 2: Draft 1 Production            (parallel agents, one per section group)
Phase 3: Dual Review                   (self-review + different-perspective review, parallel)
Phase 4: Draft 2 Production            (apply ALL fixes, parallel where possible)
Phase 5: Final Review                  (one reviewer, hunt remaining issues)
Phase 6: Draft 3 Production            (apply remaining fixes)
Phase 7: Delivery                       (artifact + download link if available + memory update)
```

---

## Phase 1 — Requirement Discovery

### Step 1.1: Classify

Pick a project type from the table above. This drives everything.

### Step 1.2: For "feature on existing codebase" — Analyze First

Before any drafting, understand the codebase:
- Tech stack actually in use
- Folder structure and naming conventions
- Existing patterns (data access, validation, error handling, auth)
- Existing schema (tables, columns, FK relationships)
- Existing API conventions (path style, response shape, auth model)
- Test setup and conventions
- Deployment/build scripts

The PRD must extend these, not contradict them.

### Step 1.3: Parse Intent Against Requirement Dimensions

These are **prompts, not requirements** — a frontend-only PRD doesn't need a database section. Use as a checklist of what *might* be relevant.

| # | Dimension | Often relevant for |
|---|-----------|---------------------|
| 1 | Product/feature type | All |
| 2 | Core domain entities/objects | Backend, full-stack, data-heavy UIs |
| 3 | Users & roles | Multi-user products |
| 4 | Authentication & authorization | Anything with users |
| 5 | Tech stack hints | All |
| 6 | External integrations | Anything that talks to other services |
| 7 | Quality/feature bar | All — but the bar varies by project |
| 8 | API exposure / public-API posture | Backend, libraries |
| 9 | Compliance / security | Regulated domains, anything with PII |
| 10 | Performance / scale targets | Anything user-facing or high-volume |
| 11 | Deployment model | Productionized work |
| 12 | Customization scope | Multi-tenant or operator-configurable products |
| 13 | Execution model (AI vs human, solo vs team) | All — drives Section 10 |
| 14 | Output / deliverable format | All |
| 15 | Existing constraints (codebase, infra, brand) | Extension projects |

### Step 1.4: Write a Requirement Summary

Capture in session memory or a working note before drafting. Keep it terse:

```markdown
## Requirement Summary
- Type: <one of the project types>
- Scope: <one paragraph>
- Key dimensions: <relevant from the list>
- Existing context: <if extension — what's already there>
- Open questions: <only ones that genuinely need answers>
```

If there are open questions, ask them in **a single batched message**. Never one at a time.

---

## Phase 2 — Draft 1 Production

### Section Structure — Adaptive, Not Fixed

Below is a **canonical menu** of sections. Use what fits the project. Add sections specific to the project type. Drop or merge sections that don't apply.

| # | Section | When to include |
|---|---------|------------------|
| 1 | **Before You Start** (user customization Q&A with defaults) | When the PRD will be executed by another agent or team and customization is expected |
| 2 | **Project Overview & Vision** | Always |
| 3 | **Technology Stack & Architecture** | New projects (or extensions where stack matters); skip for pure spec docs |
| 4 | **Conventions & Best Practices** | Most projects; for extensions, replace with "Follow existing conventions in `<paths>`" |
| 5 | **Data Model / Schema** | When data persistence is involved |
| 6 | **Core Features / Functional Requirements** | Always |
| 7 | **API / Interface Design** | When there's a service interface (HTTP, RPC, library, CLI) |
| 8 | **Security, Privacy, Compliance** | Always — even "no auth needed" deserves a sentence |
| 9 | **Milestones / Execution Plan** | When execution will be sequenced (multi-step build) |
| 10 | **Executor Instructions** | When an AI agent or external team will execute |
| + | **Project-specific sections** | Add freely. E.g. "Design System", "ML Model Spec", "Migration Plan", "Rollout Plan", "Telemetry" |

**The agent has discretion to add, merge, split, or drop sections** based on what the project actually needs. A frontend-only PRD might have: Overview, Design System, Component Architecture, State Management, Routing, Conventions, Milestones, Executor Instructions — and that's the right shape for it.

### Parallel Drafting

Split the writing across parallel background agents. Group related sections together so each agent has coherent context.

For each agent:
- Hand it the section(s) to write with bullet-point requirements
- Hand it a save location
- Tell it to be exhaustive within scope: "every edge case, every validation rule, zero ambiguity *for the things that need to be specified*"
- Tell it what's canonical elsewhere so it references rather than redefines

Run the agents in parallel. Wait for completion. Assemble into a single document.

### Per-Section Detail Expectations

Calibrate to the project type. Examples:

- **Schema sections** (when present): every column, type, constraint, index, FK behavior, seed data, migration order
- **Feature sections**: API endpoints, validation, frontend behavior, side effects, authorization, edge cases — *as relevant*
- **Component sections** (frontend-heavy): component hierarchy, props, state ownership, interaction patterns
- **CLI sections**: command tree, flags, exit codes, output formats
- **Library sections**: public API surface, types, examples, error model

The principle: **specify what's specifiable, leave room for craft where craft is needed.** Do not force a schema section into a CSS-refactor PRD.

---

## Phase 3 — Dual Review

Launch two reviewers in parallel with different perspectives. They read the assembled Draft 1 and produce structured findings.

### Reviewer A: Self-Review (Architect's perspective)

Looks for:
- **Gaps** — what's missing that the project type needs?
- **Contradictions** — does Section X say something different from Section Y?
- **Ambiguities** — could a careful executor still misinterpret this?
- **Sequencing issues** — do milestones depend on things they don't actually have yet?
- **Structural issues** — broken cross-references, duplicated specs, naming drift

### Reviewer B: Different-Perspective Review

The right "different perspective" depends on the project type. Pick one or compose a hybrid:

| Project type | Suggested perspective |
|---|---|
| Full-stack product | Security engineer + QA lead |
| Backend service | Security + reliability/SRE |
| Frontend / UI | UX + accessibility + design systems |
| Library / SDK | API designer + DX (developer experience) |
| Internal tool | Operator / day-2 maintainer |
| Data pipeline | Data engineer + cost/perf |
| Mobile | Platform-conventions expert |

Whichever you pick: instruct the reviewer to **poke holes specifically from that perspective**.

### Required Reviewer Output Format

Both reviewers produce reports in this shape:

```markdown
# Review Report

## CRITICAL — Will cause build failures or major rework
1. <short title> — <where, what, how to fix>

## HIGH — Will cause confusion or significant rework
...

## MEDIUM / LOW
...

## Master Contradiction Table
| # | Topic | Section A says | Section B says | Resolution |

## Priority Action Items (P0 / P1 / P2)
```

Insist on this format. Vague feedback ("improve security") is rejected and re-requested.

---

## Phase 4 — Draft 2 Production

Synthesize ALL findings from both reviewers into a fix list. Then launch parallel agents (typically the same split as Phase 2) with explicit per-section fix instructions.

### The Draft 2 Agent Prompt Pattern

```
You are rewriting Section <X> for Draft 2. Read the current draft at <path>.
Apply ALL of the following fixes:

1. <Specific fix — where, what, why>
2. ...

Save the rewritten section to <new path>.
```

**Critical:** Make fixes specific and copy-pasteable. Don't say "fix the password policy" — say "harmonize password minimum to N characters in Sections X.Y, X.Z, and N." Vague fix instructions create new contradictions.

### Cross-Cutting Standards to Reconcile in Draft 2

These categories of contradiction tend to appear across nearly every PRD. The *categories* are universal; the *specific values* are project-dependent. Pick one canonical answer per category and enforce it everywhere.

- **Path / endpoint conventions** — pick one style and use it everywhere
- **Response and error envelope shapes** — define ONCE, reference elsewhere
- **Pagination model** — pick one approach
- **Naming conventions** — match what fits the stack and the codebase
- **Soft vs hard delete** — pick one policy per entity type
- **Identifier generation** — application-side or DB-side, pick one
- **Migration / schema-change tooling** — one tool
- **Validation library** — one library, ideally shareable across boundaries
- **Configurable vs fixed enums** — be explicit which is which

These are *categories to check*, not *answers to enforce*. The right answer depends on the codebase, the team, and the constraints.

---

## Phase 5 — Final Review

Single reviewer, final pass. Goal: catch what the dual review missed.

Specifically check:
- Did the Draft 2 fixes actually land?
- Did any fixes introduce new contradictions?
- Are all sections internally consistent?
- Do schema references match the schema definition (if applicable)?
- Are all environment variables / configuration items documented in one place?
- Do milestones each have testable exit criteria?
- Are cross-references between sections still accurate?
- **Beyond this checklist** — this is the final review and the last chance to catch anything that might still be improved. Look for issues, gaps, ambiguities, or improvements *not* enumerated above. Trust your judgment; flag anything that would make the PRD stronger.

Output: a concise list of remaining issues — not a massive document. If the doc is solid, the reviewer says so.

---

## Phase 6 — Draft 3 Production

Single agent applies the final review's remaining fixes. These are usually small (one-line corrections, value swaps, missing items). After this pass:
- Update the version header to "Final" (or whatever finalization marker fits)
- Save to the working location

---

## Phase 7 — Delivery

Three things should happen, **adapted to whatever environment the skill is running in**:

1. **Show the result to the user.** If the host environment supports inline artifacts/canvases/previews, render the PRD there in editable form. Otherwise, surface it via whatever the environment provides (file path, message attachment, link).
2. **Make it portable.** If the host environment supports public links/downloads, produce one. Otherwise, ensure the file is saved somewhere the user can retrieve.
3. **Update working memory.** Capture what was produced, what drafts existed, and key decisions made — so future sessions or audits can reconstruct the reasoning.

The skill is environment-agnostic. Use the host's facilities for artifacts, downloads, and memory; do not assume specific paths or APIs.

---

## Quality Markers

A finished PRD should pass these checks. Adapt the list — not every marker applies to every project.

- [ ] Each concern has a single canonical section; other sections reference, never redefine
- [ ] No surviving contradictions from the master contradiction table
- [ ] All findings from the different-perspective review are addressed or explicitly accepted with rationale
- [ ] Cross-references between sections are accurate
- [ ] Every external dependency is named (libraries, services, APIs)
- [ ] Every configuration item is documented in one place
- [ ] Every milestone has testable exit criteria (when milestones are present)
- [ ] Defaults are specified for every customization question (when Q&A is present)
- [ ] Project-type-relevant cross-cutting concerns are addressed (e.g. accessibility for UIs, rate limits for APIs, observability for services)
- [ ] No surviving "TBD", "TODO", or "we'll figure this out later"
- [ ] An executor reading this PRD cold can begin work without immediately needing clarifying questions about the document itself
- [ ] The PRD respects existing codebase conventions when extending an existing project
- [ ] The PRD's depth matches the request — not bloated, not skeletal

---

## Anti-Patterns to Avoid

| Anti-pattern | Why it kills the PRD |
|---|---|
| Writing one big monolith yourself in serial | Slow, prone to forgetting sections, no parallelism gain |
| Single-reviewer feedback | Catches one perspective's blind spots only |
| Skipping Draft 2 | Critical contradictions reach implementation |
| Vague review feedback ("improve security") | Cannot be applied; produces churn |
| Inline-redefining canonical shapes (response/error/pagination) per section | Guarantees contradiction — define once, reference everywhere |
| Letting "TBD" survive into final draft | Means the executor will guess (badly) |
| Forcing the canonical 10-section template onto a project that doesn't fit it | Bloat or irrelevance |
| Specifying tech that ties the PRD to one infrastructure or vendor when the user didn't ask for it | Reduces portability — keep environment-specific bits behind a referenced skill or env var |
| Asking the user a string of one-at-a-time questions | Disrespects their time; batch them |
| Drafting before requirement summary is captured | Context drifts mid-draft |
| Writing a PRD for a feature without first reading the existing codebase | Produces a PRD that contradicts shipped code |
| Padding to hit a word/line count | Length is a side effect of completeness, not a target |

---

## Reference Workflow Checklist

Use as a live checklist when running the skill. Adapt commands and paths to the host environment.

```
Phase 1 — Discovery
  [ ] Classify project type
  [ ] If extending existing code: analyze the codebase
  [ ] Parse user prompt against Requirement Dimensions
  [ ] Write requirement summary to working memory
  [ ] Ask batched follow-up questions if anything is genuinely unclear
  [ ] Confirm scope with user before proceeding

Phase 2 — Draft 1
  [ ] Decide which canonical sections apply; add project-specific ones
  [ ] Plan section split across parallel drafting agents
  [ ] Launch all drafting agents in parallel (use whatever async/background mechanism the host provides)
  [ ] Wait for completions
  [ ] Assemble Draft 1 into a single document

Phase 3 — Dual Review
  [ ] Pick a "different perspective" appropriate to the project type
  [ ] Launch self-reviewer + different-perspective reviewer in parallel
  [ ] Wait for both reports
  [ ] Read both fully

Phase 4 — Draft 2
  [ ] Synthesize unified fix list from both reviews
  [ ] Launch parallel rewrite agents with specific per-section fix instructions
  [ ] Reconcile cross-cutting categories listed above
  [ ] Assemble Draft 2

Phase 5 — Final Review
  [ ] Single reviewer, concise output expected
  [ ] Identify remaining issues (typically small)

Phase 6 — Draft 3
  [ ] Apply final fixes
  [ ] Mark as Final
  [ ] Save to the working location

Phase 7 — Delivery
  [ ] Show the result via the host's artifact/preview mechanism (if available)
  [ ] Produce a download/share link (if the host supports it)
  [ ] Update working memory with summary of drafts and decisions
  [ ] Hand off to the user with clear pointers
```

---

## When to Update This Skill

Update when:
- A pattern emerges across multiple PRDs that should become a default consideration (add to canonical sections menu or reviewer prompts)
- A category of issue keeps appearing in final review (add it to Phase 4 cross-cutting categories)
- A user provides post-execution feedback revealing a blind spot (e.g. "the responsive part wasn't covered" → add as a project-type consideration)
- A new project type becomes common (add to the project-types table)

Do NOT update for:
- Project-specific quirks (those go in the PRD itself, not the skill)
- One-time formatting preferences
- Tech-stack-specific opinions (those belong in code-conventions skills, not here)
