# OptiScaler Deployment Strategies: Research & Testing Results

## Date: 2026-03-10/11
## Context: geode-proton OptiScaler integration

---

## 1. DXVK and Wine dxgi.dll History

### The Conflict (pre-2018)
DXVK replaced Wine's entire `dxgi.dll` because it needed to own the Vulkan pipeline — instance creation (`IDXGIVkInteropFactory`), device enumeration (`IDXGIVkInteropAdapter`), and swap chain management (`IDXGIVkSwapChain`). Wine's DXGI was designed around WineD3D (OpenGL-based), making the two fundamentally incompatible. However, replacing dxgi.dll broke Wine's own D3D12 implementation (vkd3d), which depended on Wine's DXGI for swap chain creation.

### The Resolution (late 2018)
- **Wine side**: Henri Verbeet introduced `IWineDXGISwapChainFactory` (commit `8ba075a`), allowing Wine's DXGI to delegate swap chain creation to whatever D3D backend is active.
- **DXVK side**: doitsujin implemented the interface in `d3d11.dll` (commit `1e393bf`), so DXVK could work with Wine's built-in DXGI without replacing it.

### Current State in Proton
Proton still ships DXVK's `dxgi.dll` as the default in system32 because it provides extra features (GPU spoofing, HUD, frame rate control, HDR). vkd3d-proton uses DXVK's presenter interfaces (`IDXGIVkSwapChainFactory`) for D3D12 swap chains, sharing VkInstance/VkDevice handles.

### Key Interfaces (from DXVK `src/dxgi/dxgi_interfaces.h`)
- `IDXGIVkInteropFactory` — exposes VkInstance and vkGetInstanceProcAddr
- `IDXGIVkInteropAdapter` — maps DXGI adapters to VkPhysicalDevice
- `IDXGIVkInteropDevice` — queue access, layout transitions, submission locking
- `IDXGIVkSwapChain` / `IDXGIVkSwapChainFactory` — entire Vulkan presentation pipeline
- `IDXGIVkMonitorInfo` — monitor data, color space, HDR state

### Implication for OptiScaler
**dxgi.dll in system32 is spoken for.** DXVK owns it, vkd3d-proton depends on it, and replacing it breaks the entire D3D rendering pipeline. OptiScaler as dxgi.dll MUST go in the game directory where it shadows DXVK's copy only for the game process.

---

## 2. System32 Proxy DLL Testing

### Methodology
Deployed OptiScaler to `drive_c/windows/system32/` as various proxy DLL names. Analyzed Wine startup logs (`+loaddll` trace) to determine which processes load each DLL.

### Wine Process DLL Loading Map (from Scorn test run)

| Process (PID) | Role | version | winhttp | winmm | wininet | dbghelp |
|---|---|---|---|---|---|---|
| wineboot.exe (0028) | Prefix init | - | - | - | **YES** | - |
| services.exe (0030) | Service manager | - | - | - | - | - |
| steam.exe (0020) | Proton Steam client | **YES** | - | - | **YES** | **YES** |
| xalia.exe (0120) | Accessibility | **YES** | - | **YES** | **YES** | **YES** |
| Scorn-Win64-Shipping.exe (0168) | Game | **YES** | **YES** | **YES** | - | game dir |

### Results by Proxy DLL

#### version.dll — FAILED (crashes Wine system processes)
- **Problem**: Loaded by steam.exe and xalia.exe during startup, BEFORE the game launches.
- **Crash**: OptiScaler performs GPU probing during DllMain (loads dxgi, d3d11, nvapi). In non-game processes, this hits a null pointer dereference (EXCEPTION_ACCESS_VIOLATION at 0x0000000000000000).
- **Error**: `"version.dll" failed to initialize, aborting` — kills Wine startup entirely.
- **Conclusion**: Fundamentally incompatible with system32 deployment. OptiScaler was never designed to run in Wine infrastructure processes.

#### winhttp.dll — PARTIAL SUCCESS (game-dependent)
- **Result**: Only loaded by the game process. No Wine system process interference.
- **Success**: Scorn launched with OptiScaler splash, UI overlay (Insert), FSR2 backend swap.
- **Failure**: Cyberpunk 2077 and Witcher 3 don't load winhttp.dll at all (REDengine games skip it).
- **Additional issue**: Required `winhttp-original.dll` forwarding — without it, OptiScaler couldn't forward real winhttp API calls (self-load loop when it tried to LoadLibrary the original from system32).

#### winmm.dll — FAILED (breaks launchers)
- **Problem**: When deployed to system32, REDprelauncher.exe (Witcher 3, Cyberpunk) fails with `Library WINMM.dll not found` for Qt5Core.dll.
- **Root cause**: OptiScaler's winmm proxy doesn't correctly satisfy static import table resolution at process startup time.

#### wininet.dll — NOT TESTED (rejected by analysis)
- **Problem**: Loaded by wineboot.exe — would crash during prefix initialization, same as version.dll.

#### dbghelp.dll — NOT TESTED (rejected by analysis)
- **Problem**: Loaded by steam.exe and xalia.exe. High risk of same crash as version.dll.

### Conclusion
**No single proxy DLL name works universally from system32.** Each candidate either crashes Wine system processes, isn't loaded by certain games, or breaks launcher functionality.

---

## 3. Game Directory Deployment (Final Approach)

### Why It Works
- Wine DLL search order: game directory → system32
- Game directory dxgi.dll shadows DXVK's system32 copy
- Only the game process loads from its own directory — Wine infrastructure untouched
- This is how goverlay/fgmod does it, proven across hundreds of games

### DLL Override
Set `dxgi=n,b` (native, builtin):
- Wine tries native first → finds our dxgi.dll in game directory
- Builtin fallback available for Wine system processes via system32

### Game Directory Detection Challenge
`sys.argv[2]` (the Steam launch command) often points to a launcher, not the actual game exe:

| Game | sys.argv[2] | Actual game exe |
|---|---|---|
| Scorn | `Scorn.exe` (root) | `Scorn/Binaries/Win64/Scorn-Win64-Shipping.exe` |
| Cyberpunk 2077 | `REDprelauncher.exe` (root) | `bin/x64/Cyberpunk2077.exe` |
| Witcher 3 | `REDprelauncher.exe` (root) | `bin/x64_dx12/witcher3.exe` |
| Satisfactory | `FactoryGameSteam.exe` (root) | `Engine/Binaries/Win64/FactoryGameSteam-Win64-Shipping.exe` |
| Doom Eternal | `launcher/idTechLauncher.exe` | `DOOMEternalx64vk.exe` (root) |

### Solution: Heuristic detection (from fgmod)
1. **Known launcher rewrites** — hardcoded table mapping launcher exes to game exe directories
2. **UE heuristic** — if `Engine/` exists, search `*/Binaries/Win64/*.exe` (excluding Engine binaries)
3. **Fallback** — directory of `sys.argv[2]`

---

## 4. Game-Specific Test Results

### Scorn (UE5, D3D12) — WORKS
- OptiScaler loads, splash visible, UI overlay works (Insert key)
- FSR2 backend swap: works
- FSR3 frame generation: crashes (upstream OptiScaler/AMD runtime issue under Wine)

### Witcher 3 (REDengine, D3D12) — WORKS
- OptiScaler loads from `bin/x64_dx12/`, splash visible
- FSR2: works
- FSR3/4: not tested but user reports it works with manual OptiScaler install

### Cyberpunk 2077 (REDengine, D3D12) — WORKS
- OptiScaler loads from `bin/x64/`
- Confirmed dxgi.dll loaded by game process

### Satisfactory (UE5, D3D12) — CRASHES
- OptiScaler loads from `Engine/Binaries/Win64/` (correct directory)
- Crashes during DXGI initialization — OptiScaler dxgi incompatibility with this game
- Game launches fine without OptiScaler
- Upstream OptiScaler issue, not our integration

### Doom Eternal (id Tech 7, Vulkan) — NOT COMPATIBLE via dxgi
- Vulkan game — does not load dxgi.dll
- Would need winhttp or other proxy, but UI overlay crashes on Vulkan games with UseHQFont=true
- With UseHQFont=false: overlay crash persists (likely deeper Vulkan hooking issue under Wine)

### Pacific Drive (UE5, D3D12) — LOADS BUT LIMITED
- OptiScaler loads and initializes
- OptiPatcher loads
- DLSS option hidden — UE5 DLSS plugin performs its own NVIDIA GPU check at engine level before calling DLSS API, so OptiScaler never gets a chance to intercept
- OptiPatcher doesn't cover this game's specific check

---

## 5. OptiScaler Proxy DLL Architecture

OptiScaler compiles as a **universal proxy DLL** — a single binary that exports forwarding stubs for ALL supported proxy identities simultaneously. At runtime, `CheckWorkingMode()` in `dllmain.cpp` detects its own filename via `GetModuleFileNameW` and initializes forwarding for the matching proxy only.

### Supported proxy names
dxgi, d3d12, version, winhttp, winmm, wininet, dbghelp, nvngx, _nvngx, dlss-enabler-upscaler, optiscaler, optiscaler.asi

### Forwarding resolution order (per proxy)
1. `{plugin_path}/{name}.dll`
2. `{name}-original.dll` (in same directory)
3. `{system32}/{name}.dll`

### Key insight
When deployed to system32, step 3 creates a self-load loop — OptiScaler finds itself as the "original." This is why `{name}-original.dll` forwarding was needed for the system32 approach, and why game directory deployment avoids the problem entirely (system32 still has the real builtin).

---

## 6. Key Lessons Learned

1. **System32 is hostile territory for proxy DLLs.** Wine system processes load many common DLLs during startup. Any DLL that does non-trivial initialization (like GPU probing) will crash when loaded by wineboot, services.exe, etc.

2. **Game directory deployment is the proven approach.** goverlay does it this way for a reason. The trade-off of writing files to the game directory is worth the reliability.

3. **No universal proxy DLL exists.** Different games load different system DLLs. The only DLL loaded by virtually all D3D games is dxgi.dll, which is why goverlay uses it.

4. **DXVK's dxgi.dll cannot be replaced.** The tight coupling between DXVK's DXGI, d3d11, and vkd3d-proton's presenter interface means any replacement breaks the rendering pipeline.

5. **Game exe detection is a maintenance burden.** Launchers, nested directories, and engine-specific layouts mean a hardcoded table is unavoidable. Aligning with fgmod's table reduces duplicate effort.

6. **OptiScaler.ini defaults matter.** `LoadAsiPlugins=false` (default) prevents OptiPatcher from loading. `UseHQFont=true` (default) crashes Vulkan games. Both need to be overridden.
