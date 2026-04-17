# ConTree Skill for Claude Code

A Claude Code [Skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches Claude how to use [ConTree](https://contree.dev) — Nebius's sandboxed container execution platform with Git-like branching — idiomatically and efficiently.

> **Install:** `claude skill install opencolin/contree-skill` *(once published)* or clone and drop into `~/.claude/skills/contree/`.

---

# Product Requirements Document

**Product:** ConTree Skill for Claude Code
**Author:** Colin (@opencolin)
**Status:** v0.1 — Alpha, seeking feedback from Nebius ConTree team
**Last updated:** 2026-04-16

## 1. TL;DR

Claude Code users who connect the ConTree MCP server get 17 new tools and 10 prompts — but without guidance, Claude tends to misuse them in ways that waste compute and frustrate users (forgetting `disposable=false`, spawning VMs for `ls`, re-syncing unchanged files, etc.). This skill is a ~400-line instruction set that progressively discloses ConTree's mental model, workflow patterns, and gotchas to Claude *only when the user's intent matches ConTree's use cases*. It covers both the MCP server and the Python SDK.

**Goal:** Make ConTree the default Claude-native answer to "I need to run this safely / in parallel / with rollback" — without the user having to hand-hold Claude through every invocation.

## 2. The Problem

### 2.1 What Nebius ships today

Nebius ships ConTree as three things:
1. **Managed service** — microVM-isolated container runtime with git-like branching
2. **MCP server** (`contree-mcp`) — 17 tools + 10 prompts + 7 guides, bundled with a system prompt embedded in `app.py` that teaches the CHECK-PREPARE-EXECUTE pattern
3. **Python SDK** (`contree-sdk`) — sync and async clients with image/session abstractions

The MCP server's embedded system prompt is excellent, but it only applies *inside that MCP session's model context*. And in practice, Claude Code users often:
- Don't read long system prompts carefully on every turn
- Confuse MCP tool patterns with generic shell/docker habits
- Miss the subtle distinctions (disposable vs persisted, sessions vs images, mode="any" cancelling siblings)

### 2.2 The concrete failure modes we saw

Running Claude against the raw MCP tools with no skill, we observed:

| Failure mode | Frequency | Cost |
|---|---|---|
| `pip install` without `disposable=false` → packages vanish | Very common | 1 wasted VM + confused re-runs |
| `contree_run "ls /app"` instead of `contree_list_files` | Very common | 1 wasted VM per inspection |
| Re-running `contree_rsync` before every `contree_run` | Common | Wasted upload bandwidth |
| `contree_import_image` without `contree_list_images` check | Common | Duplicate imports, wasted import VM |
| Chaining `apt update && apt install && pip install && run` in one command | Common | No rollback on failure, whole thing re-runs |
| Mixing up `files` param direction (UUID vs path) | Occasional | Hard-to-debug "file not found" |
| `wait_operations mode="any"` silently cancelling siblings | Rare but expensive | Lost work user didn't know was lost |

These aren't Claude being dumb — they're the *default* behaviors of a model that's been trained on Docker/shell idioms and doesn't know ConTree's specific model.

### 2.3 Why a Skill is the right abstraction

Claude Code Skills are loaded **progressively** based on the user's intent:

- **Level 1 (always loaded):** ~100-word description in system context
- **Level 2 (loaded on match):** Full SKILL.md body (~500 lines)
- **Level 3 (on-demand):** Bundled scripts/references the skill links to

A well-tuned description triggers *only* when the user is actually going to use ConTree — so the full workflow guidance doesn't pollute unrelated conversations. This is the right fit for tool-specific expertise.

## 3. Users & Jobs-to-be-Done

### 3.1 Primary persona: SWE-bench / agent researcher

> "I'm iterating on a coding agent. I need 7,000 preloaded environments, the ability to branch from the same checkpoint, and per-run metrics. My agent does MCTS over patches and I need sibling execution to be cheap."

**JTBD:** Run thousands of parallel, branchable, VM-isolated executions from a Claude-driven agent loop.

### 3.2 Secondary persona: Claude Code power user

> "I asked Claude to 'try a few approaches to fix this bug in parallel' and it made a mess of my working directory. I want it to do that in a sandbox."

**JTBD:** Get Claude to explore multiple solution paths without touching the host filesystem.

### 3.3 Tertiary persona: Agent builder

> "I'm building an agent that generates untrusted code. I need hardware-level isolation and rollback. I want to write Python, not YAML."

**JTBD:** Use the ConTree SDK idiomatically from Python, with clear patterns for sessions vs branching.

## 4. Product Scope

### 4.1 In scope

✅ **MCP guidance:** all 17 tools, the CHECK-PREPARE-EXECUTE pattern, tool-cost reference table, parallel execution, branching
✅ **SDK guidance:** sync/async clients, `images.use()` vs `.oci()` vs `.import_from()`, sessions vs images distinction, file handling (list/dict/`UploadFileSpec`), subprocess-like `popen`
✅ **Anti-pattern coverage:** all 7 failure modes from §2.2, plus mode="any" cancellation
✅ **Setup instructions:** for users who don't have ConTree configured yet
✅ **Mental model upfront:** the Git analogy is front-loaded so Claude's instincts map correctly

### 4.2 Out of scope (for v0.1)

❌ REST API guidance — MCP and SDK cover 95% of Claude Code usage
❌ SWE-bench integration (`mini-swe-agent` `ContreeEnvironment`) — separate follow-up skill
❌ Workflow-specific bundled scripts — save for v0.2 once we see repeated patterns in evals
❌ Registry auth UX polish — covered briefly, but a deep dive is separate
❌ Image lineage / rollback tree visualization

### 4.3 Explicit non-goals

- **Not a replacement for the MCP server's embedded system prompt.** This skill layers on top of that, providing persistent guidance across model turns.
- **Not a ConTree marketing page.** The skill is for Claude, not for humans reading docs. The README (this file) is the marketing page.

## 5. Solution Design

### 5.1 Architecture

```
User message
   │
   ▼
┌─────────────────────────────────────┐
│ Claude Code harness                 │
│                                     │
│  Level 1: Skill description         │  ← "Use when user mentions
│  (always in context)                │     ConTree OR sandboxed
│                                     │     execution OR branching..."
│         │                           │
│         ▼                           │
│  Intent match?  ─── no ───→ skip    │
│         │ yes                       │
│         ▼                           │
│  Level 2: SKILL.md body loads       │  ← CHECK-PREPARE-EXECUTE
│  (mental model + patterns +         │     pattern, gotchas, SDK
│   anti-patterns)                    │     section
│         │                           │
│         ▼                           │
│  Claude calls contree_* tools       │  ← Uses MCP server directly,
│  with correct defaults              │     with skill-guided defaults
└─────────────────────────────────────┘
```

### 5.2 Key design decisions

**Decision 1: Single SKILL.md, not domain-split references.**
ConTree's surface area is small enough (17 tools, ~10 concepts) that splitting into `references/mcp.md` and `references/sdk.md` would cost more context-switches than it saves. Revisit at v0.2 if the file exceeds 500 lines.

**Decision 2: Description-first triggering.**
The skill description uses a "pushy" style (per skill-creator best practices) and lists specific MCP tool names. This catches both (a) users who explicitly mention ConTree and (b) users whose intent matches (`"run this in a sandbox"`, `"try N approaches in parallel"`) even without the brand name.

**Decision 3: Git analogy front-loaded.**
Every ConTree concept maps cleanly to Git (image = commit, branch = branch, tag = tag, disposable = detached HEAD). Claude already has strong priors about Git, so we lean on them instead of inventing new vocabulary.

**Decision 4: Tool-cost table as a memory aid.**
Claude's biggest non-obvious failure was spawning VMs for free operations. A compact table near the end of the MCP section makes this visually memorable.

**Decision 5: Cover SDK even though most users will use MCP.**
The power-user persona (agent builder) will write Python directly. The SDK section adds ~100 lines but unlocks a separate high-value use case.

### 5.3 How the skill addresses each failure mode

| Failure mode (from §2.2) | Skill intervention |
|---|---|
| Forgetting `disposable=false` | Called out in PREPARE step + top of Common Mistakes |
| `run` for file inspection | "Inspecting Images (Free, No VM)" section + cost table |
| Re-syncing unchanged files | "Reuse it" emphasized in rsync section |
| Re-importing without checking | CHECK step is step 1 of the core pattern |
| Chaining commands | Explicit anti-pattern in Common Mistakes with reasoning |
| `files` param direction | Highlighted in both MCP and SDK sections |
| `mode="any"` cancellation | Explicit callout in wait_operations section |

## 6. Success Metrics

### 6.1 v0.1 Alpha (this release)

- [ ] **Qualitative:** Three test prompts (see `evals/`) produce outputs where Claude follows CHECK-PREPARE-EXECUTE without prompting
- [ ] **Quantitative:** Skill description triggers correctly on ≥90% of should-trigger queries and ≤10% of should-not-trigger queries (per `run_loop.py` optimizer)
- [ ] **Feedback:** Nebius ConTree team reviews and agrees the skill faithfully represents product intent

### 6.2 v0.2 (next)

- **VM-spawn rate reduction:** Compared to baseline (no skill), measure how many `contree_run` calls are spawned per task on a fixed test set. Target: **30% reduction** by redirecting to `list_files`/`read_file`.
- **`disposable=false` correctness:** On tasks that install packages, measure how often the first install run correctly sets `disposable=false`. Target: **95%+** (baseline is <50%).
- **Branching adoption:** On "try N approaches" prompts, measure how often Claude uses parallel `wait=false` + `wait_operations`. Target: **80%+**.

### 6.3 Long-term (v1.0)

- Distribution: listed in an official Claude Code skill registry, ≥1k installs
- Co-marketing: Nebius links this skill from their Claude Code integration docs
- Telemetry partnership: (with user consent) anonymized metrics on which skill sections get loaded most, feeding back into both skill improvements and ConTree product decisions

## 7. Dependencies & Risks

### 7.1 Dependencies

- **ConTree MCP server** (`contree-mcp` on PyPI, Apache 2.0) — the skill assumes `contree_*` tool names. If Nebius renames tools, the skill breaks.
- **ConTree SDK** (`contree-sdk` on PyPI, Apache 2.0, currently `0.3.0.dev1`) — the SDK is pre-alpha. API changes will require skill updates.
- **Claude Code Skills** feature — production in Claude Code 1.x+

### 7.2 Risks & mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| MCP tool signatures change (e.g., `shell` default flips) | Medium | Version the skill; ship an updater when ConTree does breaking releases |
| SDK goes through breaking API churn (it's pre-alpha) | High | Clearly version-pin the SDK section; consider splitting to a reference file that's easy to swap |
| Skill over-triggers on non-ConTree sandboxing (Docker, gVisor) | Medium | Description tuning via `run_loop.py` trigger evals |
| Skill under-triggers when user's intent is obvious | Medium | Same — optimize description against both positive and negative eval queries |
| Nebius changes tag prefix convention | Low | The convention is advisory, not enforced. Easy to update. |

## 8. Rollout Plan

### Phase 0: Internal alpha *(now)*
- [x] v0.1 draft pushed to [github.com/opencolin/contree-skill](https://github.com/opencolin/contree-skill)
- [x] MCP + SDK integration verified against live ConTree service
- [ ] 3 hand-crafted test prompts run against skill vs no-skill baseline
- [ ] Eval viewer review cycle with author

### Phase 1: Nebius review *(this week)*
- [ ] PRD (this doc) sent to ConTree PM
- [ ] Discussion: is framing accurate? Is there product roadmap this should align with?
- [ ] Any failure modes we missed from their internal testing?

### Phase 2: Public alpha *(week 2)*
- [ ] Description optimized with `run_loop.py` against 20-query trigger eval
- [ ] Expand to 10+ test prompts covering edge cases
- [ ] Package as `.skill` file, publish release
- [ ] Nebius links from their docs: "Using ConTree with Claude Code"

### Phase 3: GA *(month 2)*
- [ ] Telemetry on skill usage (opt-in)
- [ ] Bundled scripts for high-frequency patterns uncovered in evals
- [ ] SWE-bench/mini-swe-agent companion skill

## 9. Open Questions for Nebius

1. **Tool naming stability:** Is the `contree_*` prefix guaranteed? Any plans to change tool names or signatures pre-GA?
2. **SDK stability:** When does `contree-sdk` leave pre-alpha? Should the skill pin to a specific SDK version range?
3. **Tag convention:** Is `{scope}/{purpose}/{base}:{version}` Nebius-endorsed or just the MCP server's house style? Would you prefer a different convention?
4. **Anonymous access:** Will `i_accept_that_anonymous_access_might_be_rate_limited` stay? Feels like a v0 UX wart we should help users avoid.
5. **Registry auth flow:** The browser-popup flow (`registry_token_obtain`) is creative but brittle in headless environments. Is a `CONTREE_REGISTRY_*` env-var path coming?
6. **MCP prompts vs skill:** The MCP server already ships 10 prompts. Should the skill steer users *toward* those (`use the 'prepare-environment' prompt`) or encode the same logic directly? Current draft does the latter — open to flipping.
7. **Pricing signal for the model:** Would Nebius share the approximate $ cost per VM-hour so the skill can give Claude better intuitions about when to branch aggressively vs. sequentially?
8. **Co-distribution:** Interested in hosting this skill (or a forked/endorsed version) in a `nebius/contree-claude-skill` repo? Happy to transfer.

---

# Appendix

## A. Skill contents at a glance

- **`SKILL.md`** — the full skill (~400 lines)
  - Mental model (Git analogy)
  - Part 1: MCP usage
    - CHECK-PREPARE-EXECUTE pattern
    - Local files / rsync
    - Free inspection
    - Uploads, parallel execution, branching
    - `run` parameter reference
    - MCP prompts/guides pointer
    - Tool cost table
  - Part 2: SDK usage
    - Async vs sync clients
    - `images.use` / `.oci` / `.import_from`
    - Sessions vs images
    - File handling
    - Tagging, subprocess, config
  - Common mistakes (aggregated from both)
  - Setup (MCP + SDK)

## B. How this was built

This skill was drafted in an afternoon using Anthropic's `skill-creator` skill, which scaffolded the structure, helped design test prompts, and will drive the eval/iterate loop. Source material:

- [contree.dev](https://contree.dev) landing page + blog
- [docs.contree.dev](https://docs.contree.dev) (MCP, SDK, main docs)
- [github.com/nebius/contree-mcp](https://github.com/nebius/contree-mcp) (source read for exact tool signatures)
- [github.com/nebius/contree-sdk](https://github.com/nebius/contree-sdk) (source read for SDK API shape)

Discrepancies between docs and source (e.g., `run` default `timeout: 30` in MCP tool vs `60` in backend model) were resolved in favor of the source.

## C. License

Apache 2.0, matching Nebius's licensing on the underlying MCP server and SDK.

## D. Contact

- **Author:** Colin — [@opencolin](https://github.com/opencolin)
- **Issues:** [github.com/opencolin/contree-skill/issues](https://github.com/opencolin/contree-skill/issues)
- **Nebius PMs:** if you'd like to chat, I'm at `collin@dabl.club`
