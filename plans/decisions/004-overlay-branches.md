# Decision: Overlay branches based on `upstream/master`, indexed from `master`

**Status:** Accepted
**Date:** 2026-08-30
**Area:** fork maintenance, git workflow
**Supersedes:** the branch-base rule in
[`002-fork-branching-strategy.md`](002-fork-branching-strategy.md)

## Context

`002` established that `master` is a mirror and all work happens on topic
branches, and said to branch **from `master`**. That was written before the
`fork-workspace` convention existed, and before there was an actual code branch
to test it against.

Once `feat/prompt-log-capture` existed, the rule turned out to be wrong.

The overlay convention says a patch must be extractable by diffing against the
branch's merge-base, and must apply to a **stock** upstream tree. Checking that
against our branch:

```bash
$ BASE=$(git merge-base upstream/master feat/prompt-log-capture)
$ git diff $BASE..feat/prompt-log-capture --name-only
.claude/skills/fork-workflow/SKILL.md
common/arg.cpp
plans/.gitkeep
plans/000-codebase-overview.md
plans/001-prompt-log-capture.md
plans/decisions/001-docs-location.md
plans/decisions/002-fork-branching-strategy.md
plans/decisions/003-workflow-as-repo-skill.md
tools/server/server-context.cpp
tools/server/server-task.cpp
tools/server/server-task.h
```

The patch carried our internal documentation into anything it was applied to.

The cause is that `master`'s documentation commits are not in upstream, so
`git merge-base upstream/master <branch>` resolves to the **upstream tip** rather
than to `master`. Everything on `master` therefore falls inside the diff range.
This is the same hazard `002` already identified for `--ff-only` - ancestry is a
property of the commit graph, not of which paths changed - arriving from the
other direction.

## Decision

**Code branches are created from `upstream/master` directly.**

```bash
git fetch upstream
git switch -c feat/<name> upstream/master
```

`master` keeps its documentation role and gains one file: `DEPLOYMENT-BRANCHES.md`,
the catalogue of overlay branches. Per-patch-set documentation lives on its own
branch at `overlays/<name>/README.md`.

Patch extraction excludes `overlays/`, which is fork documentation:

```bash
BASE=$(git merge-base upstream/master feat/<name>)
git diff "$BASE"..feat/<name> -- . ':(exclude)overlays/' > <name>.patch
```

Verified rather than assumed:

```bash
git switch --detach upstream/master
git apply --check <name>.patch
```

`feat/prompt-log-capture` was rebased with
`git rebase --onto upstream/master master feat/prompt-log-capture`. The code was
confirmed unchanged by the rebase (`git diff` of the source paths against the
pre-rebase branch was empty), and the resulting 209-line patch applies cleanly to
a stock `upstream/master`.

## Alternatives Considered

| Alternative | Pros | Cons | Why rejected |
|---|---|---|---|
| **Keep branching from `master`, scope every diff by path** | Branches carry the docs, so context is available while working | Every extraction must remember an explicit path list; forgetting it silently ships internal docs into a stock tree. The failure is quiet and only visible to whoever applies the patch | A convention that fails silently when you forget a flag is the wrong default |
| **Move `plans/` off `master` so it is a pure mirror** | Restores fast-forward syncing; branch base becomes irrelevant | Docs would need their own branch, which nobody would land on by default; the catalogue is specifically supposed to live where consumers arrive | Loses the one property that makes `master` a useful landing place |
| **Root-level per-branch description** | Simple to find | Every overlay branch would write the same root path with different content, colliding as soon as two are composed | Directly breaks independent applicability |
| **Branch from `upstream/master`, index from `master`** | Patches contain only code with no path scoping needed; overlay docs are disjoint by construction; catalogue sits where consumers land | Branches lack `plans/` in their working tree | **Chosen** |

## Consequences

- Overlay branches do not contain `plans/`. Design rationale is read from
  `master`; the operational manifest travels with the branch in `overlays/`.
- `DEPLOYMENT-BRANCHES.md` is the entry point for anyone consuming this fork. It
  is an index; the per-set manifest wins where they disagree.
- Adding a second overlay requires re-checking that the branches compose - both
  the file-overlap check and an actual sequential `git apply`. Trivially true
  with one branch; not thereafter.
- Overlay branches are rebased onto each new `upstream/master` independently, and
  re-verified with `git apply --check`. They are expected to lag upstream between
  syncs, which is exactly why patches must be diffed against the merge-base and
  never against the upstream tip.
- The `--ff-only` guidance in `002` still applies to `master` and is unaffected.

## Related

- [`002-fork-branching-strategy.md`](002-fork-branching-strategy.md) - the
  `master`-as-mirror rule, whose branch-base guidance this supersedes
- [`003-workflow-as-repo-skill.md`](003-workflow-as-repo-skill.md) - why the
  operational checklist is a skill rather than an ADR; the same reasoning applies
  to `DEPLOYMENT-BRANCHES.md` being an index rather than a rationale document
- `.claude/skills/fork-workflow/SKILL.md` - updated alongside this decision
