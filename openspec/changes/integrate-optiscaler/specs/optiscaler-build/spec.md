## ADDED Requirements

### Requirement: OptiScaler built from source in CI
The build system SHALL compile OptiScaler.dll from source using MSVC on a GitHub Actions Windows runner. The build MUST use the upstream OptiScaler repository as a git submodule.

#### Scenario: Successful OptiScaler build
- **WHEN** the CI pipeline runs
- **THEN** OptiScaler.dll is produced as a build artifact for x86_64 Windows

#### Scenario: OptiScaler submodule is present
- **WHEN** the repository is checked out with submodules
- **THEN** the OptiScaler source is available at the expected submodule path

### Requirement: OptiPatcher built from source in CI
The build system SHALL compile OptiPatcher.asi from source using MSBuild on the same GitHub Actions Windows runner. The build MUST use the upstream OptiPatcher repository as a git submodule.

#### Scenario: Successful OptiPatcher build
- **WHEN** the CI pipeline runs
- **THEN** OptiPatcher.asi is produced as a build artifact

### Requirement: Supporting runtime DLLs packaged
The build system SHALL download and package supporting runtime DLLs (AMD FSR, Intel XeSS, Nukem FG mod) from upstream release archives alongside the built OptiScaler.dll.

#### Scenario: FSR runtimes included
- **WHEN** the build completes
- **THEN** amd_fidelityfx_dx12.dll, amd_fidelityfx_upscaler_dx12.dll, amd_fidelityfx_framegeneration_dx12.dll, and amd_fidelityfx_vk.dll are present in the distribution

#### Scenario: XeSS runtimes included
- **WHEN** the build completes
- **THEN** libxess.dll, libxess_dx11.dll, libxess_fg.dll, and libxell.dll are present in the distribution

#### Scenario: Nukem FG mod included
- **WHEN** the build completes
- **THEN** dlssg_to_fsr3_amd_is_better.dll is present in the distribution

### Requirement: Distribution layout follows Proton conventions
The build output SHALL be organized under `files/lib/wine/optiscaler/` following the same layout pattern as DXVK (`files/lib/wine/dxvk/`).

#### Scenario: Correct directory structure
- **WHEN** the build artifacts are installed into the Proton distribution
- **THEN** the layout is:
  - `files/lib/wine/optiscaler/x86_64-windows/OptiScaler.dll`
  - `files/lib/wine/optiscaler/x86_64-windows/plugins/OptiPatcher.asi`
  - `files/lib/wine/optiscaler/x86_64-windows/<supporting-dlls>`
  - `files/lib/wine/optiscaler/OptiScaler.ini`
  - `files/lib/wine/optiscaler/version`

### Requirement: Default OptiScaler.ini shipped
The build SHALL include a default OptiScaler.ini configuration file with OptiPatcher enabled and DXGI spoofing disabled.

#### Scenario: Default config enables OptiPatcher
- **WHEN** the default OptiScaler.ini is inspected
- **THEN** `LoadAsiPlugins` is set to `true`

#### Scenario: Default config disables DXGI spoofing
- **WHEN** the default OptiScaler.ini is inspected
- **THEN** `Dxgi` is set to `false`

### Requirement: Build artifact consumed by main Proton build
The Windows build step SHALL produce a compressed artifact that the main Linux container build extracts into the Proton distribution tree.

#### Scenario: Artifact integration
- **WHEN** the main Proton container build runs
- **THEN** the OptiScaler artifact is extracted and its contents are present in the final distribution
