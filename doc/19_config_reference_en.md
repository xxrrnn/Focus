# 19 Config Reference

This reference collects the most important configuration surfaces across algorithm and simulator layers.

## Algorithm Method Configs

### `algorithm/focus/configs/focus.csv`

Observed columns:

- `model`
- `dataset`
- `selected_layers`
- `alpha_list`
- `similarity_threshold`

Meaning:

- `selected_layers`: decoder layers where SEC is applied.
- `alpha_list`: retention ratios for semantic pruning at those layers.
- `similarity_threshold`: cosine-similarity threshold for SIC matching.

Default paper-aligned example:

- For `llava_vid/videomme`, the file contains `selected_layers=[3, 6, 9, 18, 26]`, `alpha_list=[0.4, 0.3, 0.2, 0.15, 0.1]`, and `similarity_threshold=0.9`, matching the narrative in `doc/paper/tex/7.evaluation.tex`.

### `algorithm/focus/configs/adaptiv.csv`

Observed column:

- `adaptiv_threshold`

This is loaded by `algorithm/focus/interface.py` when `--adaptiv` is used.

### `algorithm/focus/configs/cmc.csv`

Observed columns:

- `CMC_threshold`
- `CMC_query_threshold`
- `CMC_attn_threshold`

This is loaded by `algorithm/focus/interface.py` when `--CMC` is used.

## Important Algorithm CLI Knobs

Defined in `algorithm/run_eval.py`:

- `--focus`
- `--frame_fusion`
- `--adaptiv`
- `--CMC`
- `--SEC_only`
- `--export_focus_trace`
- `--trace_dir`
- `--trace_name`
- `--trace_meta_dir`
- `--use_median`
- `--gemm_m_size`
- `--vector_size`
- `--block_size`
- `--frame_block_size`
- `--write_accuracy`
- `--write_accuracy_table_name`

## Simulator Architecture Defaults

Key defaults in `simulator/arch/accelerator.py`:

- `frequency = 500 MHz`
- Focus `focus_m_tile_size = 1024` by default
- Focus `num_scatter_vector = 2`, which corresponds to 64 accumulators when array width is 32
- Focus layouter buffer and similarity-map/table buffers are explicitly modeled in `set_focus_config(...)`
- Dense configuration reuses much of Focus’s systolic setup via `set_dense_config()`
- AdapTiV and CMC have separate buffer dictionaries in `set_adaptiv_config()` and `set_cmc_config()`

## Modeled Layer Template

`simulator/models/models.py` hardcodes:

- hidden dimension `3584`
- `28` heads
- `28` decoder blocks
- projection sizes for `q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj`, and `attn`

It then imports workload sizing from `meta_data.csv`.

## Simulator Entry Modes

Flags in `simulator/main.py`:

- `--all_models_datasets`
- `--image_models_datasets`
- `--SEC_only`
- `--m_tile_size_dse`
- `--block_size_dse`
- `--vector_size_dse`
- `--num_scatter_dse`
- `--quantization`

## ScaleSim Configs

### `simulator/core/scalesim_cfg/config.cfg`

Purpose:

- template config rewritten before each ScaleSim call

Important fields:

- `ArrayHeight`
- `ArrayWidth`
- `Dataflow`
- SRAM sizes and offsets

Updated by:

- `simulator/core/simulator_comp.py::call_scalesim(...)`

### `simulator/core/scalesim_cfg/gemm.csv`

Purpose:

- template GEMM topology CSV rewritten before each ScaleSim call

Important columns:

- `M`
- `N`
- `K`

Updated by:

- `simulator/core/simulator_comp.py::call_scalesim(...)`

## SRAM / Memory Model Configs

### `simulator/memory/buffer_model_spec.csv`

Purpose:

- lookup table for compiler-style SRAM macro area/power estimates

Used by:

- `simulator/memory/buffer.py`
- `simulator/arch/accelerator.py`

### `simulator/memory/sram_config.json`

Purpose:

- CACTI config template for custom SRAM evaluation

Used by:

- `simulator/memory/cacti.py`

## RTL Summary Configs

Files:

- `simulator/arch/focus_rtl.csv`
- `simulator/arch/adaptiv_rtl.csv`
- `simulator/arch/cmc_rtl.csv`

Purpose:

- precomputed component-level area/power summaries derived from RTL synthesis

Used by:

- `simulator/arch/accelerator.py::get_components_area_power(...)`

## Config Design Pattern Worth Reusing

- Method-specific hyperparameters live in small CSV files under `algorithm/focus/configs/`.
- Experiment modes are selected at the CLI level in `algorithm/run_eval.py` and `simulator/main.py`.
- Paper-facing defaults are encoded close to the execution entry points instead of being buried deep in notebooks.
- Direct observation: for hardware behavior, `simulator/arch/accelerator.py` is the highest-leverage config file, because abstract parameters become concrete buffer counts, widths, and component inventories there.
