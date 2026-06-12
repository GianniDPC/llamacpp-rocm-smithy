# llamacpp-rocm-smithy

[![Latest release](https://img.shields.io/github/v/release/GianniDPC/llamacpp-rocm-smithy)](https://github.com/GianniDPC/llamacpp-rocm-smithy/releases/latest)
[![License](https://img.shields.io/github/license/GianniDPC/llamacpp-rocm-smithy)](LICENSE)
[![ROCm 7](https://img.shields.io/badge/ROCm-7-blue?logo=amd)](https://github.com/ROCm/TheRock)
[![Powered by llama.cpp](https://img.shields.io/badge/Powered%20by-llama.cpp-blue?logo=llama)](https://github.com/ggerganov/llama.cpp)
[![kernel-anvil](https://img.shields.io/badge/kernel--anvil-Smithy-orange)](https://github.com/apollosenvy/kernel-anvil)

Fresh builds of **llama.cpp** with **AMD ROCm 7** acceleration and the **kernel-anvil (Smithy)** MMVQ runtime config patch baked in.

This is a derivative of [lemonade-sdk/llamacpp-rocm](https://github.com/lemonade-sdk/llamacpp-rocm), with the kernel-anvil patch from [apollosenvy/kernel-anvil](https://github.com/apollosenvy/kernel-anvil) applied at build time. The resulting binaries load shape-specific MMVQ kernel configs at runtime via the `SMITHY_CONFIG` env var.

## Why this exists

[lemonade-sdk/llamacpp-rocm](https://github.com/lemonade-sdk/llamacpp-rocm) provides nightly AMD ROCm builds of vanilla llama.cpp. The kernel-anvil tool profiles GQA/FFN layer shapes on your actual GPU and emits JSON configs that tune `nwarps` / `rows_per_block` for each `(quant, N, K)` cell. llama.cpp needs a small patch (~15 lines) to load those configs at runtime. This repo combines both: it builds llama.cpp with the patch applied, so the resulting binaries are kernel-anvil-ready out of the box.

**Real-world impact** (RDNA3, gfx1100):
- Gemma 4 26B Q4_K_XL: 33.6 → 128.4 t/s (1.96× decode speedup)
- Qwen3 27B IQ4_NL: 2.36× on hot shapes
- Qwen3 35B A3B Q5_K_M: 6.37× on Q6_K (512, 2048)

## Supported Devices

This build targets **gfx1100** (RDNA3, e.g. RX 7900 XTX). The CI cross-compiles for this target only. To extend, edit the `GFX_TARGETS` env var in `build-smithy.yml` and re-queue a build.

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

## Required runtime env vars (Windows only)

The bundled ROCm 7.14 runtime needs a few env vars on Windows for COMGR to find the right code objects. (Linux does not need these — `amdgpu` reports gfx version consistently.)

```powershell
$env:HIP_VISIBLE_DEVICES="1"        # iGPU is index 0, dGPU is 1
$env:HSA_ENABLE_SDMA="0"            # SDMA is unstable on TheRock Windows
$env:HSA_OVERRIDE_GFX_VERSION="11.0.0"  # Adrenalin reports gfx version inconsistently
$env:ROCBLAS_USE_HIPBLASLT="1"
$env:SMITHY_CONFIG="H:/path/to/your/model.json"  # optional: kernel-anvil JSON
```

## Automated Builds

`.github/workflows/build-smithy.yml` runs:
- **Every 2 days** at 13:00 UTC (odd days of the month) — picks up the latest TheRock nightly + llama.cpp master
- **On push to main** when `patches/smithy.patch` or the workflow file changes
- **On pull requests** — verifies the build works
- **Manual** via `workflow_dispatch` — for testing specific ROCm or llama.cpp versions

Builds cross-compile on `windows-2022` and `ubuntu-22.04` GitHub-hosted runners (no GPU in CI). Releases are published automatically as `smithy-b####` tags.

## Patch Details

The patch in `patches/smithy.patch` contains 3 changes against llama.cpp master:

1. **`ggml/src/ggml-cuda/smithy-config.h`** (new file) — runtime config loader. Supports `SMITHY_CONFIG` env var and `~/.cache/smithy/<model>.json` lookup.

2. **`ggml/src/ggml-cuda/mmvq.cu`** (1 hunk) — adds `#include "smithy-config.h"` and a smithy override block inside `should_use_small_k` to force the small_k path on RDNA3 when the profile says `rows_per_block > 1`.

3. **`ggml/src/ggml-cuda/ggml-cuda.cu`** (2 hunks) — TheRock-specific workaround for `hipMemGetInfo` returning errors or zeros on Windows + iGPU/dGPU mixed systems.

The patch is regenerated against the current `master` HEAD on every release. If llama.cpp master moves the relevant code, the `git apply --check` step in the CI fails and the patch needs to be regenerated manually.

## Manual Build

See [docs/manual_instructions.md](docs/manual_instructions.md).

## Attribution

- [lemonade-sdk/llamacpp-rocm](https://github.com/lemonade-sdk/llamacpp-rocm) — the original CI pipeline this is derived from
- [ROCm/TheRock](https://github.com/ROCm/TheRock) — ROCm nightly tarball source
- [ggerganov/llama.cpp](https://github.com/ggerganov/llama.cpp) — the inference engine
- [apollosenvy/kernel-anvil](https://github.com/apollosenvy/kernel-anvil) — the runtime shape tuner

## License

MIT — see [LICENSE](LICENSE).
