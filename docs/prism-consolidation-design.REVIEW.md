# PRISM-ALL REVIEW (synthesized) - prism-consolidation-design.md - 2026-06-22

Provenance: SYNTHESIZED from the completed prism-all DISCOVERY outputs
(`docs/prism-all/prism-consolidation/o.*.json` in the seoul worktree). That run
stopped after discovery and never produced its own REPORT.md, so this is a
hand-synthesis, not the tool's official report.

Caveat: only one set of 5 angle outputs was present (appears single-engine or
merged), so there is NO cross-model agreement tier here - confidence is
"prism-level", not the 10-reviewer "prism-all-level" of the review-tooling report.

> Bottom line: unlike review-tooling (where the core idea mostly failed "do
> nothing"), the consolidation idea is NOT challenged - the 3-way duplication pain
> is real and no angle says "don't." The convergence is on HOW: (1) extract a
> shared runner/library with thin `/prism --engine` entrypoints rather than one
> conditional SKILL.md; (2) nail ~6 underspecified contracts BEFORE building;
> (3) prove parity with fixtures before deleting any old skill.

## HIGH (multi-angle agreement)

- **[conflict+improvement+code-review+robustness] `--verifier` x `--engine` matrix is undefined.**
  `--verifier=claude|codex|both` is exposed but defaults, invalid combos
  (`--engine=claude --verifier=codex`, `--engine=codex --verifier=both`), and
  dual-verifier disagreement resolution are unspecified. -> Define the full matrix
  (defaults per engine, reject-or-coerce on invalid, dual-verifier reconciliation)
  before implementation.

- **[conflict+code-review] `both`-mode verifier topology unspecified.**
  "cross-model promotion + Verifier" does not say which verifier handles Tier3
  singletons (same-engine / opposite-engine / both) or how verifier outputs merge.
  Affects parity with today's prism-all. -> Specify Tier3 routing + merge/dedupe.

- **[all 5 angles] "one-line shim" backward-compat aliases are underspecified and risky.**
  Assumes skill dispatch can reliably forward argv/options to `/prism --all`; if it
  cannot, `/prism-all` / `/prism-codex` break SILENTLY. -> Specify and TEST the shim
  mechanism as first-class cases; shims must be pure dispatch (zero copied
  prompts/parser/docs) or the "ZERO drift" claim is false.

- **[all 5 angles] parity proof is too weak to gate deletion.**
  "Run one real --all sample" cannot prove parity (LLM output is nondeterministic).
  -> Define structural/contract parity: deterministic fixtures + golden normalized
  tier results (claude / codex / both, empty findings, malformed fences, duplicates,
  singleton promotion). Keep old skills until parity artifacts are reviewed.

- **[improvement+devil] prefer a shared runner/library over one conditional SKILL.md (answers the doc's Open Q3).**
  One conditional SKILL.md is a new monolith that triad will duplicate again. ->
  Put shared prompts / parser / Codex bridge / synthesis in a versioned common
  runner; make `/prism`, `/prism-all`, `/prism-codex` thin entrypoints over it.

- **[robustness] engine preflight + partial-failure semantics missing.**
  `--engine=codex` with no Codex CLI = opaque runtime failure (add fail-fast
  preflight, no silent Claude fallback). `both` has no policy when one engine
  crashes / times-out / returns unparsable -> define fail-closed / degrade-to-single
  / `--allow-partial`, and record engine health in synthesis.

## MEDIUM

- [conflict+code-review] `--quick` / `--adversarial` are in the signature but
  unspecified per engine; may conflict with parity claims. Define, or drop from Phase 1.
- [code-review] default-engine is stated `claude` in section 1 but reopened in Open
  Q1 -> unstable CLI contract. Decide and record one rule.
- [code-review] `--all` / `--codex` convenience aliases are not normalized against
  `--engine` (e.g. `/prism --all --engine=codex`). Define precedence or reject conflicts.
- [conflict] factual nit: section 0 ("duplicated x3") vs section 2 table
  ("parse-findings.js = 2 copies"). Reconcile the baseline.
- [improvement] if centralizing anyway, keep the Codex bridge behind a small
  runner/adapter rather than inline bash (optional, declarative SKILL.md).

## LOW

- [devil+robustness] "standalone prism-codex install loss is marginal" is an
  author-self-bias dismissal (author owns the tools) -> keep wrappers or gather real
  usage evidence before removing standalone distribution.
- [devil] post-merge `prism-devil` naming stays confusing -> document it as a
  separate deep-probe tool, not an engine mode.
- [improvement] staged rollout with rollback criteria is unstated -> keep old skills
  as shims for one release, deprecate in docs, remove only after fixtures + one real
  review per mode.

## Recommended action order (synthesized)

1. **Decide the contracts FIRST (gating):** verifier x engine matrix; `both`
   verifier topology; alias-shim mechanism (+ tests); partial-failure policy;
   default-engine rule; parity-fixture definition.
2. **Extract a shared runner/library;** make prism / prism-all / prism-codex thin
   entrypoints. (`prism-devil` and `*.local-backup` stay separate.)
3. **Prove `--all` parity against fixtures;** keep old skills until the artifacts pass.
4. **Then fold in:** proposal A (structured schema output) once on the shared
   library, and the stake-gated right-size dial into the `--engine` / `--quick`
   semantics (see `token-caching-findings.md`).

## Still open (doc's own questions, narrowed by the above)

default-engine rule; shim vs hard-cut; whether one-SKILL.md is actually simpler
(this review says no - use a shared library); `prism-devil` as `--mode=devil`
(review: keep separate); real standalone-codex use case; smallest-risk cutover +
how to prove parity before deleting prism-all.
