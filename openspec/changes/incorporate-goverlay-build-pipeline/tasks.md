## 1. CI Packaging Updates

- [ ] 1.1 Add `fakenvapi.dll` and `fakenvapi.ini` to the CI extract-and-repackage step in `_job_build_optiscaler.yml`
- [ ] 1.2 Add `nvngx_dlss.dll`, `nvngx_dlssd.dll`, `nvngx_dlssg.dll` to the CI extract-and-repackage step
- [ ] 1.3 Add sed commands to modify OptiScaler.ini during packaging: set `LoadAsiPlugins=true` and `UseHQFont=false`

## 2. Extended Game Detection

- [ ] 2.1 Add Baldur's Gate 3 launcher rewrite (`Launcher/LariLauncher.exe` → `bin/`)
- [ ] 2.2 Add HITMAN 3 / WoA launcher rewrite (`Launcher.exe` + `Retail/` → `Retail/`)
- [ ] 2.3 Add SYNCED launcher rewrite (`Launcher/sop_launcher.exe` → game root)
- [ ] 2.4 Add 2K Launcher rewrite (`2KLauncher/LauncherPatcher.exe` → game root)
- [ ] 2.5 Add Warhammer Darktide rewrite (`launcher/Launcher.exe` + `binaries/` → `binaries/`)
- [ ] 2.6 Add Warhammer Vermintide 2 rewrite (`launcher/Launcher.exe` + `binaries_dx12/` → `binaries_dx12/`)
- [ ] 2.7 Add Satisfactory rewrite (`FactoryGameSteam.exe` → `Engine/Binaries/Win64/`)
- [ ] 2.8 Add FFXIV rewrite (`boot/ffxivboot.exe` → `game/`)
- [ ] 2.9 Add Dune Awakening rewrite (`Launcher/FuncomLauncher.exe` → UE shipping exe dir)

## 3. Runtime Deployment Updates

- [ ] 3.1 Add `fakenvapi.dll` and `fakenvapi.ini` to the list of files deployed to game directory
- [ ] 3.2 Add `nvngx_dlss.dll`, `nvngx_dlssd.dll`, `nvngx_dlssg.dll` to the deployed files list
- [ ] 3.3 Implement DLL backup logic: rename existing game DLLs with `.b` suffix before overwriting
- [ ] 3.4 Implement DLL restore logic: rename `.b` files back to originals during cleanup
- [ ] 3.5 Set `SteamDeck=0` environment variable when OptiScaler is enabled

## 4. Testing

- [ ] 4.1 Verify CI artifact contains all new DLLs (fakenvapi, nvngx stubs)
- [ ] 4.2 Verify OptiScaler.ini defaults are correct in CI artifact
- [ ] 4.3 Test game directory deployment with a UE game (Scorn)
- [ ] 4.4 Test game directory deployment with a REDengine game (Witcher 3 or Cyberpunk)
- [ ] 4.5 Test cleanup restores backed-up game DLLs when OptiScaler is disabled
