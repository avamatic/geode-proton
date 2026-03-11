## Context

geode-proton integrates OptiScaler as a native Proton component, deploying DLLs to the game directory at launch. The current implementation downloads pre-built packages from `benjamimgois/OptiScaler-builds` and has minimal game detection logic (UE heuristic, REDengine, id Tech). goverlay/fgmod solves the same deployment problem with a more mature launcher rewrite table and a broader set of deployed DLLs (fakenvapi, nvngx_dlss stubs).

Current state:
- CI: `_job_build_optiscaler.yml` downloads edge build from OptiScaler-builds, repackages into Proton layout
- Runtime: proton script deploys to game exe directory with 3 launcher rewrites
- Missing: fakenvapi, nvngx_dlss/dlssd/dlssg stubs, several launcher rewrites
- OptiScaler.ini: `LoadAsiPlugins` defaults to false (should be true), `UseHQFont` defaults to true (crashes Vulkan)

## Goals / Non-Goals

**Goals:**
- Import fgmod's complete launcher rewrite table (12 game-specific entries + UE fallback)
- Add missing DLLs to the CI packaging and runtime deployment (fakenvapi, nvngx stubs)
- Ship OptiScaler.ini with tested defaults (LoadAsiPlugins=true, UseHQFont=false)
- Ensure our CI artifact includes all files fgmod deploys

**Non-Goals:**
- Replacing fgmod or competing with goverlay — we complement it
- Lutris support (fgmod has Lutris integration; we're Proton-only)
- Building OptiScaler from source (we continue using pre-built packages)
- Solving game-specific OptiScaler crashes (e.g., Satisfactory) — those are upstream issues

## Decisions

### 1. Use fgmod's launcher table directly
**Decision**: Import all 12 game-specific launcher rewrites from fgmod into the proton script.
**Rationale**: fgmod's table is battle-tested across hundreds of users. Our current 3 entries (UE, REDengine, id Tech) are a subset. Maintaining a separate list adds no value.
**Alternative considered**: Auto-detect by scanning for all exe files — rejected because it's unreliable (picks up crash reporters, installers, etc.) and goverlay already solved this with their manual table.

### 2. Add fakenvapi to the distribution
**Decision**: Include `fakenvapi.dll` and `fakenvapi.ini` in the CI artifact and deploy them to the game directory.
**Rationale**: fakenvapi spoofs NVIDIA GPU detection so games expose DLSS options on AMD hardware. This complements OptiPatcher (which patches game binaries) by working at the API level. fgmod includes it by default.
**Alternative considered**: Rely solely on OptiPatcher — rejected because OptiPatcher only covers ~180 specific games, while fakenvapi works generically.

### 3. Add nvngx DLL stubs
**Decision**: Include `nvngx_dlss.dll`, `nvngx_dlssd.dll`, `nvngx_dlssg.dll` from the OptiScaler-builds package.
**Rationale**: These are DLSS stub DLLs that redirect DLSS calls to OptiScaler. Games that ship their own nvngx_dlss.dll don't need these, but games that rely on them being present in the search path do. fgmod includes all three.

### 4. Modify OptiScaler.ini defaults at packaging time
**Decision**: Modify OptiScaler.ini during CI packaging to set `LoadAsiPlugins=true` and `UseHQFont=false`.
**Rationale**: `LoadAsiPlugins=auto` defaults to false, which prevents OptiPatcher from loading. `UseHQFont=auto` defaults to true, which crashes Vulkan games. These are proven safe defaults from our testing.

### 5. Back up game DLLs before overwriting
**Decision**: Before copying supporting DLLs, rename any existing game DLLs with a `.b` suffix (matching fgmod's behavior). Restore them on cleanup.
**Rationale**: Some games ship their own copies of `amd_fidelityfx_*.dll` or other DLLs. Backing them up ensures we can restore the originals when OptiScaler is disabled.

## Risks / Trade-offs

- **[Risk] Launcher table maintenance** → We inherit fgmod's maintenance burden. Mitigation: stay aligned with fgmod updates, contribute back new entries.
- **[Risk] fakenvapi may cause issues on NVIDIA hardware** → Mitigation: OptiScaler is opt-in; users on NVIDIA who don't need it simply won't enable it. fakenvapi only activates on non-NVIDIA GPUs.
- **[Risk] Backing up game DLLs adds complexity** → Mitigation: The manifest system already tracks deployed files; backup/restore is a natural extension.
- **[Trade-off] We depend on OptiScaler-builds package layout** → If the package structure changes, our CI breaks. Mitigation: the package is stable and maintained by our potential collaborator.
