English | [한국어](README.ko.md)

# haroom_plugins

> Claude Code plugins by **haroom**.

A small collection of Claude Code plugins that target specific pain points when using Claude Code for real work: parallel sessions clobbering each other, AI reviewing its own code, design docs that AI cannot parse. Each plugin lives in its own repo; this marketplace aggregates them so one `marketplace add` covers all four.

---

## Which plugin should I pick?

| Goal | Tool |
|---|---|
| Find defects broadly, evidence-graded (SUSPECTED → SUPPORTED → REPRODUCED) | **[prism](https://github.com/minwoo-data/prism)** |
| Cross-check the same review with a second model (Claude + Codex) | **`/prism-all`** (ships inside prism) |
| Security / attacker-mindset deep probe | **`/prism <target> --redteam`** (ships inside prism; prism-devil was absorbed in 0.3.0) |
| Actually fix a file, iteratively, with cross-model review | **[mangchi](https://github.com/minwoo-data/mangchi)** |
| Check a design doc is sound (conflicts, claims vs code, failure modes) | **`/prism <doc>`** (design lens set, automatic for markdown) |
| Review how a markdown / design / spec reads, from 3 perspectives | **[triad](https://github.com/minwoo-data/triad)** |
| Run parallel Claude Code sessions without them clobbering each other | **[ddaro](https://github.com/minwoo-data/ddaro)** |

Details per plugin below.

---

## Quick Start

### 1. Add the marketplace (one time)

```
/plugin marketplace add https://github.com/minwoo-data/haroom_plugins.git
```

### 2. Install any combination

```
/plugin install ddaro
/plugin install prism
/plugin install triad
/plugin install mangchi
```

### 3. Restart Claude Code

Required after every install and update.

### 4. Update

```
/plugin update
```

One command refreshes every installed haroom plugin from this marketplace.

---

## Plugins

### [ddaro](https://github.com/minwoo-data/ddaro)

**Pain:** two Claude Code sessions editing the same repo overwrite each other's work; sessions die without saving context.

**What it does:** creates one git worktree per session (disposable folder + branch), auto-pushes every commit, writes a plain-text context snapshot to disk so a crash never loses the picture, and refuses to merge if a file on `main` would silently disappear.

**Use when:** running 2+ Claude Code sessions, doing risky refactors, keeping long-running experimental branches, or working on crash-prone machines.

### [prism](https://github.com/minwoo-data/prism)

**Pain:** a single reviewer misses what a different angle would have caught. LLM self-review has predictable blind spots.

**What it does:** five defect lenses look at the code in parallel (correctness / security / state & concurrency / integration / testability), then an Evidence pass pins every candidate to file, lines and execution path and a script checks each quote against the real file. Agreement only sets priority; evidence confirms. `--redteam` adds an attacker-mindset lens, `--reproduce` demonstrates failures with throwaway tests, `--diff` reviews a change instead of a file.

**Use when:** before merging an uncertain PR, after implementing a feature, or on anything touching auth / payments / untrusted input (`--redteam`). On a design doc it switches to design lenses (is the design sound?); triad covers how the doc reads, and `/ddaro:review` runs both.

### [triad](https://github.com/minwoo-data/triad)

**Pain:** you ship a design doc and later discover AI agents cannot parse it, the architecture ages poorly, or new users cannot figure out what it says.

**What it does:** three independent agents review one document through fixed lenses (LLM clarity / architectural longevity / end-user comprehension) across rounds until they agree or hit the cap. Consensus is written to `updated.md` without touching the original.

**Use when:** writing project instruction docs (`CLAUDE.md`, agent prompts), ADRs, public docs, or finalizing a PR justification.

### [mangchi](https://github.com/minwoo-data/mangchi)

**Pain:** one model reviewing its own code reinforces its own blind spots; the same bug survives multiple review passes.

**What it does:** alternates Claude-as-editor with Codex-as-reviewer across five rotating axes (correctness / security / readability / performance / design), catching each model's blind spots with the other until the code converges.

**Use when:** hardening production code before release, reviewing prompt-injection / security defense code, or using Claude Max tokens to drive external cross-model review without running Codex by hand.

---

## Repo layout

Each plugin is pulled in as a **git submodule** at `plugins/<name>/`, pointing at its own GitHub repo. When a plugin ships a new release, this aggregator's submodule pointer is bumped automatically every day at 06:00 UTC by [.github/workflows/daily-submodule-update.yml](./.github/workflows/daily-submodule-update.yml). `/plugin update` then picks up the new version on the next refresh.

---

## License

MIT - see [LICENSE](./LICENSE). Each plugin retains its original LICENSE file preserving provenance.

## Author

haroom - [github.com/minwoo-data](https://github.com/minwoo-data)
