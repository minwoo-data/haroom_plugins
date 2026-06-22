# prism consolidation - implementation spec (the 6 gating contracts)

Status: SUPERSEDED 2026-06-22 - DO NOT IMPLEMENT.

> **SUPERSEDED.** The `/prism --engine` merge this spec freezes was REJECTED in the
> live skills repo (`~/.claude/skills`, commit `553d7d9`) the same day, for blockers
> this spec missed: (1) a hard `context: fork` frontmatter conflict - prism forks
> Claude subagents, codex variants need the main bash context, so they cannot share
> one skill; (2) cross-skill file references are hook-blocked, so the shared-lib /
> alias-shim / "one shared core" approaches - INCLUDING this spec's Contract 3 - are
> all impossible; (3) synthesis was never duplicated, so merging only relocates
> complexity; and it would collapse 3 independent blast radii into 1. The
> duplication that actually mattered (the high-churn parsers) was instead solved by
> a single canonical `parse-findings.js` + `sync-review-parsers.sh`, keeping the 3
> skills as separate thin entrypoints. Proposal A (structured findings v1) already
> shipped (`62661dc`). The contract analysis below is kept for history only.
Date: 2026-06-22
Scope: freeze the contracts the prism-all review flagged as "underspecified to
build" before merging prism / prism-all / prism-codex into `/prism --engine`.
Builds on `prism-consolidation-design.md` + `prism-consolidation-design.REVIEW.md`.
Provenance: decisions drafted from the design doc + its review; they are proposals
for the owner to confirm, not final calls.

## Architecture decision (from the review): shared library, NOT one conditional SKILL.md

Extract a shared core - the 5 angle prompts, the Codex call convention (SANDBOX
preamble, `--output-last-message`, secret-scrub), `parse-findings.js` (ONE copy),
the v1 findings record format, and the synthesis-triage - into one versioned
location. `/prism` is a thin entrypoint that selects engine + synthesis branch.
prism-all / prism-codex become thin entrypoints (or hard-cut; Contract 3) over the
same core. prism-devil and `*.local-backup` stay separate.
[Rationale: review HIGH - one conditional SKILL.md is a monolith triad re-duplicates.]

## Contract 1 - `--verifier` x `--engine` matrix

| `--engine` | allowed `--verifier` | default | note |
|---|---|---|---|
| `claude` | claude / codex / both | **claude** | codex|both verifier needs Codex preflight |
| `codex`  | claude / codex / both | **codex**  | claude verifier = cross-model verify of codex discovery |
| `both`   | claude / codex / both | **claude** | == today's prism-all default |

Rules:
- No combo is rejected outright. Any verifier that needs an absent engine fails at
  **preflight** (Contract 4), never silently.
- `--verifier=both`: claude-verifier and codex-verifier each judge ALL singletons
  independently; a finding is CONFIRMED iff both confirm; split -> DEPENDS.
  (Today's prism-all `--verifier=both` rule, preserved.)

## Contract 2 - `both`-mode Tier3 verifier topology

- Tier1 (cross-model agreement) + Tier2 (intra-model multi-angle) auto-confirm,
  skip verification (unchanged from prism-all).
- Tier3 singletons -> the selected `--verifier` in ONE batched call, given the full
  target + all 10 discovery outputs + the singleton list.
- Singletons keyed by `(engine, angle, locus)`; identical-locus singletons from
  different angles are merged BEFORE verification (dedupe).
[Rationale: review HIGH - verifier routing for `both` was unspecified.]

## Contract 3 - entrypoint mechanism (RESOLVED 2026-06-22, empirical)

Empirical finding (claude-code-guide + existing `commands/prism.md`): `$ARGUMENTS`
substitution into a command body is DETERMINISTIC and reliable. What is NOT reliable
is a shim that re-invokes ANOTHER skill via the Skill tool - that is model-mediated
(the model may narrate instead of invoking) and has no documented alias pattern.

Decision: **do NOT build shims that re-invoke `/prism`.** Keep `/prism-all` and
`/prism-codex` as **thin entrypoints (command files) that each preset the engine in
their own body and read the ONE shared core directly** - exactly how
`commands/prism.md` already works (parse args -> read `skills/prism/SKILL.md` ->
execute). Each entrypoint: sets `engine` (both / codex), `$ARGUMENTS` carries
target+flags, reads the shared core, executes. No skill-to-skill delegation, no
duplicated prompts/parser/synthesis.

Net: triggers `/prism-all` / `/prism-codex` keep working (no muscle-memory break),
the model-optional-delegation failure mode is avoided, and drift is zero (all read
one core). This supersedes BOTH "re-invoke shim" and "hard-cut".
[Rationale: review HIGH/all-5 + empirical arg-forwarding check.]

## Contract 4 - preflight + partial-failure

- **Preflight per selected engine.** `--engine=codex|both` (or a codex verifier)
  requires `codex` on PATH at the required version; absent -> fail-fast with a clear
  message, **no silent Claude fallback**. `--engine=claude` needs no codex.
- **Mid-run engine failure** (crash / timeout / unparsable): run does NOT fail
  whole. Default = **degrade**: drop the dead engine, emit an explicit
  `ENGINE-DEGRADED` finding; in `both` mode collapse to single-engine tiering (no
  cross-model Tier1). `--strict` flips to fail-closed. Engine health in report header.
- **Single-angle parser failure** (Contract 6) = one `ANGLE-DEGRADED` finding;
  never whole-run fail, never silent-drop.
[Rationale: review HIGH robustness.]

## Contract 5 - default engine + flag precedence

- Bare `/prism <target>` = `--engine=claude` (backward-compatible). **Closes the
  design's Open Q1** - do not leave it reopened.
- Precedence: explicit `--engine=X` wins; `--all` / `--codex` are sugar that SET
  `--engine` and **error** if combined with a conflicting explicit `--engine`
  (e.g. `/prism --all --engine=codex`).
- `--quick` (skip Pass-2) and `--adversarial` (refute-before-reject) apply to ALL
  engines uniformly; in `both` they apply to both discovery sets and the verifier.
  Defined once in the shared core.
[Rationale: review MED - unstable contract + unspecified `--quick`/`--adversarial`.]

## Contract 6 - parity proof (gates deletion of old skills)

Parity = **structural / contract parity**, not output equality (LLM output is
nondeterministic).

Deterministic fixtures (captured outputs fed to parser + synthesis, no live model):
- claude / codex / both: same set of agents launched, same v1 schema, same tier
  assignments.
- edge fixtures: empty findings; malformed / fence-less output; duplicate findings
  (same locus); singleton -> verifier promotion; one-engine-down (degrade path).
- `parse-findings.js` self-test stays green; ONE copy after consolidation.

Acceptance: old `prism-all` and new `/prism --all` on the fixed fixtures produce
identical parsed findings + tier assignments. Keep old skill dirs until these
artifacts are reviewed; only then delete / alias.
[Rationale: review HIGH / all-5 angles.]

## Cutover order

1. Extract shared core (+ `parse-findings.js` to 1 copy) and author `/prism` with
   the engine fork.
2. Build Contract-6 fixtures + parser self-test; prove parity.
3. Add thin shims (Contract 3) and test argv forwarding; else hard-cut.
4. Delete / alias old prism-all + prism-codex. prism-devil + backups untouched.
5. Phase 2: same collapse for triad / triad-all / triad-codex.

## Non-goals / stays separate

prism-devil (different structure), `*.local-backup`, triad (Phase 2 follow-up), and
the 5 angles / v1 record format / `parse-findings.js` behavior (only copy-count 3->1).

## Folds in AFTER consolidation (not part of it)

- Proposal A (structured schema output) - implement once on the shared core.
- right-size dial (stake-gated angle/engine selection) - into `--engine` / `--quick`
  semantics. See `token-caching-findings.md`.

## Owner decisions (CONFIRMED 2026-06-22)

1. **default engine = `claude`** - bare `/prism` stays identical to today;
   `--all` / `--codex` escalate. (Contract 5.)
2. **degrade-by-default + a loud `ENGINE-DEGRADED` marker** on mid-run engine
   failure; `--strict` opts into fail-closed. (Contract 4.)

Contract 3 ("shims vs hard-cut") resolved empirically above: thin entrypoints over
one shared core. All 6 contracts + both owner decisions are now frozen - ready to
implement (cutover order above).
