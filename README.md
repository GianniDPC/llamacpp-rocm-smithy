# llamacpp-rocm-smithy

[![Latest release](https://img.shields.io/github/v/release/GianniDPC/llamacpp-rocm-smithy)](https://github.com/GianniDPC/llamacpp-rocm-smithy/releases/latest)
[![License](https://img.shields.io/github/license/GianniDPC/llamacpp-rocm-smithy)](LICENSE)
[![ROCm 7](https://img.shields.io/badge/ROCm-7-blue?logo=amd)](https://github.com/ROCm/TheRock)
[![Powered by llama.cpp](https://img.shields.io/badge/Powered%20by-llama.cpp-blue?logo=llama)](https://github.com/ggerganov/llama.cpp)
[![kernel-anvil](https://img.shields.io/badge/kernel--anvil-Smithy-orange)](https://github.com/apollosenvy/kernel-anvil)

Fresh **every 2 days** builds of **llama.cpp** with **AMD ROCm 7** acceleration and the **kernel-anvil (Smithy)** MMVQ runtime config patch baked in.

This is a direct derivative of [lemonade-sdk/llamacpp-rocm](https://github.com/lemonade-sdk/llamacpp-rocm), with the kernel-anvil (Smithy) patch from [apollosenvy/kernel-anvil](https://github.com/apollosenvy/kernel-anvil) applied at build time. The resulting binaries load shape-specific MMVQ kernel configs at runtime via the `SMITHY_CONFIG` env var.

## Why this exists

`lemonade-sdk/llamacpp-rocm` provides nightly AMD ROCm builds of vanilla llama.cpp. The kernel-anvil tool profiles GQA/FFN layer shapes on your actual GPU and emits JSON configs that tune `nwarps` / `rows_per_block` for each `(quant, N, K)` cell. llama.cpp needs a small patch (~15 lines) to load those configs at runtime. This repo combines both: it builds llama.cpp with the patch applied, so the resulting binaries are kernel-anvil-ready out of the box.

**Real-world impact** (RX 7900 XTX):
- Gemma 4 26B Q4_K_XL: 33.6 → 128.4 t/s (1.96× decode speedup)
- Qwen3 27B IQ4_NL: 2.36× on hot shapes
- Qwen3 35B A3B Q5_K_M: 6.37× on Q6_K (512, 2048)

## Supported Devices

This build targets **gfx1100** (Radeon RX 7900 XTX / RDNA3). The CI cross-compiles for this target only. To extend, edit the `GFX_TARGETS` env var in `build-smithy.yml` and re-queue a build.

Compatible GPUs in the gfx110X family (would require rebuilding with `-DGPU_TARGETS="gfx1100;gfx1101;gfx1102;gfx1103"`):
- PRO W7900 / W7800 / W7700 / W7600
- RX 7900 XTX / XT / GRE
- RX 7800 XT, RX 7700 XT / 7700
- RX 7600 XT / 7600
- Radeon 780M / 760M / 740M (iGPUs)

## Quick Start

1. **Download** the latest build for your OS from the [releases page](https://github.com/GianniDPC/llamacpp-rocm-smithy/releases/latest).
2. **Extract** to a directory of your choice.
3. **Profile a model** (one-time per model + GPU combination):
   ```bash
   pip install kernel-anvil
   kernel-anvil gguf-optimize ~/Models/my-model.gguf
   # writes ~/.cache/smithy/my-model.json
   ```
4. **Run** llama-server with the optimized config:
   ```bash
   ./llama-server.exe -m ~/Models/my-model.gguf -ngl 999
   # On startup: "kernel-anvil: loaded 6 shape configs from ~/.cache/smithy/my-model.json"
   ```

All releases include the complete ROCm 7 runtime — no separate ROCm installation required.

## Required runtime env vars (Windows, TheRock nightly)

```powershell
$env:HIP_VISIBLE_DEVICES="1"        # mask iGPU, force dGPU (TheRock bug workaround)
$env:HSA_ENABLE_SDMA="0"            # SDMA unstable on TheRock Windows
$env:HSA_OVERRIDE_GFX_VERSION="11.0.0"  # force gfx1100 codepath
$env:ROCBLAS_USE_HIPBLASLT="1"
$env:SMITHY_CONFIG="H:/path/to/your/model.json"  # path to kernel-anvil JSON
```

## Automated Builds

GitHub Actions workflow `.github/workflows/build-smithy.yml` runs:
- **Every 2 days** at 13:00 UTC (odd days of the month) — picks up the latest TheRock nightly + llama.cpp master
- **On push to main** when `patches/smithy.patch` or the workflow file changes
- **On pull requests** — verifies the build works
- **Manual** via `workflow_dispatch` — for testing specific ROCm or llama.cpp versions

Builds cross-compile on `windows-latest` and `ubuntu-22.04` GitHub-hosted runners. No GPU required in CI. Releases are published automatically as `smithy-b####` tags.

## Architecture

```
┌────────────────────────────────────────────────────┐
│  lemonade-sdk/llamacpp-rocm (build chain)          │
│  + apollosenvy/kernel-anvil (Smithy patch)         │
│  + this repo's patches/smithy.patch                │
└─────────────────────┬──────────────────────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │  GitHub Actions CI          │
        │  (cross-compile, no GPU)    │
        └──────────┬──────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  Release: smithy-b####           │
    │  - smithy-b1234-windows-...zip   │
    │  - smithy-b1234-ubuntu-...zip    │
    └──────────────────────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  User's GPU (RX 7900 XTX)        │
    │  - extracts zip                  │
    │  - sets env vars                 │
    │  - profiles model with anvil    │
    │  - runs llama-server             │
    └──────────────────────────────────┘
```

## Patch Details

The patch in `patches/smithy.patch` contains 3 modifications against llama.cpp commit `ac4cdde` (b9592):

1. **`ggml/src/ggml-cuda/smithy-config.h`** (new file, 331 lines) — runtime config loader, supports `SMITHY_CONFIG` env var and `~/.cache/smithy/<model>.json` lookup.

2. **`ggml/src/ggml-cuda/mmvq.cu`** (3 hunks):
   - Add `#include "smithy-config.h"`
   - In `calc_rows_per_block()`: when `kernel-anvil` profile says `rows_per_block > 1`, return 2 instead of 1
   - In `should_use_small_k` lambda: consult the Smithy config to force `small_k` path on RDNA3

3. **`ggml/src/ggml-cuda/ggml-cuda.cu`** (2 hunks) — **TheRock-specific workaround** for `hipMemGetInfo` returning errors or zeros on Windows + iGPU/dGPU mixed systems. Falls back to `hipGetDeviceProperties().totalGlobalMem` when the primary path fails. This is not in upstream kernel-anvil because it's specific to TheRock nightly Windows builds.

To regenerate the patch against a newer llama.cpp:
```bash
# In your local llama.cpp tree, with all 3 modifications applied:
cd /path/to/llama.cpp
git add -N ggml/src/ggml-cuda/smithy-config.h
git diff -- ggml/src/ggml-cuda/smithy-config.h \
           ggml/src/ggml-cuda/mmvq.cu \
           ggml/src/ggml-cuda/ggml-cuda.cu > /path/to/this/repo/patches/smithy.patch
```

## Updating the Patch

When llama.cpp master changes the relevant code, the `Apply kernel-anvil (Smithy) patch` step in the CI will fail. To recover:

1. Apply the current `patches/smithy.patch` to your local llama.cpp at the new commit
2. Resolve any conflicts manually
3. Re-generate the patch using the command above
4. Commit the new patch to this repo
5. Update the `LLAMACPP_VERSION` workflow input default (or override per-run) to match the new commit

## Manual Build

See [docs/manual_instructions.md](docs/manual_instructions.md).

## Attribution

- [lemonade-sdk/llamacpp-rocm](https://github.com/lemonade-sdk/llamacpp-rocm) — the original CI pipeline this is derived from
- [ROCm/TheRock](https://github.com/ROCm/TheRock) — ROCm nightly tarball source
- [ggerganov/llama.cpp](https://github.com/ggerganov/llama.cpp) — the inference engine
- [apollosenvy/kernel-anvil](https://github.com/apollosenvy/kernel-anvil) — the runtime shape tuner

## License

MIT — see [LICENSE](LICENSE).
