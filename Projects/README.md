# Projects Directory
This directory is a collection of various CBECC models used for testing, development, and reference.

## Year Directory Components
### `research`
Not mirrored to public repo. This directory is used to store models used for memos or other research tasks. Create new sub-folders as needed for each new group of research items so related models stay grouped and easy to find.

### Common for `non-residential` & `multi-family` & `single-family`
- `EAA` (Existing Additions and/or Alterations)
- `examples`
- `prototypes`
- `standard`
- `user-support`

### Common for `non-residential` & `multi-family`
- `ruleset-implementation`
- `sensitivity`

### Common for `multi-family` & `single-family`
- `cuac`

### Common for `multi-family`
- `schema-related`
- `contains-nonres-elements.json` - This json file is used to track the files that contain non-residential elements, such that non-res developers can easily identify and test these files in addition to the non-residential models. Any of these files when running should also assume these models could also be under the sub-directory `~not-for-release`. 

### Common for `single-family`
- `TDS`

### `~not-for-release` Directories
This sub-directory should exist in directories that are typically packaged within a release. Please reference the `Projects/release-package.json` for detail. **TODO:** Create and note the workflow noted in #772

**NOTE:** the `.gitkeep` files in these directories are used to ensure the directories are tracked by Git even if they are empty. Do not remove these `.gitkeep` files.

### `~intentional-failures` Directories
This sub-directory should exist in directories that are used to test the system's ability to handle failures. Currently it only exists as needed. Files in these directories are also not-for-release.

**Important:** Assure to include a README.md file in each of these directories to explain how/why these files are intended to fail.

### `release-compliance` Directories
Per-year directory of severed model copies used to verify positive compliance and **no water marking** on generated reports during release testing. Models do not need zero compliance margin. See `Projects/<year>/release-compliance/README.md` in each year that has this folder ([#932](https://github.com/NOR-Codes-Stds/CBECC-Dev/issues/932)).

### `standard` Directories
These models are designed for ~0% compliance margin where Proposed model = Baseline model for each end-use. Useful for ruleset checking. 

### `user-support` Directories
Not mirrored to public repo. These directories in each of the core directories are used to store and track models submitted from user-support issues which contain valuable reach-cases that the current testing suite otherwise does not test. 

**TODO: A workflow/wiki page should be created to track and manage these files.**

### `README.md` Files
Add README files as needed to any directory if there is specific information to know about models in that directory.

# Other Key Files for Automated Workflows
## `release-package.json`
This file is used to track the files that are typically packaged within a release. Please reference the `Projects/release-package.json` for detail. **TODO:** Create and note the workflow noted in #772

## `pr-checks.json`
This file defines the sample models exercised by the **Test Small Sample Models** PR check (`.github/workflows/test_select_models.yml`).

- **When it runs:** The workflow runs only when the PR modifies files under `RulesetSrc/`, or when triggered manually (`workflow_dispatch`).
- **What it runs:** `prepare_and_run.py` reads each building-type section's `all` array (for example `nonres_multifam.all`) and runs those models through CBECC.
- **Adding models:** Add paths under the appropriate building type's `all` list. Paths may be directories (all IBD files copied) or individual `.cibd##` / `.ribd##` files.
- **`by_component`:** Reserved for future ruleset-component–specific model selection; not used by CI yet.
