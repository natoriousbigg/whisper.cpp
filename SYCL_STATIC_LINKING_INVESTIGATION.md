# Investigation: Static Linking of libsycl for Ubuntu oneAPI Build

## Question
Can `libsycl7.so` (or `libsycl.so`) be statically built into the binary for Ubuntu oneAPI builds?

## Current Situation

The Ubuntu oneAPI build currently:
- Uses dynamic linking for SYCL runtime libraries
- Bundles `libsycl.so*` and related `.so` files in the release packages
- Links via CMake using either:
  - `IntelSYCL::SYCL_CXX` target (when IntelSYCL package is found)
  - `-fsycl` compiler/linker flags (fallback)

## Investigation Results

### Technical Limitations

**1. Intel's Official Position**
Intel oneAPI does NOT officially support or recommend static linking of the SYCL runtime because:
- The SYCL runtime has dependencies on system-level components (Level Zero driver, OpenCL, etc.)
- Plugin architecture requires runtime loading of backend-specific libraries (PI plugins)
- Runtime device detection and initialization happens dynamically

**2. CMake Configuration**
From `ggml/src/ggml-sycl/CMakeLists.txt`:
```cmake
find_package(IntelSYCL)
if (IntelSYCL_FOUND)
    target_link_libraries(ggml-sycl PRIVATE IntelSYCL::SYCL_CXX)
else()
    target_compile_options(ggml-sycl PRIVATE "-fsycl")
    target_link_options(ggml-sycl PRIVATE "-fsycl")
endif()
```

The `-fsycl` flag links dynamically by default. There is no `-fsycl-static` equivalent.

**3. Plugin System**
The SYCL runtime uses a plugin interface (PI) system:
- `libpi_level_zero.so` - For Intel GPUs
- `libpi_opencl.so` - For OpenCL devices
- These are loaded at runtime based on available hardware
- Cannot be statically linked as they're discovered and loaded dynamically

### Possible Workarounds (Not Recommended)

While technically possible to attempt static linking with advanced linker techniques, it would:
1. Require custom build flags and linker scripts
2. Break the plugin system (no runtime backend selection)
3. Make the binary hardware-specific (e.g., Level Zero only)
4. Not be supported by Intel
5. Likely cause runtime failures or missing functionality

## Recommendation

**DO NOT attempt to statically link libsycl** because:

1. **Current approach is correct**: Bundling the shared libraries in the release package is the proper way to distribute oneAPI/SYCL applications
2. **Users need the libraries anyway**: Even if statically linked, users would still need Intel GPU drivers and the Level Zero runtime
3. **Flexibility**: Dynamic linking allows the runtime to select appropriate backends based on available hardware
4. **Maintainability**: Follows Intel's supported distribution model

## Current Bundle Size

The current Ubuntu oneAPI package includes:
- Main binaries: whisper-cli, whisper-bench
- SYCL runtime: `libsycl.so*`, `libpi_*.so`  
- Math libraries: `libimf.so*`, `libsvml.so*`, etc.
- Total compressed size: ~24 MB (v1.8.2.2)

This is reasonable for a GPU-accelerated application bundle.

## Conclusion

Static linking of libsycl is:
- ❌ Not officially supported by Intel
- ❌ Would break runtime plugin system
- ❌ Would complicate the build process
- ❌ Would not significantly reduce distribution requirements

The current approach of bundling shared libraries is:
- ✅ The recommended way to distribute SYCL applications
- ✅ Maintains flexibility for different hardware backends
- ✅ Easier to maintain and update
- ✅ Allows runtime to adapt to user's system

**Status**: No changes needed. The current dynamic linking approach is correct and optimal.
