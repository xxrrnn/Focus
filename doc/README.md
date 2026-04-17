# Focus Repository Documentation Suite

This documentation suite was written by tracing the code in `algorithm/`, `simulator/`, `rtl/`, `evaluation_scripts/`, and the glue around `3rd_party/`. It is meant to replace a repository re-read, not restate the top-level README.

## How To Read These Docs

Observation labels used throughout the suite:

- **Direct observation**: verified directly in the repository files.
- **Reasoned inference**: inferred from several files working together.
- **Unclear**: still ambiguous after inspection.

## Document Index

| File | What it contains |
| --- | --- |
| `doc/00_repo_overview.md` | Repository purpose, trimmed tree, subsystem boundaries, and high-level data flow. |
| `doc/01_mandatory_entry_points.md` | Every important runnable entry point, what file it actually enters, and why it matters. |
| `doc/02_build_and_environment.md` | Dependency structure, submodules, environment setup, build requirements, and common failure points. |
| `doc/03_execution_map.md` | End-to-end command-to-artifact execution traces across algorithm, simulator, evaluation, and RTL linkage. |
| `doc/04_algorithm_deep_dive.md` | How Focus is injected into VLM execution, where traces come from, and where accuracy is recorded. |
| `doc/05_simulator_deep_dive.md` | Simulator architecture, trace ingestion, compute/memory/energy models, and key assumptions. |
| `doc/06_rtl_deep_dive.md` | RTL hierarchy, module roles, reconstructed hardware data flow, and the relation to simulator power/area tables. |
| `doc/07_evaluation_and_plotting.md` | How figures/tables are assembled from CSV outputs and notebooks. |
| `doc/08_paper_to_code_mapping.md` | Concept-to-code mapping table with implementation status and caveats. |
| `doc/09_code_reading_guide.md` | A practical reading order for a researcher joining the project. |
| `doc/10_open_questions_and_risks.md` | Known ambiguities, brittle spots, reproducibility risks, and good engineering patterns. |
| `doc/11_top_20_files.md` | The 20 most important files in the repository, with one paragraph each. |
| `doc/12_trace_format_reference.md` | Concrete artifact formats for Focus traces, metadata, sparsity CSVs, and simulator outputs. |
| `doc/13_config_reference.md` | Important config files and what each parameter controls. |
| `doc/14_glossary.md` | Project terminology and abbreviations. |
| `doc/15_architecture_ascii_map.md` | ASCII maps of cross-directory flow and accelerator structure. |

## Recommended Reading Orders

### Quick Overview

1. `doc/00_repo_overview.md`
2. `doc/03_execution_map.md`
3. `doc/10_open_questions_and_risks.md`

### Reproduce Results

1. `doc/02_build_and_environment.md`
2. `doc/01_mandatory_entry_points.md`
3. `doc/03_execution_map.md`
4. `doc/07_evaluation_and_plotting.md`
5. `doc/12_trace_format_reference.md`

### Learn The Algorithm Implementation

1. `doc/01_mandatory_entry_points.md`
2. `doc/04_algorithm_deep_dive.md`
3. `doc/12_trace_format_reference.md`
4. `doc/13_config_reference.md`

### Learn The Simulator Implementation

1. `doc/01_mandatory_entry_points.md`
2. `doc/05_simulator_deep_dive.md`
3. `doc/12_trace_format_reference.md`
4. `doc/13_config_reference.md`
5. `doc/15_architecture_ascii_map.md`

### Learn RTL Organization

1. `doc/06_rtl_deep_dive.md`
2. `doc/15_architecture_ascii_map.md`
3. `doc/05_simulator_deep_dive.md`

### Learn Reusable Research Engineering Patterns

1. `doc/00_repo_overview.md`
2. `doc/03_execution_map.md`
3. `doc/04_algorithm_deep_dive.md`
4. `doc/05_simulator_deep_dive.md`
5. `doc/10_open_questions_and_risks.md`

## Bottom Line

- **Direct observation**: the repository is organized as four mostly separate layers: model-side algorithm instrumentation in `algorithm/`, hardware/performance modeling in `simulator/`, synthesized hardware blocks in `rtl/`, and paper reproduction notebooks in `evaluation_scripts/`.
- **Reasoned inference**: this separation is intentional. Expensive VLM inference is done once to emit reusable sparse artifacts; simulator and plotting stages then reuse those artifacts many times without re-running the models.
- **Unclear**: there is no single script that runs the entire paper pipeline end to end across all subsystems. Reproduction is done by moving artifacts between separate stages.
