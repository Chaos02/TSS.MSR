# PCPTool (v11)

Sample/reference command-line tool for the Windows TPM Platform Crypto Provider (PCP) and TPM-based key attestation. Originally published by Microsoft in the [Platform Crypto Provider Toolkit](https://www.microsoft.com/en-us/download/details.aspx?id=52487); this `v11` tree is the modernized fork that retargets the projects from the Windows 8 SDK / `v110` toolset to the Windows 8.1 SDK / `v140` toolset.

The accompanying whitepaper is checked in alongside the sources:
- [Using the Windows 8 Platform Crypto Provider and Associated TPM Functionality.pdf](Using%20the%20Windows%208%20Platform%20Crypto%20Provider%20and%20Associated%20TPM%20Functionality.pdf)

## Layout

- [PCPTool.sln](PCPTool.sln) — solution containing two projects:
  - [dll/dll.vcxproj](dll/dll.vcxproj) — `TpmAtt.dll` (attestation API, WBCL log parser, exported via [dll/TpmAtt.def](dll/TpmAtt.def))
  - [exe/exe.vcxproj](exe/exe.vcxproj) — `PCPTool.exe`, the CLI front-end that links against `TpmAtt.lib`
- [inc/](inc/) — public headers (`TpmAtt.h`, `InlineFn.h`)
- [misc/](misc/) — sample `.cmd` scripts and `.inf` cert templates exercising the tool

## Prerequisites

- Windows 10/11 with TPM 2.0
- One of:
  - Visual Studio 2022 (any edition) with the **Desktop development with C++** workload, or
  - Visual Studio Build Tools 2022 with MSVC v143 + a Windows 10/11 SDK (≥ 10.0.22621)

The checked-in `.vcxproj` files declare `<PlatformToolset>v140</PlatformToolset>` (VS 2015). On a modern toolchain you'll override the toolset and SDK on the MSBuild command line — see below.

## Build

From a "Developer PowerShell for VS 2022" prompt, or by invoking MSBuild directly:

```powershell
& "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\MSBuild\Current\Bin\MSBuild.exe" `
    PCPTool.sln `
    -p:Configuration=Release -p:Platform=x64 `
    -p:PlatformToolset=v143 `
    -p:WindowsTargetPlatformVersion=10.0.26100.0
```

Adjust `WindowsTargetPlatformVersion` to whichever Windows SDK is installed under `C:\Program Files (x86)\Windows Kits\10\Include\`.

Outputs land in `x64\Release\`:
- `PCPTool.exe`
- `TpmAtt.dll` / `TpmAtt.lib`

`Debug` and `Win32` configurations are also defined in the solution.

## Quick smoke test

```powershell
.\x64\Release\PCPTool.exe GetVersion
```

Run the executable with no arguments to see the full command list (RNG, EK/SRK management, key creation, attestation, PCR-bound keys, etc.). The scripts in [misc/](misc/) are runnable end-to-end examples.

## Modifications relative to the upstream Microsoft sources

The Windows 10 SDK introduced `<wbcl.h>`, which now declares the same WBCL types and SIPA event structures that this project's [inc/TpmAtt.h](inc/TpmAtt.h) was carrying as private copies. Building against a modern SDK without changes produces `C2011` (struct redefinition) and `C2375` (linkage mismatch) errors. The following changes resolve that without altering runtime behavior:

- [inc/TpmAtt.h](inc/TpmAtt.h) — removed the local declarations of `SIPAEVENT_VSM_IDK_RSA_INFO`, `SIPAEVENT_VSM_IDK_INFO_PAYLOAD`, `SIPAEVENT_SI_POLICY_PAYLOAD`, `SIPAEVENT_REVOCATION_LIST_PAYLOAD`, `_WBCL_Iterator`, the `WBCL_DIGEST_*` macros, and the `WbclApi*` function prototypes. They now come from the SDK's `<wbcl.h>`, which is already included via [dll/stdafx.h](dll/stdafx.h) and [exe/stdafx.h](exe/stdafx.h).
- [dll/TpmAtt.def](dll/TpmAtt.def) — explicitly exports `WbclApiInitIterator`, `WbclApiGetCurrentElement`, and `WbclApiMoveToNextElement`. The SDK's wbcl.h declarations have no `__declspec(dllexport)`, so the in-tree implementations in [dll/PCPWbcl.cpp](dll/PCPWbcl.cpp) need to be exported by name for `PCPTool.exe` to link against them.
- [dll/dll.vcxproj](dll/dll.vcxproj) — added `<ModuleDefinitionFile>TpmAtt.def</ModuleDefinitionFile>` to all four `<Link>` configurations so the .def file is actually consumed by the linker.

If you ever need to rebuild against an SDK that predates `<wbcl.h>` (Windows SDK 8.1 or Windows 10 SDK 10240), revert these three changes.
