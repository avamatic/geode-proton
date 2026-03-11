## ADDED Requirements

### Requirement: Complete launcher rewrite table from fgmod
The proton script SHALL include all game-specific launcher rewrites from fgmod's launcher table to correctly identify the game exe directory.

#### Scenario: Cyberpunk 2077 detection
- **WHEN** the launched exe is `REDprelauncher.exe` and `bin/x64` exists
- **THEN** OptiScaler deploys to `bin/x64/`

#### Scenario: Witcher 3 detection
- **WHEN** the launched exe is `REDprelauncher.exe` and `bin/x64_dx12` exists
- **THEN** OptiScaler deploys to `bin/x64_dx12/`

#### Scenario: Baldur's Gate 3 detection
- **WHEN** the launched exe path contains `Launcher/LariLauncher.exe`
- **THEN** OptiScaler deploys to `bin/` relative to game root

#### Scenario: HITMAN 3 / World of Assassination detection
- **WHEN** the launched exe is `Launcher.exe` and `Retail/` exists
- **THEN** OptiScaler deploys to `Retail/`

#### Scenario: SYNCED detection
- **WHEN** the launched exe path contains `Launcher/sop_launcher.exe`
- **THEN** OptiScaler deploys to the game root

#### Scenario: 2K Launcher games detection
- **WHEN** the launched exe path contains `2KLauncher/LauncherPatcher.exe`
- **THEN** OptiScaler deploys to the game root (parent of 2KLauncher)

#### Scenario: Warhammer 40K Darktide detection
- **WHEN** the launched exe path contains `launcher/Launcher.exe` and `binaries/Darktide.exe` exists
- **THEN** OptiScaler deploys to `binaries/`

#### Scenario: Warhammer Vermintide 2 detection
- **WHEN** the launched exe path contains `launcher/Launcher.exe` and `binaries_dx12/` exists
- **THEN** OptiScaler deploys to `binaries_dx12/`

#### Scenario: Satisfactory detection
- **WHEN** the launched exe is `FactoryGameSteam.exe` and `Engine/Binaries/Win64/` exists
- **THEN** OptiScaler deploys to `Engine/Binaries/Win64/`

#### Scenario: FINAL FANTASY XIV detection
- **WHEN** the launched exe path contains `boot/ffxivboot.exe`
- **THEN** OptiScaler deploys to `game/` relative to game root

#### Scenario: Dune Awakening detection
- **WHEN** the launched exe path contains `Launcher/FuncomLauncher.exe`
- **THEN** OptiScaler deploys to the UE shipping exe directory under the game root

#### Scenario: id Tech games detection
- **WHEN** the launched exe is `idTechLauncher.exe`
- **THEN** OptiScaler deploys to the game root

### Requirement: Generic Unreal Engine fallback
The proton script SHALL detect UE games by the presence of an `Engine/` directory and find the shipping exe in `*/Binaries/Win64/`, excluding Engine binaries.

#### Scenario: Unknown UE game detected
- **WHEN** the game root contains an `Engine/` directory and no launcher rewrite matched
- **THEN** OptiScaler deploys to the directory containing the first `*/Binaries/Win64/*.exe` found (excluding Engine binaries)

#### Scenario: No UE exe found outside Engine
- **WHEN** the game root contains an `Engine/` directory but all exes are under `Engine/`
- **THEN** OptiScaler deploys to the `Engine/Binaries/Win64/` directory containing a `*-Shipping.exe`

### Requirement: Fallback to launched exe directory
When no launcher rewrite matches and no UE heuristic applies, the proton script SHALL deploy to the directory of the launched executable.

#### Scenario: Unknown non-UE game
- **WHEN** no launcher rewrite matches and no `Engine/` directory exists
- **THEN** OptiScaler deploys to `os.path.dirname(sys.argv[2])`
