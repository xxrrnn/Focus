# 03 执行路径图

## 为什么要看执行路径

论文在 `doc/paper/tex/5.semantic.tex`、`doc/paper/tex/6.vector.tex`、`doc/paper/tex/7.evaluation.tex` 里讲的是方法与结论；真正把这些落地的是仓库里的执行链。要理解“论文 ↔ 代码 ↔ 结果”，必须看清楚命令是怎么走的。

## 流程 1：Focus trace 生成

| 步骤 | 文件路径 | 发生了什么 |
| --- | --- | --- |
| 1 | `algorithm/run_eval.py` | 解析 Focus 相关 CLI 参数 |
| 2 | `algorithm/lmms-eval/lmms_eval/evaluator.py` | 加载模型和任务 |
| 3 | `algorithm/focus/interface.py` | 给支持的模型打补丁，替换 forward |
| 4 | `algorithm/focus/models/llava_video/modeling_llava_video.py` 等 | 计算 patch/frame/token 元数据，并调用 `self.focus.prepare(...)` |
| 5 | `algorithm/focus/models/qwen2/modeling_qwen2.py` | 在 decoder 执行中调用 SIC 和 SEC |
| 6 | `algorithm/focus/main.py` | 汇总稀疏统计并保存中位样本 trace |

### 产物

- `focus_main/<model>_<dataset>.pth`
- `meta_data.csv`

### 对应论文

- SEC：`doc/paper/tex/5.semantic.tex`
- SIC：`doc/paper/tex/6.vector.tex`

## 流程 2：精度评估

| 步骤 | 文件路径 | 发生了什么 |
| --- | --- | --- |
| 1 | `algorithm/run_*.sh` 或直接 CLI | 启动不同模型/任务组合 |
| 2 | `algorithm/run_eval.py` | 通过 `lmms-eval` 执行评测 |
| 3 | `algorithm/run_eval.py::get_score_from_results(...)` | 统一不同任务的评分口径 |
| 4 | `algorithm/run_eval.py::save_score_to_main_csv(...)` / `save_score_to_dse_csv(...)` | 写出论文用 CSV |

### 产物

- `accuracy.csv`
- `dse_a_m_tile_accuracy.csv`
- `dse_b_vector_accuracy.csv`
- `dse_c_block_accuracy.csv`

### 对应论文

- `doc/paper/tex/7.evaluation.tex` 中的 accuracy、sparsity、INT8、image-VLM 表格

## 流程 3：模拟器

| 步骤 | 文件路径 | 发生了什么 |
| --- | --- | --- |
| 1 | `simulator/main.py` | 解析模拟模式 |
| 2 | `simulator/models/models.py` | 从 `meta_data.csv` 读取序列统计信息 |
| 3 | `simulator/models/sparse_info.py` | 读取 Focus trace 或基线稀疏度 CSV |
| 4 | `simulator/arch/accelerator.py` | 配置硬件模型 |
| 5 | `simulator/core/simulator.py` | 聚合周期、访存、能耗 |
| 6 | `simulator/utils/utils.py` | 结果追加到 CSV |

### 产物

- `main_focus.csv`
- `main_dense.csv`
- `main_adaptiv.csv`
- `main_cmc.csv`
- `main_focus_SEC_only.csv`
- `int8_focus.csv`
- `dse_*.csv`

### 对应论文

- `doc/paper/tex/7.evaluation.tex` 中的性能、能耗、DSE、访存、SEC-only 消融、INT8、image-VLM 讨论

## 流程 4：绘图 / 表格

| 论文产物 | 真实生成来源 |
| --- | --- |
| 主性能与能耗图 | `evaluation_scripts/plot_scripts/ipynb_src/figure_9.ipynb` + simulator CSV + `jetson_stats/figure9_gpu.csv` |
| DSE 图 | `figure_10.ipynb` + DSE CSV |
| 消融图 | `figure_11.ipynb` + `main_focus_SEC_only.csv` |
| 访存图 | `figure_12.ipynb` |
| 精度/稀疏度表 | `table_2.ipynb` |
| INT8 表 | `table_4.ipynb` |
| 图像 VLM 表 | `table_5.ipynb` |
| 最坏情况图 | `simulator/utils/analysis.py` |

## 流程 5：RTL 派生出的面积 / 功耗

| 步骤 | 文件路径 | 发生了什么 |
| --- | --- | --- |
| 1 | `rtl/*.v`, `rtl/*.sv` | 硬件模块源文件 |
| 2 | `simulator/arch/*_rtl.csv` | 预先提取好的组件面积/功耗统计 |
| 3 | `simulator/arch/accelerator.py` | 读取组件统计并拼成整机面积/功耗 |

### 重要边界

- **直接观察**：运行时模拟器并不会执行 RTL。
- **推断**：仓库采用的是“RTL 派生统计 -> 模拟器”，而不是“RTL 联合仿真”。

## 关键 artifact 交接点

| 生产者 | artifact | 消费者 |
| --- | --- | --- |
| `algorithm/focus/main.py` | Focus `.pth` trace | `simulator/models/sparse_info.py` |
| `algorithm/focus/main.py` | `meta_data.csv` | `simulator/models/models.py` |
| `algorithm/focus/baseline_adaptiv.py` | `adaptiv_sparsity.csv` | `simulator/models/sparse_info.py` |
| `algorithm/focus/baseline_CMC.py` | `cmc_sparsity.csv` | `simulator/models/sparse_info.py` |
| `simulator/main.py` | `main_*.csv`, `dse_*.csv` | notebook |

## 结论

- **直接观察**：这个项目的真实主线是 `run_eval.py -> trace/CSV artifact -> simulator/main.py -> notebook/图表`。
- **推断**：如果你要对照论文读代码，这就是最正确的主心骨。
