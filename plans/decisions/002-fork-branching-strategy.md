# Decision: `master` stays a pristine mirror; all work on topic branches

**Status:** Accepted
**Date:** 2026-08-30
**Area:** fork maintenance, git workflow

## Context

After syncing this fork to `upstream/master` (`4cc6eb158 -> 9723942ad`, 2400 commits, clean
fast-forward), the question was where custom work should live: directly on `master`, on
topic branches, or as standalone patch files.

The sync itself made the tradeoff concrete. It was a **pure fast-forward with zero
conflicts**, and that was only possible because `master` had no local commits and no
locally-modified tracked files. Upstream had substantially rewritten `AGENTS.md`,
`vendor/` (22 files, +13.7k lines), and the entire per-model architecture in `src/models/`
during that range. Any local commit touching those paths would have turned a one-command
sync into a manual conflict resolution across a 2400-commit gap.

## Decision

**`master` is a read-only mirror of `upstream/master`. It receives no work.**

All changes go on topic branches, rebased onto `master` after each sync:

```bash
# work
git switch -c feat/my-thing
# ... commits ...

# sync (always a fast-forward, fails loudly if not)
git fetch upstream
git switch master && git merge --ff-only upstream/master

# replay work on top
git switch feat/my-thing && git rebase master
```

`--ff-only` is deliberate: it **fails** rather than silently creating a merge commit if
`master` ever diverges. That failure is the signal that something was committed to `master`
by mistake.

### Single exception: `plans/`

`plans/` is committed to `master` (commit `a2b5dc54d`), making it technically 1 ahead of
upstream. This is accepted because:

- Upstream has no `plans/` path and never will, so it cannot collide
- `git merge --ff-only` still succeeds - a fast-forward only requires that `master` is an
  ancestor of the target, which one commit on a disjoint path does not prevent
- The alternative (untracked + `.git/info/exclude`) loses the docs on any reclone

If `plans/` ever *does* start conflicting, that is the signal upstream added the path, and
this exception should be revisited.

## Alternatives Considered

| Alternative | Pros | Cons | Why rejected |
|---|---|---|---|
| **Commit to `master` directly** | Simplest day to day; no branch juggling | `master` diverges permanently. Every future sync becomes a merge or rebase with real conflict potential; `--ff-only` stops working as a safety net | Trades a permanent, compounding cost for a small convenience |
| **Patch series** (`git format-patch` / stgit) | Maximum portability; changes fully decoupled from history | Manual apply/refresh on every sync; no branch tooling; no CI | Portability we do not need - `origin` already exists as the durable store |
| **Topic branches, `master` pristine** | Sync is always one command; conflicts only where work genuinely overlaps upstream; mistakes surface immediately via `--ff-only` | Requires remembering to branch before starting | **Chosen** |

## Consequences

- Syncing stays `git fetch upstream && git merge --ff-only upstream/master`. If that ever
  fails, something was committed to `master` - fix it rather than working around it.
- Topic branches need rebasing after each sync. Given the pace upstream moves (2400 commits
  between two arbitrary points), branches should be kept short-lived.
- `plans/` must be rebased along with everything else if it is ever moved off `master`.
- Pushing is a **human action**. Per `AGENTS.md`, an agent must never run `git push` or
  `gh pr create`; doing so risks a contributor ban. Agents may commit only with explicit
  per-action approval, and must use `Assisted-by:`, never `Co-authored-by:`.

## Related

- [`001-docs-location.md`](001-docs-location.md) - why `plans/` exists instead of writing
  to `CLAUDE.md` / `AGENTS.md`. Same underlying principle: keep the conflict surface with
  upstream at zero.
