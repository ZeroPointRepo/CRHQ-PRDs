# Code Refactor Skill

> **Purpose:** Produce executable refactoring reports for an existing codebase, using a multi-round, multi-agent methodology. Two modes: **AUDIT** (high-level — find & rank opportunities) and **PLAN** (deep-dive — produce verifiable per-target refactor specs).

---

## When to Use This Skill

Trigger on any of:

- "Audit this codebase for refactor opportunities"
- "What should we refactor in <project>?"
- "Tech debt review", "code quality audit", "maintainability review"
- "Do a deep-dive refactor plan for <feature/module>"
- "I want to give a developer a refactor task they can execute end-to-end"
- User wants either a high-level prioritized list **or** a specific, executable refactor proposal

**Do NOT use for:**

- Bug fixes (the code is wrong; this skill is for code that *works* but should be better)
- Behavior changes / new features (refactor = same behavior, better internals)
- Renames or trivial reorganization (a chat reply suffices)
- Production incidents — fix first, refactor later

---

## The Two Modes

This skill has **two operating modes**. Pick one at the start; never run both in the same invocation.

| Mode | Purpose | Input | Output |
|---|---|---|---|
| **AUDIT** | Find and rank refactor opportunities across a project (or subsystem) | Project/subsystem path; optional focus areas | Prioritized list of opportunities, top-to-bottom by impact × leverage / risk. Each item is a paragraph plus tags — *not* an execution spec. |
| **PLAN** | Produce executable, verifiable refactor specs for 1–3 named targets | 1–3 targets (often picked from a prior AUDIT) plus the project path | One deep-dive spec per target, each with full pre/post verifiability loop, step-by-step instructions, and an HTML artifact template. |

The two modes share the same multi-round methodology. They differ in *what gets drafted* and *what the reviewers are looking for*.

**Typical sequence:** user runs AUDIT → reviews ranked list → picks top 1–3 → user runs PLAN on those targets → executor developer follows each PLAN.

---

## Core Principles

1. **Behavioral parity is non-negotiable.** A refactor changes internals, never observable behavior. Every PLAN must prove parity with a pre-baseline and a post-comparison. If behavior must change, that's a feature/fix, not a refactor — say so and stop.
2. **Evidence binds every claim.** "This is slow", "this is hard to maintain", "this is duplicated" — each must cite file paths and line numbers. No vibes-based recommendations.
3. **The executor closes the loop.** A PLAN's success is measured by whether the receiving developer can: capture baseline → do the work → re-run the same checks → produce a verdict. If the loop doesn't close, the PLAN failed.
4. **Specific over abstract.** "Extract to a helper" is not a plan. "Move lines 47–93 of `src/foo.ts` into a new `src/lib/normalizeFoo.ts` exporting `normalizeFoo(input: Foo): NormalizedFoo` — see proposed diff below" is a plan.
5. **Right-size the report.** A 5-file project does not need a platform audit. A subsystem PLAN does not need a full-stack ceremony. The methodology is fixed; depth adapts.
6. **Multi-round, multi-perspective.** Draft 1 → Dual Review → Draft 2 → Final Review → Draft 3. Two reviewer perspectives in parallel catch different categories of issues. Skipping rounds costs more downstream than it saves now.
7. **Project conventions win.** Read the project's own coding standards / dev guidelines first. The refactor must end up *more* aligned with the project's conventions, never less.
8. **Defaults are decisions.** Every customization in a PLAN (test command, baseline location, artifact path) gets a default the executor can run with. The executor never blocks.
9. **The PLAN is also a runbook.** Final section instructs the executor end-to-end, including how to produce the HTML artifact. The PLAN is *what* and *how*, not just *what*.

---

## The Seven Phases (shared by both modes)

```
Phase 1: Scoping & Context Load           (mode, scope, project conventions, test infra)
Phase 2: Discovery — Draft 1              (parallel agents, one per facet or per target)
Phase 3: Dual Review                      (skeptic + Reviewer B, parallel)
Phase 4: Draft 2                          (apply ALL fixes, parallel where possible)
Phase 5: Final Review                     (single reviewer, hunt remaining issues)
Phase 6: Draft 3                          (final fixes, mark Final)
Phase 7: Delivery                         (HTML artifact + working memory + handoff)
```

Phases are identical in shape across AUDIT and PLAN. The *contents* of each phase differ — see the per-mode sections.

---

## Phase 1 — Scoping & Context Load

Identical for both modes.

### 1.1 Lock the mode

- AUDIT or PLAN?
- If PLAN: which 1–3 targets? (max three; if user offers more, push back and ask them to pick)
- Scope: full project, or a subsystem path / set of paths?

If the user is ambiguous, ask in a **single batched message**. Never one question at a time.

### 1.2 Load project context

Always read, in this order:

1. Project's CLAUDE.md / agents.md (if present)
2. Project's development guidelines / coding standards document (in DB project documents, or in repo)
3. Any PRD or spec the project follows
4. The repo's README, package.json (or equivalent), test config

Identify and capture:

- **Tech stack** — language, framework, runtime, package manager
- **Test infra** — unit / integration / e2e commands, test framework, location of tests
- **Build / typecheck / lint commands** — exact strings
- **Linter & formatter** — which configs are authoritative
- **Conventions** — naming, file org, error model, validation library, logging shape
- **Branch & commit conventions** — Conventional Commits? squash policy? PR template?
- **Deployment** — what "shipped" looks like (CI? PM2? Docker?)
- **Forbidden zones** — anything the project explicitly says do-not-touch

Save this context to a working note before drafting — it gets referenced in every subsequent phase.

### 1.3 Decide the discovery split

For **AUDIT**, run the 7 default facets (drop any that don't apply); add extension facets if the project clearly needs them — see Phase 2 — AUDIT.

For **PLAN**, allocate one deep-dive analysis agent per target.

Plan the split before launching anything.

---

## Phase 2 — Discovery / Draft 1

The modes diverge here.

### Phase 2 — AUDIT mode

Launch parallel agents, one per facet group. Each agent scans its facet across the scoped paths and returns **candidate opportunities with evidence**.

#### Default facet split (7) — run all that apply

| Facet | What the agent looks for |
|---|---|
| **1. Architecture & boundaries** | Tangled layers, cyclic deps, leaky abstractions, mis-located logic, modules doing too much, undocumented contracts at module seams |
| **2. Data layer** | N+1 queries, missing indexes, unindexed FKs, schema drift, transaction misuse, ORM anti-patterns, query hotspots |
| **3. API surface / Component architecture** | (Backend) Inconsistent endpoint shapes, ad-hoc error envelopes, drift from spec, missing validation at boundaries. (Frontend) Component sprawl, prop drilling, state-management drift, render-perf hotspots |
| **4. Testing** | Coverage gaps in critical paths, slow tests, flaky tests, mocked-too-much tests, missing integration coverage, brittle e2e selectors |
| **5. Performance** | Hotspots (CPU, IO, memory), unbounded loops, sync-over-async, missing caching, oversized bundles, slow startup |
| **6. Code quality & conventions** | Drift from project's own coding standards; inconsistent naming; `any`/escape-hatch type usage; dead code; duplicated logic; missing strict-mode flags |
| **7. Observability** | Swallowed errors, inconsistent logging shape, missing context, no structured logs, missing telemetry on critical paths, error model drift |

Drop any facet that doesn't apply (e.g. drop **Data layer** for a static site).

#### Extension facets — add only when relevant

| Facet | When to add |
|---|---|
| **Build & tooling** | When CI is slow, scripts are brittle, dependency drift / advisories suspected |
| **Security posture** | When the project handles PII, auth, payments, or has a public-internet attack surface |
| **Configuration & env** | When env / config is suspected to have multiple sources of truth, or secrets-in-code risk |
| **Frontend / UI** *(separate from Component architecture)* | When the project has both backend and frontend, and you're already running the API facet — split UI into its own agent |

#### Per-agent prompt pattern (AUDIT)

```
You are scanning <facet> in the project at <path>.
Project conventions are summarized at <working note path>.

For every refactor opportunity you find, return:
- Short title (≤80 chars)
- One-paragraph summary
- Evidence: 3–5 file:line references (real, verifiable)
- Impact dimensions affected: maintainability / reliability / speed / accuracy / security (rate each H/M/L/—)
- Estimated effort: S / M / L / XL  (S=<1 dev-day, M=1–3, L=3–10, XL=>10 — XL must be split before PLAN)
- Estimated risk: S / M / L
- Why now (or: why later) — one sentence

Constraints:
- Do not propose feature changes, only refactors (same behavior, better internals).
- Do not propose anything without evidence.
- Do not duplicate findings already covered by a more general one — note overlaps.
- Be honest about effort and risk; underestimating burns the executor.

Save findings to <agent-specific path>.
```

#### Effort/risk anchors

```
Effort:
  S  = <1 dev-day
  M  = 1–3 dev-days
  L  = 3–10 dev-days
  XL = >10 dev-days  — MUST be split into smaller items before any PLAN runs

Risk:
  S = isolated; rollback is `git revert`; no shipped state affected
  M = touches multiple callsites or shared abstractions; rollback is straightforward
  L = touches data, migrations, or cross-service contracts; rollback requires explicit plan
```

#### Assemble Draft 1 (AUDIT)

1. **Cluster duplicates.** Two facets often surface the same root issue. Merge with a unified title; keep evidence from all angles.
2. **Compute priority score** (advisory, not verdict):
   ```
   priority = (impact_sum × confidence) / (effort × risk)
   where impact_sum = sum of dimension ratings (H=3, M=2, L=1, —=0)
         confidence = 1.0 if evidence is strong, 0.7 if partial, 0.4 if weak
         effort     = S=1, M=2, L=4, XL=8
         risk       = S=1, M=2, L=4
   ```
3. **Order top-to-bottom by score, then editorially re-order if judgment disagrees.** When the agent overrides the score-based order, it must add a one-sentence rationale next to the moved item:
   > *"Moved REFACTOR-005 above REFACTOR-003 despite lower formula score: blocks 4 other opportunities by clearing a shared abstraction."*
   Reviewers can challenge both the formula inputs and any overrides.
4. **One paragraph per opportunity** in AUDIT — *not* an execution spec. The user's next step is to pick top 1–3 for PLAN mode.

### Phase 2 — PLAN mode

For each target (1–3), launch one deep-dive analysis agent.

#### Per-agent prompt pattern (PLAN — discovery half)

```
You are deeply analyzing the refactor target: <target name + scope>.
Project at <path>. Conventions at <working note path>.

Map the target's full topology:
1. Exact files / modules / functions / endpoints / tables in scope
2. Every callsite of the target's public surface (file:line refs)
3. Every test that exercises the target (file:line refs)
4. Every external dependency the target uses
5. Behavior contract (what does this currently do, observably)
6. Test commands that exercise the target
7. Any manual / browser-driven behaviors that matter
8. Performance characteristics (if measurable)
9. Documented or undocumented edge cases
10. Anything currently broken in this target (separate from the refactor — flag, don't fix)

Save topology to <topology path>. Keep file:line refs precise.
```

#### Draft 1 production (PLAN)

After topology agents return, produce one PLAN spec per target, following the **PLAN Report Schema** below. Drafting can be parallelized (one drafting agent per target) once topology is in.

---

## Phase 3 — Dual Review

Two reviewers in parallel. Different perspectives. Reviewer A is **the same in both modes**; Reviewer B **changes by mode** because the right "different perspective" genuinely differs.

### Reviewer A — The Skeptic (both modes)

Reads Draft 1, challenges *the claims*.

Looks for:

- Opportunities (AUDIT) or proposals (PLAN) **not backed by evidence**
- **Overstated impact** — "this is critical" when it's a 50-line module nobody touches
- **Understated risk** — refactors that look small but ripple across many callsites
- **Effort estimates that are clearly wrong** in either direction (especially: any XL hiding as L)
- **Hidden behavior changes** sneaking into a "refactor" — flag any proposed change that would alter observable output
- **Missing alternatives** — was a smaller / less invasive option considered?
- **Conventions drift** — does the proposed end-state align with the project's own guidelines?

### Reviewer B — by mode

#### AUDIT → Reviewer B = The Prioritizer

Asks: *is this ordering defensible?*

Looks for:

- Items mis-ranked by the formula (effort or risk under/overstated)
- Editorial re-orderings that lack rationale
- Top items that are *easy* but *low impact* crowding out *higher leverage* items
- Bottom items that should not even be on the list (out of scope, behavior change in disguise, already mitigated)
- Themes / cross-cuts that should have been called out (e.g. five separate items all rooted in the same broken abstraction)
- Confidence ratings that don't match the strength of cited evidence

#### PLAN → Reviewer B = The Executor's Eye

Asks: *can a developer actually run this end-to-end?*

Looks for:

- **Baseline capture is concrete enough to actually run** (commands present, outputs defined, save locations specified)
- **Refactor steps are concrete enough to follow** without re-thinking the design
- **Callsite hunt list is complete** (every place that touches the target is enumerated)
- **Post-refactor parity check is a real comparison**, not "the tests pass"
- **Rollback plan is real** (not "git revert" — what if the migration is partially applied?)
- **HTML artifact template is producible** with the data the executor will have
- **Drift log discipline** is preserved — no hidden behavior changes smuggled in via §9.6

### Required Reviewer Output Format

Both reviewers produce reports in this shape:

```
# Review Report — <Reviewer A | Reviewer B>

## CRITICAL — Will produce a wrong / unsafe / unexecutable report
1. <short title> — <where, what, how to fix>

## HIGH — Will produce confusion or rework
...

## MEDIUM / LOW
...

## Specific challenges
- Claim X (in section Y) is not supported by evidence — needs <citation>
- Effort estimate Z is undercounted because ...
- ...

## Priority Action Items (P0 / P1 / P2)
```

Vague feedback ("strengthen the verifiability section") is rejected and re-requested. The reviewer must say *what* line, *what* fact is missing, *what* concrete change.

---

## Phase 4 — Draft 2

Synthesize ALL findings from both reviewers into a unified fix list. Then launch parallel rewrite agents (typically same split as Phase 2) with explicit per-section fix instructions.

### Draft 2 agent prompt pattern

```
You are rewriting <section / target> for Draft 2. Read the current draft at <path>.
Apply ALL of the following fixes:

1. <Specific fix — where, what, why> — e.g., "REFACTOR-007's evidence cites no file:line; add 3+ refs from the topology in <topology path>"
2. ...

Save the rewritten section to <new path>.
```

Fixes must be specific and copy-pasteable. "Strengthen security" is rejected. "Add input validation step at line N of REFACTOR-003's refactor instructions, using the project's `zod` schema convention from `src/lib/validation/`" is accepted.

### Cross-cutting consistency to reconcile in Draft 2

- **Test commands** — one canonical command per test tier; same string used in baseline and post-check
- **Path conventions** — match the project's actual layout, not generic guesses
- **Naming of new code** — match project's existing naming
- **Validation / error model** — must match what the project already uses
- **Logging / observability** — match the project's existing logger and shape
- **Environment / config** — match the project's single source of truth for env vars
- **HTML artifact location & filename pattern** — pick once, reference everywhere

---

## Phase 5 — Final Review

Single reviewer, final pass. Goal: catch what the dual review missed.

Specifically check:

- Did Draft 2 fixes actually land?
- Did any fixes introduce new contradictions?
- Are all evidence citations real (file:line) and accurate?
- Is the priority ordering (AUDIT) or per-target ordering (PLAN) defensible?
- Are all test commands runnable as written?
- (PLAN) Does each target's pre-baseline have a 1:1 partner in the post-comparison?
- (PLAN) Is the HTML artifact template self-contained and producible?
- (PLAN) If §7.6 (Known pre-existing failures) has entries, are they justified?
- (PLAN) If §9.6 (Drift log) is anticipated, are the four discipline rules visible?
- Does each milestone / step have a testable exit criterion?
- **Beyond this checklist** — last chance to catch anything that would make the report stronger.

Output: a concise list of remaining issues. If the doc is solid, the reviewer says so explicitly.

---

## Phase 6 — Draft 3

Single agent applies the final review's remaining fixes. Usually small. After this pass:

- Mark the report **Final**
- Stamp date and project name
- Save to working location

---

## Phase 7 — Delivery

Three things must happen, adapted to the host environment:

1. **Render the report as a shareable HTML artifact** — the AUDIT or PLAN goes through the project's artifact-display mechanism so the user sees and can share it.
2. **Save a portable copy** — a markdown file at a stable path; if the host supports public links, produce one.
3. **Update working memory** — drafts produced, key decisions, what mode was used, what targets if PLAN.

For PLAN mode, **the deliverable to the executor developer must include the HTML artifact template they'll fill in** — see "Executor's HTML Artifact Template" below.

---

## AUDIT Report Schema

Header:
- **Mode:** AUDIT
- **Project:** <name>
- **Scope:** <paths or "full project">
- **Date:** <date>
- **Methodology:** Multi-round, multi-agent (Code Refactor skill)

Sections:

### Executive Summary
3–5 sentences. Overall health, top 1–3 themes, suggested next step (typically: "run PLAN on REFACTOR-001..003").

### Scope & Method
- Paths reviewed
- Facets reviewed (note any default facets dropped, any extension facets added)
- Conventions referenced
- Tools / agents used

### Priority Scoring
Show the formula. Show the H/M/L scale and effort/risk anchors. Note that the score is **advisory** — the agent may editorially re-order items, with one-line rationale per override.

### Opportunities (ranked)

For each opportunity, in priority order:

```
### REFACTOR-NNN — <Title>

**Scope:** <files / modules / endpoints>

<One paragraph: what's wrong, what's the better state.>

**Evidence:**
- `path/to/file.ts:47–93` — <what's there>
- `path/to/other.ts:12` — <what's there>
- ...

**Impact:**
- Maintainability: H / M / L / —
- Reliability: H / M / L / —
- Speed: H / M / L / —
- Accuracy: H / M / L / —
- Security: H / M / L / —

**Effort:** S / M / L / XL
**Risk:** S / M / L
**Confidence:** 1.0 / 0.7 / 0.4
**Priority Score:** <number>
**Editorial override (if any):** <one sentence rationale, omit otherwise>

**Why now / why later:** <one sentence>
```

No execution detail in AUDIT — that's PLAN's job.

### Themes & Cross-Cuts
If multiple opportunities share a root cause (e.g. "validation drift across endpoints"), call it out as a theme. Helps the user pick PLAN targets that have leverage.

### Recommended PLAN Targets
Top 1–3 the user should consider for deep-dive. State why each made the cut.

---

## PLAN Report Schema (per target)

For each of 1–3 targets, the PLAN contains exactly the sections below. Numbering is per-target (REFACTOR-007 → P-007.1 through P-007.12).

### P-NNN.1 Title & ID
- **ID:** matches the AUDIT ID if applicable, else a fresh REFACTOR-NNN
- **Title**
- **Date:** <date>
- **Project:** <name>
- **Branch convention:** <e.g. `refactor/<slug>` per project guidelines>

### P-NNN.2 Scope
Exact files / modules / endpoints / tables / migration files in scope. **Anything not listed is out of scope.**

### P-NNN.3 Current State
- What's there now
- Code excerpts (≤30 lines each, file:line cited)
- Callsite map (every place that uses the target's public surface)
- Tests that currently exercise it (file:line)
- Behavior contract — what the target observably does today, in plain prose

### P-NNN.4 Problem Statement
Why this needs refactoring. Concrete, evidence-bound. Not "code smell" — *what* smell, *where*, *why it matters*.

### P-NNN.5 Refactor Goal
What the better state looks like. Architecturally, in prose. Plus: the **non-goals** — what this PLAN deliberately does *not* change. (Non-goals stop scope creep.)

### P-NNN.6 Impact Dimensions
H/M/L per dimension (maintainability, reliability, speed, accuracy, security), with a sentence justifying each rating.

### P-NNN.7 Pre-Refactor Verifiability — BASELINE

**The executor MUST complete this section in full before touching any code.**

#### 7.1 Environment baseline
Exact commands; expected: pass.
- `<typecheck command>` — expect: 0 errors
- `<lint command>` — expect: 0 errors
- `<build command>` — expect: success
- `<full test command>` — expect: all pass

If any of the above does **not** currently pass on the base branch, the executor STOPS and chooses one of:
- **(a) Fix the failing check first**, in a separate PR. Refactor PR rebases on top once base is green. *Default option.*
- **(b) Scope the failing check out of the parity guarantee** — see §7.6.

#### 7.2 Targeted test capture
- List of tests that exercise the target (commands that run them in isolation)
- Save outputs verbatim to `<baseline path>/tests-pre.txt`

#### 7.3 Behavioral baseline (manual smoke tests)
For each user-visible behavior the target affects:
- Step-by-step (click here, enter this, hit submit)
- Expected output (exact strings, screenshot if applicable)
- Save observations to `<baseline path>/manual-pre.md`

#### 7.4 Performance baseline (if relevant)
- Exact command(s) to capture timing / memory / bundle size
- Capture 3 runs, record median
- Save raw numbers to `<baseline path>/perf-pre.txt`

#### 7.5 Baseline checkpoint
Commit baseline files to a fresh branch (`refactor/<slug>-baseline`) so they're recoverable.

#### 7.6 Known pre-existing failures (escape hatch — use sparingly)

If §7.1's hard STOP rule is invoked under option (b), each excluded check is documented here:

```
- Check: `<exact command>`
- Failure: <one-line description; link to issue if open>
- Why scoped out: <why fixing-first is impractical for this PLAN>
- Parity impact: <which behaviors this check normally guards; what risk this exclusion creates>
- Reviewer/PM acknowledgment: <name + date>
```

Rules:
- §7.6 is empty by default. Only entries explicitly invoked under §7.1(b) appear here.
- A PLAN with **more than one entry in §7.6** signals the AUDIT mis-scoped this target. Reviewer A flags this in Phase 3.
- Each entry must include reviewer or PM acknowledgment before the PLAN ships to an executor.

The baseline must be **concrete and runnable**. If the executor has to invent commands, the PLAN failed.

### P-NNN.8 Refactor Instructions — THE WORK

**Pre-conditions:**
- Branch `refactor/<slug>` created from `<base>`
- Baseline section 7 fully completed
- All baselines green (or §7.6 entries acknowledged)

**Step-by-step changes (numbered):**

For each step:
- What changes (which file, which lines)
- Proposed diff or precise edit instruction
- Why this step (one sentence)
- Convention notes (link to project guidelines)
- Gotchas / things to pay attention to

**New abstractions / interfaces introduced:**
- Name, signature, location
- Why it earns its keep

**Migration approach:**
- In-place edit, parallel-implementation-with-cutover, or shim-and-deprecate?
- If multi-step, the cutover sequence

**Callsite hunt list:**
- Every callsite (file:line) that must be updated
- For each: "before / after" delta description

**Things to pay attention to:**
- Cross-cutting concerns the executor will hit (logging, error model, validation)
- Project conventions to honor

**Style & convention notes:**
- Direct references to the project's coding standards
- Naming, file org, formatting expectations

### P-NNN.9 Post-Refactor Verifiability — PARITY CHECK

**The executor MUST complete this section in full before declaring done.**

#### 9.1 Re-run environment baseline
- Same commands as 7.1, expect: same results (still all green, except any §7.6-acknowledged checks)

#### 9.2 Re-run targeted tests
- Same commands as 7.2
- Save outputs to `<baseline path>/tests-post.txt`
- **Diff against `tests-pre.txt`.** Differences require explicit justification (e.g. test was renamed in this refactor, or covered under §9.6 drift log) — note in §9.6.

#### 9.3 Re-run manual smoke tests
- Same scripts as 7.3
- Save observations to `<baseline path>/manual-post.md`
- Behavioral parity checklist:
  - [ ] Behavior 1 — same as pre
  - [ ] Behavior 2 — same as pre
  - …

#### 9.4 Re-run performance baseline
- Same commands as 7.4 (3 runs, median)
- Save to `<baseline path>/perf-post.txt`
- Compute delta vs pre. **Speed-target refactors** must show measurable improvement; **non-speed refactors** must show no regression beyond <X%> noise floor (define X% per project).

#### 9.5 New tests added (optional but encouraged)
If the refactor surfaces a previously-untested edge case, add a test for it. Note location.

#### 9.6 Drift log — intentional behavior changes (use sparingly, strict rules)

If, during the refactor, the executor finds a pre-existing bug whose fix is unavoidable in-flight, document each occurrence here. **Empty by default.** Each entry must satisfy ALL four rules:

```
Drift entry:
  Location:        <file:line of the changed behavior>
  Pre-existing bug: <commit / issue / code reference proving the bug existed before this refactor>
  Diff justification: <pre-output → post-output; why it differs>
  New test added:   <path:line of test that locks in the corrected behavior>
```

**Four discipline rules:**
1. **Pre-existing only.** Drift may only correct behavior that was *already* wrong before the refactor began. Net-new functionality is not drift.
2. **Documented in §9.6.** Every diff vs. baseline that is not byte-identical must be either covered here or in a renamed-test note in §9.2.
3. **Test required.** Every drift entry has a corresponding new test that locks the corrected behavior.
4. **Size cap.** If the drift would change behavior in **more than ~3 places**, the executor STOPS. The work splits: refactor PR (no drift) ships first; bug fixes ship as a separate PR. The PLAN itself does not absorb the fix scope.

If the executor cannot satisfy all four rules for a behavioral diff, the refactor has changed behavior — STOP and treat as a feature/fix, not a refactor.

### P-NNN.10 Acceptance Criteria

All must be true to declare done:

- [ ] All env baseline checks green (or §7.6 entries acknowledged)
- [ ] All targeted tests pass
- [ ] `tests-pre.txt` ≡ `tests-post.txt` (or differences justified in §9.2 / §9.6)
- [ ] All manual smoke tests pass with same observable behavior (modulo §9.6 drift entries)
- [ ] Perf check passes (improvement for speed targets; no regression otherwise)
- [ ] No new lint / typecheck errors
- [ ] Project conventions honored (per project guidelines)
- [ ] §9.6 drift log: ≤ ~3 entries, all four rules satisfied, all backed by new tests
- [ ] HTML artifact (§12) produced and saved
- [ ] PR opened with link to artifact

### P-NNN.11 Rollback Plan

- **If parity fails partway:** branch is disposable; revert to baseline branch.
- **If a partial migration is shipped:** explicit unwind procedure (DB migration down-migration, feature-flag flip, cutover reversal).
- **If post-merge regression discovered:** the revert PR command, plus any data fixes needed.

The rollback must be specific. "Git revert" is not a plan unless the change is purely code with no migrations / no shipped state.

### P-NNN.12 Final Report — HTML Artifact

The executor produces an HTML artifact summarizing the run. Use the **Executor's HTML Artifact Template** (below) as the starting point.

Required artifact contents:
- Header: target name, date, executor, branch, PR link
- Pre-baseline summary (commands + key outputs)
- Changes summary (files changed, LOC delta, key abstractions added)
- Post-baseline summary (commands + key outputs)
- Diff vs baseline (tests, behavior, perf)
- §7.6 entries (if any) — acknowledged exclusions
- §9.6 drift log (if any) — entries with proof
- Acceptance checklist (filled in)
- Verdict: **PASS** or **FAIL** (with reason)

Save to `<artifact path>` and surface via the project's artifact-display mechanism.

---

## Executor's HTML Artifact Template

A self-contained HTML document the executor fills in at the end of a PLAN. The PLAN must hand this template (or a pointer to it) to the executor.

Structure (in plain HTML/CSS, no external deps):

```
- <header>: Project, Target ID, Title, Date, Executor, Branch, PR
- <section id="verdict">: Big PASS / FAIL banner, one-sentence summary
- <section id="pre-baseline">:
    - Env checks: command + result table
    - Test results: pre snapshot
    - Manual checks: pre snapshot
    - Perf numbers: pre snapshot (3 runs + median)
    - §7.6 acknowledged exclusions (if any)
- <section id="changes">:
    - Files changed (table)
    - LOC delta
    - New abstractions introduced
    - Notable design decisions
- <section id="post-baseline">:
    - Env checks: command + result table
    - Test results: post snapshot
    - Manual checks: post snapshot
    - Perf numbers: post snapshot (3 runs + median)
- <section id="diff">:
    - Test output diff (pre vs post) — must be empty or fully justified
    - Behavior parity checklist (filled)
    - Perf delta table
    - §9.6 drift log entries (if any)
- <section id="acceptance">: checklist with checks filled in
- <section id="rollback">: link to rollback plan, plus disposal instructions if shipped
```

Style: simple, monospace for command/output blocks, clean tables, no external CDN. Mobile-responsive. The template itself is project-agnostic; the PLAN customizes contents.

---

## Anti-Patterns to Avoid

| Anti-pattern | Why it kills the report |
|---|---|
| Recommending refactors without file:line evidence | Reviewer can't validate; executor can't locate |
| "Improve maintainability" with no concrete what/how | Unactionable; produces churn |
| Skipping baseline capture | No way to prove parity; executor improvises |
| Skipping post-refactor parity check | Behavior drift ships unnoticed |
| Recommending behavior change disguised as refactor | Refactor = same behavior, better internals — anything else is a feature/fix |
| Listing 30 opportunities in AUDIT with no prioritization | The user can't act on a flat list |
| Treating the priority formula as a verdict | Score is advisory; editorial judgment can override with a one-line rationale, and reviewers can challenge both |
| One reviewer | Misses categories of issues a second perspective would catch |
| Vague reviewer feedback ("strengthen security") | Cannot be applied; produces churn |
| PLAN that requires the executor to design things mid-stride | Either the PLAN specifies, or it leaves a labeled "design choice X" with options and a recommended default |
| Letting "TBD" survive into Final | Means executor will guess (badly) |
| Ignoring the project's own coding standards | Refactor produces less-aligned code — net negative |
| Padding the report to look thorough | Length is a side effect of completeness, not a target |
| Running both modes in one invocation | Confuses scope; AUDIT and PLAN are sequential, not concurrent |
| Letting an XL item ship as a single PLAN target | XL must be split during AUDIT; if it survives to PLAN, the AUDIT was wrong |
| Stuffing §7.6 with multiple exclusions | Means the AUDIT mis-scoped this target — go back |
| Drift log entries without all four discipline rules | Either it's not really a refactor, or scope creep — STOP |

---

## Quality Markers

A finished report should pass:

- Mode is unambiguous; outputs match the mode's schema
- Every claim has evidence (file:line) — none are hand-waved
- (AUDIT) Priority ordering is defensible; formula is visible; editorial overrides are justified
- (AUDIT) No XL items remain — they've been split
- (PLAN) Pre and post baselines are 1:1 mirrors — every check has a partner
- (PLAN) §7.6 has 0 or 1 entries (more = AUDIT was wrong); each entry is acknowledged
- (PLAN) §9.6 drift log is empty by default; if populated, all four rules satisfied per entry
- (PLAN) Every step in §8 is concrete enough to execute without re-thinking
- (PLAN) Callsite hunt list is exhaustive (the executor doesn't have to grep for more)
- (PLAN) Acceptance criteria are testable, not aspirational
- (PLAN) Rollback plan is real (not just "git revert")
- (PLAN) HTML artifact template is self-contained and producible
- Project conventions are honored end-to-end
- No surviving "TBD", "TODO", or "we'll figure this out later"
- The receiving developer can run the loop without coming back with clarifying questions

---

## Reference Workflow Checklist

```
Phase 1 — Scoping & Context Load
  [ ] Mode locked: AUDIT or PLAN
  [ ] Targets locked (if PLAN): 1–3 specific items
  [ ] Scope locked: paths / subsystems
  [ ] Project conventions loaded (CLAUDE.md, dev guidelines, PRD)
  [ ] Test/build/lint commands captured
  [ ] Discovery split planned (default 7 facets for AUDIT, drop/add as fits)

Phase 2 — Discovery / Draft 1
  [ ] (AUDIT) Default facets selected; extension facets added if relevant
  [ ] (AUDIT) Facet agents launched in parallel
  [ ] (PLAN) One topology agent per target; launched in parallel
  [ ] All agents returned
  [ ] Findings clustered (AUDIT) / topology assembled (PLAN)
  [ ] (AUDIT) XL items split before they reach the ranked list
  [ ] Draft 1 written per the appropriate schema

Phase 3 — Dual Review
  [ ] Skeptic reviewer launched
  [ ] Mode-appropriate Reviewer B launched (Prioritizer for AUDIT, Executor's Eye for PLAN)
  [ ] Both returned with required output format
  [ ] Findings unified into a single fix list

Phase 4 — Draft 2
  [ ] Per-section fix instructions specific & copy-pasteable
  [ ] Rewrite agents launched in parallel
  [ ] Cross-cutting consistency reconciled
  [ ] Draft 2 assembled

Phase 5 — Final Review
  [ ] Single reviewer pass
  [ ] Remaining issues listed (concise)

Phase 6 — Draft 3
  [ ] Final fixes applied
  [ ] Marked Final + dated

Phase 7 — Delivery
  [ ] Rendered as HTML artifact via host's artifact mechanism
  [ ] Markdown copy saved to stable path
  [ ] Working memory updated
  [ ] Handed off to user with clear pointers (and PLAN handoff instructions if applicable)
```

---

## When to Update This Skill

Update when:

- A new facet emerges that AUDIT consistently misses (add to the default 7 or extension menu)
- A new category of "executor blocker" appears in PLAN reviews (add to Executor's Eye checklist)
- A new project type needs a different verifiability shape (e.g. mobile app smoke testing)
- A pattern emerges that should become a default

Do **not** update for:

- Project-specific quirks (those go in the report itself, not the skill)
- One-time formatting preferences
- Tech-stack-specific opinions (those belong in code-conventions skills)
