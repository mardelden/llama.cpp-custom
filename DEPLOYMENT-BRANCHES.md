# Deployment branches

Index of the overlay branches carried by this fork. Start here.

`master` is a **documentation-only mirror** of `upstream/master`. It carries no
code changes and is never rebased or deleted, so it is the stable thing to land
on and branch from. Every code change lives on its own overlay branch.

**This file is an index.** Where it disagrees with an overlay's own
`overlays/<name>/README.md`, **the per-set manifest wins** - it travels with the
branch and is updated alongside the code.

> Note on branch naming: upstream llama.cpp uses `master`, not `main`. The
> fork-workspace convention says `main`; read it as "the mirror branch" here.

## Branches

| Branch | Scope | Activation | Rebuild? | Manifest |
|---|---|---|---|---|
| `feat/prompt-log-capture` | fleet-wide, inert unless activated | `--log-prompts-dir PATH` (off by default) | **yes** - C++ | [`overlays/prompt-log-capture/README.md`](https://github.com/mardelden/llama.cpp-custom/blob/feat/prompt-log-capture/overlays/prompt-log-capture/README.md) |

### `feat/prompt-log-capture`

Makes `--log-prompts-dir` record the full request envelope and the completion as
paired JSON documents keyed on the completion id, instead of upstream's
prompt-only file named after the current millisecond.

- **Requested by:** agent-ledger, for turn reconstruction and replay without a
  reverse proxy in the serving path.
- **Files touched:** `common/arg.cpp`, `tools/server/server-context.cpp`,
  `tools/server/server-task.{h,cpp}` - about 209 patch lines.
- **Removal condition:** upstream ships equivalent capture, the agent-ledger
  inference adapter is retired, or capture moves outside the server.
- **Design rationale:** `plans/001-prompt-log-capture.md` on this branch.

Carries an upstream bug fix as a side effect: two requests arriving in the same
millisecond previously resolved to the same filename and the second silently
truncated the first. That part is genuinely upstreamable on its own and could be
split from the record-format change to keep the carried surface smaller.

## Working with overlays

### Branches are based on `upstream/master`, not on `master`

Code branches start from `upstream/master` directly so their patches contain
only code. A branch based on `master` inherits its documentation commits, and
because those commits are not in upstream, `git merge-base` resolves to the
upstream tip - so the extracted patch drags `plans/` and `.claude/` into any
stock tree it is applied to.

```bash
git fetch upstream
git switch -c feat/<name> upstream/master
```

### Extracting a patch

Diff against the branch's merge-base, never `upstream/master` directly. Branches
deliberately lag upstream, and diffing the tip reverts every upstream commit in
between. Exclude `overlays/`, which is fork documentation:

```bash
BASE=$(git merge-base upstream/master feat/<name>)
git diff "$BASE"..feat/<name> -- . ':(exclude)overlays/' > <name>.patch
```

Verify against a stock tree before shipping:

```bash
git switch --detach upstream/master
git apply --check <name>.patch
```

### Checking that overlays compose

Each branch must apply to a stock tree independently, in any order. With one
overlay this is trivially true; re-check when a second is added:

```bash
# do any two branches touch the same file?
for b in $(git branch --list 'feat/*' --format='%(refname:short)'); do
  git diff --name-only "$(git merge-base upstream/master "$b")".."$b" -- . ':(exclude)overlays/'
done | sort | uniq -d          # any output = collision risk
```

If patch B needs patch A, B's branch must carry A's changes too. Do not stack
branches and rely on apply order.

### Syncing

`master` carries documentation commits, so it is not an ancestor of
`upstream/master` and cannot be fast-forwarded:

```bash
git fetch upstream
git diff --name-only upstream/master..master   # expect ONLY plans/, .claude/, this file
git switch master && git rebase upstream/master
git push --force-with-lease origin master
```

Overlay branches are rebased onto the new `upstream/master` independently, and
re-verified with `git apply --check`.

## Conventions

- **Never push to `upstream`.** Its push URL is deliberately set to
  `DISABLE_PUSH_TO_UPSTREAM`. If that is ever missing, restore it.
- **Never commit code to `master`.** Documentation only.
- **Gate behaviour changes off by default** so a deployment that sets nothing
  behaves exactly as stock.
- **Check upstream first** (`gh issue list`, `gh pr list --search`) before
  carrying a fix. A bug present in stock upstream belongs in an upstream PR, not
  in permanent local divergence.
- **The fork is where you author, not necessarily what runs.** Confirm what the
  deployed artifact actually contains before concluding a fix is live.

## See also

- `.claude/skills/fork-workflow/SKILL.md` - the operational checklist
- `plans/decisions/` - why the fork is arranged this way
- `plans/000-codebase-overview.md` - llama.cpp architecture and build commands
