---
name: fork-workflow
description: >-
  How to work in this llama.cpp fork - syncing with upstream, branching, committing, and
  which files are off-limits. Use before syncing with upstream, before starting any change,
  or when unsure whether something belongs on master.
---

# Working in this fork

`llama.cpp-custom` is a fork of `ggml-org/llama.cpp`. `master` is a **mirror** of
`upstream/master` plus local documentation under `plans/` and `.claude/`. Nothing else.

Full rationale lives in `plans/decisions/`. This file is the operational summary - the
"normal way of operating" - so you do not have to re-derive it.

## The steady-state sync loop

```bash
git fetch upstream

# sanity check: master must carry NOTHING but our local docs
git diff --name-only upstream/master..master
#   expected: only plans/**, .claude/**, overlays/**/DEPLOY.md, DEPLOYMENT-BRANCHES.md
#   anything else means work landed on master by mistake - move it to a branch first

git switch master && git rebase upstream/master
git push --force-with-lease origin master
```

### Why rebase and not `--ff-only`

A fast-forward requires `master` to be an **ancestor** of the target commit. Ancestry is a
property of the commit graph, not of which files changed - so the local `plans/` commits
disqualify it even though upstream never touches that path:

```bash
$ git merge-base --is-ancestor master upstream/master; echo $?
1    # not an ancestor -> `git merge --ff-only` fails
```

The rebase is conflict-free by construction, because our commits touch only paths upstream
does not use.

### Why `--force-with-lease`

Rebasing gives the local commits new SHAs, which `origin` no longer contains, so the push
must overwrite. `--force-with-lease` aborts if `origin` moved since your last fetch, turning
a silent overwrite into a visible error. Never use bare `--force`.

### The tripwire you gave up

`--ff-only` used to fail loudly if work were committed to `master` by accident. Rebase has no
such guard - it will happily replay a stray commit. The `git diff --name-only` check above is
the replacement. Run it.

## Branching

`master` receives no code. Code lives on **overlay branches based on
`upstream/master`** - not on `master`:

```bash
git fetch upstream
git switch -c feat/my-thing upstream/master
# ... work ...
git rebase upstream/master        # after each sync, then re-verify (below)
```

**Base on `upstream/master`, not `master`.** `master`'s documentation commits are not
in upstream, so `git merge-base upstream/master <branch>` resolves to the upstream tip
and every doc file falls inside the patch range. A branch based on `master` therefore
ships `plans/` and `.claude/` into any stock tree its patch is applied to. See
`plans/decisions/004-overlay-branches.md`.

Keep branches short-lived. Upstream moves fast - there were 2400 commits between two
arbitrary points in this repo's history.

### Extracting a patch

Diff against the branch's merge-base, never `upstream/master` directly - branches
deliberately lag upstream, and diffing the tip reverts every upstream commit in
between. Exclude `overlays/`, which is fork documentation:

```bash
BASE=$(git merge-base upstream/master feat/my-thing)
git diff "$BASE"..feat/my-thing -- . ':(exclude)overlays/' > my-thing.patch
```

Verify against a stock tree rather than assuming:

```bash
git switch --detach upstream/master
git apply --check my-thing.patch
```

### Overlay rules

- **One topic branch per patch set.** The branch is the atomic unit.
- **Branches must apply independently, in any order.** If B needs A, B carries A's
  changes; do not stack branches and rely on apply order. Re-check whenever a second
  overlay is added - both file overlap and an actual sequential `git apply`.
- **Document each set in `overlays/<name>/README.md`** on its own branch. Never at
  the repo root: every branch would claim the same path and collide when composed.
- **Deploy notes go on `master`**, at `overlays/<name>/DEPLOY.md`. Their audience
  never checks out the branch, so a note that only exists there is findable only by
  someone who already knows the branch name. Keeping them on `master` also makes
  them structurally unable to reach a stock tree. No collision risk: they sit in
  per-name directories and `master` is never composed into a stock tree.
- **Index them in `DEPLOYMENT-BRANCHES.md` on `master`.** That file is the entry
  point for anyone consuming the fork. It is an index; the per-set manifest wins on
  conflict.
- **Gate behaviour changes off by default** - env var, config key, or capability
  check - so a deployment that sets nothing behaves exactly as stock.
- **Check upstream first** (`gh issue list`, `gh pr list --search`) before carrying a
  fix. A bug present in stock upstream belongs in an upstream PR, not in permanent
  local divergence.
- **The fork is where you author, not necessarily what runs.** Confirm what the
  deployed artifact contains before concluding a fix is live.

## Files that are off-limits

| Path | Why |
|---|---|
| `CLAUDE.md` | Upstream-owned. Byte-identical to `upstream/master`. It is upstream's pointer to `AGENTS.md`, not personal config. |
| `AGENTS.md` | Upstream-owned and actively rewritten upstream (it grew ~200 lines and reversed its AI policy in one sync). |
| `skills/` | Upstream-owned (`add-new-model`, `code-review`). Our skills go in `.claude/skills/`. |
| `vendor/` | Generated by `scripts/sync_vendor.py`. `check-vendor.yml` fails CI on any hand edit. Change the script and re-run it. |

Local documentation goes in `plans/` (docs and ADRs), `.claude/` (skills), or
`DEPLOYMENT-BRANCHES.md` (the overlay index) on `master`, and `overlays/<name>/` on an
overlay branch. Upstream uses none of these paths, so they never collide.

Verify at any time:

```bash
git diff --stat upstream/master -- CLAUDE.md AGENTS.md   # must be empty
```

## Committing

Write a real message with a body. The subject says what changed - which the diff already
shows. The body carries the *why*: the reasoning, the constraint discovered, what was
rejected. That is the part that is unrecoverable later.

```
<module>: <imperative summary>

Motivation and reasoning, wrapped around 72 characters. Explain trade-offs
and rejected alternatives when a choice was made. If the commit corrects an
earlier error, say what was wrong and why.

Assisted-by: <assistant name>
```

`AGENTS.md` mandates `Assisted-by:` and **forbids** `Co-authored-by:`.

### Agent restrictions

`AGENTS.md` exempts private forks from the contributor rules, so local work here is
unconstrained. These still apply without exception:

- **Never** run `git push` or `gh pr create` on the user's behalf without explicit, per-action
  approval. Automated PR submission risks a contributor ban from the project.
- **Never** write PR descriptions, review comments, or replies to reviewers. This is
  non-overridable and applies everywhere, fork or not.
- Commit only with explicit approval for that specific action.

Anything headed upstream is a different matter: you must fully understand every line, be able
to debug it without AI, and disclose meaningful AI contribution. Read `AGENTS.md` first.

## Before changing llama.cpp itself

- Read `AGENTS.md` and `CONTRIBUTING.md`.
- Upstream ships its own skills - prefer them over improvising: `skills/add-new-model/` for
  porting a model architecture, `skills/code-review/` for review.
- Code and comments are **ASCII only** - no em-dash or unicode punctuation.
- Architecture, module layout and build/test commands: `plans/000-codebase-overview.md`.

## Recovering from a bad sync

`git reflog` has everything. To undo an in-progress rebase:

```bash
git rebase --abort
```

To get back to where `master` was before a rebase that already finished:

```bash
git reflog master              # find the pre-rebase SHA
git branch -f master <sha>
```

Take a safety branch first when doing anything unusual:

```bash
git branch -f backup/pre-<thing> master
```
