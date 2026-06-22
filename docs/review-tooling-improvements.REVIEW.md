# PRISM-ALL REPORT - review-tooling-improvements.md - 2026-06-22

Mode: default (10 discovery + inline singleton verification)
Engines: Claude 5 + Codex 5 (gpt-5.5)
Verifier: inline (Pass 2 short-circuited - only ~4 singletons; near-total cross-model agreement)

> Bottom line: the review is brutally consistent across BOTH engines. The doc's
> own §1 framing ("where does each proposal live") is the central flaw - it
> never makes each proposal beat "do nothing." Against that bar, **A survives,
> B-G mostly do not** (as standalone builds). The 10 reviewers independently
> converged on this.

---

## CRITICAL (cross-model, must address before any build)

- **[cross-model/devil+conflict+improvement] §1 launders "should we build?" into "where does it live?"**
  The altitude split lets every proposal survive by finding it a home. Both
  engines say: evaluate each proposal against "do nothing" FIRST; assign
  altitude only to survivors. Codex adds the ownership-bias angle (author owns
  all the tooling being expanded -> default to "no new command / no new gate /
  no new lens unless a smaller patch demonstrably fails").
  -> Fix: reframe the doc decision-first, not placement-first.

- **[cross-model/devil] B + the new `/ddaro:audit` command may automate an N=1 problem and re-invent existing primitives.**
  R2 was one milestone audit; no evidence it recurs. B also duplicates the
  Workflow primitive (pipeline/parallel/resume) AND `/ddaro:review` (already
  fans out prism-all + triad). Both engines: do NOT create `/ddaro:audit`;
  if batch is ever needed, add `/ddaro:review --batch` over a list, and gate it
  behind real recurrence (Codex: "≥2-3 separate audits with the same pain").

- **[cross-model/devil+robustness] D (shared-facts memo) is a systemic false-negative machine.**
  One wrong "established fact" suppresses real findings across EVERY target -
  exactly how systemic bugs survive broad audits. Codex rated this CRITICAL
  ("CUT or invert"). -> If it exists at all: rename to "claims to challenge /
  assume-unless-contradicted", every fact carries evidence + verified-at + a
  dispute channel, and it may NEVER disable a check (advisory only).

- **[cross-model/devil+improvement+robustness] G (scorecard merge gate) = Goodhart / checkbox theater.**
  A PASS/FLAG/FAIL gate fed by imperfect findings manufactures false confidence;
  authors optimize the score. Codex rated CRITICAL ("CUT scorecard"). -> If
  anything: a deterministic severity-rollup (any unresolved CRIT -> FAIL,
  partial-coverage -> never PASS) + structured blockers (`must_fix` /
  `accepted_risk` / `needs_test` / `unknown`) consumed by the EXISTING
  `/ddaro:check`. No new synthetic lens-scoring pass, no new gate command.

---

## HIGH (cross-model)

- **[cross-model/all-angles] C and D are the SAME mechanism (context injection) in opposite directions.**
  C amplifies ("also watch for X"), D suppresses ("don't re-audit X"). Shipping
  both = two prompt-plumbing paths, two failure modes, conflicting instructions
  with no precedence. UNANIMOUS resolution: ONE typed `ReviewContext` contract
  with tagged sections (`heuristics`=advisory/amplify, `verified_facts`=
  disputable-presumed, optional `blast_radius`, `scope_limits`); reviewers
  consume one slot, the orchestrator populates it; **amplify always wins over
  suppress on the same locus** (fail toward surfacing).

- **[cross-model/devil + claude-singleton:CONFIRMED] context injection (C/D/E) degrades prism-all's CORE value.**
  prism-all's selling point is INDEPENDENT angles + cross-model agreement =
  high confidence. Injecting shared context into every discovery angle
  CORRELATES them - correlated reviewers agreeing is worthless. Strongest single
  argument against C/D/E. -> Keep discovery angles blind; any shared context
  belongs in a SEPARATE synthesis/gate step, never in discovery prompts. (If C
  ships at all: route heuristics to ONE dedicated 6th "known-modes" lens, keep
  the other 5 blind.)

- **[cross-model/conflict+devil+improvement] F (fix-plan cross-verify) overlaps `mangchi` and `/ddaro:review`/`/ddaro:check`.**
  mangchi already does cross-model adversarial verify (with mature stop/abort
  rules F lacks). -> Don't build a parallel loop. Either fold F into
  `/ddaro:review` as an optional second pass on a fix plan, OR extend mangchi to
  accept a plan artifact. F's ONLY genuinely-new value = multi-target fix
  ordering / cross-target conflict detection (which fixes touch the same shared
  module) - mangchi can't see that because it sees one file.

- **[cross-model/devil+improvement+conflict] E (blast-radius) - cut the "grand dependency engine".**
  A wrong map is worse than none (false confidence under dynamic dispatch -
  exactly when you need it). Both engines: keep ONLY a lightweight evidence
  attachment (changed files, `rg` callers, routes/tests touched), computed ONCE
  for the whole list, presented as metadata - NOT a 6th reviewer lens (overlaps
  Robustness/state-transitions otherwise).

- **[cross-model/all] A (structured schema) is the consensus survivor - but scope it honestly.**
  The real brittleness is the Codex-side scrape, and Codex CANNOT be hard-forced
  into a schema - so A only swaps "regex scrape" for "JSON + retry that can
  still fail." Both engines converge: Claude-side forced structured output
  (real), Codex-side = validate + ONE retry + on failure emit a visible
  `parser_error`/`ANGLE-DEGRADED` finding (NEVER silent-drop, NEVER whole-run
  fail), keep the scrape as a fallback. Version the schema (per-plugin copies
  WILL drift) and treat it as the cross-altitude interface contract.

## MEDIUM (cross-model + spec gaps)

- **[cross-model/robustness] partial coverage must never read as PASS.** Engine-
  down mid-batch, budget skip, or schema-degraded = explicit UNKNOWN/NOT-RUN
  state; overall verdict FAIL-if-any-FAIL else FLAG-if-any-UNKNOWN else PASS.
- **[cross-model/robustness] concurrency cap ownership undefined.** N targets x
  10 inner calls = up to 300 concurrent calls. One global semaphore owned by the
  conductor; prism-all's inner fan-out draws from it.
- **[cross-model/code-review] every proposal is underspecified to build** (list
  format, ledger schema, A field types/enums, C file location, E method, F
  engine-selection + plan schema, G lens enumeration). Spec only the survivors.
- **[codex-singleton:CONFIRMED] "yangcheuk geomjeung" (F) violates the
  English-labels-only project rule.** Rename to `opposite-engine cross-verify`.
- **[claude-singleton:CONFIRMED] incremental / changed-targets scoping is the
  biggest omitted cost lever.** A re-audit should re-run only changed targets +
  their blast radius (carry-forward PASS otherwise). Adjacent to Codex's "triage
  pass" + "cost governor + coverage.md" findings.

## Rejected singleton

- **[codex/code-review: REJECTED]** "prism described as Claude subagents but
  skills say Codex." Factual error by Codex - `prism` IS Claude subagents;
  `prism-codex` is the Codex variant. §0 is correct.

## Smallest-subset vote (the doc's Open Q5)

| Reviewer | pick |
|---|---|
| Claude Devil | A only (cut B/D/E/F/G) |
| Claude Improvement | B + D |
| Codex Devil | A + narrowly-bounded C |
| Codex Improvement | B + A |

**A = 3/4 votes (the clear winner). B = 2/4 (contested).** No proposal beats
"do nothing" as strongly as A.

## Recommended action order (synthesized)

1. **Ship A only**, scoped honestly (Claude forced-output; Codex validate+retry+
   visible-degrade-finding; versioned schema). This is the one clear win and it
   is the substrate everything else would need anyway.
2. **Defer B/D/E/F/G.** Do not build `/ddaro:audit`. Revisit ONLY when a 2nd
   batch review of >=10 targets is concretely scheduled.
3. If/when batch recurs: `/ddaro:review --batch` (not a new command); E =
   lightweight evidence attachment; D = one disputable `ReviewContext` (merged
   with C, advisory-only, amplify-wins); F = fold into review or mangchi; G =
   severity-rollup into existing `/ddaro:check`.
4. **C: hold.** Its Goodhart + angle-correlation risk may cost more than it buys
   (pre-commit hooks already catch this project's listed modes deterministically).
   If pursued, route to ONE dedicated lens, keep the other angles blind.
