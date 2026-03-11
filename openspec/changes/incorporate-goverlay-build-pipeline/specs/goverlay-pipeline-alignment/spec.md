## ADDED Requirements

### Requirement: fakenvapi included in distribution
The CI packaging step SHALL include `fakenvapi.dll` and `fakenvapi.ini` from the OptiScaler-builds package in the distribution artifact.

#### Scenario: fakenvapi present in artifact
- **WHEN** the CI packaging completes
- **THEN** `fakenvapi.dll` and `fakenvapi.ini` are present in the `x86_64-windows/` directory of the distribution

### Requirement: fakenvapi deployed to game directory
The proton script SHALL copy `fakenvapi.dll` and `fakenvapi.ini` to the game directory when OptiScaler is enabled.

#### Scenario: fakenvapi in game directory
- **WHEN** OptiScaler is enabled and a game launches
- **THEN** `fakenvapi.dll` and `fakenvapi.ini` are present in the game exe directory

#### Scenario: fakenvapi tracked for cleanup
- **WHEN** OptiScaler deploys fakenvapi files
- **THEN** both files are recorded in the optiscaler_files manifest

### Requirement: DLSS stub DLLs included in distribution
The CI packaging step SHALL include `nvngx_dlss.dll`, `nvngx_dlssd.dll`, and `nvngx_dlssg.dll` from the OptiScaler-builds package.

#### Scenario: DLSS stubs present in artifact
- **WHEN** the CI packaging completes
- **THEN** `nvngx_dlss.dll`, `nvngx_dlssd.dll`, and `nvngx_dlssg.dll` are present in the distribution

### Requirement: DLSS stub DLLs deployed to game directory
The proton script SHALL copy DLSS stub DLLs to the game directory when OptiScaler is enabled.

#### Scenario: DLSS stubs in game directory
- **WHEN** OptiScaler is enabled and a game launches
- **THEN** `nvngx_dlss.dll`, `nvngx_dlssd.dll`, and `nvngx_dlssg.dll` are present in the game exe directory

### Requirement: OptiScaler.ini ships with tested defaults
The CI packaging step SHALL modify the default OptiScaler.ini to set `LoadAsiPlugins=true` and `UseHQFont=false`.

#### Scenario: ASI plugins enabled by default
- **WHEN** the packaged OptiScaler.ini is inspected
- **THEN** `LoadAsiPlugins` is set to `true`

#### Scenario: HQ font disabled by default
- **WHEN** the packaged OptiScaler.ini is inspected
- **THEN** `UseHQFont` is set to `false`

### Requirement: Game DLLs backed up before overwriting
The proton script SHALL rename existing game DLLs with a `.b` suffix before copying OptiScaler supporting DLLs that may conflict.

#### Scenario: Existing game DLL backed up
- **WHEN** OptiScaler is enabled and the game directory contains `amd_fidelityfx_dx12.dll`
- **THEN** the existing file is renamed to `amd_fidelityfx_dx12.dll.b` before OptiScaler's copy is deployed

#### Scenario: Backed up DLLs restored on cleanup
- **WHEN** OptiScaler is disabled and cleanup runs
- **THEN** any `.b` backup files are renamed back to their original names

### Requirement: SteamDeck environment variable set
The proton script SHALL set `SteamDeck=0` when OptiScaler is enabled, matching fgmod's behavior.

#### Scenario: SteamDeck disabled
- **WHEN** OptiScaler is enabled
- **THEN** the environment variable `SteamDeck` is set to `0`
