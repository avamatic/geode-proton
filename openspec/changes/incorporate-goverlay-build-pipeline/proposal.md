## Why

Our current OptiScaler CI workflow downloads pre-built edge packages from `benjamimgois/OptiScaler-builds` and repackages them. This works, but the OptiScaler-builds repo is maintained by the same person who maintains goverlay/fgmod — the tool that solves the same problem we do (easy OptiScaler deployment on Linux). Rather than maintaining parallel packaging logic, we should align with goverlay's build pipeline to share game detection heuristics, DLL selection, and packaging conventions. This reduces maintenance burden, keeps us current with upstream fixes, and positions us for collaboration with the goverlay maintainer.

## What Changes

- Replace our custom `_job_build_optiscaler.yml` repackaging logic with alignment to goverlay/fgmod's packaging conventions
- Adopt fgmod's complete game launcher detection table (currently we only handle UE, REDengine, and id Tech)
- Import fgmod's DLL selection and default configuration choices
- Ensure our OptiScaler.ini defaults match fgmod's proven defaults (LoadAsiPlugins=true, UseHQFont=false for Vulkan compat)
- Add any supporting DLLs that fgmod includes but we currently miss (e.g., fakenvapi)

## Capabilities

### New Capabilities
- `goverlay-pipeline-alignment`: Align CI packaging, game detection, and DLL selection with goverlay/fgmod conventions
- `extended-game-detection`: Import fgmod's full launcher rewrite table for broader game compatibility

### Modified Capabilities

## Impact

- `.github/workflows/_job_build_optiscaler.yml` — packaging logic update
- `proton` script — expanded game directory detection, possible new DLLs
- OptiScaler.ini defaults — align with fgmod's tested configuration
- CI artifact contents — may include additional DLLs (fakenvapi, nvngx stubs)
