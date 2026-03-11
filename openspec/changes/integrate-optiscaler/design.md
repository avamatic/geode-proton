## Context

Geode-proton is a fork of Valve's Proton (Wine-based compatibility layer for running Windows games on Linux via Steam). It follows the GE-Proton model of incorporating additional components. The build system uses GNU Make with a containerized SDK for cross-compilation, plus GitHub Actions for CI.

The project currently builds and deploys components like DXVK, dxvk-nvapi, and vkd3d-proton using a well-established pattern:
1. **Build**: Components are git submodules built via `rules-source`/`rules-meson` macros in `Makefile.in`, producing DLLs installed to `dist/files/lib/wine/<component>/`
2. **Deploy**: The `proton` Python script copies DLLs from the distribution into each game's Wine prefix (`system32`/`syswow64`) at launch, sets DLL overrides, and tracks files for cleanup

OptiScaler is a Windows DLL that intercepts upscaler API calls (DLSS, FSR, XeSS) and redirects them across GPU vendors. It is built with MSVC (not mingw), which means it cannot use the existing container-based build pipeline. OptiPatcher is a companion ASI plugin that patches game executables at runtime to bypass GPU vendor checks, eliminating the need for DXGI spoofing.

## Goals / Non-Goals

**Goals:**
- Integrate OptiScaler as an opt-in Proton component following established patterns
- Build from source in CI (reproducible, auditable)
- Deploy to Wine prefix system32 — zero game directory modifications
- Support multiple proxy DLL names for compatibility testing
- Ship OptiPatcher for clean vendor-check bypassing
- Maintain compatibility with both AMD and NVIDIA hardware
- Design for eventual upstream inclusion in GE-Proton

**Non-Goals:**
- Using `dxgi.dll` as proxy name (conflicts with DXVK; deferred)
- Per-game OptiScaler configuration (upstream limitation)
- Shipping NVIDIA DLSS runtime DLLs (available via driver on NVIDIA, unused on AMD/Intel)
- Replacing existing Proton components (dxvk-nvapi, FSR upgrade flags)
- Cross-compiling OptiScaler with mingw (MSVC is upstream's toolchain)
- 32-bit (i386) support (OptiScaler is x86_64 only)

## Decisions

### 1. Build via GitHub Actions Windows runner, not container

**Decision**: Build OptiScaler and OptiPatcher on a Windows runner in GitHub Actions, producing artifacts consumed by the main Proton build.

**Alternatives considered**:
- *Mingw cross-compilation in container*: Would match DXVK's build pattern, but OptiScaler uses MSVC-specific features (Detours, MSVC project files) and upstream only supports MSVC builds. Risk of subtle compatibility issues.
- *Pre-built binary download*: Simpler but not reproducible, not auditable, depends on external release cadence.

**Rationale**: Matches upstream's build system exactly. The Windows build step produces a tarball artifact that the Linux container build extracts into the distribution tree. This is a proven pattern (many Proton forks do similar for components that require Windows toolchains).

### 2. Default proxy DLL name: `version.dll`

**Decision**: When `PROTON_USE_OPTISCALER=1`, default to deploying as `version.dll`.

**Alternatives considered**:
- *`dxgi.dll`*: Most tested in goverlay/fgmod ecosystem, but directly conflicts with DXVK's dxgi.dll. Would require either replacing DXVK's dxgi or chaining (OptiScaler proxies to DXVK's dxgi), both adding complexity.
- *`winmm.dll`*: Viable alternative, but version.dll is more universally loaded by games.

**Rationale**: `version.dll` avoids DXVK conflicts entirely. OptiScaler's Detours-based hooking intercepts LoadLibrary/GetProcAddress regardless of its own filename, so the proxy name only affects how OptiScaler gets loaded into the process — not what it can intercept. All proxy names are exposed for testing via the env var.

### 3. Support all proxy names via env var value

**Decision**: `PROTON_USE_OPTISCALER` accepts both `1` (default=version) and explicit DLL names (`version`, `winmm`, `winhttp`, `wininet`, `dbghelp`, `d3d12`).

**Rationale**: Different games may require different proxy DLLs depending on their DLL loading patterns. Exposing all names lets users and testers find what works per-game without code changes. Once testing identifies the best defaults, we can simplify.

### 4. Deploy supporting DLLs alongside OptiScaler in system32

**Decision**: Copy all supporting runtime DLLs (FSR, XeSS, Nukem FG) into the same `system32` directory as OptiScaler.

**Alternatives considered**:
- *Separate directory with path configuration*: OptiScaler loads supporting DLLs relative to its own location (`GetModuleFileNameW` → parent path). No env var override exists upstream.
- *Symlinks*: Could symlink from system32 to proton dist, but Wine's DLL loader may not follow symlinks consistently for PE DLLs.

**Rationale**: OptiScaler's hardcoded DLL-relative path resolution means supporting DLLs must be in the same directory. Placing everything in system32 matches how DXVK deploys and ensures OptiScaler finds its dependencies.

### 5. Skip fakenvapi, reuse dxvk-nvapi

**Decision**: Do not ship `fakenvapi.dll`. Rely on Proton's existing `dxvk-nvapi` (`nvapi64.dll`).

**Alternatives considered**:
- *Ship fakenvapi alongside dxvk-nvapi*: Risk of conflicts between two nvapi implementations in the same prefix.

**Rationale**: dxvk-nvapi already provides GPU spoofing capabilities in Proton. OptiPatcher eliminates most need for spoofing anyway. If testing reveals dxvk-nvapi is insufficient for specific OptiScaler scenarios, fakenvapi can be added later.

### 6. Ship default OptiScaler.ini with OptiPatcher enabled

**Decision**: Include a default `OptiScaler.ini` with `LoadAsiPlugins=true` and `Dxgi=false`.

**Rationale**: OptiPatcher provides cleaner game compatibility than DXGI spoofing (no performance overhead, no crashes on Intel Arc). Having it enabled by default means most games work without user configuration. Users can still override by placing a custom `OptiScaler.ini` in the prefix's system32.

## Risks / Trade-offs

- **OptiScaler from system32 is untested territory** — It has always been deployed to game directories. DLL search paths and hooking behavior may differ when loaded from system32 vs game directory. → *Mitigation*: Multi-proxy-name support enables rapid testing. If system32 fails, we can investigate Wine-specific DLL loading behavior.

- **Proxy DLL compatibility varies per game** — Some games may not load `version.dll` (or other chosen proxy) early enough for OptiScaler's Detours hooks to intercept all needed calls. → *Mitigation*: Six proxy names available for testing. Community feedback will identify per-game recommendations.

- **300MB+ distribution size increase** — Supporting DLLs (FSR, XeSS runtimes) are large. → *Trade-off*: Acceptable for the functionality. These are the same DLLs that goverlay/fgmod install per-game; shipping once in the proton dist is more efficient than per-game copies.

- **Global OptiScaler.ini** — Config is per-prefix but not per-game (no per-executable sections upstream). → *Mitigation*: Default config works for most games. Users can manually edit prefix config. Long-term, contribute env var config path override to upstream OptiScaler.

- **MSVC build dependency** — Requires Windows runner in CI, adding build time and complexity. → *Mitigation*: Separate workflow step; only rebuilds when OptiScaler submodule changes. Windows runners are available in GitHub Actions.

- **dxvk-nvapi may not fully replace fakenvapi** — Some OptiScaler features may specifically require fakenvapi's behavior (Force Reflex, LatencyFlex). → *Mitigation*: Test without fakenvapi first. Add it back as an option if specific features fail.

## Open Questions

- Will OptiScaler's Detours hooks work correctly when loaded from system32 rather than the game directory? (Requires testing)
- Which proxy DLL name has the broadest game compatibility? (Requires testing across multiple titles)
- Does OptiPatcher's `PatchResult()` correctly communicate with OptiScaler when both are in system32? (Requires testing)
- Should the NVIDIA DLSS runtime DLLs be included for NVIDIA users, or does the existing driver-provided path (`NVIDIA_WINE_DLL_DIR`) suffice? (Requires NVIDIA hardware testing)
