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

Capture is off unless the flag is set, so deploying the build itself is safe.
Deploying the binary and *not* switching capture on is a valid and expected outcome.

### Current state on this fleet (2026-09-01)

Two things differ from the above, both worth a decision:

- **`llamacpp_log_requests: true`** is set in `inventory/host_vars/llamacpp/vars.yml`,
  so capture is on continuously. There is no scheduled disable and nothing prunes.
  That is the situation this note asks you to avoid; either add a disable, or accept
  it knowingly with the runway table below in view.
- **The deployed patch is stale.** `roles/llamacpp/files/patches/prompt-log-capture.patch`
  is 322 lines and predates the fix that stopped writing the prompt into *both* files.
  It therefore uses roughly **twice** the disk the table below predicts. Refresh it
  (command under "How this is delivered here") and rebuild before drawing volume
  conclusions from a run.

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

Six files change, about 327 lines:

```
common/arg.cpp                    directory mode + help text
tools/server/server-context.cpp   write helper, request record, response hook
tools/server/server-task.cpp      to_json_log()
tools/server/server-task.h        declaration
tools/cli/README.md               regenerated args table
tools/server/README.md            regenerated args table + Request logging section
```

Source of truth: `git@github.com:mardelden/llama.cpp-custom.git`, branch
`feat/prompt-log-capture`.

### How this is delivered here (the ansible path)

The fleet does not track this fork. `roles/llamacpp` clones
`unslothai/llama.cpp` at branch `glm5next/upstream` (needed for GLM-5.3-Flash,
which upstream master does not support) and applies source patches after clone,
before cmake:

```
roles/llamacpp/files/patches/
  anthropic-output-config-effort.patch
  prompt-log-capture.patch          <- this overlay
```

That is the right mechanism - keep using it. To refresh the patch file after a
change on our side:

```bash
# from a clone of this fork
BASE=$(git merge-base upstream/master feat/prompt-log-capture)
git diff "$BASE"..feat/prompt-log-capture -- . ':(exclude)overlays/' \
  > roles/llamacpp/files/patches/prompt-log-capture.patch
```

The `':(exclude)overlays/'` matters: `overlays/` is fork documentation and must
not reach a build tree.

### Verified against the branch you actually build

The patch applies cleanly to `unslothai/llama.cpp @ glm5next/upstream`
(tip `949f7efb0`, 2026-08-31), and to stock `upstream/master`. It touches
`common/arg.cpp` and three files under `tools/server/`, none of which the
glm5next work modifies, so the two compose without interaction.

**Check rather than assume**, on every base change:

```bash
git apply --check prompt-log-capture.patch   # verifies, applies nothing
```

If that fails, the overlay needs rebasing on our side - come back to us rather
than hand-resolving conflicts in a deployment tree.

### Building it standalone, if you need to

```bash
git clone git@github.com:mardelden/llama.cpp-custom.git
cd llama.cpp-custom && git switch feat/prompt-log-capture
```

That branch is stock `upstream/master` plus these six files - **no GLM-5.3
support**. Only useful for isolating this change; it is not what the fleet runs.

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

In this fleet it is two ansible variables, not a hand-written flag:

```yaml
# inventory/group_vars/all/inference_logging.yml - fleet default, currently false
inference_log_requests: false

# inventory/host_vars/llamacpp/vars.yml - the per-host opt-in
llamacpp_log_requests: true
```

The role turns that into `--log-prompts-dir {{ inference_log_dir }}`, which is the
`/var/log/inference` bind mount, backed by the dedicated `nvme-vg/inference-logs`
LV on `pve-ai`. Carrying the patch does **not** enable capture - that needs the
variable, and it needs a rebuild for the patch itself.

Directly, the equivalent is:

```bash
llama-server -m <model> --log-prompts-dir /var/log/inference
```

The directory is created if missing, mode `0700`. Records are written `0600`.
That is deliberate and set explicitly rather than left to umask - these files
contain fully assembled prompts, including system prompts and any source code
pulled into context. They are the most sensitive content the server handles.

### It is already on its own filesystem - keep it that way

`/var/log/inference` is backed by the dedicated `nvme-vg/inference-logs` LV, which
is the right shape. This section is here so it does not get consolidated later.

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

The directory already exists as a bind mount on its own LV, so there is nothing to
create. Toggle capture with the ansible variable rather than editing unit files:

```bash
# enable for a run
just llamacpp-configure                       # with llamacpp_log_requests: true

# stop capture
ansible-playbook playbooks/15-llamacpp.yml -e inference_log_requests=false
```

Start `llama-server` with `--log-prompts-dir /var/log/inference`, run about an
hour of real traffic, then **stop capture** by restarting without the flag.

Schedule the disable up front so it cannot outlive the experiment - a `systemd`
timer that restarts the unit without the flag, or a calendar reminder if the unit
is managed by hand. Do not rely on remembering.

Then report back:

```bash
# size
du -sh /var/log/inference

# file count - report this, not just bytes
find /var/log/inference -type f | wc -l

# inode headroom on that filesystem
df -i /var/log/inference

# bytes per request
bytes=$(du -sk /var/log/inference | cut -f1)
reqs=$(find /var/log/inference -name '*.req.json' | wc -l)
echo "$(( bytes / reqs )) KB per request, over $reqs requests"
```

### Sizing, for the reported target

You reported **ext4 with 6,553,600 inodes**. At ext4's default ratio of one inode per
16 KiB that implies a **100 GiB** filesystem - which matches the `nvme-vg/inference-logs`
LV backing `/var/log/inference`. That settles the question:

**Space runs out long before inodes do.** The crossover is 32 KB per request pair -
below that inodes bind first, above it space does. These records are far above it.

A request pair costs roughly **twice the conversation size**, plus about 2 KB. The
request record deliberately holds both the wire message array and the rendered prompt
(both were asked for), and they are near-identical in size.

| rendered prompt | pair on disk | requests before full | at 1 req/s | at 10 req/s |
|---|---|---|---|---|
| 64 KB | 0.1 MB | 806,596 | 224 h | 22 h |
| 256 KB | 0.5 MB | 204,003 | 57 h | 5.7 h |
| 575 KB | 1.1 MB | 91,022 | 25 h | **2.5 h** |
| 1 MB | 2.0 MB | 51,150 | 14 h | 1.4 h |

The inode ceiling is 3.28M pairs, which at any realistic rate you will never reach -
you will fill the disk first.

**For a one-hour run this is fine**, at any of the rates above. Set the quota anyway:
the failure mode without one is a full filesystem, and if that filesystem is shared
with anything else, you take that out too. It is the reason for the dedicated volume.

Still worth reporting back: actual bytes per request at your prompt sizes, and the
request count. `df -i` is no longer interesting given the above.

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
find /var/log/inference -type d -exec ls -ld {} \;
```

Expect every directory `drwx------`, records `-rw-------`, laid out as
`YYYY-MM-DD/HH/`, with files in pairs sharing an id stem.

```bash
# every record is valid JSON
find /var/log/inference -name '*.json' | while read -r f; do
  python3 -c "import json,sys; json.load(open(sys.argv[1]))" "$f" || echo "INVALID: $f"
done

# every request has at least one response (pairs share a directory)
find /var/log/inference -name '*.req.json' | while read -r r; do
  d=$(dirname "$r"); id=$(basename "$r" .req.json)
  n=$(ls "$d/$id".res.*.json 2>/dev/null | wc -l)
  [ "$n" -eq 0 ] && echo "UNPAIRED: $id"
done

# completions are actually populated
python3 - <<'PY'
import json, glob
for f in sorted(glob.glob('/var/log/inference/*/*/*.res.*.json')):
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

Note the fleet runs this on `unslothai/llama.cpp @ glm5next/upstream` serving
GLM-5.3-Flash, not on the stock upstream this was exercised against. The patch
applies cleanly there and touches no code the glm5next work modifies, but "applies
and compiles" is not "behaves identically" - a first run on that base is worth
checking on its own terms.

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
