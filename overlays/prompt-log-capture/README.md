# Overlay: prompt-log capture

**Branch:** `feat/prompt-log-capture`
**Base:** `upstream/master` (see Extracting the patch)
**Scope:** fleet-wide, but inert unless activated
**Status:** implemented, not yet deployed

## What it does

Turns `--log-prompts-dir` into an archivable record of what the server actually
served. Upstream's version writes only the rendered prompt, into a file named
after the current millisecond. This overlay writes the full request envelope and
the completion, as two JSON documents keyed on the completion id.

```
<completion_id>.req.json          written before the task is dispatched
<completion_id>.res.<index>.json  written when the task finishes
```

## Why

Requested by the agent-ledger team, who need to reconstruct and replay turns
without putting a preservation component in the serving path. A reverse proxy
was considered and rejected: a proxy failure costs inference mid-turn, whereas a
dropped log record costs only an archive entry. On-disk records keep
preservation strictly downstream of serving.

Rationale and the full field inventory live in the fork's
`plans/001-prompt-log-capture.md` on `master`.

## Activation

**Off by default.** A deployment that sets nothing behaves exactly as stock:

```bash
llama-server -m <model> --log-prompts-dir /var/log/llama-prompts
```

The only behaviour change for someone already using the flag is the record
format and the filenames. Nothing else in the server is affected.

## Record format

`*.req.json`

| field | meaning |
|---|---|
| `id` | completion id, also the filename stem and the join key |
| `timestamp` | `ggml_time_ms()` at request time |
| `model` | resolved model name |
| `request` | the full request object: rendered prompt, sampling params, tools, and any extra client fields the oaicompat parser passes through |

`*.res.<index>.json`

| field | meaning |
|---|---|
| `id`, `timestamp` | as above |
| `response.content` | the completion text |
| `response.message` | parsed message: `content`, `reasoning_content`, `tool_calls` |
| `response.prompt` | prompt as seen by the slot |
| `response.generation_settings` | full sampling configuration |
| `response.tokens_predicted` / `tokens_evaluated` | generated / prompt tokens |
| `response.tokens_cached` / `tokens_prompt_cache` | cache reuse |
| `response.stop_type`, `stopping_word`, `truncated` | stop condition |
| `response.timings` | slot timing stats |

The shape does not vary with endpoint, streaming mode, or a client-supplied
`response_fields` - that is why it uses a dedicated `to_json_log()` rather than
the existing `to_json()`, whose output changes with all three.

### Properties the archiver depends on

- **Write-once, never reopened.** No in-place mutation, no rename.
- **Truncation is self-detecting** - a torn write is invalid JSON.
- **Content-free filenames.** The id is server-generated; nothing user-derived
  reaches a path, since paths reach object keys and logs.
- **Failure is soft.** A write error is logged and ignored; capture never breaks
  serving. A full disk degrades to no-capture, not no-inference - provided the
  directory is on its own filesystem.
- **A `.req` with no `.res`** means the server died mid-request. Unambiguous,
  not a torn record.

## Files touched

```
common/arg.cpp                    directory mode + help text
tools/server/server-context.cpp   write helper, request record, response hook
tools/server/server-task.cpp      to_json_log()
tools/server/server-task.h        declaration
```

Four files, ~209 patch lines. No overlap with any other overlay in this fork.

## Extracting the patch

Diff against the branch's merge-base, never against `upstream/master` directly -
the branch deliberately lags upstream, and diffing the tip would revert every
upstream commit in between. Exclude `overlays/`, which is fork documentation and
must not reach a stock tree:

```bash
BASE=$(git merge-base upstream/master feat/prompt-log-capture)
git diff "$BASE"..feat/prompt-log-capture -- . ':(exclude)overlays/' > prompt-log-capture.patch
```

Verify before shipping:

```bash
git switch --detach upstream/master
git apply --check prompt-log-capture.patch
```

This overlay changes C++ and therefore requires a rebuild. It cannot be
delivered as a file overlay onto a stock image.

## Verification performed

Built and exercised against `stories260K`:

| case | result |
|---|---|
| OAI `/v1/chat/completions`, non-streaming | completion captured |
| OAI `/v1/chat/completions`, streaming | completion captured |
| Anthropic `/v1/messages`, streaming | completion captured |
| `n=2` parallel completions | two sibling files, distinct indices, no overwrite |
| all records | valid JSON, paired by id |
| permissions | directory `0700`, files `0600` |
| client `session_id` passthrough | survives into `request` on the OAI path |

One defect was found this way and fixed: in streaming mode the final result
carries only the last delta, so reading `content` naively recorded an **empty
completion for every streamed request**. `to_json_log()` falls back to the
accumulated message content. Compilation alone would not have caught this.

## Known gaps

- **No rotation.** Growth is unbounded, and because the rendered prompt is
  logged, every turn writes the whole conversation - roughly O(n^2) in
  conversation length, independent of cache reuse. Multimodal base64 is kept
  inline deliberately, so records stay replayable, and adds to this. Bound it
  outside the server: dedicated filesystem, plus `systemd-tmpfiles` or a sweep
  keyed on completed `.res` files. Never prune a `.req` whose `.res` has not
  landed - a slow request leaves one sitting alone legitimately.
- **Writes are synchronous** on the request thread, before dispatch. Accepted
  for NVMe-backed storage; revisit for slower or shared storage.
- **Anthropic `metadata.user_id` is dropped** by
  `server_chat_convert_anthropic_to_oai`, which builds a fresh object and copies
  only `system`, `messages`, `thinking`. Anthropic clients therefore cannot
  inject a session id. ~3 lines to fix if needed.
- **No inbound header capture.**

## Removal condition

Drop this overlay if any of:

- upstream ships equivalent request/response capture
- the agent-ledger inference adapter is retired
- capture moves to a mechanism outside the server

The filename-collision fix is a genuine upstream bug fix and could be submitted
separately from the record-format change, which is more opinionated. Splitting
them keeps the carried surface smaller.
