# 01 必看入口点

## 什么才算“真实入口点”

在这个仓库里，真实入口点不是“某个可能重要的模块”，而是用户真的会运行、并且会驱动论文工作流某一阶段的命令或 notebook。

## 一级入口点

| 阶段 | 用户命令 | 实际入口文件 | 对应论文位置 | 主要输出 |
| --- | --- | --- | --- | --- |
| 算法评估 / trace 导出 | `cd algorithm && python -m run_eval ...` | `algorithm/run_eval.py` | `doc/paper/tex/5.semantic.tex`、`6.vector.tex`、`7.evaluation.tex` | trace、metadata、accuracy CSV |
| 架构模拟 | `cd simulator && python main.py ...` | `simulator/main.py` | `doc/paper/tex/7.evaluation.tex` | 延迟/能耗/访存 CSV |
| 面积功耗汇总 | `cd simulator && python arch/accelerator.py --output_dir ...` | `simulator/arch/accelerator.py` | `doc/paper/tex/7.evaluation.tex` | `accelerator_area_power_buffer.csv` |
| 最坏情况分析 | `cd simulator && python utils/analysis.py --trace_dir ...` | `simulator/utils/analysis.py` | `doc/paper/tex/7.evaluation.tex` 的 worst/best-case 分析 | `figure_13.svg` |
| 绘图 / 表格 | 运行 `evaluation_scripts/plot_scripts/ipynb_src/` 下 notebook | notebook 自身 | `doc/paper/tex/7.evaluation.tex` | 论文图表 |

## 算法侧真正的入口

### `algorithm/run_eval.py`

- **直接观察**：所有算法脚本最后都会落到 `python -m run_eval`。
- **直接观察**：`algorithm/run_eval.py` 统一负责：
  - Focus / CMC / Adaptiv / FrameFusion 开关
  - trace 导出参数
  - 量化参数
  - 精度 CSV 写出
- **直接观察**：它再进入 `algorithm/lmms-eval/lmms_eval/evaluator.py`。

### 为什么它最重要

- 它是真正承载论文算法实验的控制面。
- 论文里需要的 `accuracy.csv`、DSE accuracy 表，都是它写出来的。

## 算法包装脚本

| 脚本 | 底层入口 | 在论文中的角色 |
| --- | --- | --- |
| `algorithm/run_focus.sh` | `algorithm/run_eval.py` | Focus 视频任务主实验：trace 生成 + 全量精度 |
| `algorithm/run_original.sh` | `algorithm/run_eval.py` | 稠密基线精度 |
| `algorithm/run_framefusion.sh` | `algorithm/run_eval.py` | FrameFusion 精度 |
| `algorithm/run_adaptiv.sh` | `algorithm/run_eval.py` | Adaptiv 稀疏度 / 精度 |
| `algorithm/run_cmc.sh` | `algorithm/run_eval.py` | CMC 稀疏度 / 精度 |
| `algorithm/run_dse.sh` | `algorithm/run_eval.py` | Focus DSE trace 与粗略精度 |
| `algorithm/run_focus_image.sh` | `algorithm/run_eval.py` | 论文 Discussion 中的图像 VLM Focus 实验 |
| `algorithm/run_adaptiv_image.sh` | `algorithm/run_eval.py` | 图像 VLM 的 Adaptiv 基线 |
| `algorithm/run_original_image.sh` | `algorithm/run_eval.py` | 图像 VLM 稠密基线 |

## 模拟器真正的入口

### `simulator/main.py`

- **直接观察**：所有模拟器脚本最终都调用 `python main.py`。
- **直接观察**：`simulator/main.py` 统一负责：
  - 主对比实验
  - DSE 模式
  - INT8 模式
  - SEC-only 消融
  - image-task 模式

### 模拟器包装脚本

| 脚本 | 底层入口 | 在论文中的角色 |
| --- | --- | --- |
| `simulator/run_main_sim.sh` | `simulator/main.py` | 主性能与能耗结果 |
| `simulator/run_dse_sim.sh` | `simulator/main.py` | DSE 图的输入 |
| `simulator/run_image_sim.sh` | `simulator/main.py` | 图像 VLM 讨论表格的输入 |

## 重要但不是用户命令的内部入口

| 文件 | 为什么重要 |
| --- | --- |
| `algorithm/lmms-eval/lmms_eval/evaluator.py` | 这里真正触发 `apply_focus(...)` 和 `apply_framefusion(...)` |
| `algorithm/focus/interface.py` | 这里决定支持哪类模型、替换哪些 forward |
| `simulator/core/simulator.py` | 这里把 Focus trace 变成延迟、stall、DRAM、能耗结果 |

## Notebook 入口

| Notebook | 对应论文产物 |
| --- | --- |
| `figure_9.ipynb` | 主性能 / 能耗 / 功耗面积 breakdown |
| `figure_10.ipynb` | DSE |
| `figure_11.ipynb` | 消融 |
| `figure_12.ipynb` | DRAM / activation 分析 |
| `table_2.ipynb` | 精度 + 计算稀疏度 |
| `table_4.ipynb` | INT8 研究 |
| `table_5.ipynb` | 图像 VLM 研究 |

## 主入口与辅助入口

### 主入口

- `algorithm/run_eval.py`
- `simulator/main.py`
- `simulator/arch/accelerator.py`
- `simulator/utils/analysis.py`
- `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`

### 辅助入口

- `algorithm/run_*.sh`
- `simulator/run_*.sh`
- `algorithm/lmms-eval/lmms_eval/evaluator.py`
- `algorithm/focus/interface.py`
- `simulator/core/simulator.py`

## 结论

- **直接观察**：这个仓库最核心的两个可运行程序其实只有 `algorithm/run_eval.py` 和 `simulator/main.py`。
- **推断**：其他脚本要么是它们的包装层，要么是内部调度边界，要么是下游的图表呈现层。
