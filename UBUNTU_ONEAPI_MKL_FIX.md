# Ubuntu oneAPI MKL Libraries Fix - Summary

## Problem
The Ubuntu oneAPI build for whisper.cpp was missing required Intel MKL (Math Kernel Library) runtime libraries in the distributed binary package. Specifically, users reported errors about missing `libmkl_sycl_blas.so.5` when trying to run the whisper-cli or whisper-bench executables.

Example error message:
```
error while loading shared libraries: libmkl_sycl_blas.so.5: cannot open shared object file: No such file or directory
```

## Root Cause

### Build Configuration
The whisper.cpp project uses Intel oneAPI SYCL for GPU acceleration on Intel hardware. The CMake configuration in `ggml/src/ggml-sycl/CMakeLists.txt` explicitly links against Intel MKL for BLAS operations:

```cmake
if (GGML_SYCL_TARGET STREQUAL "INTEL")
    # Intel devices use Intel oneMKL directly instead of oneMath
    find_package(MKL REQUIRED)
    target_link_libraries(ggml-sycl PRIVATE MKL::MKL_SYCL::BLAS)
    target_compile_definitions(ggml-sycl PRIVATE GGML_SYCL_USE_INTEL_ONEMKL)
endif()
```

### Missing Libraries
The build workflow installed `intel-oneapi-mkl-devel` which includes:
- `libmkl_sycl_blas.so*` - SYCL BLAS interface library (the primary missing library)
- `libmkl_intel_ilp64.so*` - MKL core library with ILP64 interface
- `libmkl_intel_thread.so*` - Threading layer for parallel operations
- `libmkl_core.so*` - Core MKL computational library
- `libmkl_sycl.so*` - General SYCL support in MKL

However, the packaging step in `.github/workflows/build.yml` was only copying SYCL runtime libraries from `compiler/latest/lib/`, not the MKL libraries from `mkl/latest/lib/`.

## Solution

### Changes to `.github/workflows/build.yml`

Added lines 493-499 in the "Package whisper-cli and whisper-bench (Ubuntu)" step:

```yaml
# Copy Intel MKL libraries required for SYCL BLAS operations
# These libraries are linked via MKL::MKL_SYCL::BLAS in CMakeLists.txt
cp ${{ env.ONEAPI_INSTALL_PATH }}/mkl/latest/lib/libmkl_sycl_blas.so* dist/ 2>/dev/null || echo "Warning: libmkl_sycl_blas.so not found"
cp ${{ env.ONEAPI_INSTALL_PATH }}/mkl/latest/lib/libmkl_intel_ilp64.so* dist/ 2>/dev/null || echo "Warning: libmkl_intel_ilp64.so not found"
cp ${{ env.ONEAPI_INSTALL_PATH }}/mkl/latest/lib/libmkl_intel_thread.so* dist/ 2>/dev/null || echo "Warning: libmkl_intel_thread.so not found"
cp ${{ env.ONEAPI_INSTALL_PATH }}/mkl/latest/lib/libmkl_core.so* dist/ 2>/dev/null || echo "Warning: libmkl_core.so not found"
cp ${{ env.ONEAPI_INSTALL_PATH }}/mkl/latest/lib/libmkl_sycl.so* dist/ 2>/dev/null || true
```

### Updated README
Also updated the README.txt included in the distribution package to mention MKL libraries:

```
This package includes Intel oneAPI SYCL runtime libraries and Intel MKL libraries.
```

## Technical Details

### Library Dependencies
The MKL SYCL BLAS library has dependencies on other MKL libraries:
- `libmkl_sycl_blas.so.5` depends on:
  - `libmkl_core.so`
  - `libmkl_intel_ilp64.so` or `libmkl_intel_lp64.so`
  - `libmkl_intel_thread.so` or `libmkl_sequential.so`

All of these must be present in the distribution for the binary to run.

### RPATH Configuration
The binaries are built with `CMAKE_BUILD_RPATH='$ORIGIN'` and use `patchelf` to ensure they look for libraries in the same directory as the executable. This allows the package to be self-contained and work without requiring users to install Intel oneAPI.

## Expected Outcome
After this fix, the Ubuntu oneAPI binary package will:
- ✅ Include all required MKL libraries for SYCL BLAS operations
- ✅ Successfully load `libmkl_sycl_blas.so.5` and dependencies at runtime
- ✅ Allow users to run whisper-cli and whisper-bench without errors
- ✅ Not require users to separately install Intel oneAPI or MKL

## Verification
To verify the fix works:
1. Download the Ubuntu oneAPI package
2. Extract it
3. Check that the following libraries are present:
   ```bash
   ls -la libmkl_*.so*
   ```
4. Run the binary:
   ```bash
   ./whisper-cli --help
   ```
5. Should execute without library loading errors

## Related Files
- `.github/workflows/build.yml` - Main workflow file with the fix
- `ggml/src/ggml-sycl/CMakeLists.txt` - Links against MKL::MKL_SYCL::BLAS
- `SYCL_STATIC_LINKING_INVESTIGATION.md` - Explains why static linking isn't used
- `README_sycl.md` - Documentation for SYCL builds

## Related Issues
This complements the previous fix for libsycl.so loading issues. The SYCL runtime and MKL libraries work together:
- SYCL runtime provides the execution framework
- MKL provides optimized math operations (BLAS, etc.)
