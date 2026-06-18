# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

CBECC (California Building Energy Code Compliance) software. This repo is a **distribution + working tree**, not the engine's source tree: the C++ compliance engine ships here as **prebuilt Windows binaries** (`CBECC/*.exe`, `CBECC/*.dll`). The editable source that actually lives here is:

- `RulesetSrc/` — the **ruleset source** (the building energy code expressed as a data model + rules), compiled into `.bin` files the engine loads.
- `CBECC/Modelkit/` — **Ruby** scripts and templates that bridge CBECC and EnergyPlus for Hybrid HVAC modeling.

Day-to-day work is almost always editing rulesets or Modelkit templates, then recompiling/simulating — not building C++.

## Project background & current goal

We are **fixing bugs in the hybrid evaporative cooling system** as modeled end-to-end by CBECC — specifically in **(a) the EnergyPlus source code** that simulates the hybrid evaporation equipment and **(b) the CBECC Modelkit** integration that generates the EnergyPlus input for it. **This repository is a build of CBECC used to test those fixes**; the EnergyPlus source itself is built elsewhere and the resulting binaries are dropped in here (see below).

The iteration loop for validating a fix:

1. **Update Modelkit** — adjust `CBECC/Modelkit/hybrid-hvac.rb` and/or the `CBECC/Modelkit/templates/*.pxt` hybrid templates.
2. **Swap in the new EnergyPlus build** — a fresh EnergyPlus build (containing the source fix) is pasted into `sim-EPlus/` as `EnergyPlus-PR-<NNNN>/`, then its runtime "engine elements" are migrated into `sim-EPlus/current/`, which is the engine CBECC actually uses. This is a **selective copy, not a wholesale folder replace** — see `sim-EPlus/CLAUDE.md` for the exact procedure and caveats.
3. **Recompile the rulesets** — run the appropriate `CompileRules-*.bat` from `CBECC/` (see "Compiling rulesets" below).
4. **Test the CBECC models** — run the models in `Test/` (e.g. `OffSml-AdvIDEC`, `OffSml-Non-AdvIDEC`, `OffSml-VentOnly`) and validate **both** the CBECC compliance outputs (`<model> - AnalysisResults.xml`, `NRCCPRF`) **and** the underlying EnergyPlus run outputs under `Test/<model> - run/` (`.csv`/`.sql`/`.err`).

The `Test/` models are deliberately paired to isolate the hybrid path — an advanced-IDEC case vs. a non-advanced-IDEC case vs. a ventilation-only case — so a fix's effect on the hybrid system can be compared against the controls.

Representative fixes fall into the two categories above: EnergyPlus-side modeling of the hybrid equipment (e.g. changing the interpolation method on the performance lookup `Table:IndependentVariable` objects in the `.pxt` templates) and Modelkit/CBECC plumbing (e.g. passing the correct space-type availability schedule through to the hybrid system).

## Version / code-year naming

Binaries and rulesets are suffixed by code cycle, and these suffixes appear everywhere:

- `22c` = 2022 (Commercial/Nonres), `25` = 2025, `25c` = 2025 Commercial, `28` = 2028.
- Per cycle there is a matched set: `CBECC-25.exe` (GUI), `CBECC-CLI25.exe` (headless analysis), `BEMCmpMgr25.dll` (Compliance Manager — the core), `BEMProc25.dll` (Building Energy Model processor), `BEMCompiler25.exe` (ruleset compiler), `OS_Wrap25.dll` (OpenStudio wrapper).

**The current hybrid-cooling work targets the 2025 cycle exclusively** — `T24_2025` (Nonres/Multifamily) and `CA Res 2025` (Single-Family), `.cibd25x` projects. Use the `25` toolset (`BEMCompiler25.exe`, `CBECC-CLI25.exe`, `CBECC-25.exe`) and the `CompileRules-T24_2025.bat` / `CompileRules_ALL_2025.bat` scripts. Other cycles (`22c`, `28`) are present but not the focus.

When touching anything version-specific, keep the suffix consistent across the ruleset source, the compiler used, and the output `.bin`.

## Compiling rulesets (the primary build step)

Ruleset source (`RulesetSrc/`) must be compiled into `Data/Rulesets/*.bin` before the engine sees changes. **All compile scripts assume the working directory is `CBECC/`** (paths inside are relative to it). From `CBECC/`:

- `CompileRules-T24_2025.bat` — 2025 Nonres/Multifamily (`T24N_2025`).
- `CompileRules_ALL_2025.bat` — 2025 Nonres **and** Single-Family in one run.
- `CompileRules_SFam_2025.bat` — 2025 Single-Family only (`CA Res 2025`).
- `CompileRules-T24N_2022.bat`, `CompileRules-T24_2028.bat` — other cycles.

Each invokes `BEMCompiler<ver>.exe ... --compileDM --compileRules`, writing e.g. `Data/Rulesets/T24_2025.bin` and copying screens/tooltips/RTF assets into `Data/Rulesets/<ruleset>/`. **Check the exit code / the `_*.out` log** (e.g. `_T24-2025 Rules Log.out`) for compile errors — the scripts `pause` on failure. Recompiling is the change that produces the modified `.bin` and `- Input/Sim Data Model.txt` files seen in commits.

## Ruleset architecture

The ruleset is a **data model plus rules**, both plain text compiled by `BEMCompiler`:

- **Data model** — `RulesetSrc/BEMBase.txt` (Nonres/Multifamily) and `RulesetSrc/BEMBase-SFam.txt` (Single-Family) define the BEM object hierarchy (components, properties, defaults). The compiler emits two views per ruleset: `<name> - Input Data Model.txt` (user input objects) and `<name> - Sim Data Model.txt` (simulation objects), plus the binary `<name>.bin`.
- **Rules** — `RulesetSrc/T24NRMF/` (Nonres/Multifamily) and `RulesetSrc/T24SFam/` (Single-Family) hold `*.rule` / `*.txt` rule sources, with `RulesetSrc/shared/` for cross-cutting screens/assets. Rules are evaluated by the Compliance Manager against the data model.
- Rules run in **phases** keyed off the model lifecycle — e.g. `Rules_PreSim.rule` (before each simulation) and `Rules_Modelkit.rule` (the hook that invokes the Modelkit/EnergyPlus integration described below). Trace this chain when a value isn't where you expect: data model definition → rule that sets/defaults it → sim phase that consumes it.

## Modelkit ↔ EnergyPlus integration (Hybrid HVAC)

For Hybrid HVAC systems, the engine cannot express the equipment directly in its IDF, so it delegates to Ruby Modelkit during simulation:

1. A ruleset rule (`RulesetSrc/T24NRMF/Rules_Modelkit.rule`) invokes `CBECC/Modelkit/hybrid-hvac.rb` via `CBECC/Modelkit/modelkit-catalyst/bin/modelkit.bat` (a bundled Ruby + OpenStudio runtime — do not treat `modelkit-catalyst/` as project source; it's vendored).
2. CBECC writes, into the simulation processing dir, `<base> - HybridHVAC-initial.idf` (partial IDF) and `<base> - HybridHVAC.csv` (per-zone parameter values).
3. `hybrid-hvac.rb` reads the CSV, and for each zone calls `Modelkit::Parametrics.template_compose` on `templates/hybrid-hvac.pxt`, substituting the CSV values, and appends the composed EnergyPlus objects to the final `<base>.idf`.

The `templates/*.pxt` files are **parametric EnergyPlus IDF templates** (`.pxt` = Modelkit parametric template) defining `Table:IndependentVariableList` / `Table:IndependentVariable` / lookup objects that describe hybrid-equipment performance. Variants encode the equipment + control combinations:

- `hybrid-tables-common.pxt` — shared independent-variable / lookup scaffolding.
- `standard` vs `advanced`, and `idec` vs `iec` — direct vs indirect evaporative cooling at standard/advanced tiers (e.g. `hybrid-tables-advanced-idec.pxt`).

Editing hybrid performance means changing these `.pxt` tables and/or `hybrid-hvac.rb`'s parameter mapping; the values flow CSV → `.pxt` → composed IDF → EnergyPlus.

## Simulation engines & supporting dirs

- `sim-EPlus/` — EnergyPlus engine, in versioned subdirs plus `current/`; `hybrid-hvac.rb` receives the EnergyPlus path as its second argument and loads `Energy+.idd` from it.
- `sim-CSE/` — California Simulation Engine (used for residential calculations).
- `Projects/` — CBECC test/reference models organized by building type and use (see `Projects/README.md`). `Projects/research/` is **not mirrored to the public repo**.
- `Test/` — local simulation outputs; git-ignored (see `.gitignore`). Keep generated run artifacts here so they stay untracked.

## Git LFS

`.gitattributes` tracks specific large ruleset binaries (`**/T24N_2022.bin`, `**/T24_2025.bin`) via Git LFS. Note that many other large `.bin`/`.dll` files are committed **directly** (not LFS) and GitHub flags them on push — that's expected for this repo, not an error to fix.

## Commits

- Do **not** add any Claude or AI attribution to commits. Specifically:
  - No `Co-Authored-By: Claude ...` trailer.
  - No "Generated with Claude Code" (or similar) footer.
  - No mention of Claude, Anthropic, or AI in commit messages.
- Author commits using the repository's configured git identity only.
