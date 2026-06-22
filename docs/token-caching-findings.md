# Findings: Prompt caching does NOT cut prism/mangchi token cost (as built)

Status: VERIFIED (doc-grounded), 2026-06-22
Scope: prism / prism-all / mangchi token efficiency
Author: Minwoo Park

## TL;DR

Prompt caching was floated as a "lossless ~75% token cut" for prism's fan-out.
**Verification overturns that for prism.** Caching is structurally unavailable to
prism/prism-all as built, and only partially + conditionally available to mangchi.
Do NOT build a prism caching feature. The real prism lever is right-sizing the
review (lossy, stake-gated). The one remaining lossless caching lever is a mangchi
prompt re-order, which is worth a small experiment but capped by file mutation.
(2026-06-22 update below: adversarial review confirms the batch-review token lever
is incremental / changed-targets scoping, NOT a shared-facts memo; and right-size
lands in a consolidated `/prism --engine` dispatch.)

## Why this was checked

prism re-sends the full target file verbatim to each of 5 (prism) or 10 (prism-all)
agents plus the verifier -> the target file is ~75% of total tokens (modeled).
mangchi re-sends the full file to Codex on every round. The obvious question:
"can prompt caching make the repeated identical content near-free?" Answer was
assumed yes; verification says mostly no.

## Method

- Anthropic prompt-caching reference (claude-api skill -> `shared/prompt-caching.md`),
  read directly, not from memory.
- Claude Code subagent caching behavior (claude-code-guide).
- OpenAI / Codex CLI caching behavior (OpenAI prompt-caching guide).
- Cross-checked against the actual skill sources: `prism/SKILL.md`,
  `prism-all/SKILL.md`, `mangchi/SKILL.md`.

## Finding 1 - prism / prism-all: caching does NOT apply. Three independent blockers.

1. **Parallel dispatch loses the cache race.**
   `prompt-caching.md` (Concurrent-request timing): a cache entry becomes readable
   only after the first response begins streaming; N parallel requests with an
   identical prefix all pay full price. prism launches all 5-10 agents in a single
   message (`prism/SKILL.md` Pass 1; `prism-all/SKILL.md` "10 byeongnyeol balsa").
   No entry exists yet -> every agent pays full input price.

2. **Prefix mismatch (angle-first ordering).**
   Caching is a strict prefix match (stable content first, volatile last; render
   order tools -> system -> messages). Each prism agent's prompt = [unique per-angle
   instruction] + [shared target]. The shared target is therefore NOT a common
   prefix across agents, so even a serialized fan-out could not share a cache entry
   for the target.

3. **Skill code cannot set the breakpoint.**
   `cache_control` on subagent prompts is harness-controlled; a skill cannot place
   it. Whether Claude Code applies any caching to subagent dispatch is undocumented.

Plus: Opus 4.8 minimum cacheable prefix is 4096 tokens; smaller targets silently
never cache.

Conclusion: prism's ~75% target-duplication cost is NOT recoverable via caching.

## Finding 2 - mangchi (Codex/OpenAI): partial + conditional.

OpenAI caching is automatic, prefix-based, and persists across separate process
invocations within ~5-10 min (per-org, server-side). mangchi is serial, so the
parallel-race blocker does not apply. But it is currently defeated by:

- **Ordering:** prompt is assembled rubric-first (axis changes each round) then file
  (`mangchi/SKILL.md` Phase 2). A per-round-varying prefix breaks the match before
  the file is reached.
- **File mutation:** the working file (`updated.<ext>`) is edited each round, so even
  with file-first ordering the cacheable prefix only extends to the first changed
  line.
- **Unverified key:** whether the Codex CLI emits a stable `prompt_cache_key` and
  keeps the prefix byte-identical across `codex exec` runs is undocumented.

The one lossless lever worth trying: **re-order the mangchi review prompt to
file-first, rubric/digest-after.** Potential cross-round Codex cache hit on the
unedited file prefix. Low risk, zero quality loss. Effect bounded by the first edit
point each round; needs empirical confirmation of `prompt_cache_key` + `cached_tokens`.

## Decision

- prism/prism-all: **do not build a caching feature.** Reduce cost via right-sizing
  the review (fewer angles / auto --quick / prism-all only on deliberate escalation),
  gated on target risk. This is a deliberate quality<->cost trade, not lossless -
  pull it only where stakes are low.
- mangchi: keep Codex-token tracking; add Claude-side token tracking (instrumentation,
  lossless). Optionally run the file-first re-order experiment above.

## Update 2026-06-22 - adversarial-review verdicts (review-tooling + prism-consolidation)

Two related design docs were drafted and prism-all-reviewed (currently untracked in
expense-us-seoul): `review-tooling-improvements.md` (7 proposals A-G) and
`prism-consolidation-design.md` (collapse prism / prism-all / prism-codex into one
`/prism --engine`). What their reviews change for token efficiency:

### Batch-review token cost: incremental scoping, NOT a shared-facts memo
Single-review caching is dead (Findings 1-2 above). For BATCH review over N targets,
the review-tooling verdict (10 reviewers, cross-model) names the real lever:
**incremental / changed-targets scoping** - the "biggest omitted cost lever" - re-run
only changed targets + their blast radius, carry-forward PASS otherwise; plus a cost
governor and one global concurrency semaphore (N x 10 = up to ~300 concurrent calls).

The tempting alternative - a shared-facts "do not re-discover" memo (proposal D) - was
rated CRITICAL ("cut or invert"): injecting shared context into the discovery angles
CORRELATES the reviewers, which destroys prism-all's independent-ensemble value, and
one wrong "fact" suppresses real findings across every target. So **do not use
context-injection to save batch tokens** - any shared context belongs in a separate
synthesis/gate step, never in discovery prompts.

### right-size (the prism lever) is validated, and lands in the unified dispatch
The same review independently flagged cost-scoping as the missing lever, which
validates the right-size direction here. The prism-consolidation review further
recommends extracting a shared runner/library with thin `/prism --engine` entrypoints
(rather than one conditional SKILL.md). Implication: build right-size ONCE into that
unified dispatch / `--engine` + `--quick` flag semantics, not into three skills.

### Net build order
1. prism-consolidation as a shared library + thin entrypoints - nail the
   verifier/engine matrix, the alias-shim mechanism, partial-failure semantics, and
   fixture-based parity proof first. This is the substrate.
2. Proposal A (structured schema output) - the one clear review-tooling survivor -
   implemented once on the shared library.
3. right-size (stake-gated angle/engine selection) folded into the unified dispatch.
4. Defer B/D/E/F/G; batch cost = incremental scoping if/when batch review recurs.

## Open empirical questions (not yet run)

- Does Claude Code apply ANY prompt caching to Agent-tool subagents? (moot for prism
  given blockers 1-2, but useful to know.)
- Does the Codex CLI set a stable `prompt_cache_key` per `codex exec`? Measure
  `cached_tokens` across two consecutive mangchi rounds with file-first ordering.

## Sources

- Anthropic `shared/prompt-caching.md` - prefix-match invariant, concurrent-request
  timing, minimum cacheable tokens, TTL/economics, verification via
  `usage.cache_read_input_tokens`.
- OpenAI prompt-caching guide - automatic, prefix-based, 1024-token minimum,
  cross-invocation persistence, order sensitivity.
- Skill sources: `plugins/prism/skills/prism/SKILL.md`,
  `plugins/prism/skills/prism-all/SKILL.md`,
  `plugins/mangchi/skills/mangchi/SKILL.md`.
