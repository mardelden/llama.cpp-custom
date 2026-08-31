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

# sync master (rebase, NOT --ff-only - see below)
git fetch upstream
git switch master && git rebase upstream/master

# replay work on top
git switch feat/my-thing && git rebase master
```

### Single exception: `plans/`, and what it costs

`plans/` is committed to `master` (commits `a2b5dc54d`, `f8194c080`), making it 2 ahead of
upstream. This is accepted because upstream has no `plans/` path, so the *content* can never
conflict, and the alternative (untracked + `.git/info/exclude`) loses the docs on any reclone.

**But it does break `--ff-only`.** A fast-forward requires `master` to be an *ancestor* of
the target commit. That is a property of the commit graph, not of which files changed - so
any local commit, even on a path upstream never touches, disqualifies it:

```
$ git merge-base --is-ancestor master upstream/master
# exit 1 -> master is NOT an ancestor -> `git merge --ff-only` fails
```

So the sync command is **`git rebase upstream/master`**, which replays the `plans/` commits
on top of the new upstream head. Because those commits touch only `plans/`, the rebase is
conflict-free by construction.

The tradeoff: `--ff-only` was going to serve as a tripwire that catches work accidentally
committed to `master`. Rebasing gives that up - it will happily replay a stray commit too.
Substitute check before syncing:

```bash
git diff --name-only $(git merge-base upstream/master master)..master   # ONLY local docs
```

(Superseded scope: `.claude/` was added in
[`003-workflow-as-repo-skill.md`](003-workflow-as-repo-skill.md). The operational copy of
this loop lives in `.claude/skills/fork-workflow/SKILL.md`.)

## Alternatives Considered

| Alternative | Pros | Cons | Why rejected |
|---|---|---|---|
| **Commit to `master` directly** | Simplest day to day; no branch juggling | `master` diverges permanently. Every future sync becomes a merge or rebase with real conflict potential; `--ff-only` stops working as a safety net | Trades a permanent, compounding cost for a small convenience |
| **Patch series** (`git format-patch` / stgit) | Maximum portability; changes fully decoupled from history | Manual apply/refresh on every sync; no branch tooling; no CI | Portability we do not need - `origin` already exists as the durable store |
| **Topic branches, `master` near-pristine** | Sync is one command; conflicts only where work genuinely overlaps upstream | Requires remembering to branch before starting; `plans/` on `master` costs the `--ff-only` tripwire | **Chosen** |

## Consequences

- Syncing is `git fetch upstream && git switch master && git rebase upstream/master`.
  Before syncing, verify
  `git diff --name-only $(git merge-base upstream/master master)..master` lists only local docs.
- Because `plans/` lives on `master`, the fork's own commits get new SHAs on every rebase.
  `origin` already holds the old SHAs, so each sync needs
  `git push --force-with-lease origin master`. Use `--force-with-lease`, never plain
  `--force`: it refuses the push if `origin` moved since the last fetch, which turns a
  silent overwrite into a visible error.
- The full steady-state loop:

  ```bash
  git fetch upstream
  git diff --name-only $(git merge-base upstream/master master)..master   # sanity: ONLY local docs
  git switch master && git rebase upstream/master
  git push --force-with-lease origin master
  ```
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
