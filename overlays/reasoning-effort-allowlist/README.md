# Overlay: reasoning-effort allowlist

**Branch:** `feat/reasoning-effort-allowlist`
**Scope:** fleet-wide, inert unless activated
**Activation:** `--reasoning-effort-levels a,b,c` (off by default; empty = stock)
**Status:** implemented

## What it does

Adds `--reasoning-effort-levels`, a declared vocabulary of `reasoning_effort` values
the deployed model's chat template actually distinguishes. When set, a request
carrying any other value is rejected with a 400 naming the supported levels, instead
of the template silently folding it to a fallback.

Chat templates commonly clamp rather than raise - GLM-5.3's is
`reasoning_effort if reasoning_effort in ['low','high'] else 'max'`, so a caller
asking for `minimal` silently got `max`, the most expensive mode. A silently
remapped setting is indistinguishable from an honored one at the call site.

Enforced at the single point where every client surface converges before template
rendering: top-level OAI `reasoning_effort`, `chat_template_kwargs`, the Responses
API `reasoning.effort`, converted Anthropic requests, and the server-side
`--reasoning-effort` default. One check, all adapters.

`none` is a listable level: the OAI path maps `reasoning_effort: "none"` to
"disable thinking", which a model that always thinks cannot honor - listing it only
for models that can actually not-think turns that silent lie into an error too.

## Files touched

```
common/common.h                   reasoning_effort_levels on common_params
common/arg.cpp                    the flag
tools/server/server-common.h      field on server_chat_params
tools/server/server-common.cpp    validation at the choke point
tools/server/server-context.cpp   wiring into chat_params
```

## Verified

stories260K, all through `/v1/chat/completions`:

| case | result |
|---|---|
| no flag, any effort | 200 (stock passthrough) |
| flag set, listed level (top-level or kwargs) | 200 |
| flag set, unlisted level (top-level or kwargs) | 400 naming supported levels |
| flag set without `none`, effort=none | 400 |
| flag set with `none`, effort=none | 200, thinking disabled |

## Removal condition

Drop if upstream grows an equivalent declared-vocabulary rejection, or if templates
start raising on unknown levels themselves.
