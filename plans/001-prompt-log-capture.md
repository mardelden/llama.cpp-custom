# Plan: Capture full request and completion records in the prompt log

**Status:** Proposed
**Date:** 2026-08-30
**Snapshot:** `9723942ad`
**Requested by:** agent-ledger team, via
`/Users/mardel/src/agent-ledger/plans/handovers/reply-inference-prompt-log-capture.md`

## Context

The agent-ledger team needs an archivable record of what this inference host actually
served: the rendered prompt as sent, the completion, and enough of the request envelope to
replay it. They will run an out-of-band agent on the LXC that tails the log directory. They
explicitly **rejected a reverse proxy** - not for performance, but for blast radius: a proxy
puts a preservation component in the interactive serving path, where a hiccup costs
inference mid-turn rather than costing an archive entry.

`llama-server` already has `--log-prompts-dir PATH` (`common/arg.cpp:3895`,
`common/common.h:517`), which writes one file per request. It is not sufficient as shipped.

### What the existing feature does

At `tools/server/server-context.cpp:4254`, inside `handle_completions_impl`:

```cpp
const auto file_path = std::filesystem::path(params.path_prompts_log_dir)
                     / string_format("%012" PRId64 ".txt", ggml_time_ms());
std::ofstream f(file_path);
if (f) {
    f << (prompt.is_string() ? prompt.get<std::string>().c_str() : prompt.dump(2).c_str());
}
```

Verified properties:

| Property | Status |
|---|---|
| Rendered prompt as sent (post chat-template) | **yes** - templating happens upstream at `server-common.cpp:1340` |
| Covers Anthropic `/v1/messages` | **yes** - converted to OAI at `server-context.cpp:5013`, then same path |
| Completion | **no** - the write happens before inference is dispatched |
| Sampling params, model identity, token counts | **no** - only `data["prompt"]` is written |
| Request id in the record | **no** - `completion_id` exists 15 lines earlier (line 4239) but is unused |
| Write-once, no mutation, no rename | **yes** - single `ofstream`, closed at scope exit |
| Filename collision safety | **no** - see below |

### Two defects in the existing code

1. **Silent data loss.** The filename is `ggml_time_ms()` alone. Two requests in the same
   millisecond produce the same path and the second truncates the first. Under batched
   serving this is routine, and it fails silently.
2. **Permissions.** `create_directories` plus a default `ofstream` yields `0755`/`0644`
   (umask-dependent). These files are the most content-dense artifacts on the box - fully
   assembled prompts including system prompt and any source code pulled into context. The
   handover requires `0700`/`0600`.

## Approach

Three changes, one coherent unit:

1. Write the **whole request object** as JSON, not just the prompt string.
2. Write a **second file** carrying the completion, on task finalisation.
3. Key both filenames on **`completion_id`** rather than a timestamp.

### Why the full request object

`oaicompat_chat_params_parse` ends with a passthrough loop (`server-common.cpp:1395`):

```cpp
for (const auto & item : body.items()) {
    if (!llama_params.contains(item.key()) || item.key() == "n_predict") {
        llama_params[item.key()] = item.value();
    }
}
```

Unknown client fields survive into `data`. So logging `data` rather than `data["prompt"]`
captures sampling params, model, tools, **and any client-injected session id** - closing
handover requirements 3, 4, 5, 9 and 10 in a single line change, with no new plumbing.

### Why two files rather than one appended file

The handover accepts either append-only or write-once-then-closed, but breaks on in-place
mutation. Two write-once files avoid the question entirely:

```
<completion_id>.req.json    written before inference dispatch
<completion_id>.res.json    written on task finalisation
```

- No mutation, no rename, no shared handle, therefore no locking between request threads.
- Filenames are deterministic and content-free; the filename **is** the join key.
- Truncation is self-detecting: a torn write yields unparseable JSON.
- Failure degrades legibly - a `.req` with no `.res` means the server died mid-request,
  which is unambiguous rather than a torn record.

JSONL into one shared file was rejected: with synchronous writes from multiple request
threads it needs locking on a shared handle, reintroducing the hot-path coupling that
killed the proxy option.

### Why `completion_id`

`completion_id` (`gen_chatcmplid()`, `"chatcmpl-" + random_string()`) is generated at
`server-context.cpp:4239`, stored as `task.params.oaicompat_cmpl_id` at line 4301, and
emitted as `"id"` in every response variant including streaming chunks and the Anthropic
message shape (`server-task.cpp:400,448,480,496,508,780,1094,1127,1336`).

Using it as the filename fixes the collision defect **and** supplies the join key the
harness can correlate against, in the same change.

## Files to Modify

| File | Change |
|------|--------|
| `tools/server/server-context.cpp` | Write `data` as JSON to `<completion_id>.req.json`; add the `.res.json` write at finalisation |
| `common/arg.cpp` | Update `--log-prompts-dir` help text; create the directory `0700` |
| `tools/server/README.md` | Document the file pair, the JSON shape, and the absence of rotation |

## Implementation Details

### 1. Request record

Replace the body of the existing `if (!params.path_prompts_log_dir.empty())` block at
`server-context.cpp:4254`. Keep it in the same place - before dispatch, so a request is
recorded even if inference then fails.

- Filename `completion_id + ".req.json"`; `completion_id` is already in scope (line 4239).
- Serialise `data` with `dump()`. `data` is already an `nlohmann::json`.
- Wrap in an envelope carrying what `data` does not hold: the id, a UTC request timestamp,
  and the resolved model name (`meta->model_name`).

### 2. Completion record

Hook `server_task_result_cmpl_final` (`server-task.h:320`). This is the right seam because
it is produced in **both** streaming and non-streaming modes, and already carries almost
everything required:

| Handover field | Source on the struct |
|---|---|
| Completion text | `content` |
| Reasoning / thinking content | `oaicompat_msg.reasoning_content` (`common/chat.h:85`) |
| Tool calls | `oaicompat_msg.tool_calls` |
| Sampling params | `generation_params` |
| Token counts incl. cache reuse | `n_decoded`, `n_prompt_tokens`, `n_prompt_tokens_cache`, `n_tokens_cached` |
| Stop reason | `stop`, `stopping_word` |
| Join key | `oaicompat_cmpl_id` |
| Model | `oaicompat_model` |

`n_tokens_cached` matters specifically: the team measured 94-96% cache reuse, so naive
reading of `n_prompt_tokens` would misrepresent what was actually prefilled.

Writing one `.res.json` per finalised task covers both modes without touching the streaming
partial path.

### 3. Filenames and permissions

- `<completion_id>.req.json` / `<completion_id>.res.json`, flat in the log directory.
- `completion_id` is server-generated and content-free - nothing user-derived reaches a
  filename, which the handover requires because filenames propagate to object keys and logs.
- Create the directory `0700`; create files `0600`. This is in scope for this change, not a
  follow-up. Today `create_directories` plus a default `ofstream` gives `0755`/`0644`
  (umask-dependent), which does not meet the handover's containment requirement. Set the mode
  explicitly rather than relying on the server's umask, since that is environment-dependent
  and would silently regress under systemd.

## Edge Cases

- **Multimodal payloads: keep inline (decided).** `data["prompt"]` may carry
  `multimodal_data` as base64 (`server-common.cpp:962`), so dumping `data` wholesale writes
  base64 images into the log. That is intended - the record stays complete and replayable,
  and eliding would make a request unreproducible. The cost is size: base64 inflates ~33%
  over the raw image, and these bytes repeat on every turn of a conversation because the
  rendered prompt is re-logged each time. Fold this into the volume measurement below rather
  than treating it separately.
- **`n_cmpl > 1`.** Child tasks are added at `server-context.cpp:4305`. Multiple finals may
  share one `completion_id`; the `.res.json` write must not have sibling completions
  overwrite each other. Suffix with the child index.
- **Aborted or errored requests.** A `.req.json` with no matching `.res.json`. Intentional
  and unambiguous; the archiver treats it as an incomplete pair, not a torn file.
- **Disk full.** The write is synchronous on the request thread, so a full disk stalls
  inference. Accepted for now (NVMe, low expected latency) - see Deferred.
- **No rotation exists.** Growth is unbounded, and because the *rendered* prompt is logged,
  every turn writes the full conversation - roughly O(n^2) in conversation length,
  independent of cache reuse. The handover is explicit that the archiver will never delete
  source data, so bounding growth stays on this side. Measure bytes/hour under real load
  before enabling continuously.

## Open Questions

- **Rotation window.** The team's archive lag target is 15 minutes and they want the
  rotation window declared explicitly so they can alert when lag approaches it. No rotation
  mechanism exists today; deleting on a timer would recreate the composed-policies failure
  that destroyed six months of transcripts in May.
- **Upstreamable?** The collision fix is a genuine bug fix and plausibly PR-able on its own.
  The record-format change is more opinionated and probably fork-local. Splitting them makes
  the bug fix independently submittable.

## Deferred (agreed, not oversights)

- **Asynchronous writes.** The write is synchronous on the serving thread and stays that
  way for now - NVMe-backed, and the record is small relative to inference time. Revisit if
  the log directory ever lands on slower or shared storage.
- **Dedup / prefix-diffing.** Stays out of llama.cpp entirely. The server writes raw and
  immutable; agent-ledger maintains a deduped view as an incrementally-updated projection.
  Keeping raw and projection separate means a bug in the dedup logic is recoverable - the
  raw archive is the thing that cannot be reconstructed.
- **Session-id correlation.** Deferred until both sides are instrumented and the actual gap
  is visible. Ranked options, cheapest first:
  1. Harness records the response `id` - **zero server changes**, works on both APIs today
  2. OAI clients inject a custom field - already survives the passthrough loop, needs only
     the `data` change above
  3. Anthropic `metadata.user_id` - currently **dropped**, because
     `server_chat_convert_anthropic_to_oai` (`server-chat.cpp:334`) builds a fresh
     `json oai_body` and copies only `system`, `messages`, `thinking`. ~3 lines to preserve
  4. Inbound HTTP headers - nothing exists; most invasive, skip unless 1-3 all fail
- **Continuous enablement.** The team asked for a time-boxed run with a disk quota and a
  scheduled disable, **not** continuous logging, until their Phase 1 storage exists. Nothing
  drains the directory yet.

## Related

- `plans/decisions/002-fork-branching-strategy.md` - this lands on a topic branch;
  `master` stays a mirror
- `.claude/skills/fork-workflow/SKILL.md` - branch, rebase and commit conventions
