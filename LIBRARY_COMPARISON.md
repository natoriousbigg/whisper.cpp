# oneAPI Library Comparison: Windows vs Ubuntu Builds

This document provides a detailed comparison of the shared libraries included in the Windows and Ubuntu oneAPI builds to ensure feature parity and correct runtime dependencies.

## Summary

After comprehensive analysis, the Ubuntu build has been updated to match Windows library coverage for Intel oneAPI 2025.0, ensuring users have a consistent experience across platforms.

## Library Categories and Comparison

### 1. Core Compiler Runtime Libraries

| Category | Windows | Ubuntu | Status |
|----------|---------|--------|--------|
| SVML (Short Vector Math Library) | `svml_dispmd.dll` | `libsvml.so*` | ✅ Both covered |
| Math Functions | `libmmd.dll` | `libimf.so*` | ✅ Both covered |
| OpenMP Runtime | `libiomp5md.dll` | Included via libmkl_intel_thread | ✅ Both covered |
| Compiler Runtime | - | `libintlc.so*` | ✅ Ubuntu specific |
| Random Number Generator | - | `libirng.so*` | ✅ Ubuntu specific |

**Conclusion:** ✅ Full coverage on both platforms

### 2. SYCL Runtime Libraries

| Library | Windows | Ubuntu | Status |
|---------|---------|--------|--------|
| SYCL Runtime | `sycl.dll`, `sycl7.dll`, `sycl8.dll` | `libsycl.so*` | ✅ Both covered |

**Conclusion:** ✅ Full coverage on both platforms

### 3. Plugin Interface (PI) Libraries

| Backend | Windows | Ubuntu | Status |
|---------|---------|--------|--------|
| Level Zero (Intel GPU) | `pi_level_zero.dll` | `libpi_level_zero.so` | ✅ Both covered |
| OpenCL | `pi_opencl.dll` | `libpi_opencl.so` | ✅ Both covered |
| CUDA (NVIDIA) | `pi_cuda.dll` | `libpi_cuda.so` | ✅ Both covered |
| HIP (AMD) | `pi_hip.dll` | `libpi_hip.so` | ✅ Both covered |

**Conclusion:** ✅ Full coverage via wildcard pattern `libpi_*.so`

### 4. OpenCL Runtime

| Library | Windows | Ubuntu | Status |
|---------|---------|--------|--------|
| OpenCL | Bundled with drivers | `libOpenCL.so*` | ✅ Both covered |

**Conclusion:** ✅ Full coverage on both platforms

### 5. Intel MKL (Math Kernel Library) - CRITICAL

| Library | Purpose | Windows | Ubuntu (Before) | Ubuntu (After Fix) |
|---------|---------|---------|-----------------|-------------------|
| MKL SYCL BLAS | BLAS operations for SYCL | `mkl_sycl_blas.5.dll` | `libmkl_sycl_blas.so*` | ✅ |
| MKL Core | Core computational routines | `mkl_core.2.dll` | `libmkl_core.so*` | ✅ |
| MKL ILP64 Interface | 64-bit integer interface | - | `libmkl_intel_ilp64.so*` | ✅ |
| MKL Intel Thread | OpenMP-based threading | `mkl_intel_thread.2.dll` | `libmkl_intel_thread.so*` | ✅ |
| MKL TBB Thread | TBB-based threading | `mkl_tbb_thread.2.dll` | ❌ MISSING | ✅ **FIXED** |
| MKL SYCL | General SYCL support | - | `libmkl_sycl.so*` | ✅ |
| MKL SYCL LAPACK | Linear algebra package | `mkl_sycl_lapack.5.dll` | ❌ | ⚠️ Not needed* |
| MKL SYCL VM | Vector math operations | `mkl_sycl_vm.5.dll` | ❌ | ⚠️ Not needed* |

\* **Note:** LAPACK and VM libraries are included in Windows builds but not actively used by whisper.cpp. The CMake configuration only links `MKL::MKL_SYCL::BLAS`, so these libraries are not required for Ubuntu builds.

**Conclusion:** ✅ **FIXED** - `libmkl_tbb_thread.so` now included to match Windows

### 6. Threading Building Blocks (TBB)

| Library | Purpose | Windows | Ubuntu (Before) | Ubuntu (After Fix) |
|---------|---------|---------|-----------------|-------------------|
| TBB Runtime | Threading library | `tbb12.dll` | ❌ MISSING | ✅ **ADDED** |
| TBB Malloc | TBB memory allocator | - | ❌ MISSING | ✅ **ADDED** (optional) |

**Why Added:** The `libmkl_tbb_thread.so.2` library (MKL TBB threading layer) has a runtime dependency on Intel TBB libraries. Windows includes `tbb12.dll` to satisfy this dependency.

**Conclusion:** ✅ **FIXED** - TBB libraries now included

### 7. Unified Runtime (oneAPI 2024.x and newer)

| Library | Purpose | Windows | Ubuntu (Before) | Ubuntu (After Fix) |
|---------|---------|---------|-----------------|-------------------|
| Unified Runtime | New SYCL runtime abstraction layer | `ur_win_proxy_loader.dll` | ❌ MISSING | ✅ **ADDED** |

**Why Added:** Intel oneAPI 2025.0 uses the Unified Runtime (UR) as an abstraction layer for SYCL backends. This provides better compatibility and performance. Windows includes this for forward compatibility.

**Conclusion:** ✅ **ADDED** for oneAPI 2025.0 compatibility

### 8. oneDNN (Deep Neural Network Library)

| Library | Purpose | Windows | Ubuntu | Status |
|---------|---------|---------|--------|--------|
| oneDNN Runtime | Deep learning operations | `dnnl.dll` | ❌ | ⚠️ Not enabled |
| oneDNN Debug | Debug variant | `dnnld.dll` | ❌ | ⚠️ Not enabled |

**Why Not Included:** The CMake configuration does not set `GGML_SYCL_DNN=ON`, so oneDNN support is disabled in both builds. The Windows build includes these DLLs but they are not used. No action needed for Ubuntu.

**Conclusion:** ⚠️ Not required - oneDNN disabled in build configuration

## Changes Made to Ubuntu Build

### 1. Added libmkl_tbb_thread.so (PRIMARY FIX)
**File:** `.github/workflows/build.yml`, line 512  
**Change:** Added `libmkl_tbb_thread` to the MKL libraries list

```bash
for lib in libmkl_sycl_blas libmkl_intel_ilp64 libmkl_intel_thread libmkl_tbb_thread libmkl_core libmkl_sycl; do
```

**Rationale:** This library provides TBB-based threading for MKL operations and is required for full compatibility with Intel oneAPI 2025.0.

### 2. Added Intel TBB Libraries (DEPENDENCY)
**File:** `.github/workflows/build.yml`, lines 526-544  
**Added Libraries:**
- `libtbb.so*` or `libtbb12.so*` - Main TBB runtime
- `libtbbmalloc.so*` - TBB memory allocator (optional but included)

**Rationale:** Runtime dependency for `libmkl_tbb_thread.so.2`

### 3. Added Unified Runtime Library (FORWARD COMPATIBILITY)
**File:** `.github/workflows/build.yml`, lines 546-558  
**Added Library:**
- `libur_loader.so*` - Unified Runtime loader

**Rationale:** Required for Intel oneAPI 2025.0 and newer versions. Provides abstraction layer for SYCL backends.

### 4. Updated Package README
**File:** `.github/workflows/build.yml`, lines 560-564  
Updated README to mention all included library categories.

## Build Configuration Analysis

### What whisper.cpp Actually Uses

Based on `ggml/src/ggml-sycl/CMakeLists.txt`:

1. **SYCL Compiler and Runtime** (lines 39-47)
   - Required for SYCL compilation and execution

2. **Intel MKL BLAS** (line 118)
   ```cmake
   target_link_libraries(ggml-sycl PRIVATE MKL::MKL_SYCL::BLAS)
   ```
   - Only BLAS operations are linked
   - LAPACK and VM libraries are NOT used

3. **oneDNN** (lines 52-84)
   - Controlled by `GGML_SYCL_DNN` flag
   - NOT enabled in current build configuration
   - Windows includes DLLs but they are unused

## Verification Steps

After building with these changes, the package should include:

### Core Libraries (20+ files)
```bash
# SYCL Runtime
libsycl.so.7

# Plugin Interfaces
libpi_level_zero.so
libpi_opencl.so

# Core Compiler Runtime
libsvml.so
libimf.so
libintlc.so
libirng.so
libOpenCL.so

# MKL Libraries (6 files)
libmkl_sycl_blas.so.5
libmkl_intel_ilp64.so.2
libmkl_intel_thread.so.2
libmkl_tbb_thread.so.2    # ← FIXED
libmkl_core.so.2
libmkl_sycl.so.2

# TBB Libraries (2+ files)
libtbb.so.12              # ← ADDED
libtbbmalloc.so.2         # ← ADDED (optional)

# Unified Runtime
libur_loader.so.0         # ← ADDED
```

### Testing Commands

```bash
# Extract package
unzip whisper-cli-*-ubuntu-x64-intelgpu.zip
cd dist/

# Check included libraries
ls -la *.so* | wc -l  # Should be 20+ files

# Check for critical libraries
ls libmkl_tbb_thread.so* && echo "✓ MKL TBB Thread found"
ls libtbb*.so* && echo "✓ TBB found"
ls libur_loader.so* && echo "✓ Unified Runtime found"

# Test execution (requires Intel GPU)
./whisper-cli --help
```

## Platform Parity Summary

| Component | Windows | Ubuntu | Status |
|-----------|---------|--------|--------|
| SYCL Runtime | ✅ | ✅ | Parity achieved |
| Plugin Interfaces | ✅ | ✅ | Parity achieved |
| Core Compiler Runtime | ✅ | ✅ | Parity achieved |
| MKL BLAS Libraries | ✅ | ✅ | **Parity achieved** (with fixes) |
| TBB Libraries | ✅ | ✅ | **Parity achieved** (added) |
| Unified Runtime | ✅ | ✅ | **Parity achieved** (added) |
| oneDNN | ⚠️ Unused | ⚠️ Disabled | Intentionally not enabled |

## Conclusion

After this PR, the Ubuntu oneAPI build achieves **feature parity** with the Windows build for all libraries actually required by whisper.cpp. The key improvements are:

1. ✅ **Fixed missing `libmkl_tbb_thread.so.2`** - Primary issue resolved
2. ✅ **Added TBB runtime libraries** - Satisfies dependencies
3. ✅ **Added Unified Runtime** - Future-proofs for oneAPI 2025.0+

Libraries not included (LAPACK, VM, oneDNN) are intentionally excluded as they are not used by the current whisper.cpp build configuration.
