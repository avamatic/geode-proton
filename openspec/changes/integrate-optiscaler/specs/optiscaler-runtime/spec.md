## ADDED Requirements

### Requirement: Opt-in via environment variable
The proton script SHALL support `PROTON_USE_OPTISCALER` as an opt-in environment variable. When not set, OptiScaler MUST NOT be deployed and all existing behavior MUST be unchanged.

#### Scenario: OptiScaler disabled by default
- **WHEN** a game launches without `PROTON_USE_OPTISCALER` set
- **THEN** no OptiScaler files are copied to the prefix and no OptiScaler-related DLL overrides are set

#### Scenario: OptiScaler enabled with default proxy
- **WHEN** `PROTON_USE_OPTISCALER=1` is set
- **THEN** OptiScaler is deployed using the default proxy DLL name (`version`)

### Requirement: Proxy DLL name selection
The proton script SHALL support explicit proxy DLL name selection via the value of `PROTON_USE_OPTISCALER`. Valid values MUST include: `1` (default=version), `version`, `winmm`, `winhttp`, `wininet`, `dbghelp`, `d3d12`.

#### Scenario: Explicit proxy name
- **WHEN** `PROTON_USE_OPTISCALER=winmm` is set
- **THEN** OptiScaler.dll is copied to the prefix as `winmm.dll` and the DLL override for `winmm` is set to `n`

#### Scenario: Invalid proxy name
- **WHEN** `PROTON_USE_OPTISCALER=invalid` is set
- **THEN** OptiScaler deployment is skipped (no crash, no error to user beyond log)

#### Scenario: Value of 1 uses default
- **WHEN** `PROTON_USE_OPTISCALER=1` is set
- **THEN** OptiScaler.dll is deployed as `version.dll` with DLL override `version=n`

### Requirement: OptiScaler DLL deployed to prefix system32
When enabled, the proton script SHALL copy OptiScaler.dll from the Proton distribution to the Wine prefix `drive_c/windows/system32/` renamed to the selected proxy DLL name.

#### Scenario: DLL copied to system32
- **WHEN** OptiScaler is enabled with proxy name `version`
- **THEN** `OptiScaler.dll` is copied to `<prefix>/drive_c/windows/system32/version.dll`

### Requirement: Supporting DLLs deployed alongside
When enabled, the proton script SHALL copy all supporting runtime DLLs (FSR, XeSS, Nukem FG) from the Proton distribution to the Wine prefix `drive_c/windows/system32/`.

#### Scenario: Supporting DLLs in system32
- **WHEN** OptiScaler is enabled
- **THEN** all supporting DLLs (amd_fidelityfx_*.dll, libxess*.dll, libxell.dll, dlssg_to_fsr3_amd_is_better.dll) are present in `<prefix>/drive_c/windows/system32/`

### Requirement: OptiPatcher plugin deployed
When enabled, the proton script SHALL copy the `plugins/` directory containing OptiPatcher.asi to `drive_c/windows/system32/plugins/`.

#### Scenario: OptiPatcher available
- **WHEN** OptiScaler is enabled
- **THEN** `<prefix>/drive_c/windows/system32/plugins/OptiPatcher.asi` exists

### Requirement: OptiScaler.ini deployed
When enabled, the proton script SHALL copy the default OptiScaler.ini to `drive_c/windows/system32/OptiScaler.ini`.

#### Scenario: Config file present
- **WHEN** OptiScaler is enabled
- **THEN** `<prefix>/drive_c/windows/system32/OptiScaler.ini` exists

### Requirement: DLL override set for proxy name
When enabled, the proton script SHALL set the Wine DLL override for the chosen proxy DLL name to `n` (native), ensuring Wine loads the OptiScaler DLL instead of its builtin.

#### Scenario: DLL override applied
- **WHEN** OptiScaler is enabled with proxy name `version`
- **THEN** `g_session.dlloverrides["version"]` is set to `"n"`

### Requirement: All deployed files tracked for cleanup
All files deployed by OptiScaler MUST be registered with Proton's tracked_files mechanism so they are automatically cleaned up on prefix update or when OptiScaler is disabled.

#### Scenario: Files tracked
- **WHEN** OptiScaler deploys files to the prefix
- **THEN** all deployed file paths are written to the tracked_files list

#### Scenario: Cleanup on disable
- **WHEN** a game previously had OptiScaler enabled but now launches without it
- **THEN** on the next prefix update, all OptiScaler files are removed from system32

### Requirement: No game directory modifications
The OptiScaler integration MUST NOT write any files to the game's installation directory. All files MUST be deployed exclusively to the Wine prefix.

#### Scenario: Game directory untouched
- **WHEN** OptiScaler is enabled and a game launches
- **THEN** no files are created, modified, or deleted in the game's installation directory

### Requirement: Compat config integration
The proton script SHALL register `PROTON_USE_OPTISCALER` with the compat_config system using `check_environment`, following the pattern of existing Proton environment variables.

#### Scenario: Compat config flag
- **WHEN** `PROTON_USE_OPTISCALER` is set to any truthy value
- **THEN** `"optiscaler"` is present in `g_session.compat_config`
