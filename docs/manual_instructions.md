# Manual Build Instructions

> Reference only. For the current build process, see `.github/workflows/build-smithy.yml`.

The CI workflow is the recommended way to build. This document covers manual local builds.

## Prerequisites

- Windows 11 or Ubuntu 22.04
- Visual Studio 2022 Build Tools (Windows) with C++ workload
- CMake 3.31+
- Ninja
- Python 3.10+ (for kernel-anvil profiling, optional at build time)

## Windows

```powershell
# 1. Install dependencies (choco)
choco install visualstudio2022buildtools -y --params "--add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 --add Microsoft.VisualStudio.Component.VC.CMake.Project --add Microsoft.VisualStudio.Component.VC.ATL --add Microsoft.VisualStudio.Component.Windows11SDK.22621"
choco install cmake ninja python -y

# 2. Download TheRock nightly (gfx110X-all)
$rocmUrl = "https://therock-nightly-tarball.s3.amazonaws.com/therock-dist-windows-gfx110X-all-7.14.0a20260609.tar.gz"
Invoke-WebRequest -Uri $rocmUrl -OutFile "rocm.tar.gz"
New-Item -ItemType Directory -Force -Path "C:\opt\rocm" | Out-Null
tar -xzf rocm.tar.gz -C C:\opt\rocm --strip-components=1

# 3. Clone llama.cpp at the commit matching your patch
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
git checkout ac4cdde

# 4. Apply the Smithy patch
git apply ..\patches\smithy.patch

# 5. Build (in x64 Native Tools Command Prompt)
set HIP_PATH=C:\opt\rocm
set HIP_PLATFORM=amd
set PATH=%HIP_PATH%\lib\llvm\bin;%HIP_PATH%\bin;%PATH%

mkdir build && cd build
cmake .. -G Ninja ^
  -DCMAKE_C_COMPILER="C:\opt\rocm\lib\llvm\bin\clang.exe" ^
  -DCMAKE_CXX_COMPILER="C:\opt\rocm\lib\llvm\bin\clang++.exe" ^
  -DCMAKE_CXX_FLAGS="-IC:\opt\rocm\include" ^
  -DCMAKE_CROSSCOMPILING=ON ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DGPU_TARGETS="gfx1100" ^
  -DBUILD_SHARED_LIBS=ON ^
  -DLLAMA_BUILD_TESTS=OFF ^
  -DGGML_HIP=ON ^
  -DGGML_OPENMP=OFF ^
  -DGGML_CUDA_FORCE_CUBLAS=OFF ^
  -DGGML_RPC=ON ^
  -DGGML_HIP_ROCWMMA_FATTN=OFF ^
  -DLLAMA_BUILD_BORINGSSL=ON ^
  -DGGML_NATIVE=OFF ^
  -DGGML_STATIC=OFF ^
  -DCMAKE_SYSTEM_NAME=Windows
cmake --build . -j 24

# 6. Copy runtime DLLs
cd bin
Copy-Item C:\opt\rocm\bin\amdhip64_*.dll .
Copy-Item C:\opt\rocm\bin\rocblas.dll,rocsolver.dll,hipblas.dll,libhipblas.dll,hipblaslt.dll,libhipblaslt.dll,rocm_kpack.dll,amd_comgr.dll,amdhip64_7.dll .
Copy-Item -Recurse C:\opt\rocm\bin\hipblaslt .
```

## Ubuntu

```bash
# 1. Install dependencies
sudo apt update
sudo apt install -y cmake ninja-build unzip curl patchelf

# 2. Download TheRock nightly
sudo mkdir -p /opt/rocm
curl -sL "https://therock-nightly-tarball.s3.amazonaws.com/therock-dist-linux-gfx110X-all-7.14.0a20260609.tar.gz" \
  | sudo tar --use-compress-program=gzip -xf - -C /opt/rocm --strip-components=1

# 3. Set env
export HIP_PATH=/opt/rocm
export ROCM_PATH=/opt/rocm
export HIP_PLATFORM=amd
export HIP_DEVICE_LIB_PATH=/opt/rocm/lib/llvm/amdgcn/bitcode
export PATH=/opt/rocm/bin:/opt/rocm/llvm/bin:$PATH
export LD_LIBRARY_PATH=/opt/rocm/lib:/opt/rocm/lib64:${LD_LIBRARY_PATH:-}

# 4. Clone + patch + build
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
git checkout ac4cdde
git apply ../patches/smithy.patch
mkdir build && cd build
cmake .. -G Ninja \
  -DCMAKE_C_COMPILER=/opt/rocm/llvm/bin/clang \
  -DCMAKE_CXX_COMPILER=/opt/rocm/llvm/bin/clang++ \
  -DCMAKE_CXX_FLAGS="-I/opt/rocm/include" \
  -DCMAKE_CROSSCOMPILING=ON \
  -DCMAKE_BUILD_TYPE=Release \
  -DGPU_TARGETS="gfx1100" \
  -DBUILD_SHARED_LIBS=ON \
  -DLLAMA_BUILD_TESTS=OFF \
  -DGGML_HIP=ON \
  -DGGML_OPENMP=OFF \
  -DGGML_CUDA_FORCE_CUBLAS=OFF \
  -DGGML_RPC=ON \
  -DGGML_HIP_ROCWMMA_FATTN=OFF \
  -DLLAMA_BUILD_BORINGSSL=ON \
  -DGGML_NATIVE=OFF \
  -DGGML_STATIC=OFF \
  -DCMAKE_SYSTEM_NAME=Linux
cmake --build . -j $(nproc)

# 5. Copy runtime libs
cd bin
sudo cp -r /opt/rocm/lib/rocblas/library ./rocblas/library
sudo cp -r /opt/rocm/lib/hipblaslt/library ./hipblaslt/library
for p in libhipblas librocblas libamdhip64 librocsolver libhipblaslt libamd_comgr libhsa-runtime64 librocm_kpack; do
  sudo cp -v /opt/rocm/lib/${p}*.so* .
done

# 6. Set RPATH
for f in *.so* llama-*; do
  [ -f "$f" ] && [ ! -L "$f" ] && patchelf --set-rpath '$ORIGIN' "$f" 2>/dev/null || true
done
```

## Profiling your model (one-time per model)

```bash
# Install kernel-anvil (one time)
pip install kernel-anvil

# Profile each GGUF you want to use
kernel-anvil gguf-optimize ~/Models/my-model-Q4_K_M.gguf
# writes ~/.cache/smithy/my-model-Q4_K_M.json

# Speculative decoding: profile both target and draft
kernel-anvil gguf-optimize ~/Models/target.gguf --draft ~/Models/draft.gguf
```

## Running

```bash
# Linux
export SMITHY_CONFIG=~/.cache/smithy/my-model-Q4_K_M.json
./llama-server -m ~/Models/my-model-Q4_K_M.gguf -ngl 999

# Windows (PowerShell)
$env:SMITHY_CONFIG = "$HOME\.cache\smithy\my-model-Q4_K_M.json"
.\llama-server.exe -m H:\Models\my-model-Q4_K_M.gguf -ngl 999
```

On startup you'll see:
```
kernel-anvil: loaded 6 shape configs from /path/to/config.json
```

Without a config file, behavior is identical to stock llama.cpp.
