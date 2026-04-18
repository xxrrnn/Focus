# 18 Trace Format Reference

The algorithm/simulator boundary in this repo is a serialized trace contract. The main producer is `algorithm/focus/main.py`; the main consumer is `simulator/models/sparse_info.py`.

## Main Output Directories

| Path under trace root | Produced by | Consumed by | Contents |
| --- | --- | --- | --- |
| `focus_main/` | `algorithm/run_focus.sh` and `algorithm/focus/main.py::post_process(...)` | `simulator/models/sparse_info.py` when `type='focus'` | Focus `.pth` traces |
| `focus_int8/` | `algorithm/run_focus.sh int8` | `SparseInfo(..., dse='quantization')` | INT8 Focus `.pth` traces |
| `m_tile_size_dse/` | `algorithm/run_dse.sh` | `SparseInfo(..., dse='m_tile_size_*')` | DSE traces for tile size |
| `block_size_dse/` | `algorithm/run_dse.sh` | `SparseInfo(..., dse='block_size_*')` | DSE traces for block size |
| `vector_size_dse/` | `algorithm/run_dse.sh` | `SparseInfo(..., dse='vector_size_*')` | DSE traces for vector size |
| `meta_data.csv` | `algorithm/focus/main.py::post_process(...)` | `simulator/models/models.py` | per-model/dataset sequence metadata |
| `adaptiv_sparsity.csv` | `algorithm/focus/baseline_adaptiv.py` | `simulator/models/sparse_info.py` | aggregate baseline sparsity |
| `cmc_sparsity.csv` | `algorithm/focus/baseline_CMC.py` | `simulator/models/sparse_info.py` | aggregate baseline sparsity |

## `meta_data.csv`

Observed columns from `algorithm/example_output/meta_data.csv`:

- `Model`
- `Dataset`
- `Sequence length`
- `Num frames`
- `Num patches`
- `Median index`

Producer:

- `algorithm/focus/main.py::post_process(...)` writes this via `save_result_to_csv(...)` in `algorithm/focus/utils.py`.

Consumer:

- `simulator/models/models.py::ModelConfig.add_seq_len()` reads it and sets `seq_len`, `num_frames`, and `num_patches`.

## Focus `.pth` Trace Structure

Producer:

- `algorithm/focus/main.py::prepare(...)` allocates the trace tensors.
- `algorithm/focus/main.py::focus_similarity_concentration(...)` fills them.
- `algorithm/focus/main.py::post_process(...)` saves the selected trace.

Top-level keys:

- `mask_zero`
- `mask_similar`
- `group_idx`

Inner operation keys:

- `q_proj`
- `o_proj`
- `query`
- `gate_proj`
- `down_proj`

Tensor meanings:

- `mask_zero`: boolean mask for vectors that are zero, often due to SEC-induced token dropping.
- `mask_similar`: boolean mask for vectors that were matched as redundant.
- `group_idx`: integer representative/group assignment used to reconstruct vector sharing.

Observed allocation shapes in `algorithm/focus/main.py::prepare(...)`:

- first dimension: `num_layers`
- second dimension: currently `1`
- third dimension: `image_token_length`
- fourth dimension: number of vectors per token, based on `hidden_dim` or `intermediate_dim` divided by `vector_size`

Observed tensor dtypes from `algorithm/focus/main.py::prepare(...)`:

- `mask_zero[name]`: `torch.bool`
- `mask_similar[name]`: `torch.bool`
- `group_idx[name]`: `torch.int32`

## Baseline CSV Formats

### `adaptiv_sparsity.csv`

Observed columns from `algorithm/example_output/adaptiv_sparsity.csv`:

- `Model`
- `Dataset`
- `Sparsity`

### `cmc_sparsity.csv`

Observed columns from `algorithm/example_output/cmc_sparsity.csv`:

- `Model`
- `Dataset`
- `linear_sparsity`
- `query_sparsity`
- `attn_score_sparsity`

## Accuracy CSVs

### `accuracy.csv`

Producer:

- `algorithm/run_eval.py::save_score_to_main_csv(...)`

Consumers:

- `evaluation_scripts/plot_scripts/ipynb_src/table_2.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_4.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_5.ipynb`

Observed columns from `algorithm/example_output/accuracy.csv`:

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

### DSE accuracy CSVs

Producer:

- `algorithm/run_eval.py::save_score_to_dse_csv(...)`

Observed file names:

- `dse_a_m_tile_accuracy.csv`
- `dse_b_vector_accuracy.csv`
- `dse_c_block_accuracy.csv`

Typical fields:

- sweep parameter such as `m_tile_size`, `vector_size`, or `block_size`
- accuracy value

## Simulator Result CSVs

### Main simulator outputs

Producer:

- `simulator/main.py`, through the simulator save helpers

Common files:

- `main_focus.csv`
- `main_dense.csv`
- `main_adaptiv.csv`
- `main_cmc.csv`
- `main_focus_SEC_only.csv`
- `int8_focus.csv`

Observed columns from `simulator/example_sim_results/main_focus.csv`:

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

### DSE simulator outputs

Observed file names:

- `dse_a_m_tile_size.csv`
- `dse_b_vector_size.csv`
- `dse_c_block_size.csv`
- `dse_d_num_scatter_accumulator.csv`

Typical extra columns:

- `m_tile_size`
- `vector_size`
- `block_size`
- `buffer_area`
- `num_scatter`

## Power / Area Breakdown CSVs

Files:

- `accelerator_area_power_buffer.csv`
- `detailed_power_area_breakdown.csv`

Producers:

- `simulator/arch/accelerator.py`
- `simulator/core/simulator.py::get_detailed_power_area_breakdown(...)`

Typical breakdown fields:

- `component`
- `area`
- `power`
- `energy`

## Important Contract Details

- Direct observation: the simulator does not require live model code; it only needs `meta_data.csv` plus trace files or sparsity CSVs.
- Direct observation: `simulator/models/sparse_info.py` validates Focus trace sequence length against `ModelConfig.seq_len`.
- Reasoned inference: this file-format boundary is one of the repository’s best engineering decisions because it cleanly decouples expensive model execution from cheap simulation reruns.
- Direct observation: the Focus `.pth` trace is the single most structurally important artifact in the repo, because it is where paper-level sparsity becomes simulator-consumable hardware evidence.
