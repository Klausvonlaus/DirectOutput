# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

DirectOutput (DOF) is a Windows .NET Framework 4.8 add-in for Visual Pinball and
other virtual pinball cabinet software. It receives table-event data from the
simulator (via ROM/PinMAME element updates) and drives physical feedback
devices — lights, solenoids, shaker motors, RGB LED strips, etc. — through a
variety of USB/serial output controllers (LedWiz, PacLed, Pinscape, DudesCab,
DMX, Philips Hue, SSF, addressable LED strips, and more).

This is a C#/VB/C++ Visual Studio solution (`DirectOutput.sln`), not a
cross-platform or web project. There is no npm/pip/cargo tooling here.

## Build

Building requires **Windows with Visual Studio 2022** (MSBuild + .NET Framework
4.8 targeting pack); this cannot be built on Linux/macOS. If you're working in
a non-Windows environment, you can still read/edit/reason about the code, but
you cannot compile or run it — say so explicitly rather than claiming a build
succeeded.

- Open `DirectOutput.sln` in Visual Studio and build. It resolves project
  build order automatically.
- Configurations are combinations of `Debug`/`Release` and `x86`/`x64`
  (see `README_BuildSetup.md`). The main library (`DirectOutput.csproj`)
  targets `net48`.
- Command-line full release build: `BuildRelease.bat AUTHOR` (run from a VS
  Developer Command Prompt). It cleans and builds `Release|x86` and
  `Release|x64`, then produces `.msi`/`.zip` artifacts under `.\Builds`.
- Building the `.msi` installer (`DOFSetup` project) additionally requires the
  **WiX Toolset v3.14.0 exactly** (not 3.14.1 — see `README_BuildSetup.md` for
  why). Everything else builds fine without WiX; only the last MSI step will
  fail.
- `BuildGithubDocu.bat` / `BuildLocalWIPDocu.bat` generate the Doxygen-based
  API docs (see `.github/workflows/docs.yml`, which runs on `windows-latest`
  and publishes to GitHub Pages on pushes touching `DirectOutput/`,
  `Documentation/`, `DocumentationHelper/`, or `DoxygenConfig/`).

There is no automated test suite. The `Test` project (`Test/Program.cs`) is a
manual scratch/smoke-test console app, not a unit test project — there's no
`dotnet test` / xUnit / NUnit harness to run. Validate behavioral changes by
reasoning through the code and, where possible, by building and exercising the
relevant tester tool (see below) on Windows.

## Solution layout

The `.sln` contains ~20 projects. The important ones:

- **`DirectOutput/`** — the core framework (the actual DOF engine). Everything
  else either hosts it, configures it, or tests it. This is almost always
  where a real feature or bug fix lives.
- **`Extensions/`** — a small standalone library of generic C# extension
  methods (`StringExtensions`, `LinqExtensions`, etc.), referenced by other
  projects. Distinct from `DirectOutput/Extensions/`, which is a *second*,
  narrower set of extension methods used only inside the core framework.
- **`B2SServerPlugin/`** — plugin adapter that lets DOF run as a Backglass
  Server (B2S.Server) plugin.
- **`DirectOutput PinballX Plugin/`** — plugin/adapter for the PinballX
  frontend (game selection, ROM-to-table mapping via `Table2RomMapping/`).
- **`ProPinballBridge/`** (C++) and **`ProPinballSlave/`** — bridge process
  for integrating with Pro Pinball, communicating with the main DOF process
  via COM (`IDirectOutputCom`).
- **`DirectOutputComObject/`** (VB) + **`DirectOutputComObjectRegister/`** —
  expose DOF as a COM object for hosts that can't load a .NET assembly
  directly, plus its registration helper.
- **GUI/CLI tools for authoring and testing config**:
  `GlobalConfigEditor/` (edits per-host `GlobalConfig_*.xml`),
  `DirectOutputConfigTester/` (loads a cabinet/table config and lets you
  trigger table elements to see the resulting effects), `LedControlFileTester/`
  (validates/tests legacy `DirectOutput*.ini` "ledcontrol" files).
- **`DOFSetup/`**, `DOFSetupB2SFixup/`, `DOFSetupPBXFixup/` — WiX installer
  project and its custom actions.
- **`Documentation/`, `DocumentationHelper/`, `DoxygenConfig/`** — Doxygen-based
  API documentation generation, published to
  http://directoutput.github.io/DirectOutput/.
- **`AssemblyInfo/`** — shared assembly version info linked into other
  projects (`SharedAssemblyInfo.cs`).
- **`UpdateNotification/`** — small WinForms popup used to notify users of new
  releases.

## Core architecture (`DirectOutput/`)

Data flows in one direction, event-driven, through four layers:

1. **Ingress — `DirectOutput.cs` (`DirectOutputHandler`) / `Pinball.cs`**
   The static `DirectOutputHandler` class is the framework's public API
   surface, called by host apps (VPX, B2S, PinballX, Pro Pinball) via
   `Init()`, `UpdateTableElement()` / `UpdateNamedTableElement()`, and
   `Finish()`. It owns a single `Pinball` instance — the framework's root
   object, holding the `Table`, the `Cabinet`, the `GlobalConfig`, and the
   `AlarmHandler`. `Pinball.Setup()` loads the global config, table config,
   and cabinet config; `Pinball.Init()` starts the processing thread(s).
   Incoming data is pushed onto an internal queue (`PinballSupport/InputQueue.cs`)
   and processed asynchronously rather than inline on the caller's thread.

2. **`Table/`** — represents game state. A `Table` holds a `TableElementList`
   of `TableElement` objects (each identified by a type char + number, e.g.
   `S23`, or a name), keyed by config-driven mappings. Updates to a
   `TableElement`'s value are what trigger effects.

3. **`FX/`** — the effects layer, organized by sub-namespace per effect
   family: `ValueFX` (pass raw values), `RGBAFX`/`AnalogToyFX` (color/level
   effects), `MatrixFX` (bitmap/shape-driven effects for LED matrices,
   including `BitmapShapes`), `ListFX`, `ConditionFX`, `TimmedFX`, `NullFX`.
   All effects implement `IEffect`/derive from `EffectBase`. A `TableElement`
   is connected to one or more effects via an `AssignedEffect`
   (`AssignedEffectList`), which is the binding between "this table element
   changed" and "run this effect."

4. **`Cab/`** — the physical cabinet model that effects ultimately act on.
   - `Toys/` — logical output devices (lamps, shakers, RGB LEDs, layered/
     virtual toys, LedWiz-equivalent toys) that effects target.
   - `Out/` — one subfolder per physical output-controller protocol/vendor
     (`LW` = LedWiz, `Pac`, `FTDIChip`, `ComPort`, `DMX`, `PhilipsHue`, `SSF`,
     `PinOne`, `PS`/`PSPico` = Pinscape, `DudesCab`, `AdressableLedStrip`,
     `NullOutputController` for a no-op sink). Controllers implementing
     `IAutoConfigOutputController` are auto-detected by `Cabinet.AutoConfig()`.
   - `Schedules/`, `Sequencer/`, `Overrides/` — timing/scheduling of effect
     output and manual overrides.
   - `Cabinet` is serialized to/from XML (`xCabinet.xml`) and is the object
     graph edited by `GlobalConfigEditor`/`DirectOutputConfigTester`.

5. **`LedControl/`** — a parallel, legacy config format ("ledcontrol.ini"
   style, `LedControl/Loader/`) used to import table-specific LedWiz-era
   effect mappings (`TableConfig`, `TableConfigSetting`, `ColorConfig`, etc.)
   and translate them into the modern `FX`/`Cab` object model. `LedControl/Setup/Configurator.cs`
   does this translation. This is a distinct configuration pipeline from
   `xCabinet.xml` — expect to touch both if changing how legacy configs map
   to toys/effects.

Supporting namespaces: `General/` (curves, color models, bitmap handling,
generic collections, file-pattern matching, reflection helpers — utility code
used across the framework, not business logic), `GlobalConfiguration/`
(per-host `GlobalConfig` — logging, paths, etc.), `PinballSupport/` (input
queue, alarm/timer handling), `Frontend/` (the WinForms UI shown via
`DirectOutputHandler.ShowFrontend()` inside a hosting app).

## Configuration files (`DirectOutput/Config/`)

Ship-with-the-repo samples/templates:
- `GlobalConfig_B2SServer.xml` — per-host global config example.
- `xCabinet.xml` — cabinet/toy/output-controller definition example.
- `ledcontrol.ini`, `ledcontrol2.ini` (+ `.nofeedback` variants) — legacy
  LedControl-format table effect definitions.

At runtime, DOF locates its install folder and config files relative to the
running assembly; see the detailed lookup logic and fallback rules documented
in `DirectOutputHandler.GetInstallFolder()` and `GetGlobalConfigFileName()` in
`DirectOutput/DirectOutput.cs` — this handles both the legacy flat install
layout and the newer side-by-side `x86`/`x64` subfolder layout introduced for
dual 32/64-bit installs.

## Conventions worth knowing

- **32-bit/64-bit dual support is load-bearing.** Since the 2024 rework, the
  installer places binaries under `x86\`/`x64\` subfolders sharing one
  `Config\` folder at the parent level. Path/install-folder logic must keep
  working under both the old flat layout and this new layout — don't assume
  the assembly directory is the install root.
- **.NET Framework version upgrades need their own commit/PR**, isolated from
  functional changes (per `README_BuildSetup.md`), because tooling/version
  bumps can affect compatibility with older Windows versions that some pin
  cab users still run, and mixing them with logic changes makes bisecting
  regressions much harder.
- **WiX version is pinned to exactly 3.14.0** — do not "helpfully" upgrade it;
  3.14.1 is known to produce broken installs (see `README_BuildSetup.md`).
  WiX v4/v5 would require a full installer rewrite and is out of scope.
- Namespaces mirror folder structure closely (e.g. `DirectOutput.Cab.Out.LW`
  lives in `DirectOutput/Cab/Out/LW/`) — use that to navigate rather than
  grepping blindly.
- XML doc comments (`///`) are used throughout the core library and feed the
  Doxygen-generated public API docs — keep them accurate when changing public
  members in `DirectOutput/`.
- This is a long-lived community project (originally by "SwissLizard", now
  maintained as the "Grand Unified" edition); multiple output-controller and
  effect implementations exist side by side for hardware DOF still supports,
  so don't remove an apparently-unused `Cab/Out/*` or `FX/*` implementation
  without checking whether it's the only support for a real device.
