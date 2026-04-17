# 13 Config Reference

This file summarizes the most important configuration sources that shape repository behavior.

## Algorithm Defaults

## `algorithm/focus/configs/focus.csv`

### Purpose

- Default Focus hyperparameters per model and dataset.

### Key Columns

- `model`
- `dataset`
- `selected_layers`
- `alpha_list`
- `similarity_threshold`

### Read By

- `algorithm/focus/interface.py::replace_focus_forward(...)`

### Meaning

- `selected_layers`: decoder layers where SEC is applied
- `alpha_list`: keep ratios used by SEC at those layers
- `similarity_threshold`: SIC merge threshold

## `algorithm/focus/configs/adaptiv.csv`

### Purpose

- Default Adaptiv thresholds per model and dataset.

### Key Column

- `adaptiv_threshold`

### Read By

- `algorithm/focus/interface.py::replace_focus_forward(...)`

## `algorithm/focus/configs/cmc.csv`

### Purpose

- Default CMC thresholds per model and dataset.

### Key Columns

- `CMC_threshold`
- `CMC_query_threshold`
- `CMC_attn_threshold`

### Read By

- `algorithm/focus/interface.py::replace_focus_forward(...)`

## User-Facing Algorithm Knobs

Important CLI parameters in `algorithm/run_eval.py`:

| Parameter | Meaning |
| --- | --- |
| `--focus` | Enable Focus patch path. |
| `--CMC` | Enable CMC patch path. |
| `--adaptiv` | Enable Adaptiv patch path. |
| `--frame_fusion` | Enable FrameFusion patch path. |
| `--similarity_threshold` | Override Focus SIC threshold. |
| `--block_size` | Spatial SIC block range. |
| `--frame_block_size` | Temporal SIC block range. |
| `--alpha_list` | SEC keep-ratio schedule. |
| `--selected_layers` | SEC target layers. |
| `--gemm_m_size` | Focus GEMM tile restriction for matching. |
| `--vector_size` | SIC vector granularity. |
| `--SEC_only` | Disable SIC behavior and keep only SEC. |
| `--export_focus_trace` | Write Focus `.pth` trace. |
| `--trace_dir` | Trace file output directory. |
| `--trace_name` | Trace filename stem. |
| `--trace_meta_dir` | Metadata and accuracy output directory. |
| `--use_median` | Choose median sparse sample when exporting trace. |

## Simulator Structure Configs

## `simulator/arch/accelerator.py`

This is the main hardware configuration source.

### Focus knobs

- `focus_m_tile_size`
- `num_scatter_vector`
- `block_size`
- `SEC_only`

### Fixed architectural assumptions

- frequency = 500 MHz
- DRAM bandwidth = 64 GB/s
- DRAM energy per byte = `99.98 * 1e-9` mJ/byte
- default Focus array = 32x32

## `simulator/models/models.py`

This file hardcodes:

- hidden dimension = 3584
- heads = 28
- blocks = 28
- projection sizes for modeled layer types

It also imports runtime workload size from `meta_data.csv`.

## ScaleSim Configs

## `simulator/core/scalesim_cfg/config.cfg`

### Purpose

- Template config rewritten before each ScaleSim call.

### Important fields

- `ArrayHeight`
- `ArrayWidth`
- `Dataflow`
- SRAM sizes and offsets

### Updated By

- `simulator/core/simulator_comp.py::call_scalesim(...)`

## `simulator/core/scalesim_cfg/gemm.csv`

### Purpose

- Template topology CSV rewritten before each ScaleSim call.

### Important columns

- `M`
- `N`
- `K`

### Updated By

- `simulator/core/simulator_comp.py::call_scalesim(...)`

## SRAM Modeling Configs

## `simulator/memory/buffer_model_spec.csv`

### Purpose

- Small lookup table for compiler-like SRAM macro area/power estimates.

### Used By

- `simulator/memory/buffer.py`
- `simulator/arch/accelerator.py`

## `simulator/memory/sram_config.json`

### Purpose

- Default CACTI config template used when the simulator must evaluate a custom SRAM configuration.

### Used By

- `simulator/memory/cacti.py`

## RTL Summary Configs

### Files

- `simulator/arch/focus_rtl.csv`
- `simulator/arch/adaptiv_rtl.csv`
- `simulator/arch/cmc_rtl.csv`

### Purpose

- Component-level area/power lookup tables derived from RTL synthesis.

### Used By

- `simulator/arch/accelerator.py::get_components_area_power(...)`

## Which Configs Are Most Important To A Researcher

- **Direct observation**: for algorithm behavior, the most important files are `algorithm/focus/configs/focus.csv`, `adaptiv.csv`, and `cmc.csv`.
- **Direct observation**: for hardware behavior, the most important file is `simulator/arch/accelerator.py`, because that is where many parameters stop being abstract and become actual buffer counts and component counts.
- **Reasoned inference**: for extending the repo, the highest-leverage config change is usually either Focus defaults on the algorithm side or buffer/array settings on the simulator side.
