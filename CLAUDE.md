# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository manages build workflows for PJRT (XLA's Portable JAX Runtime) and StableHLO libraries. It focuses on cross-platform builds (Windows, Linux, macOS) for these machine learning infrastructure components.

## Project Components

1. **PJRT Build Workflow**: Builds XLA's PJRT CPU plugin library for different platforms
   - Uses Bazel as the build system
   - Generates platform-specific binaries (.so for Linux/Mac, .dll for Windows)

2. **StableHLO Build Workflow**: Builds the StableHLO compiler for different platforms
   - Uses CMake as the build system
   - Generates platform-specific binaries (stablehlo-opt)

## Key Files and Directories

- `.github/workflows/build.yaml`: GitHub Actions workflow for building PJRT
- `.github/workflows/stablehlo.yaml`: GitHub Actions workflow for building StableHLO

## Common Commands

### Building PJRT

For CPU backend:
```bash
git clone https://github.com/openxla/xla
cd xla
python configure.py --backend=cpu
bazel build --config=clang_local //xla/pjrt/c:pjrt_c_api_cpu_plugin.so
```

Platform-specific configuration:
- Windows: `--backend=cpu --os=windows --host_compiler=clang --clang_path=C:/PROGRA~1/LLVM/bin/clang.exe`
- Linux/Mac: `--backend=cpu`

### Building StableHLO

```bash
git clone https://github.com/openxla/stablehlo
cd stablehlo
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
git checkout $(cat ../build_tools/llvm_version.txt)
cd ..
MLIR_ENABLE_BINDINGS_PYTHON=OFF build_tools/build_mlir.sh "${PWD}"/llvm-project/ "${PWD}"/llvm-build
mkdir -p build && cd build
cmake .. -GNinja \
  -DLLVM_ENABLE_LLD="$LLVM_ENABLE_LLD" \
  -DCMAKE_BUILD_TYPE='Release' \
  -DLLVM_ENABLE_ASSERTIONS='ON' \
  -DSTABLEHLO_ENABLE_BINDINGS_PYTHON='OFF' \
  -DMLIR_DIR="${PWD}"/../llvm-build/lib/cmake/mlir
cmake --build .
```

## Workflow Execution

The workflows can be triggered manually through GitHub Actions with the following parameters:
- PJRT: Specify XLA commit hash (default: 6319f0d)
- StableHLO: Specify release tag (default: v1.12.1)

## Platform-Specific Considerations

### Windows
- Uses Chocolatey to install LLVM
- Uses clang-cl as compiler
- Packages output as ZIP files
- Note: To avoid path issues on Windows, use environment variables to set LLVM paths:
  ```yaml
  setup: |
    choco install llvm -y
    echo "LLVM_PATH=C:/PROGRA~1/LLVM" >> $GITHUB_ENV
  configure_args: --backend=cpu --os=windows --host_compiler=clang --clang_path=${{ env.LLVM_PATH }}/bin/clang.exe
  ```
- For Bazel, explicitly set the linker path with:
  ```yaml
  bazel build --config=clang_local //xla/pjrt/c:pjrt_c_api_cpu_plugin.so --linkopt="--ld-path=${{ env.LLVM_PATH }}/bin/ld.lld.exe"
  ```

### Linux
- May require installing lld
- Removes unnecessary directories to free up space
- Packages output as tar.gz files

### macOS
- Uses standard build configuration
- Packages output as tar.gz files