# Decision: Keep codebase docs in `plans/`, never in `CLAUDE.md` or `AGENTS.md`

**Status:** Accepted
**Date:** 2026-08-30
**Area:** documentation, fork maintenance

## Context

Running `/codebase-docs init` on this repo surfaced a conflict.

The skill's `init` step says to create `CLAUDE.md` with build/architecture/patterns
sections. Its own Guidelines section says the opposite — "Never modify `CLAUDE.md`. It's
user-maintained." So the skill contradicts itself.

Inspecting the repo resolved the contradiction in an unexpected direction: **`CLAUDE.md`
here is not user-maintained at all — it is upstream's file.**

| File | Tracked? | vs `upstream/master` | Notes |
|---|---|---|---|
| `CLAUDE.md` | yes | **byte-identical** | One line, pointing at `AGENTS.md`. Added upstream in `3595ae596` |
| `AGENTS.md` | yes | older copy, no local edits | Upstream's AI-usage policy. Grew ~200 lines in commits we were behind - and reversed its stance |

This fork (`mardelden/llama.cpp-custom`) is **0 commits ahead, 2400 behind**
`upstream/master` — a clean mirror with no local modifications.

The preferred general pattern was to treat `CLAUDE.md` as a thin pointer and `AGENTS.md`
as the source of truth. That pattern is sound in general — and upstream llama.cpp already
implements exactly it. The problem is purely one of ownership: upstream owns both files
and actively edits them.

## Decision

**All codebase documentation lives under `plans/`. `CLAUDE.md` and `AGENTS.md` stay
byte-identical to upstream and are never edited in this fork.**

- `plans/000-codebase-overview.md` — build commands, architecture, patterns, critical rules
- `plans/decisions/` — ADRs and lessons
- `plans/NNN-*.md` — implementation plans

`plans/` is safe because upstream llama.cpp has no such directory, so it can never
collide during a sync.

## Alternatives Considered

| Alternative | Pros | Cons | Why rejected |
|---|---|---|---|
| **Append to `AGENTS.md`** | Adopts the "AGENTS.md as source of truth" pattern literally; one obvious place to look | Upstream-tracked and *actively changed* upstream (3 recent commits, +200 lines). Every `git pull upstream master` that touches it conflicts. Content would appear in any upstream PR | Recurring manual merge conflicts on a file we get no benefit from owning |
| **Overwrite `CLAUDE.md`** | Matches the skill's literal `init` instruction | Same conflict problem; also destroys upstream's mandated pointer to the AI-usage policy | Would silently drop a policy pointer the project marks IMPORTANT |
| **Local `AGENTS.local.md` + `.git/info/exclude`** | Keeps the pointer-pattern spirit; zero conflicts; cannot leak into a PR | A second, invisible convention on top of `plans/`; `.git/info/exclude` is not portable across clones | Adds a mechanism without adding value over `plans/` |
| **`plans/` only** ✅ | Zero conflict surface; upstream files stay pristine; survives clones; already the skill's own directory convention | Docs are one directory away rather than at the repo root | **Chosen** |

## Consequences

- Syncing with upstream stays a clean fast-forward. This is the main win, given the fork
  is 2400 commits behind and will need to sync.
- Any agent or human starting work reads `AGENTS.md` first (per `CLAUDE.md`), then
  `plans/000-codebase-overview.md` for structure.
- If this fork ever gains real local commits, revisit whether a root-level pointer is
  worth the conflict cost. Until then, no.
- `plans/` content must never be pushed upstream. It is a local reading aid.
  (Note: upstream has since *reversed* its AI policy - AI-generated code is now allowed,
  and private forks are explicitly exempt. That does not change this decision, which rests
  on merge-conflict cost, not on the AI policy.)

## Related skill change

The global `codebase-docs` skill was updated in the same session to:

1. Write `CLAUDE.md` as a thin pointer to `AGENTS.md`, with `AGENTS.md` as the source of
   truth (the general pattern).
2. **Detect the fork case first** — if `CLAUDE.md` or `AGENTS.md` is already git-tracked
   and matches an upstream remote, leave both alone and write to `plans/` instead.
3. Resolve the `init`-vs-Guidelines self-contradiction that started this.

## Postscript (2026-08-30, same session)

The fork was synced immediately after this decision: `vendor/` was restored, then
`git merge --ff-only upstream/master` fast-forwarded `4cc6eb158 -> 9723942ad` cleanly
(2400 commits). Result: 0 ahead / 0 behind, clean tree, `plans/` untouched.

This validated the decision in practice - because the root files were left pristine, the
sync was a pure fast-forward with **zero conflicts**. Had we written into `AGENTS.md`, it
would have conflicted: upstream rewrote that file substantially, including reversing the
AI policy.
