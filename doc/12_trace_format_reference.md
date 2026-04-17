# 12 Trace Format Reference

This file documents the concrete artifact formats that connect `algorithm/` to `simulator/` and `evaluation_scripts/`.

## Focus Trace File

### Producer

- `algorithm/focus/main.py::Focus.post_process(...)`

### Consumer

- `simulator/models/sparse_info.py`

### File Location

- Default main traces: `focus_main/<model>_<dataset>.pth`
- INT8 traces: `focus_int8/<model>_<dataset>.pth`
- DSE traces:
  - `m_tile_size_dse/<model>_<dataset>_<m_tile_size>.pth`
  - `block_size_dse/<...>.pth`
  - `vector_size_dse/<model>_<dataset>_<vector_size>.pth`

### File Structure

- **Direct observation**: the saved object is a Python dict written with `torch.save(...)`.
- **Direct observation**: top-level keys are:
  - `mask_zero`
  - `mask_similar`
  - `group_idx`
- **Direct observation**: each top-level key maps to another dict keyed by operation name:
  - `q_proj`
  - `o_proj`
  - `query`
  - `gate_proj`
  - `down_proj`

### Tensor Shapes

From `algorithm/focus/main.py::prepare(...)`:

- `mask_zero[name]`: `torch.bool`
- `mask_similar[name]`: `torch.bool`
- `group_idx[name]`: `torch.int32`

Shape formulas:

- For `q_proj`, `o_proj`, `query`, `gate_proj`:
  - `(num_layers, 1, image_token_length, ceil(hidden_dim / vector_size))`
- For `down_proj`:
  - `(num_layers, 1, image_token_length, ceil(intermediate_dim / vector_size))`

### Meaning Of The Fields

| Field | Meaning |
| --- | --- |
| `mask_zero` | Whether a vector position is already zero, often due to SEC token dropping. |
| `mask_similar` | Whether a vector position was merged away because it is similar to a previous vector. |
| `group_idx` | Cluster/group assignment used for concentration and later hardware modeling. |

## `meta_data.csv`

### Producer

- `algorithm/focus/main.py::Focus.post_process(...)`

### Consumer

- `simulator/models/models.py::ModelConfig.add_seq_len(...)`

### Columns

From `algorithm/example_output/meta_data.csv` and `algorithm/focus/main.py`:

- `Model`
- `Dataset`
- `Sequence length`
- `Num frames`
- `Num patches`
- `Median index`

### Purpose

- `Sequence length`, `Num frames`, and `Num patches` feed workload sizing in the simulator.
- `Median index` is used when traces are exported without `--use_median`; the code can look up the stored median sample index and reuse it.

## `adaptiv_sparsity.csv`

### Producer

- `algorithm/focus/baseline_adaptiv.py::post_process(...)`

### Consumer

- `simulator/models/sparse_info.py`

### Columns

From `algorithm/example_output/adaptiv_sparsity.csv`:

- `Model`
- `Dataset`
- `Sparsity`

## `cmc_sparsity.csv`

### Producer

- `algorithm/focus/baseline_CMC.py::post_process(...)`

### Consumer

- `simulator/models/sparse_info.py`

### Columns

From `algorithm/example_output/cmc_sparsity.csv`:

- `Model`
- `Dataset`
- `linear_sparsity`
- `query_sparsity`
- `attn_score_sparsity`

## `accuracy.csv`

### Producer

- `algorithm/run_eval.py::save_score_to_main_csv(...)`

### Consumer

- `evaluation_scripts/plot_scripts/ipynb_src/table_2.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_4.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_5.ipynb`

### Columns

From `algorithm/example_output/accuracy.csv`:

- `Models`
- `Dataset`
- `Metric`
- `Dense`
- `FF`
- `Adaptiv`
- `CMC`
- `Focus`
- `INT8_Dense`
- `INT8_Focus`

## DSE Accuracy CSVs

### Producer

- `algorithm/run_eval.py::save_score_to_dse_csv(...)`

### File Names

- `dse_a_m_tile_accuracy.csv`
- `dse_b_vector_accuracy.csv`
- `dse_c_block_accuracy.csv`

### Columns

- `m_tile_size`
- `block_size`
- `vector_size`
- `accuracy`

## Main Simulator Result CSVs

### Producer

- `simulator/main.py` through `simulator/utils/utils.py::save_result(...)`

### Common Files

- `main_focus.csv`
- `main_dense.csv`
- `main_adaptiv.csv`
- `main_cmc.csv`
- `main_focus_SEC_only.csv`
- `int8_focus.csv`

### Typical Columns

From `simulator/example_sim_results/main_focus.csv`:

- `model`
- `dataset`
- `accelerator`
- `execution_time`
- `total_cycles`
- `total_compute_cycles`
- `total_stall_cycles`
- `total_energy`
- `total_dram_access`
- `dram_bandwidth`
- `dense_ops`
- `num_ops`
- `mem_counter_total`
- `total_activation`
- `total_compression_ratio`
- `dram_energy`
- `sram_energy`
- `core_energy`

## DSE Simulator CSVs

### File Names

- `dse_a_m_tile_size.csv`
- `dse_b_vector_size.csv`
- `dse_c_block_size.csv`
- `dse_d_num_scatter_accumulator.csv`

### Additional Columns

- `m_tile_size`
- `block_size`
- `vector_size`
- `buffer_area`
- `num_scatter`

depending on the DSE mode.

## Power / Area Breakdown CSVs

### Files

- `accelerator_area_power_buffer.csv`
- `detailed_power_area_breakdown.csv`

### Producer

- `simulator/arch/accelerator.py`
- `simulator/core/simulator.py::get_detailed_power_area_breakdown(...)`

### Typical Breakdown Columns

- `component`
- `area`
- `power`
- `energy`

## Practical Reading Notes

- **Direct observation**: the Focus trace `.pth` file is the most structurally important artifact in the repository.
- **Direct observation**: the simulator does not need raw model logs; it only needs `meta_data.csv` and either the Focus `.pth` or the baseline sparsity CSVs.
- **Reasoned inference**: if you extend the project, preserving these file contracts is more important than preserving any one internal helper function.
