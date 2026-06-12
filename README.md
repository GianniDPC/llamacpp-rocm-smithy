# llamacpp-rocm-smithy

[![Latest release](https://img.shields.io/github/v/release/GianniDPC/llamacpp-rocm-smithy)](https://github.com/GianniDPC/llamacpp-rocm-smithy/releases/latest)
[![License](https://img.shields.io/github/license/GianniDPC/llamacpp-rocm-smithy)](LICENSE)
[![ROCm 7](https://img.shields.io/badge/ROCm-7-blue?logo=amd)](https://github.com/ROCm/TheRock)
[![Powered by llama.cpp](https://img.shields.io/badge/Powered%20by-llama.cpp-blue?logo=llama)](https://github.com/ggerganov/llama.cpp)
[![kernel-anvil](https://img.shields.io/badge/kernel--anvil-Smithy-orange)](https://github.com/apollosenvy/kernel-anvil)

Fresh **every 2 days** builds of **llama.cpp** with **AMD ROCm 7** acceleration and the **kernel-anvil (Smithy)** MMVQ runtime config patch baked in.

This is a direct derivative of [lemonade-sdk/llamacpp-rocm](https://github.com/lemonade-sdk/llamacpp-rocm), with the kernel-anvil patch from [apollosenvy/kernel-anvil](https://github.com/apollosenvy/kernel-anvil) applied at build time. The resulting binaries load shape-specific MMVQ kernel configs at runtime via the `SMITHY_CONFIG` env var.

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
3. **Verify the build** (Windows):
   ```cmd
   test-llama.bat
   ```
   This sets the required env vars, runs a 1-token smoke test, and prints the exit code. See [Verifying a release](#verifying-a-release) below.
4. **Profile a model** (one-time per model + GPU combination):
   ```bash
   pip install kernel-anvil
   kernel-anvil gguf-optimize ~/Models/my-model.gguf
   # writes ~/.cache/smithy/my-model.json
   ```
5. **Run** llama-server with the optimized config:
   ```bash
   ./llama-server.exe -m ~/Models/my-model.gguf -ngl 999
   # On startup: "kernel-anvil: loaded 6 shape configs from ~/.cache/smithy/my-model.json"
   ```

All releases include the complete ROCm 7 runtime — no separate ROCm installation required.

## Verifying a release

Each Windows release ships with `test-llama.bat` in the `bin/` directory — a 1-token smoke-test harness that sets the right env vars and verifies the HIP kernel launches successfully. After extracting the zip:

```cmd
cd path\to\extracted\build
test-llama.bat
```

**What it does:** sets the 5 env vars from [Required runtime env vars](#required-runtime-env-vars-windows-therock-nightly), loads your model (default: `gemma-4-12b-it-UD-Q8_K_XL.gguf` — edit the `MODEL` line in the .bat if yours is in a different path), generates 1 token, exits.

**Expected output on success:** `Loading model...` followed by the model metadata (build hash, model name, modalities), then `[ Prompt: N t/s | Generation: 1000000.0 t/s ]` and `Exiting...`. Exit code 0.

**On failure:** if you see `ROCm error: device kernel image is invalid`, the env vars didn't take effect — try running the .bat from a fresh shell, or set them by hand.

The .bat is a smoke test (1 token), not a chat session. To actually chat, edit `test-llama.bat` and uncomment the `llama-server.exe` or `-i` lines at the bottom.

## Required runtime env vars (Windows, TheRock nightly)

The CI bundles ROCm 7.14 (from TheRock nightly) — no separate AMD driver install needed. The HIP runtime has two known quirks on Windows that need env vars to work around:

1. **Adrenalin kernel-mode driver (`amdkmpfd.sys`) reports gfx version inconsistently** for RDNA3 silicon. The compiled code object is for `gfx1100`, but the driver sometimes reports `gfx1101` or a SKU-specific variant. HIP's COMGR then can't match the code object and crashes with `ROCm error: device kernel image is invalid`. **`HSA_OVERRIDE_GFX_VERSION=11.0.0` forces the gfx1100 codepath**. (Linux's `amdgpu` driver reports consistently, so this is Windows-only.)
2. **SDMA is unstable on TheRock Windows** — async GPU→CPU transfers can hang. `HSA_ENABLE_SDMA=0` falls back to slower but reliable paths.
3. **`HIP_VISIBLE_DEVICES=1`** masks the iGPU (index 0 is usually the iGPU on a dGPU+iGPU system, 1 is the dGPU). Optional if you don't have an iGPU.

```powershell
$env:HIP_VISIBLE_DEVICES="1"
$env:HSA_ENABLE_SDMA="0"
$env:HSA_OVERRIDE_GFX_VERSION="11.0.0"
$env:ROCBLAS_USE_HIPBLASLT="1"
$env:SMITHY_CONFIG="H:/path/to/your/model.json"  # optional: kernel-anvil JSON
```

If `HSA_OVERRIDE_GFX_VERSION=11.0.0` doesn't help, your Adrenalin driver may be reporting something exotic. Try `12.0.0` (RDNA4) or other values. Or roll back the Adrenalin driver.

## Automated Builds

GitHub Actions workflow `.github/workflows/build-smithy.yml` runs:
- **Every 2 days** at 13:00 UTC (odd days of the month) — picks up the latest TheRock nightly + llama.cpp master
- **On push to main** when `patches/smithy.patch` or the workflow file changes
- **On pull requests** — verifies the build works
- **Manual** via `workflow_dispatch` — for testing specific ROCm or llama.cpp versions

Builds cross-compile on **`windows-2022`** and **`ubuntu-22.04`** GitHub-hosted runners. No GPU required in CI. Releases are published automatically as `smithy-b####` tags.

### Why `windows-2022` and not `windows-latest`

`windows-latest` currently maps to a Windows Server image with **Visual Studio 18 (MSVC 14.51)**. MSVC 14.51's `<cmath>` declares the C99 math comparison functions (`isgreater`, `isgreaterequal`, `isless`, `islessequal`, `islessgreater`, `isunordered`) as `__host__ __device__` via `_CLANG_BUILTIN2`. TheRock 7.14's bundled Clang 23 headers (`__clang_cuda_math_forward_declares.h`) try to redeclare them as just `__device__` — and `__device__` cannot overload `__host__ __device__`, so every `.cu` file fails to compile with ~20 errors.

`windows-2022` ships **Visual Studio 17 (MSVC 14.40)**, whose `<cmath>` doesn't have the `_CLANG_BUILTIN2` redeclarations, so the conflict doesn't happen. Same cmake flags, same TheRock tarball, same source — just an older MSVC.

The fix is upstream — either TheRock adjusts their HIP headers or AMD updates the Adrenalin driver / bundled clang version to not pull in MSVC 14.51's `<cmath>` redeclarations. Until then, `windows-2022` is the workaround.

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
        │  windows-2022 + ubuntu-22.04│
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
    │  - runs test-llama.bat          │
    │  - sets env vars                 │
    │  - profiles model with anvil    │
    │  - runs llama-server             │
    └──────────────────────────────────┘
```

## Patch Details

The patch in `patches/smithy.patch` is regenerated against the current `master` HEAD on every CI run. It contains 3 file changes:

1. **`ggml/src/ggml-cuda/smithy-config.h`** (new file, ~330 lines) — runtime config loader. Supports `SMITHY_CONFIG` env var and `~/.cache/smithy/<model>.json` lookup.

2. **`ggml/src/ggml-cuda/mmvq.cu`** (1 hunk) — adds `#include "smithy-config.h"` and inserts a smithy override block inside the `should_use_small_k` lambda that forces the `small_k` path on RDNA3 when the profile says `rows_per_block > 1`.

3. **`ggml/src/ggml-cuda/ggml-cuda.cu`** (2 hunks) — **TheRock-specific workaround** for `hipMemGetInfo` returning errors or zeros on Windows + iGPU/dGPU mixed systems. Falls back to `hipGetDeviceProperties().totalGlobalMem` when the primary path fails. This is not in upstream kernel-anvil because it's specific to TheRock nightly Windows builds.

### Patch is master-specific

The patch is regenerated against `master` HEAD on every CI run. It **will not apply cleanly** to other commits — the `should_use_small_k` lambda structure differs between versions. If you need to build against a specific llama.cpp commit, regenerate the patch manually against that commit (instructions below).

## Updating the Patch

When llama.cpp master changes the relevant code, the `Apply kernel-anvil (Smithy) patch` step in the CI will fail with a hunk mismatch. To recover:

1. `git checkout master && git pull` in your local llama.cpp clone
2. Apply the current `patches/smithy.patch` to your local llama.cpp
3. Resolve any conflicts manually
4. Re-generate the patch:
   ```bash
   # In your local llama.cpp tree, with all modifications applied:
   git add -N ggml/src/ggml-cuda/smithy-config.h
   git diff -- ggml/src/ggml-cuda/smithy-config.h \
              ggml/src/ggml-cuda/mmvq.cu \
              ggml/src/ggml-cuda/ggml-cuda.cu > /path/to/this/repo/patches/smithy.patch
   ```
5. Commit the new patch to this repo and push to main
6. The path-filter on `patches/smithy.patch` triggers a fresh build automatically — no other config changes needed

## Manual Build

See [docs/manual_instructions.md](docs/manual_instructions.md).

## Attribution

- [lemonade-sdk/llamacpp-rocm](https://github.com/lemonade-sdk/llamacpp-rocm) — the original CI pipeline this is derived from
- [ROCm/TheRock](https://github.com/ROCm/TheRock) — ROCm nightly tarball source
- [ggerganov/llama.cpp](https://github.com/ggerganov/llama.cpp) — the inference engine
- [apollosenvy/kernel-anvil](https://github.com/apollosenvy/kernel-anvil) — the runtime shape tuner

## License

MIT — see [LICENSE](LICENSE).
