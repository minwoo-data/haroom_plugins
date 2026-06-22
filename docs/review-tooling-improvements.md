# Review-Tooling Improvements - Design Proposal

Status: DRAFT for adversarial review (prism-all)
Author: Minwoo Park
Date: 2026-06-22
Scope: improvements to the `prism` / `prism-all` / `triad` / `triad-all` review
skills and the `ddaro` worktree-workflow plugin. NOT a change to the
`expense-us` product. This doc is the deliberation artifact that precedes any
code change; it exists to be attacked before we build.

---

## 0. Context (self-contained - reviewers may not have the repos)

**The tools in play:**

- `prism` - multi-angle CODE review. 5 Claude subagents review one target file
  in parallel (Conflict / Improvement / Devil / CodeReview / Robustness), then a
  Verifier cross-checks singleton findings. Read-only; emits a report.
- `prism-all` - same 5 angles run on BOTH Claude (5) and Codex CLI (5) = 10
  discovery calls, then Verifier. Cross-model agreement = highest-confidence
  tier. Reviews ONE target.
- `triad` / `triad-all` - 3-lens deliberation (LLM-clarity / architecture-
  longevity / end-user-comprehension) on ONE document or file, multi-round until
  consensus. triad-all adds the Codex engine.
- `ddaro` - worktree-based parallel-workflow plugin. Has a lifecycle conductor:
  `/ddaro:spec` (design doc), `/ddaro:review` (fans out prism-all + triad and
  collates findings), `/ddaro:check` (pre-merge gate). So ddaro ALREADY
  orchestrates the review skills around the worktree cycle.

**The limitation that motivates this doc:** every reviewer above reviews exactly
ONE target. Real review work is over a LIST. In this project's "R2 30-feature
review" the operator needed to review 30 features (F01-F30) and had to hand-build
the orchestration: a CHECKLIST.md (30 features, batched, with per-feature tier
assignments), a dependency-map.md (cross-feature blast radius), an
AUDIT-production-readiness.md ("12 systemic facts already verified - do NOT
re-audit per feature"), 3 parallel worktrees each running `prism` + `mangchi` /
`triad` per feature, then a FIX-PLAN -> FIX-PLAN-REVIEW -> CODEX-VERIFY loop.
That hand-built pipeline is the de-facto spec for what the tooling is missing.

**Precedent that the upstream flow works:** prism-all's mandatory "SANDBOX SAFETY
PREAMBLE + secret-scrub" (invariants 6/7) was born from a real secret-leak
incident in THIS project (2026-05-22, `docker compose config` resolved env_file
secrets into Codex output). A project lesson became a generic skill safeguard.
This doc asks whether more lessons should flow the same way - and importantly,
which should NOT (to avoid polluting generic tools with project specifics).

---

## 1. Core architectural recommendation (the spine)

Split the proposals by altitude, and do NOT bloat the generic reviewers:

- **Per-target reviewers (`prism-all` / `triad-all`) stay single-target.** They
  only absorb improvements that make a single review better and stay
  project-agnostic: proposals **A** and **C**.
- **List-level orchestration lives in the `ddaro` conductor**, most likely a new
  `/ddaro:audit` command (sibling to `/ddaro:review`). It owns proposals **B, D,
  E, F, G**. Rationale: ddaro already fans out prism-all + triad in
  `/ddaro:review`; batch-over-a-list is the natural extension, and it keeps the
  generic skills lean. The R2 KICKOFF.md is the prototype spec for this command.

If this split is wrong, the whole doc is wrong - reviewers should attack it
first.

---

## 2. Reviewer-level proposals (prism-all / triad-all)

### A. Structured schema output instead of YAML-text parsing

**Problem.** Both prism-all and triad-all parse Codex stdout by hunting for a
`^codex$` marker, reading until `^tokens used$`, and extracting hand-rolled
YAML. The SKILLs even document model-specific parser quirks ("5.5 does not use
yaml fences"). This is brittle: a model output-format change silently breaks
parsing, and malformed YAML degrades a whole angle.

**Proposal.** Define a fixed findings JSON schema (severity / locus / problem /
proposed_fix / verdict). Claude-side agents return it via forced structured
output; Codex-side calls instruct strict JSON and validate-with-one-retry.
Synthesis consumes validated objects, not scraped text.

**Why now.** Parser fragility is a standing maintenance tax tied to external
model behaviour we do not control.

**Tradeoffs / risks.** Codex CLI cannot be forced into tool-call schema the way
Claude can, so "structured" for Codex is still prompt-discipline + validation,
not a hard guarantee; the retry adds latency/cost. Net: reduces Claude-side
fragility fully, Codex-side partially.

**Belongs in:** prism-all + triad-all (shared schema, each keeps its own copy
per the plugins' independence rule).

### C. Project-local review-heuristics injection

**Problem.** This project's pre-commit guardrails encode hard-won failure modes
(silent JS `catch` -> ghost-Loading; broad `except` -> swallowed GA-only bug;
route-map line drift; Windows hardcoded paths; receipt-image-commit PII). A
generic reviewer rediscovers these from scratch every run, or misses them.
Hardcoding them into the skill would pollute it for every other project.

**Proposal.** A generic, opt-in mechanism: the reviewer reads a project-local
heuristics file (e.g. `.prism/heuristics.md`, or a tagged section of CLAUDE.md)
and injects "known failure modes for THIS project" into each angle's prompt. The
skill stays project-agnostic; each project supplies its own anti-patterns.

**Why now.** Lets breadth scale with the project's accumulated lessons without
forking the skill per project.

**Tradeoffs / risks.** Injected heuristics could bias reviewers toward only the
known modes (Goodhart - they look where the flashlight points and miss novel
bugs); large heuristics files inflate every prompt; a project could inject bad
guidance. Mitigation: keep injection additive ("also watch for..."), cap size,
never replace the base angle prompt.

**Belongs in:** prism-all + triad-all (and by inheritance prism / triad).

---

## 3. Orchestration-level proposals (new `/ddaro:audit`)

### B. List batch review (deterministic fan-out over many targets)

**Problem.** Reviewing N targets today = the operator manually issues N reviews,
tracks which are done, and aggregates by hand (the R2 CHECKLIST + 3 terminals).

**Proposal.** `/ddaro:audit <list>` runs each item through review stages as a
deterministic pipeline (one item can be verifying while another is still in
discovery), resumable on crash, with a progress ledger. Built on the Workflow
primitive (pipeline/parallel), not on the main agent remembering to parallelize.

**Why now.** The single-target ceiling forced a bespoke, error-prone manual
process for the one time real breadth was needed.

**Tradeoffs / risks.** Large token blast radius (N targets x 10 calls each);
needs hard caps + a "what was skipped" log so silent truncation never reads as
"covered everything". Concurrency cap interplay with prism-all's own fan-out.

### D. Shared-facts "do not re-discover" memo

**Problem.** When many targets are reviewed, every reviewer independently
re-derives the same global truths (CSRF is global, auth is centralized, ...).
Wasted tokens + repeated noise. R2 solved this with AUDIT-production-readiness.md
listing 12 systemic facts marked "do not re-audit per feature."

**Proposal.** `/ddaro:audit` produces (or is given) a shared-context memo of
verified global facts and feeds it to every per-target reviewer as read-only
context, with the instruction "these are established; focus per-target evidence."

**Why now.** Directly cuts the dominant cost driver in batch review (redundant
horizontal re-discovery).

**Tradeoffs / risks.** A wrong "established fact" in the memo suppresses real
findings everywhere (single point of failure). Needs the memo itself to be
verified, and a way for a reviewer to dispute a "fact."

### E. Cross-feature blast-radius lens

**Problem.** All five prism angles look INSIDE the target. R2 needed a
dependency-map "break-glass" so each reviewer knew what else a change touched.
Nothing in the skills surfaces cross-module impact.

**Proposal.** An orchestration-level lens: before/with the per-target review,
compute and attach the target's blast radius (callers, shared state, downstream
consumers) so findings account for cross-target impact.

**Why now.** The densest defect cluster in R2 was cross-feature (matching <->
self-match <-> vendor-norm <-> trip-builder), exactly what single-target angles
miss.

**Tradeoffs / risks.** Blast-radius computation is itself error-prone
(dynamic dispatch, late binding); a wrong map misleads. Partial overlap with
prism's Robustness "state transitions" axis - must be positioned as distinct
(cross-target, not intra-target) or it is noise.

### F. Fix-plan -> opposite-engine cross-verify ("yangcheuk geomjeung")

**Problem.** prism-all stops at the findings report (read-only). Acting on
findings - producing a remediation plan and trusting it - is unguarded. R2 did
this by hand: FIX-PLAN -> FIX-PLAN-REVIEW -> CODEX-VERIFY. This is the
"verify both sides" pattern the operator explicitly wants.

**Proposal.** A `--fix-plan` stage: after findings, one engine drafts a
remediation plan; the OPPOSITE engine (cross-model) verifies the plan before any
code is touched. Distinct from `mangchi` (which iterates on a single file's
code); this verifies a PLAN across findings.

**Why now.** The highest-leverage, highest-risk moment (deciding what to change)
currently has no second set of eyes.

**Tradeoffs / risks.** Scope creep toward "auto-fix" - must stay plan-level and
read-only unless explicitly told to apply. Cross-model verify can deadlock on
disagreement; needs a tie-break (defer to human) rule.

### G. Production-readiness scorecard / merge gate

**Problem.** prism output is severity-tiered findings, not a go/no-go verdict.
R2 used an 8-lens PASS/FLAG/FAIL scorecard with file:line evidence where FAIL =
merge blocker. `/ddaro:check` is a gate but consumes prism findings ad hoc.

**Proposal.** A `--scorecard` mode that maps findings to a fixed lens set and
emits a PASS/FLAG/FAIL verdict per lens + an overall merge recommendation, so
`/ddaro:check` has a structured gate instead of prose.

**Why now.** Aligns the review output with the actual decision it feeds (merge or
not) and with ddaro's existing gate.

**Tradeoffs / risks.** A scorecard invites Goodharting (optimize the score, not
the code) and false confidence ("all PASS" != safe). The lens set is a value
judgment that must be defensible and stable.

---

## 4. Open questions (for the reviewers to press on)

1. Is the §1 split correct, or should `prism-all` itself grow a batch mode (i.e.
   is ddaro the right home for B/D/E/F/G)?
2. Which proposals are not worth their complexity? Rank by (value / cost).
3. Do any proposals conflict? (e.g. D's shared-facts memo vs C's heuristics
   injection - overlapping "context injection" mechanisms that should perhaps be
   ONE mechanism, not two.)
4. F and `mangchi` overlap in spirit - is plan-level cross-verify distinct
   enough to justify a separate path, or should mangchi absorb it?
5. What is the smallest subset that captures most of the R2 value? (If we ship
   only 2 of the 7, which 2?)

## 5. Non-goals

- Auto-applying fixes without explicit opt-in (F stays plan-level).
- Baking any project-specific heuristic into a generic skill (that is exactly
  what C's injection mechanism exists to avoid).
- Replacing `mangchi` (single-file iterative refinement) or the existing
  single-target reviewers.
