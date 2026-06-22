# Prism Family Consolidation - Design Proposal

Status: SUPERSEDED 2026-06-22 - the merge proposed here was REJECTED. DO NOT BUILD.

> **SUPERSEDED.** This `/prism --engine` merge was rejected in the live skills repo
> (`~/.claude/skills`, commit `553d7d9`) the same day: a hard `context: fork`
> conflict (prism forks Claude subagents; codex variants need the main bash
> context - cannot share one skill), hook-blocked cross-skill file references (so
> shared-lib / alias-shim / one-shared-core are ALL impossible), and a synthesis
> that was never duplicated. The real duplication (the high-churn parsers) was
> instead single-sourced via canonical `parse-findings.js` + `sync-review-parsers.sh`
> (`--check` drift gate), keeping the 3 skills separate. Proposal A (structured
> findings v1) already shipped (`62661dc`). See `prism-consolidation-spec.md` (also
> superseded) and `prism-consolidation-design.REVIEW.md`. Kept for history.
Author: Minwoo Park
Date: 2026-06-22
Scope: collapse the engine-variant skills `prism` / `prism-all` / `prism-codex`
into ONE skill `/prism` with an `--engine` option. Same idea later for triad.
NOT a change to the `expense-us` product. Attack before building.

---

## 0. Context (self-contained - reviewers may not have the repos)

**The prism family today = 4 SEPARATE skills** (separate dirs, separate SKILL.md,
separate triggers), all under `~/.claude/skills/`:

- `/prism`        - 5 Claude subagents review one target in parallel across 5
                    angles (Conflict / Improvement / Devil / CodeReview /
                    Robustness), then a Claude Verifier cross-checks singletons.
- `/prism-all`    - the SAME 5 angles run on BOTH Claude (5) and Codex CLI (5) =
                    10 discovery calls; cross-model agreement = top tier.
- `/prism-codex`  - the SAME 5 angles, all on Codex CLI (5) + a Codex Verifier.
- `/prism-devil`  - a SINGLE-agent deep adversarial probe. DIFFERENT STRUCTURE
                    (not a 5-angle ensemble).

**The key fact:** prism / prism-all / prism-codex differ ONLY by ENGINE. The 5
angle prompts, the Codex call convention, the findings record format (v1), the
deterministic parser (`parse-findings.js`), and the synthesis logic are
otherwise the SAME - and are physically DUPLICATED across the 3 skills.

**The cost of that duplication (observed this session):** every change - the v1
record format, the `--output-last-message` clean-capture wiring, the
`parse-findings.js` parser, and TWO rounds of parser bug-fixes - had to be
edited into ALL of prism-all + prism-codex (and triad-all + triad-codex)
identically. Four-file edits for one logical change, every time. The skills'
own docs even warn about "독립 사본 간 drift" (drift between independent copies).

**The concept:** prism-all is the "both-engines OPTION of prism"; prism-codex is
the "Codex-engine OPTION." They are MODES of one review concept, not different
tools. The implementation (3 independent skills) does not match the concept (one
skill, an engine option).

---

## 1. Proposal

**Collapse `prism` + `prism-all` + `prism-codex` into ONE `/prism` skill** whose
engine is an option. `prism-devil` stays separate (different structure).

```
/prism <target> [--engine=claude|codex|both] [--quick] [--adversarial] [--verifier=claude|codex|both]
  # convenience aliases: --all == --engine=both ; --codex == --engine=codex
  # default engine = claude  (so bare `/prism <target>` == today's prism, backward compatible)
```

Engine dispatch:
- `claude` : 5 Claude `Agent` angles + Claude Verifier.            (== today's /prism)
- `codex`  : 5 Codex CLI angles (--output-last-message + parser) + Codex Verifier. (== /prism-codex)
- `both`   : 5 Claude + 5 Codex = 10, cross-model promotion + Verifier.            (== /prism-all)

**Shared ONCE (the whole point):** the 5 angle prompts, the Codex call convention
(SANDBOX preamble, `--output-last-message`, secret-scrub, `parse-findings.js`),
the v1 findings record format, and `parse-findings.js` itself (1 copy, not 3).

**Synthesis branches on engine:**
- `claude` / `codex` : tiers = multi-angle agreement (2+ angles) -> Verifier on
  singletons. (single-engine; no cross-model tier)
- `both` : Tier1 cross-model agreement, Tier2 intra-model multi-angle, Tier3
  singleton -> Verifier. (the current prism-all triage)

So the SKILL.md carries the shared material once and a synthesis section that
forks by engine. Net: ~1/3 the total surface, ZERO cross-copy drift.

---

## 2. What changes / what does not

| Item | Before | After |
|---|---|---|
| Skills | prism, prism-all, prism-codex (3) | prism (1) |
| `parse-findings.js` | 2 identical copies (all + codex) | 1 |
| Angle prompts / Codex convention / v1 format | duplicated x3 | once |
| Trigger | `/prism-all`, `/prism-codex` | `/prism --all`, `/prism --codex` |
| `prism-devil` | separate | UNCHANGED (separate; different structure) |
| `*.local-backup` | separate | UNCHANGED |
| triad family | separate | follow-up Phase 2 (same pattern) |

Backward-compat option: keep thin alias skills `/prism-all` -> `/prism --all`
(one-line shims) so existing muscle memory / docs keep working, OR hard-cut and
update references.

---

## 3. Migration plan

- **Phase 1 (prism):** author the merged `/prism` SKILL.md (engine fork) + move
  `parse-findings.js` to the single prism skill. Keep its self-test green. Run a
  real `--all` review on a sample to confirm parity with today's prism-all.
- **Phase 2 (triad):** same collapse for triad / triad-all / triad-codex into
  `/triad --engine`.
- Delete (or alias) the absorbed skill dirs. Keep prism-devil + backups.
- The skills now live in a git repo (`~/.claude/skills`), so this is a tracked,
  revertible refactor.

---

## 4. Costs / risks (decide with eyes open)

1. **Reverses the deliberate "independent plugin" design.** Each current SKILL.md
   asserts "하나만 설치해도 동작" (works if only one is installed). After merge you
   cannot install `prism-codex` standalone without the rest. Is "Codex-only
   install" a real need? Likely marginal (one skill is small).
2. **A refactor of CORE review tooling.** Merge bugs would degrade every review.
   Mitigation: `parse-findings.js` self-test + a real `--all` parity run before
   cutover.
3. **One larger SKILL.md with conditional (per-engine) sections** - more branching
   in one file, but far less total text and no cross-file drift.
4. **Trigger UX change** (`/prism-all` -> `/prism --all`). Alias shims soften it.
5. **Codex bridge stays bash** regardless - `--engine` just selects whether the
   bash Codex stage runs. (Not a regression; same as today.)

---

## 5. Open questions (press here)

1. Default engine = `claude` (backward-compatible) - agree, or should bare
   `/prism` stay undefined and require an explicit `--engine`?
2. Keep `/prism-all` / `/prism-codex` as alias shims, or hard-cut?
3. Is folding three modes into one SKILL.md with engine-conditional synthesis
   actually SIMPLER than 3 skills, or does the conditional complexity offset the
   dedup win? (the crux - attack it)
4. Is excluding `prism-devil` correct, or should it be `--mode=devil` for symmetry?
5. Any real distribution use case lost by removing standalone `prism-codex`?
6. Smallest-risk cutover order, and how to prove `--all` parity with today's
   prism-all before deleting it.

## 6. Non-goals

- Touching `prism-devil`'s structure or the `*.local-backup` skills.
- Changing the 5 angles, the v1 record format, or `parse-findings.js` behavior
  (only its NUMBER OF COPIES: 2 -> 1).
- Consolidating triad in Phase 1 (separate, identical-pattern follow-up).
