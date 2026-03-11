# Draft: Collaboration Message to goverlay Maintainer

## Target: GitHub Discussion on benjamimgois/goverlay

---

**Title:** Collaboration opportunity: OptiScaler as a native Proton component

Hi! I've been working on [geode-proton](https://github.com/avamatic/geode-proton), a fork of GE-Proton that integrates OptiScaler directly into Proton as a built-in component — similar to how DXVK is shipped.

The goal is the same as goverlay/fgmod: make OptiScaler easy to use on Linux. The difference is the approach — instead of a wrapper script that copies files before launch, OptiScaler is packaged into the Proton distribution and deployed automatically when the user sets `PROTON_USE_OPTISCALER=1` in their Steam launch options.

What this gets us:
- Automatic deployment and cleanup tied to Proton's prefix lifecycle
- No persistent files left behind when disabled
- Works through Steam's standard launch options (no external tools needed)
- Manifest-based file tracking so everything gets cleaned up properly

I want to be upfront: this project already builds heavily on your work. We pull the pre-built OptiScaler edge packages from [OptiScaler-builds](https://github.com/benjamimgois/OptiScaler-builds) in our CI, and our game directory detection logic is directly inspired by fgmod's approach (UE heuristic, REDengine launcher rewrites, etc).

A few things I'd love your input on:
- Are there game-specific quirks in fgmod's launcher table that we should know about beyond what's in the script today?
- We found that `dxgi.dll` in the game directory is really the only reliable proxy approach (system32 deployment with version/winhttp/winmm all have issues). Has your experience been the same?
- Would you be interested in this kind of native Proton integration? The long-term goal is to get GloriousEggroll interested in incorporating it upstream into GE-Proton.

Happy to collaborate however makes sense — whether that's sharing findings, aligning on packaging, or just comparing notes on what works and what doesn't.
