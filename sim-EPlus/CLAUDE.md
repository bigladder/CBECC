# CLAUDE.md — sim-EPlus

This file provides guidance to Claude Code (claude.ai/code) when working in `sim-EPlus/`.

## Purpose

This folder holds the EnergyPlus simulation engine(s) that CBECC drives. CBECC is hard-wired to use **`sim-EPlus/current/`** (it is passed as the EnergyPlus path — e.g. `hybrid-hvac.rb` loads `current/Energy+.idd` and the engine runs `current/energyplus.exe`). To test an EnergyPlus source fix, the freshly built engine must be migrated **into `current/`**; you do not point CBECC at a different folder.

Subfolders:

- **`current/`** — the active engine CBECC uses. **Git-tracked** (~7k files). This is what you update when testing a fix.
- **`9-4/`** — an older EnergyPlus kept for reference. Git-tracked. Leave alone.
- **`EnergyPlus-PR-<NNNN>/`** — a **staging copy of a fresh build** pasted in from the EnergyPlus PR under test (e.g. `EnergyPlus-PR-11642/`). This is a raw CMake build-output dir (contains `.lib`, `.exp`, `energyplus_tests.exe`, `TestAPI_*.exe` — none needed at runtime). **Untracked and should stay that way** — do not commit it (it's ~800 MB of build artifacts). It exists only as the source to copy engine elements from.

## Migrating a new EnergyPlus build into `current/`

A build-output dir is **not** an EnergyPlus install, so you must copy a **subset** — the runtime "engine elements" — into `current/`, while preserving `current/`'s install-only files. This mirrors the historical migration commit (`dc6006e5`, "EnergyPlus engine elements updates based on new built").

### Copy these from `EnergyPlus-PR-<NNNN>/` → `current/` (overwrite)

- `energyplus.exe`, `energyplusapi.dll` — the engine + API DLL (the core of the fix).
- `Energy+.idd`, `Energy+.schema.epJSON` — input data dictionary + schema. Copy whenever the fix changes EnergyPlus input objects/fields (hybrid-equipment fixes often do).
- `BasementGHT.idd`, `SlabGHT.idd` — preprocessor IDDs.
- `ConvertInputFormat.exe`, `parser.exe` — input tooling rebuilt alongside the engine.
- `pyenergyplus/` — the Python API package (copy whole dir).
- `python_lib/` and `python<ver>.dll` — the embedded Python runtime. **Copy these together and matched to each other.** If the build bumps Python (it recently went 3.12 → 3.13), the DLL name changes; copy the new `python_lib/` + new `python<ver>.dll`, and delete the now-stale old `python<ver>.dll` from `current/` (the last migration left `python312.dll` behind next to `python313.dll` — avoid repeating that).
- `workflows/` — if changed in the build.

### Do NOT copy (build-only artifacts)

All `*.lib` (`energypluslib.lib`, `energyplusapi.lib`, `btwxt.lib`, etc.), `*.exp`, `energyplus_tests.exe`, and `TestAPI_*.exe`. CBECC never loads these; copying them just bloats `current/`.

### Do NOT delete from `current/` (install-only — absent from the build dir)

`EP-Launch.exe`, `Epl-run.bat`, `RunEPlus.bat`, `ExpandObjects.exe`, `PreProcess/`, `SetupOutputVariables.csv`, `Energy+500CstmMtrs.idd`, the docs, and especially the **MSVC runtime DLLs** (`vcruntime140*.dll`, `msvcp140*.dll`, `concrt140.dll`, `vcomp140.dll`). A wholesale folder replace would wipe these and break the engine — that's why the migration is selective, not a mirror copy.

> Caveat: the MSVC runtime DLLs above ship from a VC++ redistributable, not the build. If `energyplus.exe` fails to start after a migration with a missing/incompatible DLL error, the build was compiled against a newer MSVC toolset than `current/`'s runtime DLLs — update the VC++ redistributable files in `current/` to match.

### Example (PowerShell, run from repo root)

```powershell
$src = "sim-EPlus\EnergyPlus-PR-11642"
$dst = "sim-EPlus\current"
# Engine elements (files)
'energyplus.exe','energyplusapi.dll','Energy+.idd','Energy+.schema.epJSON',
'BasementGHT.idd','SlabGHT.idd','ConvertInputFormat.exe','parser.exe' |
  ForEach-Object { Copy-Item "$src\$_" "$dst\$_" -Force }
# Python runtime (match dll name to the build's version)
Copy-Item "$src\python313.dll" "$dst\python313.dll" -Force
robocopy "$src\pyenergyplus" "$dst\pyenergyplus" /MIR | Out-Null
robocopy "$src\python_lib"   "$dst\python_lib"   /MIR | Out-Null
# If Python version changed, remove the stale dll, e.g.:
# Remove-Item "$dst\python312.dll" -ErrorAction SilentlyContinue
```

Adjust the source folder name to the current `EnergyPlus-PR-<NNNN>` build.

## After migrating

1. Recompile the rulesets (see the repo-root `CLAUDE.md` — run `CompileRules-*.bat` from `CBECC/`).
2. Run the `Test/` models and validate **both** the CBECC compliance outputs (`<model> - AnalysisResults.xml`, `NRCCPRF`) **and** the underlying EnergyPlus run outputs under `Test/<model> - run/` (`.csv`/`.sql`/`.err`).
3. Commit the resulting `sim-EPlus/current/` changes with a message noting the source build (date or PR number), following the historical style: *"EnergyPlus engine elements updates based on new built dated <date>"*. Do **not** commit the `EnergyPlus-PR-<NNNN>/` staging dir.
