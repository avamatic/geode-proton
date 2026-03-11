## 1. Git Submodules

- [x] 1.1 Add OptiScaler as a git submodule (from https://github.com/optiscaler/OptiScaler)
- [x] 1.2 Add OptiPatcher as a git submodule (from https://github.com/optiscaler/OptiPatcher)

## 2. GitHub Actions Build (Windows Runner)

- [x] 2.1 Create a new workflow job `_job_build_optiscaler.yml` that runs on a Windows runner
- [x] 2.2 Add MSVC/MSBuild build step for OptiScaler.dll (x64 Release)
- [x] 2.3 Add MSBuild build step for OptiPatcher.asi (x64 Release)
- [x] 2.4 Add step to download supporting runtime DLLs (FSR, XeSS, Nukem FG) from upstream release archives
- [x] 2.5 Package all build outputs into a compressed artifact (tarball or zip)
- [x] 2.6 Wire the OptiScaler build job into the existing `devel.yml` and `release.yml` workflows

## 3. Distribution Integration

- [x] 3.1 Add build system logic to extract the OptiScaler artifact into `dist/files/lib/wine/optiscaler/`
- [x] 3.2 Create the distribution directory layout: `x86_64-windows/` with OptiScaler.dll, plugins/, and supporting DLLs
- [x] 3.3 Create default `OptiScaler.ini` with `LoadAsiPlugins=true` and `Dxgi=false` (using upstream defaults — auto values adapt correctly)
- [x] 3.4 Add version tracking file (`files/lib/wine/optiscaler/version`)

## 4. Proton Script — Environment Variable

- [x] 4.1 Add `PROTON_USE_OPTISCALER` handling via `check_environment()` in `setup_session()` (maps to `"optiscaler"` compat_config)
- [x] 4.2 Add proxy name parsing logic: `1` → `version` (default), or explicit name from allowed list (`version`, `winmm`, `winhttp`, `wininet`, `dbghelp`, `d3d12`)
- [x] 4.3 Add validation that rejects invalid proxy names (log warning, skip deployment)

## 5. Proton Script — Prefix Deployment

- [x] 5.1 Add OptiScaler directory path resolution in the `Proton` class (similar to `arch_pe_dir("wine/dxvk", ...)`)
- [x] 5.2 Add deployment logic in `setup_prefix()`: copy OptiScaler.dll as `<proxy_name>.dll` to system32
- [x] 5.3 Add deployment of supporting DLLs to system32 (FSR, XeSS, Nukem FG runtimes)
- [x] 5.4 Add deployment of `plugins/OptiPatcher.asi` to `system32/plugins/`
- [x] 5.5 Add deployment of `OptiScaler.ini` to system32
- [x] 5.6 Set DLL override for the proxy name (`g_session.dlloverrides[proxy_name] = "n"`)
- [x] 5.7 Register all deployed files with `tracked_files` for automatic cleanup
- [x] 5.8 Add `use_optiscaler` flag to `prefix_info` string (triggers prefix update when flag changes)

## 6. Testing

- [ ] 6.1 Build the project via GitHub Actions and verify OptiScaler artifacts are present in the distribution
- [ ] 6.2 Install locally (`make install`) and verify OptiScaler files in the Proton distribution directory
- [ ] 6.3 Test `PROTON_USE_OPTISCALER=1` — verify files deployed to prefix system32 with correct names
- [ ] 6.4 Test each proxy DLL name (version, winmm, winhttp, wininet, dbghelp, d3d12) on AMD hardware
- [ ] 6.5 Verify OptiScaler overlay initializes in at least one game
- [ ] 6.6 Verify game directory remains untouched (no files written)
- [ ] 6.7 Verify cleanup: disable OptiScaler, relaunch, confirm files removed from prefix
- [ ] 6.8 Test on NVIDIA hardware to verify compatibility with existing dxvk-nvapi (deferred — needs NVIDIA tester)
