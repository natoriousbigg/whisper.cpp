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
- `libmkl_sycl_lapack.so*` - SYCL LAPACK interface library for linear algebra operations
- `libmkl_sycl_vm.so*` - SYCL Vector Math library for vectorized math functions
- `libmkl_intel_ilp64.so*` - MKL core library with ILP64 interface
- `libmkl_intel_thread.so*` - Threading layer for parallel operations (OpenMP-based)
- `libmkl_tbb_thread.so*` - Threading layer for parallel operations (TBB-based)
- `libmkl_core.so*` - Core MKL computational library
- `libmkl_sycl.so*` - General SYCL support in MKL

However, the packaging step in `.github/workflows/build.yml` was only copying SYCL runtime libraries from `compiler/latest/lib/`, not the MKL libraries from `mkl/latest/lib/`.

## Solution

### Changes to `.github/workflows/build.yml`

Updated lines 511-529 in the "Package whisper-cli and whisper-bench (Ubuntu)" step to include all required MKL libraries and search in the correct directory path (`mkl/latest/lib/intel64/`):

```bash
echo "Packaging Intel MKL libraries..."
for lib in libmkl_sycl_blas libmkl_sycl_lapack libmkl_sycl_vm libmkl_intel_ilp64 libmkl_intel_thread libmkl_tbb_thread libmkl_core libmkl_sycl; do
  found=false
  # Search in multiple possible MKL library locations
  for mkl_path in "${{ env.ONEAPI_INSTALL_PATH }}/mkl/latest/lib/intel64" "${{ env.ONEAPI_INSTALL_PATH }}/mkl/latest/lib"; do
    if [ -d "$mkl_path" ]; then
      for file in ${mkl_path}/${lib}.so*; do
        if [ -f "$file" ]; then
          echo "  - Copying $(basename $file)"
          cp "$file" dist/
          found=true
        fi
      done
    fi
  done
  if [ "$found" = "false" ]; then
    echo "Warning: ${lib}.so not found"
  fi
done
```

**Key Fix**: The script now searches in `mkl/latest/lib/intel64/` first (where MKL libraries are typically installed on Linux), then falls back to `mkl/latest/lib/` for compatibility. This ensures versioned libraries like `libmkl_core.so.2` are found and packaged correctly.

### Updated README
Also updated the README.txt included in the distribution package to mention MKL libraries:

```
This package includes Intel oneAPI SYCL runtime libraries and Intel MKL libraries.
```

## Technical Details

### Library Dependencies
The MKL SYCL BLAS library has dependencies on other MKL libraries. The whisper.cpp build uses:
- `libmkl_sycl_blas.so.5` - Main SYCL BLAS interface
- `libmkl_core.so` - Core MKL computational routines
- `libmkl_intel_ilp64.so` - ILP64 interface (64-bit integers for large data)
- `libmkl_intel_thread.so` - Multi-threaded execution layer (OpenMP-based)
- `libmkl_tbb_thread.so.2` - Multi-threaded execution layer (TBB-based)
- `libmkl_sycl.so` - General SYCL support library

Note: This build configuration uses the ILP64 interface and threaded execution. Alternative variants like `libmkl_intel_lp64.so` (LP64 interface) or `libmkl_sequential.so` (sequential execution) are not used in this configuration.

Both OpenMP-based (`libmkl_intel_thread`) and TBB-based (`libmkl_tbb_thread`) threading libraries are required for full compatibility with newer oneAPI versions.

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
