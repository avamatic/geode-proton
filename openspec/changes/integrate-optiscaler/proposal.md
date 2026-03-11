## Why

OptiScaler enables games that only support NVIDIA DLSS to use AMD FSR or Intel XeSS instead, making upscaling and frame generation available on non-NVIDIA GPUs. Currently on Linux, installing OptiScaler requires tools like goverlay/fgmod that copy 15+ DLLs into each game directory on every launch, leaving behind stateful files that modify game behavior and are cumbersome to clean up. By treating OptiScaler as a Proton component — built, distributed, and deployed like DXVK — we eliminate game directory pollution entirely while making cross-vendor upscaling opt-in via a simple environment variable.

## What Changes

- OptiScaler and OptiPatcher added as git submodules built from source
- New GitHub Actions workflow step builds OptiScaler (MSVC) and OptiPatcher (MSBuild) on a Windows runner
- Supporting runtime DLLs (FSR, XeSS, Nukem FG mod) packaged into the Proton distribution under `files/lib/wine/optiscaler/`
- Proton script gains `PROTON_USE_OPTISCALER` environment variable (opt-in, disabled by default)
- When enabled, OptiScaler + supporting DLLs deployed to Wine prefix `system32` using the established DXVK pattern (try_copy, dlloverrides, tracked_files)
- Multiple proxy DLL names supported (`version`, `winmm`, `winhttp`, `wininet`, `dbghelp`, `d3d12`) for compatibility testing
- Default `OptiScaler.ini` shipped with OptiPatcher enabled and DXGI spoofing disabled
- No `dxgi.dll` proxy name initially (conflicts with DXVK); deferred until testing proves necessary
- Existing Proton components (dxvk-nvapi) reused rather than shipping duplicates (fakenvapi)

## Capabilities

### New Capabilities
- `optiscaler-build`: Building OptiScaler and OptiPatcher from source via GitHub Actions Windows runner, packaging with supporting runtime DLLs
- `optiscaler-runtime`: Deploying OptiScaler into Wine prefix at game launch, controlled by environment variable, with proxy DLL name selection and automatic cleanup

### Modified Capabilities

None. This change adds new functionality without modifying existing Proton behavior.

## Impact

- **Build system**: New GitHub Actions workflow step (Windows runner) added to build pipeline; artifact consumed by existing container build
- **Proton distribution**: ~300MB additional DLLs in `files/lib/wine/optiscaler/`
- **Proton script** (`proton`): New env var handling in `setup_session()`, new deployment logic in `setup_prefix()`
- **Git submodules**: Two new submodules (OptiScaler, OptiPatcher)
- **Dependencies**: MSVC build toolchain required in CI (Windows runner); FSR/XeSS SDK runtimes downloaded during build
- **No breaking changes**: Feature is opt-in, all existing behavior unchanged when `PROTON_USE_OPTISCALER` is not set
