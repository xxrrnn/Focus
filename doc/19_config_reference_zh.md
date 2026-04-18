# 19 配置参考

这份参考文档把算法层与模拟器层最重要的配置入口集中列出来。

## 算法方法配置

### `algorithm/focus/configs/focus.csv`

可直接观察到的列：

- `model`
- `dataset`
- `selected_layers`
- `alpha_list`
- `similarity_threshold`

含义：

- `selected_layers`：在哪些 decoder layer 上启用 SEC。
- `alpha_list`：这些层对应的 semantic pruning 保留比例。
- `similarity_threshold`：SIC 的 cosine similarity 阈值。

与论文默认值最接近的例子：

- 对 `llava_vid/videomme`，文件中给出 `selected_layers=[3, 6, 9, 18, 26]`、`alpha_list=[0.4, 0.3, 0.2, 0.15, 0.1]`、`similarity_threshold=0.9`，与 `doc/paper/tex/7.evaluation.tex` 的叙述一致。

### `algorithm/focus/configs/adaptiv.csv`

可观察到的关键列：

- `adaptiv_threshold`

该配置在 `algorithm/focus/interface.py` 中由 `--adaptiv` 路径加载。

### `algorithm/focus/configs/cmc.csv`

可观察到的关键列：

- `CMC_threshold`
- `CMC_query_threshold`
- `CMC_attn_threshold`

该配置在 `algorithm/focus/interface.py` 中由 `--CMC` 路径加载。

## 重要算法 CLI 参数

定义在 `algorithm/run_eval.py` 中：

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

## 模拟器架构默认值

`simulator/arch/accelerator.py` 中的关键默认值：

- `frequency = 500 MHz`
- Focus 默认 `focus_m_tile_size = 1024`
- Focus 默认 `num_scatter_vector = 2`，结合阵列宽度 32，对应 64 accumulators
- Focus 的 layouter buffer、similarity-map、similarity-table 都在 `set_focus_config(...)` 中显式建模
- Dense 配置通过 `set_dense_config()` 复用了大量 Focus 的 systolic 基础设置
- AdapTiV 与 CMC 在 `set_adaptiv_config()`、`set_cmc_config()` 中有独立 buffer 配置

## 建模层模板

`simulator/models/models.py` 中硬编码了：

- hidden dimension `3584`
- `28` 个 attention heads
- `28` 个 decoder blocks
- `q_proj`、`k_proj`、`v_proj`、`o_proj`、`gate_proj`、`up_proj`、`down_proj`、`attn` 的投影尺寸

然后再通过 `meta_data.csv` 注入 workload 规模信息。

## 模拟器入口模式

`simulator/main.py` 中的主要模式开关：

- `--all_models_datasets`
- `--image_models_datasets`
- `--SEC_only`
- `--m_tile_size_dse`
- `--block_size_dse`
- `--vector_size_dse`
- `--num_scatter_dse`
- `--quantization`

## ScaleSim 配置

### `simulator/core/scalesim_cfg/config.cfg`

用途：

- 每次调用 ScaleSim 前会被重写的模板配置

重要字段：

- `ArrayHeight`
- `ArrayWidth`
- `Dataflow`
- 各类 SRAM 大小与偏移

更新位置：

- `simulator/core/simulator_comp.py::call_scalesim(...)`

### `simulator/core/scalesim_cfg/gemm.csv`

用途：

- 每次调用 ScaleSim 前会被重写的 GEMM 拓扑模板 CSV

重要列：

- `M`
- `N`
- `K`

更新位置：

- `simulator/core/simulator_comp.py::call_scalesim(...)`

## SRAM / 内存模型配置

### `simulator/memory/buffer_model_spec.csv`

用途：

- 近似 memory-compiler 风格的 SRAM 宏面积/功耗查找表

使用位置：

- `simulator/memory/buffer.py`
- `simulator/arch/accelerator.py`

### `simulator/memory/sram_config.json`

用途：

- 自定义 SRAM 配置时给 CACTI 使用的模板

使用位置：

- `simulator/memory/cacti.py`

## RTL 汇总配置

文件：

- `simulator/arch/focus_rtl.csv`
- `simulator/arch/adaptiv_rtl.csv`
- `simulator/arch/cmc_rtl.csv`

用途：

- 保存来自 RTL 综合的组件级面积/功耗汇总

使用位置：

- `simulator/arch/accelerator.py::get_components_area_power(...)`

## 值得复用的配置模式

- 方法超参用小型 CSV 放在 `algorithm/focus/configs/` 下，便于管理和审计。
- 实验模式在 `algorithm/run_eval.py` 与 `simulator/main.py` 的 CLI 层集中切换。
- 面向论文的默认值被放在靠近执行入口的位置，而不是埋在 notebook 深处。
- 直接观察：就硬件行为而言，`simulator/arch/accelerator.py` 是最关键的配置文件，因为很多抽象参数都会在这里落成具体 buffer 数量、位宽和组件清单。
