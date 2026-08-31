# Deployment note: llama-server prompt/completion capture

**Audience:** whoever builds and runs `llama-server` on the inference hosts.
**Self-contained** - you do not need to read anything else in this fork.

---

## Read this first

**Do not enable this continuously yet.**

The archive that is meant to drain the log directory does not exist yet. Turning
capture on permanently today gives you unbounded growth of plaintext prompts on a
serving host, with nothing consuming or removing them.

What is wanted right now is a **time-boxed run**: an hour of real traffic, with a
hard disk quota and a scheduled disable so it cannot outlive the experiment, plus
two or three sample files and a rough bytes-per-hour figure. Details in
[Time-boxed run](#time-boxed-run) below.

Capture is off unless you pass the flag, so deploying the build itself is safe.
Deploying the binary and *not* switching it on is a valid and expected outcome.

---

## What this changes

`llama-server` already has a `--log-prompts-dir` flag. Stock, it writes only the
prompt, into a file named after the current millisecond.

This overlay changes it to record the full request **and** the completion, as two
JSON documents per request, named after the completion id:

```
<log-dir>/YYYY-MM-DD/HH/chatcmpl-<random>.req.json          before inference starts
<log-dir>/YYYY-MM-DD/HH/chatcmpl-<random>.res.<index>.json  when the request finishes
```

Records are sharded into UTC date and hour directories. The shard is chosen when
the request arrives and reused for its completion, so a request that crosses an
hour boundary still writes both files into the same directory - you never have to
look in two places to find a pair.

It also fixes a real bug in the stock version: two requests arriving in the same
millisecond resolved to the same filename, and the second silently overwrote the
first. Under batched serving that is routine, so stock capture loses records with
no error.

**Nothing else in the server changes.** If you do not pass `--log-prompts-dir`,
behaviour is identical to stock.

### Why not a proxy

A reverse proxy in front of `llama-server` was considered and rejected. Not for
performance - for blast radius. A proxy puts a log-capture component in the
serving path, so when it fails you lose inference mid-turn, not just an archive
entry. Writing files keeps capture strictly downstream of serving: if the disk
fills, capture stops and inference continues.

---

## Getting the code

Repo: `git@github.com:mardelden/llama.cpp-custom.git`
Branch: `feat/prompt-log-capture`

Four files change, about 209 lines:

```
common/arg.cpp
tools/server/server-context.cpp
tools/server/server-task.cpp
tools/server/server-task.h
```

### Option A - build the branch directly

```bash
git clone git@github.com:mardelden/llama.cpp-custom.git
cd llama.cpp-custom
git switch feat/prompt-log-capture
```

The branch is based on upstream `9723942ad` (2026-08-30). It is not a fork of
llama.cpp with other changes - it is stock upstream at that commit plus these
four files.

### Option B - apply as a patch to your own upstream checkout

Preferred if you pin a specific upstream revision.

```bash
# from a clone of this fork
BASE=$(git merge-base upstream/master feat/prompt-log-capture)
git diff "$BASE"..feat/prompt-log-capture -- . ':(exclude)overlays/' > prompt-log-capture.patch
```

Then against your tree:

```bash
git apply --check prompt-log-capture.patch   # verify first, applies nothing
git apply prompt-log-capture.patch
```

`--check` is not optional. If it fails, do not force it - see below.

### If you deploy against a newer upstream

The patch was verified to apply cleanly to upstream `daef7b687` (2026-08-31), one
commit past its base. It touches `common/arg.cpp` and three files under
`tools/server/`, so it will keep applying until upstream edits those specific
regions - the server's completion handler and task-result serialisation.

Upstream moves very fast (2400 commits between two arbitrary points in this repo's
recent history), so **check, do not assume**:

```bash
git apply --check prompt-log-capture.patch
```

If that fails, the overlay needs rebasing onto the newer upstream. That is a
change to this fork, not something to resolve at deploy time - come back to us
rather than hand-resolving conflicts in a deployment tree.

---

## Building

This changes C++. **It requires a rebuild** - it cannot be delivered by overlaying
files onto a stock image.

```bash
cmake -B build -DGGML_CUDA=ON     # or your usual backend flags
cmake --build build --target llama-server -j"$(nproc)"
```

Use whatever backend flags you build with today; this overlay is backend-agnostic
and touches no ggml or backend code.

---

## Enabling capture

```bash
llama-server -m <model> --log-prompts-dir /var/log/llama-prompts
```

The directory is created if missing, mode `0700`. Records are written `0600`.
That is deliberate and set explicitly rather than left to umask - these files
contain fully assembled prompts, including system prompts and any source code
pulled into context. They are the most sensitive content the server handles.

### Put it on its own filesystem

This is the load-bearing operational decision.

Writes are fail-soft: if a write fails, the server logs an error and continues
serving. So a full disk degrades to *no capture*, not *no inference* - **provided
the directory is not sharing a filesystem with anything that matters.** On a
shared volume, filling it takes out whatever else lives there.

### Do not route these into journald or your log shipper

Two concrete reasons, not a policy preference:

- journald's `LineMax` defaults to 48K. Real prompts here serialise to several
  hundred KB. Records would be truncated to a fraction of their content.
- journald rate-limits by default (`RateLimitIntervalSec=30s`,
  `RateLimitBurst=10000`) and **silently drops** messages over the burst.

Beyond truncation, the observability pipeline has a much broader access model than
these records are allowed to have. Prompt content, file content, and file paths
must not enter it. "We already have log shipping" is not a shortcut here.

---

## Record format

`chatcmpl-<id>.req.json`

| field | meaning |
|---|---|
| `id` | completion id; also the filename stem, and the key that pairs the two files |
| `timestamp` | milliseconds, at request time |
| `model` | resolved model name |
| `request` | full request object: the **rendered** prompt after chat templating, sampling params, tools, and any extra fields the client sent |

`chatcmpl-<id>.res.<index>.json`

| field | meaning |
|---|---|
| `id`, `timestamp` | as above |
| `response.content` | the completion text |
| `response.message` | parsed message: `content`, `reasoning_content`, `tool_calls` |
| `response.generation_settings` | full sampling configuration |
| `response.tokens_predicted` / `tokens_evaluated` | generated / prompt tokens |
| `response.tokens_cached` / `tokens_prompt_cache` | cache reuse |
| `response.stop_type`, `stopping_word`, `truncated` | how generation ended |
| `response.timings` | slot timing stats |

The shape is the same regardless of which endpoint served the request
(OpenAI-compatible or Anthropic) and regardless of streaming.

`<index>` is `0` for normal requests. A request asking for several completions
(`n > 1`) produces one `.res.<index>.json` per completion, all sharing one `.req`.

### Properties worth knowing

- **Write-once.** Files are written once, closed, and never reopened or renamed.
- **A truncated file is invalid JSON**, so incomplete records are detectable by
  parsing - there is no separate "complete" marker and none is needed.
- **A `.req` with no `.res`** means the server died or the request was aborted
  mid-flight. That is unambiguous, not corruption.
- **Filenames contain no user data.** Ids are server-generated, so paths are safe
  to put in logs and object keys.
- **Directories stay small.** One hour of traffic each, so whatever reads this
  directory does not get slower the longer capture has been running.

Every directory in the tree is `0700` and every record `0600`, set explicitly
rather than left to umask.

---

## Time-boxed run

What is actually being asked for right now.

```bash
# dedicated, quota'd location
sudo mkdir -p /var/log/llama-prompts
sudo chown llama:llama /var/log/llama-prompts
sudo chmod 0700 /var/log/llama-prompts
```

Start `llama-server` with `--log-prompts-dir /var/log/llama-prompts`, run about an
hour of real traffic, then **stop capture** by restarting without the flag.

Schedule the disable up front so it cannot outlive the experiment - a `systemd`
timer that restarts the unit without the flag, or a calendar reminder if the unit
is managed by hand. Do not rely on remembering.

Then report back:

```bash
# size
du -sh /var/log/llama-prompts

# file count - report this, not just bytes
find /var/log/llama-prompts -type f | wc -l

# inode headroom on that filesystem
df -i /var/log/llama-prompts

# bytes per request
bytes=$(du -sk /var/log/llama-prompts | cut -f1)
reqs=$(find /var/log/llama-prompts -name '*.req.json' | wc -l)
echo "$(( bytes / reqs )) KB per request, over $reqs requests"
```

**Report the file count and `df -i` output, not only the byte total.** Each request
writes two files, so a host serving 10 req/s creates about 1.7M files a day. On ext4
the inode count is fixed at mkfs time and can run out well before the disk is full;
XFS allocates them dynamically and is less exposed. Which of those you are on
changes the retention answer, so we need the number.

and hand over **two or three sample file pairs** (a `.req.json` and its matching
`.res.<index>.json`). Sanitise the content if needed - the shape matters more than
the text. Note that sanitising by hand is itself a reason to keep the run short.

### Expect this to be large

Every request logs the *whole rendered conversation*, because that is what the
model is actually sent. So a 20-turn conversation writes turn 1 twenty times
across twenty files. Volume grows roughly with the square of conversation length.

Prompt-cache reuse does **not** reduce this - caching saves prefill compute, not
what gets written.

Multimodal requests embed images as base64 inline, deliberately, so records stay
replayable. Base64 adds about 33% over the raw image, repeated on every turn.

This is why the volume figure is being asked for before anything is enabled
permanently.

---

## Verifying it works

After the first request:

```bash
find /var/log/llama-prompts -type d -exec ls -ld {} \;
```

Expect every directory `drwx------`, records `-rw-------`, laid out as
`YYYY-MM-DD/HH/`, with files in pairs sharing an id stem.

```bash
# every record is valid JSON
find /var/log/llama-prompts -name '*.json' | while read -r f; do
  python3 -c "import json,sys; json.load(open(sys.argv[1]))" "$f" || echo "INVALID: $f"
done

# every request has at least one response (pairs share a directory)
find /var/log/llama-prompts -name '*.req.json' | while read -r r; do
  d=$(dirname "$r"); id=$(basename "$r" .req.json)
  n=$(ls "$d/$id".res.*.json 2>/dev/null | wc -l)
  [ "$n" -eq 0 ] && echo "UNPAIRED: $id"
done

# completions are actually populated
python3 - <<'PY'
import json, glob
for f in sorted(glob.glob('/var/log/llama-prompts/*/*/*.res.*.json')):
    d = json.load(open(f))['response']
    print(f.split('/')[-1], '->', repr(d['content'][:60]))
PY
```

Empty `content` on **every** record means something is wrong - report it. A few
empty completions are normal (a request that was aborted, or one that produced
only tool calls).

---

## Rolling back

Capture is opt-in, so the fastest rollback is to stop passing the flag and
restart. Existing files are left alone; nothing in this change deletes anything.

To remove the code, redeploy your stock build, or:

```bash
git apply -R prompt-log-capture.patch
```

There is no state, no schema, and no migration. Nothing else in the server depends
on this.

---

## What is not included

- **No rotation or retention of any kind.** Files accumulate until something else
  removes them. Nothing in `llama-server` will ever delete them. Bounding growth
  is a deployment responsibility - a dedicated filesystem plus
  `systemd-tmpfiles` ageing, or a sweep keyed on completed pairs.

  If you add a sweep: **never delete a `.req.json` whose `.res` has not landed.** A
  slow request legitimately leaves one sitting alone for minutes. Prune on the
  `.res` file and remove its `.req` sibling, rather than ageing both independently.

  The hour directories make this easier - an hour directory two hours old contains
  no in-flight requests, so it can be removed wholesale without inspecting pairs.
  Prune whole hours rather than individual files, and the pairing hazard disappears.

  If disk pressure ever becomes the binding constraint, compressing a *completed
  hour directory as a single stream* is worth far more than compressing files
  individually. Measured on simulated multi-turn traffic: `zstd --long` over one
  concatenated stream was about **10x smaller** than the sum of the same records
  compressed separately, because every turn re-logs the whole conversation and a
  long-window compressor can deduplicate that. Per-file compression cannot see
  across files. Do this on completed hours only, never the current one.

  Whatever window you choose, declare it explicitly - the archive side needs to
  alert when its lag approaches your deletion window. A retention policy and a
  copy policy that are each individually correct can compose into silent data
  loss; that has happened before and is the specific failure being designed
  against here.

- **Writes are synchronous** on the request thread, before inference starts. On
  NVMe this is negligible against inference time. On slow or network storage it
  would add latency to every request - another reason for a dedicated local
  filesystem.

- **No session id.** Records carry a per-request completion id, not a
  conversation or session identifier. Correlating requests into sessions is not
  solved yet. If clients send an extra JSON field on OpenAI-compatible endpoints
  it is preserved in `request`, which may be usable - but nothing is wired up.
  Anthropic's `metadata.user_id` is currently dropped before it reaches the log.

- **No inbound HTTP headers** are captured.

---

## Validation status - read before trusting this

Verified by the author on macOS with a tiny test model (`stories260K`), covering:

| case | result |
|---|---|
| OpenAI-compatible, non-streaming | completion captured |
| OpenAI-compatible, streaming | completion captured |
| Anthropic `/v1/messages`, streaming | completion captured |
| `n=2` parallel completions | separate files, no overwrite |
| all records | valid JSON, correctly paired |
| permissions | `0700` directory, `0600` files |

**Not yet verified:** Linux, CUDA builds, production models, real concurrency, or
sustained load. Treat the time-boxed run as the first real test, and check the
pairing and volume numbers before drawing conclusions from the data.

One bug was found during that testing and fixed: in streaming mode the naive
implementation recorded an **empty completion for every streamed request**. If you
see that symptom, you are running a build from before that fix - confirm your
build includes commit `068cc3589` or later on `feat/prompt-log-capture`.

---

## Questions

This note lives on `master`, alongside `DEPLOYMENT-BRANCHES.md`, which indexes every
overlay this fork carries.

Deeper material, if you need it:

- `plans/001-prompt-log-capture.md` on `master` - design rationale, full field
  inventory, and the alternatives that were rejected
- `overlays/prompt-log-capture/README.md` on the **`feat/prompt-log-capture`
  branch** - the patch manifest, which travels with the code
