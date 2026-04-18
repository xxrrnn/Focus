# 18 Trace 格式参考

这个仓库里，算法层与模拟器层之间的关键边界不是共享内存对象，而是序列化后的 trace 契约。主要生产者是 `algorithm/focus/main.py`，主要消费者是 `simulator/models/sparse_info.py`。

## 主要输出目录

| trace 根目录下的路径 | 生产者 | 消费者 | 内容 |
| --- | --- | --- | --- |
| `focus_main/` | `algorithm/run_focus.sh` 与 `algorithm/focus/main.py::post_process(...)` | `simulator/models/sparse_info.py` 在 `type='focus'` 时 | Focus `.pth` trace |
| `focus_int8/` | `algorithm/run_focus.sh int8` | `SparseInfo(..., dse='quantization')` | INT8 Focus `.pth` trace |
| `m_tile_size_dse/` | `algorithm/run_dse.sh` | `SparseInfo(..., dse='m_tile_size_*')` | tile size 的 DSE trace |
| `block_size_dse/` | `algorithm/run_dse.sh` | `SparseInfo(..., dse='block_size_*')` | block size 的 DSE trace |
| `vector_size_dse/` | `algorithm/run_dse.sh` | `SparseInfo(..., dse='vector_size_*')` | vector size 的 DSE trace |
| `meta_data.csv` | `algorithm/focus/main.py::post_process(...)` | `simulator/models/models.py` | 每个 model/dataset 的序列元数据 |
| `adaptiv_sparsity.csv` | `algorithm/focus/baseline_adaptiv.py` | `simulator/models/sparse_info.py` | baseline 的聚合 sparsity |
| `cmc_sparsity.csv` | `algorithm/focus/baseline_CMC.py` | `simulator/models/sparse_info.py` | baseline 的聚合 sparsity |

## `meta_data.csv`

从 `algorithm/example_output/meta_data.csv` 可直接观察到的列：

- `Model`
- `Dataset`
- `Sequence length`
- `Num frames`
- `Num patches`
- `Median index`

生产者：

- `algorithm/focus/main.py::post_process(...)` 通过 `algorithm/focus/utils.py` 中的 `save_result_to_csv(...)` 写出。

消费者：

- `simulator/models/models.py::ModelConfig.add_seq_len()` 读取该文件，并设置 `seq_len`、`num_frames`、`num_patches`。

## Focus `.pth` Trace 结构

生产者：

- `algorithm/focus/main.py::prepare(...)` 负责分配 trace tensor。
- `algorithm/focus/main.py::focus_similarity_concentration(...)` 负责填充。
- `algorithm/focus/main.py::post_process(...)` 负责保存选中的 trace。

顶层 key：

- `mask_zero`
- `mask_similar`
- `group_idx`

内部 operation key：

- `q_proj`
- `o_proj`
- `query`
- `gate_proj`
- `down_proj`

这些 tensor 的含义：

- `mask_zero`：为零的向量掩码，很多时候来自 SEC 引入的 token dropping。
- `mask_similar`：被判定为冗余相似向量的布尔掩码。
- `group_idx`：代表向量/分组编号，用于后续重建共享关系。

从 `algorithm/focus/main.py::prepare(...)` 可观察到的分配形状：

- 第一维：`num_layers`
- 第二维：当前实现固定为 `1`
- 第三维：`image_token_length`
- 第四维：每个 token 被切成多少个 vector，由 `hidden_dim` 或 `intermediate_dim` 与 `vector_size` 决定

从 `algorithm/focus/main.py::prepare(...)` 可观察到的 dtype：

- `mask_zero[name]`：`torch.bool`
- `mask_similar[name]`：`torch.bool`
- `group_idx[name]`：`torch.int32`

## Baseline CSV 格式

### `adaptiv_sparsity.csv`

从 `algorithm/example_output/adaptiv_sparsity.csv` 可观察到的列：

- `Model`
- `Dataset`
- `Sparsity`

### `cmc_sparsity.csv`

从 `algorithm/example_output/cmc_sparsity.csv` 可观察到的列：

- `Model`
- `Dataset`
- `linear_sparsity`
- `query_sparsity`
- `attn_score_sparsity`

## Accuracy CSV

### `accuracy.csv`

生产者：

- `algorithm/run_eval.py::save_score_to_main_csv(...)`

消费者：

- `evaluation_scripts/plot_scripts/ipynb_src/table_2.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_4.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_5.ipynb`

从 `algorithm/example_output/accuracy.csv` 可观察到的列：

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

### DSE accuracy CSV

生产者：

- `algorithm/run_eval.py::save_score_to_dse_csv(...)`

可观察到的文件名：

- `dse_a_m_tile_accuracy.csv`
- `dse_b_vector_accuracy.csv`
- `dse_c_block_accuracy.csv`

常见字段：

- 扫描参数，例如 `m_tile_size`、`vector_size`、`block_size`
- 对应 accuracy 值

## 模拟器结果 CSV

### 主实验输出

生产者：

- `simulator/main.py` 及其结果写出辅助逻辑

常见文件：

- `main_focus.csv`
- `main_dense.csv`
- `main_adaptiv.csv`
- `main_cmc.csv`
- `main_focus_SEC_only.csv`
- `int8_focus.csv`

从 `simulator/example_sim_results/main_focus.csv` 可观察到的列：

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

### DSE 输出

可观察到的文件名：

- `dse_a_m_tile_size.csv`
- `dse_b_vector_size.csv`
- `dse_c_block_size.csv`
- `dse_d_num_scatter_accumulator.csv`

常见附加字段：

- `m_tile_size`
- `vector_size`
- `block_size`
- `buffer_area`
- `num_scatter`

## 功耗 / 面积分解 CSV

文件：

- `accelerator_area_power_buffer.csv`
- `detailed_power_area_breakdown.csv`

生产者：

- `simulator/arch/accelerator.py`
- `simulator/core/simulator.py::get_detailed_power_area_breakdown(...)`

常见字段：

- `component`
- `area`
- `power`
- `energy`

## 这个契约最值得注意的地方

- 直接观察：simulator 不需要加载真实模型代码，只需要 `meta_data.csv` 和 trace 文件或 sparsity CSV。
- 直接观察：`simulator/models/sparse_info.py` 会把 Focus trace 的 sequence length 与 `ModelConfig.seq_len` 做一致性校验。
- 推断：这种文件格式边界，是整个仓库最值得学习的工程设计之一，因为它把昂贵的模型执行与便宜的模拟重跑彻底解耦了。
- 直接观察：Focus 的 `.pth` trace 是全仓库结构上最重要的产物，因为论文中的稀疏性正是通过它被转译成 simulator 可消费的硬件证据。
