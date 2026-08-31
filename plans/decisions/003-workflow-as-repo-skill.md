# Decision: Ship the fork workflow as a repo-local skill, and require commit message bodies

**Status:** Accepted
**Date:** 2026-08-30
**Area:** documentation, git workflow, tooling

## Context

Two gaps surfaced after the fork was synced and documented.

**1. The operational knowledge was in ADRs, not where work happens.**
`001-docs-location.md` and `002-fork-branching-strategy.md` record *why* `master` is a mirror
and *why* syncing uses rebase. But someone (human or agent) sitting down to work does not
naturally read `plans/decisions/` first. They run `git pull` or `git merge --ff-only`, hit a
failure, and re-derive the reasoning - or worse, work around it by committing to `master`.

The rules are also easy to get wrong in a specific way: the `--ff-only` mistake documented in
`002` was made *while writing the ADR that documents the workflow*. Reasoning has to be
re-verified each time it is applied; a checklist does not.

**2. Commit messages were too thin.**
The first three `plans/` commits had minimal messages, and `f8194c080` ("docs: record fork
branching strategy") had **no body at all** - it recorded a decision with alternatives and
trade-offs behind a single subject line. The user flagged this directly.

This came from over-applying an `AGENTS.md` rule: *"Do NOT write PR descriptions, commit
messages, or reviewer responses."* That rule exists so upstream contributors can defend their
own text to maintainers. `AGENTS.md` explicitly exempts private forks, so it does not govern
local docs commits here. Reading it literally produced worse history for no benefit.

## Decision

**1. The workflow ships as a repo-local skill at `.claude/skills/fork-workflow/SKILL.md`.**

It carries the sync loop, the branching rule, the off-limits file table, the commit format,
and recovery steps - as commands to run, not reasoning to follow. The ADRs remain the
authoritative *why*; the skill is the *how*, and it links back.

`.claude/` was chosen over `skills/`. Upstream now owns `skills/` (`add-new-model`,
`code-review`), so adding a file there risks a name collision on a path upstream actively
develops. Upstream tracks nothing under `.claude/`:

```bash
$ git ls-tree -r --name-only upstream/master | grep '^\.claude/'
$    # empty
```

**2. Commits require a body explaining why.**

```
<module>: <imperative summary>

Motivation and reasoning, wrapped around 72 characters. Trade-offs and
rejected alternatives when a choice was made. If correcting an earlier
error, what was wrong and why.

Assisted-by: <assistant name>
```

`Assisted-by:` is mandated by `AGENTS.md`, which forbids `Co-authored-by:`.

Applied retroactively: the four existing `plans/` commits were rewritten with full bodies.
The rewrite was verified content-neutral by comparing tree hashes before and after
(`0078ced64a7b62e29e795ebb47e151d7b071bef9` in both cases), so only messages changed.

## Alternatives Considered

| Alternative | Pros | Cons | Why rejected |
|---|---|---|---|
| **Put the workflow in `plans/` only** | One location for everything; no new mechanism | ADRs explain *why* and are long; nobody reads them before running `git pull`. The failure mode is silent - you only learn the rule when a sync breaks | Wrong artifact for the job: reference material, not a checklist |
| **Add it to upstream's `skills/`** | Sits beside `add-new-model` and `code-review` where an agent already looks | `skills/` is upstream-owned and actively developed; a future upstream skill could collide by name, reintroducing exactly the conflict `001` avoided | Recreates the problem `001` solved |
| **Put it in `CLAUDE.md`** | Loaded automatically every session, no invocation needed | `CLAUDE.md` is upstream-owned and byte-identical to upstream - editing it is what `001` forbids | Directly violates `001` |
| **`.claude/skills/fork-workflow/`** | Conflict-free path; invocable on demand; sits next to the code | Must be invoked or discovered rather than always-loaded | **Chosen** |

## Consequences

- Anyone working in this fork can read one file to learn the normal way of operating, instead
  of reconstructing it from three ADRs.
- `.claude/` joins `plans/` as a local-only path on `master`. The pre-sync check widens
  accordingly - it must now list only `plans/**` and `.claude/**`:

  ```bash
  git diff --name-only $(git merge-base upstream/master master)..master
  ```

  This is recorded in the skill itself and supersedes the `plans/`-only phrasing in `002`.
- The skill duplicates facts that live in the ADRs (the sync loop, the off-limits table). That
  duplication is deliberate but is now a drift risk: **when a workflow rule changes, update
  both.** The skill links to the ADRs to make the pairing visible.
- History rewrites on `master` are acceptable on this single-user mirror, but require
  `--force-with-lease` and a `backup/` branch beforehand.

## Related

- [`001-docs-location.md`](001-docs-location.md) - why local docs avoid `CLAUDE.md`/`AGENTS.md`
- [`002-fork-branching-strategy.md`](002-fork-branching-strategy.md) - why `master` is a mirror
  and why syncing rebases
- The global `codebase-docs` skill was updated in the same session to detect upstream-owned
  root files and to record a fork's workflow as a skill during `init`.
