# AC2026.1.2 — AutoCAD 2026.1.2 reference assemblies

This folder supplies the AutoCAD assemblies that the shim (`OPMNetExt/asdkOPMNetExt`)
references at compile time. `OPMNetExt.vcxproj` adds it as a `#using` directory
(`AdditionalUsingDirectories` = `$(MSBuildThisFileDirectory)..\AC2026.1.2`) for both
Debug and Release.

The DLLs are Autodesk binaries and are **not committed** (see `.gitignore`). Copy them
here yourself before building.

## Required files

| File          | Version (expected) | Why it is needed |
|---------------|--------------------|------------------|
| `acdbmgd.dll` | 25.1.179.0         | `#using <acdbmgd.dll>` in `StdAfx.h`: managed database API (`RXObject`, `RXClass`, …) |
| `acmgd.dll`   | 25.1.179.0         | `#using <acmgd.dll>` in `StdAfx.h`: managed application API |
| `acdb25.dll`  | 25.1.179.0         | Native database module that `acdbmgd.dll` depends on; kept next to `acdbmgd.dll` so the mixed-mode assembly can be resolved |

## Where to get them

Copy them from the program folder of an **AutoCAD 2026 installation updated to 2026.1.2**
(`C:\Program Files\Autodesk\AutoCAD 2026\`). Check that the file version is 25.1.179.
An installation without that update has older builds (e.g. 25.1.164), and the shim
should be compiled against the same build it will be loaded into.

`OPMNetSample` does not use this folder: it references `AcDbMgd` / `AcMgd` /
`AcCoreMgd` from the ObjectARX SDK `inc\` folder through `<AssemblySearchPaths>`.

## Retargeting

When moving to a different AutoCAD release or update, create a matching folder (e.g.
`AC2027`), put that release's files in it (`acdb26.dll` for 2027), and update the
`AdditionalUsingDirectories` path in `OPMNetExt.vcxproj`. The other places to update
are listed in `CLAUDE.md`, under "SDK / version paths".
