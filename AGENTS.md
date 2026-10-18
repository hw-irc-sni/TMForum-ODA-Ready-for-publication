# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A publication staging repo for TM Forum ODA (Open Digital Architecture) components. It is mostly **generated content**: the factory code lives in private repos (ODA CI, ODA Component Conformance). The git history is almost entirely automated commits ("Updating api index from GitHub Actions"), and `Reports/` holds generated CSVs and S3 listings. Treat hand edits to generated assets with care; they may be overwritten by the pipeline.

## Layout

One top-level directory per component, named `TMFCnnn-ComponentName/` (about 36 of them), each containing:

- `Specification/TMFCnnn-*.yaml`: the **golden component specification**, an `oda.tmforum.org/v1beta4` Component resource. The key sections are `spec.componentMetadata`, `coreFunction`, `managementFunction`, `securityFunction` (with `dependentAPIs` and `exposedAPIs`) and event notifications. `README.md` at the root documents the attributes and the v1beta3 → v1beta4 changes: metadata moved under `componentMetadata`, `specification` became a list of `{url, version}`, `controllerRole` was renamed `canvasSystemRole`, and `eTOMs` and `functionalFrameworkFunctions` were added.
- `CTK/`: the Component Test Kit.
  - `Dockerfile` builds an image with Python and Node that runs `componentCTK/run.sh`.
  - `componentCTK/` is the CTK itself (see below).
  - `ComponentRI/` (only some components, e.g. TMFC001, 005 and 006) is a Helm chart for a reference implementation with stubs. It has a `Component` CR template plus mongodb, API and metrics deployments.
- `ComponentConformanceProfile/` (only some components) and the `TMFC*.pdf` hold the human-readable conformance profile and spec.

`Reports/{ApiOverview,FunctionalOverview,Metadata}/` hold timestamped CSV reports aggregated across components. The `s3_list_*.txt` files are listings of the TM Forum API index bucket.

## CTK architecture (`CTK/componentCTK/`)

The CTK is **copied per component**. `src/` is identical across components (checked by diffing TMFC001 against TMFC005). A change to the CTK logic must therefore be applied to every `TMFC*/CTK/componentCTK/` copy. Per-component differences are mainly `features/*.feature` files and the spec YAML. Before editing, diff a copy against another to confirm which parts really are shared.

Flow:
1. `scripts/CTK_Executor.py` is the orchestrator, driven by `CHANGE_ME.json` (release name, component under test, namespace, CTK version, optional-API toggles, `standardComponentDownload` pointing at the GitHub branch that hosts these specs). It downloads the golden spec of the component from this repo, then fetches the TM Forum runtime API index (`tmf-open-api-table-documents.s3...Indexes/index.json`) and converts it to a legacy format. For each exposed, dependent or security API it downloads the matching API-level CTK zip into `resources/api-ctks/` and runs it against the live canvas, then consolidates the results into JSON.
2. `src/` is a Node app (`index.js`, `tests/{configuration,CoreFunction,ManagementFunction,deployment,Reporting}.js`, `tests/SecurityFunction.json`). It compares the component actually deployed on the ODA Canvas with the golden spec and renders HTML and JSON reports using `reporting/index.mustache`. `features/` holds Cucumber feature files and step definitions for some components. The `package.json` scripts are `start` (`node index.js`) and `test` (`cucumber-js` with html and json formatters). `.mocharc.json` configures mochawesome output to `../resources/results/baseline-ctk`.

Run it (it needs a reachable Kubernetes ODA Canvas with the component already installed, plus Docker or Python and Node):
```bash
# edit CHANGE_ME.json first (releaseName, component_to_run, ctkconfig.*)
cd TMFCnnn-*/CTK/componentCTK
./MAC-LINUX-RUN.sh        # or WINDOWS-RUN.bat; installs scripts/requirements.txt, runs CTK_Executor.py
# or via Docker, from TMFCnnn-*/CTK:  docker build -t ctk . && docker run ... (CMD runs componentCTK/run.sh)
```
Reports are written to `Reports/<ComponentName>/` under the run directory. There is no repo-wide build, lint or unit-test setup. `componentCTK/README` has the full instructions and the `CHANGE_ME.json` field reference.

## Notes

- Current working branch convention: the default/main branch is `v1.0.0`. The CTK's `gitBranch` setting references the branch that serves specs, so keep spec file paths (`Specification/<name>.yaml`) stable.
- The `ctkVersion` in `CHANGE_ME.json` selects which API CTK versions are fetched from the index.
