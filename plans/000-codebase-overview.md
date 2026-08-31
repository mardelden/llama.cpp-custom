# Codebase Overview: llama.cpp-custom

**Status:** Implemented
**Date:** 2026-08-30
**Snapshot:** `9723942ad` (synced to `upstream/master`; `master` is 2 ahead, `plans/` only)

> This file exists because `CLAUDE.md` and `AGENTS.md` are both upstream-owned in this
> fork and must stay byte-identical to upstream. See
> [`decisions/001-docs-location.md`](decisions/001-docs-location.md) for the rationale.

## What this repo is

A personal fork of [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) - C/C++
inference for LLMs, built on the `ggml` tensor library.

| | |
|---|---|
| `origin` | `git@github.com:mardelden/llama.cpp-custom.git` |
| `upstream` | `git@github.com:ggml-org/llama.cpp.git` (push disabled) |
| Divergence | Synced 2026-08-30. **2 ahead** (`plans/` docs only), 0 behind |
| Language | C11 / C++17, CMake build |

**There are no local modifications to upstream code** - the only local commits add `plans/`.
Everything below describes upstream llama.cpp at `9723942ad`.

## AI usage policy (read this first)

Upstream **reversed** its AI policy. The current `AGENTS.md` states:

> AI-generated code is allowed. What is **not** allowed is submitting code you do not
> understand. You are 100% responsible for every line, however it was produced.

Key points for working here:

- **Private forks are exempt** from the contributor rules entirely. Local work in this
  fork is unconstrained.
- For anything headed upstream: you must fully understand it, be able to debug it
  independently, and discuss it with reviewers without AI help. Disclosure is mandatory
  when AI meaningfully contributed.
- **Hard prohibitions** (these cause PR closure or a contributor ban):
  - Never write PR descriptions, commit messages, or reviewer responses
  - Never run `git push` or `gh pr create` on the user's behalf
  - If asked to commit on the user's behalf: use `Assisted-by:`, never `Co-authored-by:`
- Upstream ships its own agent skills in `skills/` - **`skills/add-new-model/SKILL.md`**
  (guided model-porting workflow) and **`skills/code-review/SKILL.md`**. Prefer these over
  improvising.

### Style rules that apply to code and comments

- **ASCII only.** No em-dash, no `->` arrows as unicode, no `x`/`...` unicode forms. Use
  `-`, `->`, `x`, `...`
- Comments: 1-2 lines, written *after* the code, only where genuinely needed. Simplified
  Technical English. No hard-wrapping to a column width.
- Do not add files under `tests/` without maintainer approval.
- llama.cpp does **not** use Minja. It has its own Jinja engine in `common/jinja`.

## Build & Development Commands

### Core build

```bash
cmake -B build
cmake --build build --config Release -j 8
```

Binaries land in `build/bin/`, prefixed `llama-` (`llama-cli`, `llama-server`,
`llama-bench`, `llama-quantize`, ...). There is also a **unified `llama` binary** built
from `app/`, which links every tool implementation into one multi-call executable.

### Backend-enabled builds

Backends are opt-in via one `GGML_*` flag each:

```bash
cmake -B build -DGGML_CUDA=ON      # also GGML_METAL, GGML_VULKAN, GGML_SYCL, GGML_OPENCL,
                                   # GGML_HIP, GGML_CANN, GGML_WEBGPU, GGML_RPC, GGML_ZDNN, ...
```

`CMakePresets.json` has ready-made triples: `cmake --preset arm64-apple-clang-release`,
`x64-windows-vulkan-debug`, `x64-linux-gcc-debug`. See `docs/build.md` and `docs/backend/`.

### Useful configure flags

| Flag | Purpose |
|---|---|
| `LLAMA_BUILD_TESTS` / `_TOOLS` / `_EXAMPLES` / `_SERVER` | Default ON standalone; turn off to cut build time |
| `LLAMA_FATAL_WARNINGS=ON` | `-Werror` |
| `LLAMA_SANITIZE_ADDRESS` / `_THREAD` / `_UNDEFINED` | Sanitizer builds |
| `BUILD_SHARED_LIBS=OFF` | Static build |
| `LLAMA_CURL=OFF` / `LLAMA_OPENSSL=OFF` | Drop network deps |

### Tests

```bash
ctest --test-dir build --output-on-failure
ctest --test-dir build -L main      # self-contained, no model download
ctest --test-dir build -L model     # requires real GGUF files
```

Two CTest labels: **`main`** and **`model`**. Sources are `tests/test-*.cpp`, registered
via the `llama_build` / `llama_test` helpers in `tests/CMakeLists.txt`.

```bash
cd tools/server/tests && ./tests.sh     # server, pytest (SLOW_TESTS=1 for full)
cd tools/ui && npm test                 # webui: vitest + playwright
bash ci/run.sh ./tmp/results ./tmp/mnt  # full local CI (GG_BUILD_CUDA=1 etc.)
```

### Lint / format

```bash
pre-commit run --all-files    # trailing whitespace, EOF, YAML, flake8
```

- C/C++: `.clang-format` (clang-tools v15+), 4-space indent, brackets on same line
- Python: `.flake8` + `flake8-no-print`, mypy (`mypy.ini`), pyright, **`ty.toml`**
  (astral `ty` type checker)
- All files: `.editorconfig`, enforced by the `editorconfig.yml` workflow

### Python / conversion

```bash
python convert_hf_to_gguf.py <model-dir>     # HF -> GGUF
python convert_lora_to_gguf.py <lora-dir>    # LoRA -> GGUF
```

Conversion logic now lives in **`conversion/`** (~90 per-architecture Python modules,
`base.py` plus one file per model family) rather than one monolithic script.
`gguf-py/` remains the standalone `gguf` package (reader, writer, tensor mapping, quants,
vocab).

## Architecture Overview

```
ggml/          tensor library + backends   (ggml.c, ggml-backend.cpp, ggml-<backend>/)
  |
src/           libllama: model loading, graph building, KV cache, sampling, vocab
  |
common/        shared app utilities: arg parsing, chat templates, sampling, jinja
  |
tools/         end-user binaries (cli, server, bench, quantize, mtmd, ui, tuning, ...)
app/           unified multi-call `llama` binary linking all tool impls
examples/      minimal usage demos
```

Public API surface is small: `include/llama.h` (C API), `include/llama-cpp.h` (RAII
wrappers), `ggml/include/ggml.h` + `ggml-backend.h`.

### Module structure

| Path | Role |
|---|---|
| `ggml/src/ggml.c` | Tensor ops, graph construction |
| `ggml/src/ggml-backend-reg.cpp` | Backend registry - where every backend registers itself |
| `ggml/src/ggml-cpu/` | CPU backend, `arch/{x86,arm,riscv,powerpc,s390,loongarch,wasm}` SIMD kernels |
| `ggml/src/ggml-{cuda,metal,vulkan,sycl,opencl,cann,hexagon,webgpu,blas,rpc,...}/` | One dir per backend |
| `src/llama-arch.{h,cpp}` | `llm_arch` enum (**150 archs**) + KV/tensor-name mapping tables |
| `src/llama-model.cpp` | Model base class, arch dispatch (now ~3200 lines, was ~9000) |
| `src/models/` | **151 per-architecture files** - hparams, tensors and graph per model family |
| `src/llama-graph.{h,cpp}` | `llm_graph_context` - shared graph-building primitives |
| `src/llama-kv-cache*.cpp` | KV cache variants (standard, iSWA, recurrent, hybrid, dsv4) |
| `src/llama-vocab.cpp`, `unicode*.cpp` | Tokenizers (SPM/BPE/WPM/UGM/RWKV) |
| `src/llama-sampler.cpp` | Sampling chains |
| `common/jinja/` | llama.cpp's **own** Jinja engine (not Minja) |
| `common/chat*.cpp`, `common/peg-parser.cpp` | Chat templating and tool-call parsing |
| `tools/server/` | `llama-server`: HTTP API, router mode, slots |
| `tools/ui/` | Webui (SvelteKit + Tailwind + Storybook). **Moved here from `tools/server/webui`** |
| `tools/tuning/` | Backend/kernel tuning harness |
| `conversion/` | Per-architecture HF -> GGUF Python modules |
| `skills/` | Upstream-maintained agent skills (`add-new-model`, `code-review`) |
| `vendor/` | Vendored single-header deps, **generated** by `scripts/sync_vendor.py` |

## Key Patterns

- **Backend registry.** Backends self-register (`ggml_backend_<name>_reg()` in
  `ggml-backend-reg.cpp`) and can be compiled in or loaded dynamically
  (`ggml-backend-dl.cpp`). Adding a backend means implementing `ggml-backend-impl.h` and
  adding one `register_backend()` call - no changes under `src/`.

- **Arch enum + name-mapping tables.** Every model is an `LLM_ARCH_*` entry in
  `src/llama-arch.h`, paired with tables mapping arch -> GGUF metadata keys (`llm_kv`) and
  -> tensor names (`llm_tensor`). `LLM_TN` builds the tensor name strings.

- **One class per model family** (the current shape, after a large refactor). Each
  `src/models/<name>.cpp` defines:

  ```cpp
  struct llama_model_llama : public llama_model_base {
      void load_arch_hparams(llama_model_loader & ml) override;
      void load_arch_tensors(llama_model_loader & ml) override;

      template <bool embed>
      struct graph : public llm_graph_context {
          graph(const llama_model & model, const llm_graph_params & params);
      };
  };
  ```

  Hparams parsing, tensor allocation and graph construction for a model now live together
  in one file, declared in `src/models/models.h`. This is why `llama-model.cpp` shrank
  from ~9000 to ~3200 lines and `llama-arch.cpp` from ~2900 to ~1150.

- **Typed graph inputs.** `llm_graph_input_*` classes (`llama-graph.h`) encapsulate each
  runtime-varying input (positions, masks, KV state, cross-attention embeddings) and know
  how to fill their own tensors, keeping the decode path uniform.

- **Memory abstraction.** `llama_memory_i` implementations cover standard KV cache, iSWA
  (sliding window), recurrent (Mamba/RWKV state), hybrids and DSv4 - so attention and
  state-space models share one decode path.

- **Vendored deps are generated, not edited.** `vendor/` is reproduced by
  `scripts/sync_vendor.py`; `check-vendor.yml` fails the build if a re-run produces a diff.
  Each vendor subdir now carries its own `CMakeLists.txt`, and `vendor/hash/` absorbed what
  used to be `examples/gguf-hash/deps`.

## Critical Rules

1. **AI-generated code is allowed upstream, but you own every line.** Understand it, be
   able to debug and defend it without AI, and disclose meaningful AI contribution.
   **Private forks are exempt.** Never write PR text or reviewer replies; never push or
   open a PR on the user's behalf.

2. **Never hand-edit `vendor/`.** Edit `scripts/sync_vendor.py` and re-run it. CI verifies
   byte-for-byte.

3. **Never edit `CLAUDE.md` or `AGENTS.md` in this fork.** Both are upstream-tracked and
   byte-identical to `upstream/master`. See `decisions/001-docs-location.md`.

4. **ASCII only in code and comments.** No em-dash or unicode punctuation.

5. **Coding style** (CONTRIBUTING.md): `snake_case`; 4 spaces; brackets on same line;
   `void * ptr`, `int & a`; `struct foo {}` not `typedef struct`; sized ints (`int32_t`)
   in the public API; naming optimizes for longest common prefix (`number_small`, not
   `small_number`); enum values upper-case, prefixed with the enum name.

6. **Avoid new third-party dependencies**, fancy modern STL and heavy templates. Prefer
   reusing existing infrastructure over adding subsystems.

7. **ggml changes need `test-backend-ops`** across at least 2 backends; a new operator
   needs a new test case there.

8. **New model/feature PRs should be CPU-only first.** Other backends land in follow-ups.

9. **Tensor conventions.** Row-major; dim 0 = columns, 1 = rows, 2 = matrices.
   `ggml_mul_mat(ctx, A, B)` computes `C^T = A B^T`, i.e. `C = B A^T`.

10. **Never hack RoPE with a custom sin/cos implementation.** Past PRs doing this were
    closed. Use `ggml_rope_ext`; if it truly cannot express the model, open an issue first.

## Fork maintenance

```bash
git fetch upstream
git switch master && git rebase upstream/master
```

Rebase rather than `--ff-only`: `master` carries the `plans/` commits, so it is not an
ancestor of `upstream/master` and a fast-forward is impossible. The rebase is conflict-free
because those commits touch only `plans/`, a path upstream does not use.

Before syncing, confirm nothing else drifted onto `master`:

```bash
git diff --name-only $(git merge-base upstream/master master)..master
#   must list ONLY plans/, .claude/, overlays/**/DEPLOY.md, DEPLOYMENT-BRANCHES.md
```

All real work belongs on topic branches - see `decisions/002-fork-branching-strategy.md`.

## Adding a new model

Use **`skills/add-new-model/SKILL.md`** - it is upstream-maintained and more current than
the docs. The rough path:

1. `conversion/<name>.py` + `gguf-py/gguf/tensor_mapping.py` - conversion side
2. `src/llama-arch.h` / `.cpp` - new `LLM_ARCH_*`, KV keys, tensor names
3. `src/models/<name>.cpp` + declaration in `src/models/models.h` - hparams, tensors, graph
4. `tests/test-llama-archs.cpp` - coverage

Also read `docs/development/HOWTO-add-model.md`, and check `git log --oneline -- src/models`
for recent convention.

## Reference docs

| Topic | File |
|---|---|
| AI usage policy (**read first**) | `AGENTS.md` |
| Contributing, coding & naming guidelines | `CONTRIBUTING.md` |
| Upstream agent skills | `skills/add-new-model/`, `skills/code-review/` |
| Building, all platforms | `docs/build.md` |
| Per-backend build/setup | `docs/backend/` |
| Adding a model | `docs/development/HOWTO-add-model.md` |
| Debugging tests | `docs/development/debugging-tests.md` |
| Server internals | `tools/server/README-dev.md` |
| Local CI | `ci/README.md` |
| Op support matrix | `docs/ops.md` |
