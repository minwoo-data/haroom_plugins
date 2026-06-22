# prism consolidation - implementation spec (the 6 gating contracts)

Status: SPEC - proposed contract resolutions (confirm / adjust before building)
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

## Contract 3 - alias-shim mechanism (the riskiest unknown)

Decision: **thin command-file shims**, not SKILL.md copies. `/prism-all` and
`/prism-codex` are one-line command files that invoke the prism skill with the
engine preset and forward `$ARGUMENTS` verbatim. ZERO copied prompts/parser/synthesis.

Acceptance (MUST pass before deleting anything): `/prism-all <t> --quick` and
`/prism-codex <t> --adversarial` produce identical dispatch (same engine, same
flags) as `/prism <t> --all --quick` / `--codex --adversarial`. **If argv
forwarding proves unreliable in Claude Code, fall back to HARD-CUT + a deprecation
note - never silent breakage.**
[Rationale: review HIGH / all-5 angles - shims assumed to forward argv reliably.]

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

## Decisions that still want owner sign-off

1. default engine = `claude` (vs require explicit `--engine`).
2. shims vs hard-cut (depends on whether Claude Code reliably forwards `$ARGUMENTS`
   from one command to another - the one empirical unknown to test first).
3. degrade-by-default vs strict-by-default on mid-run engine failure.
