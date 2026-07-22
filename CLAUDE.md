# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A C++/CLI interop shim that exposes AutoCAD's **Object Properties Manager (OPM / Properties Palette)** COM functionality to managed .NET languages (C#/VB.NET). AutoCAD's Properties Palette normally requires C++ to implement its COM interfaces; this library wraps those native interfaces so .NET code can add custom dynamic properties to AutoCAD objects (e.g. adding a property to every `AcDbLine`).

The output is a mixed-mode ObjectARX module (`asdkOPMNetExt.dll`) that a managed AutoCAD plugin references and `Assembly.LoadFrom`s at runtime.

## Two projects (`OPMNetExt.sln`)

- **`OPMNetExt/` (`asdkOPMNetExt`)** — the shim itself. A **C++/CLI** mixed-mode DLL (`.vcxproj`, `ArxAppType=arxnet`, `CLRSupport=NetCore`). **x64 only.** This is where all interop lives. Depends on the ObjectARX SDK and an AutoCAD installation.
- **`OPMNetSample/` (`asdkOPMNetSample`)** — a pure C# example plugin (`.csproj`, SDK-style) showing how to consume the shim. References `asdkOPMNetExt.vcxproj`. Not required to build the shim.

## Architecture (the shim)

Everything lives in the `Autodesk::AutoCAD::Windows::OPM` namespace, deliberately mirroring where these APIs would sit in the managed AutoCAD assemblies.

- **Managed COM interfaces** — `IDynamicProperty2` / `IDynamicPropertyNotify2` (`IDynamicProperty2.h`) and `IPropertyManager2` (`IPropertyManager2.h`). These are `[ComVisible]` `interface class`es whose **GUIDs, `InterfaceType`, and marshalling attributes exactly mirror the native OPM interfaces** in ObjectARX's `dynprops.h`. They are the managed face of the native COM contract.
- **RXObject wrappers** — `AcMgdOPMPropertyExtension` (`OPMPropertyExtension.*`) and `AcMgdOPMPropertyExtensionFactory` (`OPMPropertyExtensionFactory.*`). These derive from managed `RXObject`, carry a native `OPMPropertyExtension*`/`OPMPropertyExtensionFactory*` (via `GetImpObj()`), and marshal calls across the managed/native boundary using `Marshal::GetObjectForIUnknown` / `GetIUnknownForObject`.
- **`xOPM`** (`xOPM.*`) — the static entry helper. `xGET_OPMPROPERTY_MANAGER(RXClass^)` is the key call: it looks up the native `OPMPropertyExtensionFactory` in the class dictionary, creates the OPM protocol object for a given `RXClass`, and returns its `IPropertyManager2`.
- **ARX plumbing** — `acrxEntryPoint.cpp` (`IMPLEMENT_ARX_ENTRYPOINT`, RDS = `asdk`), `OPMNetExt.cpp` (`DllMain`), `StdAfx.*` (PCH), `AssemblyInfo.cpp`, `OPMNetExt.rc`/`Resource.h`.

### Consumption pattern (see `OPMNetSample/customPropertySample.cs`)

1. A `[ComVisible]` C# class with `Guid` / `ProgId` / `ClassInterface(None)` / `ComDefaultInterface(typeof(IDynamicProperty2))` implements `IDynamicProperty2` (display name, value type e.g. `VT_I4`, get/set value, connect/disconnect notify sink).
2. In `IExtensionApplication.Initialize`, `Assembly.LoadFrom("asdkOPMNetExt.dll")`, resolve the target `RXClass` from `SystemObjects.ClassDictionary` (e.g. `"AcDbLine"`), get the `IPropertyManager2` via `xOPM.xGET_OPMPROPERTY_MANAGER`, and call `AddProperty`.
3. `Terminate` calls `RemoveProperty`.

When editing interop signatures, **the managed interface GUIDs and marshalling attributes must stay in lock-step with the native OPM headers** — a mismatch silently breaks COM QueryInterface at runtime rather than failing at compile time.

## Building

x64 only. Requires an installed AutoCAD and matching ObjectARX SDK, plus VS 2026 (`v145` toolset).

```bash
msbuild /t:build /p:Configuration=Debug;Platform=x64 OPMNetExt.sln
msbuild /t:build /p:Configuration=Release;Platform=x64 OPMNetExt.sln
```

There is **no test suite**. This is an in-process AutoCAD module — it can only be verified by `NETLOAD`ing the built DLLs into a running AutoCAD and inspecting the Properties Palette.

### SDK / version paths — retargeting AutoCAD versions

The AutoCAD/ARX version is pinned in **three** places that must be kept consistent:

- `arxConfig.props` — `<AcadDir>` and `<ArxSdk>` UserMacros, **and** the versioned link libraries in `AdditionalDependencies` (e.g. `acdb26.lib`, `ac1st26.lib`, `acge26.lib`, `acui26.lib`, `acismobj26.lib` — the `26` corresponds to the AutoCAD release, not the year).
- `OPMNetSample/asdkOPMNetSample.csproj` — the hard-coded `<AssemblySearchPaths>` pointing at the ARX SDK `inc\` folder (for `AcDbMgd` / `AcMgd` / `AcCoreMgd`).
- `.vcxproj` / `.csproj` `TargetFramework` values.

The SDK/AutoCAD paths are fixed as UserMacros in `arxConfig.props`; edit that file directly to point at a different install.

The project currently targets **AutoCAD 2027 / ARX 2027 SDK** and **.NET 10** (`net10.0` / `net10.0-windows`, lib suffix `26`).
