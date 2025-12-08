# Windows oneAPI DLL Fix - Summary

## Problem
The Windows oneAPI build for release v1.8.2.2 failed to copy required Intel oneAPI runtime DLLs, showing:
```
Warning: sycl7.dll not found
Warning: pi_*.dll not found (0 files copied)
```

## Root Causes

### 1. Incorrect DLL Name
- **Issue**: Build script tried to copy `sycl7.dll`
- **Reality**: Intel oneAPI 2024.x uses `sycl.dll` (not `sycl7.dll`)
- **History**: Older versions (2023.x) used versioned names like `sycl6.dll`, `sycl7.dll`

### 2. CMD Wildcard Limitations
- **Issue**: `copy "%ONEAPI_ROOT%\compiler\latest\bin\pi_*.dll"` returned 0 files
- **Reality**: Windows CMD `copy` command doesn't expand wildcards for source files
- **Behavior**: The literal string `pi_*.dll` was searched for, which doesn't exist

## Solution

### Changes to `.github/workflows/build.yml`
1. **Switched to PowerShell** for packaging steps (lines 883-945)
   - Better file handling and scripting capabilities
   - Native support for iteration and conditionals

2. **Fixed DLL Names**
   - Try both `sycl.dll` and `sycl7.dll` for compatibility
   - Explicitly enumerate PI plugin DLLs: `pi_level_zero.dll`, `pi_opencl.dll`, etc.

3. **Search Multiple Locations**
   - Check `compiler/latest/bin`
   - Check `compiler/latest/bin/plugins`
   - Some oneAPI installations place PI DLLs in a plugins subdirectory

4. **Better Logging**
   - Reports which DLLs are successfully copied
   - Clear warnings for missing files
   - Count of PI plugin DLLs found

### Code Structure
```powershell
# List of core runtime DLLs to copy
$dllsToCopy = @('svml_dispmd.dll', 'libmmd.dll', 'libiomp5md.dll', 'sycl.dll', 'sycl7.dll')

# Try each DLL with Test-Path
foreach ($dll in $dllsToCopy) {
  if (Test-Path $sourcePath) {
    Copy-Item $sourcePath dist\
  }
}

# Search multiple locations for PI plugins
$piLocations = @("$env:ONEAPI_ROOT\compiler\latest\bin", "$env:ONEAPI_ROOT\compiler\latest\bin\plugins")
$piDlls = @('pi_level_zero.dll', 'pi_opencl.dll', 'pi_cuda.dll', 'pi_hip.dll')

# Nested loop to check all combinations
foreach ($location in $piLocations) {
  foreach ($piDll in $piDlls) {
    if (Test-Path $piPath) {
      Copy-Item $piPath dist\
    }
  }
}
```

## Expected Outcome
After this fix, the Windows oneAPI build should:
- ✅ Successfully find and copy `sycl.dll` (primary)
- ✅ Copy PI plugin DLLs (at least `pi_level_zero.dll` for Intel GPUs)
- ✅ Include all necessary runtime dependencies in the release package
- ✅ Allow users to run whisper-cli.exe and whisper-bench.exe with Intel GPU acceleration

## Verification
To verify the fix worked:
1. Check CI workflow logs for "Copied X PI plugin DLLs" message
2. No warnings about sycl7.dll
3. Non-zero count of PI plugin DLLs copied
4. Download the release package and verify DLL files are present

## Related Files
- `.github/workflows/build.yml` - Main workflow file with the fix
- `SYCL_STATIC_LINKING_INVESTIGATION.md` - Documentation on why static linking isn't viable
