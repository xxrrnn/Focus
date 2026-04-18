# 00 仓库总览

## 文档目的

这个仓库不是单纯“实现了一个算法”，而是一个把论文方法、真实 VLM 推理、硬件模拟、RTL 模块、论文图表复现串起来的全栈研究工程。论文主线由 `doc/paper/main.tex` 及其分节文件 `doc/paper/tex/*.tex` 定义，尤其是：

- `doc/paper/tex/4.overview.tex`
- `doc/paper/tex/5.semantic.tex`
- `doc/paper/tex/6.vector.tex`
- `doc/paper/tex/7.evaluation.tex`

**直接观察**：代码仓库并不是按论文章节组织，而是按执行流水线组织。  
**推断**：这种组织方式更适合复现实验，也更适合把“昂贵的模型推理”和“可重复的硬件评估”解耦。

## 它解决什么问题

- **论文表述**：`doc/paper/tex/1.introduction.tex` 和 `doc/paper/tex/2.background.tex` 说明，VLM 的主要瓶颈来自视觉 token 数量极大、跨帧冗余高、LLM 主干计算占比极高。
- **代码证据**：仓库把问题分成四层处理：
  - `algorithm/`：在真实模型前向中注入 Focus
  - `simulator/`：把稀疏行为投影到加速器性能/能耗
  - `rtl/`：给出硬件模块与面积功耗来源
  - `evaluation_scripts/`：把 CSV 汇总为论文图表

## 仓库结构与论文对应关系

| 目录 | 实际职责 | 对应论文位置 | 典型产物 |
| --- | --- | --- | --- |
| `algorithm/` | 真实 VLM 推理、Focus/基线注入、trace 导出、精度评估 | `doc/paper/tex/5.semantic.tex`、`doc/paper/tex/6.vector.tex`、`doc/paper/tex/7.evaluation.tex` | `.pth` trace、`meta_data.csv`、`accuracy.csv`、基线稀疏度 CSV |
| `simulator/` | 周期/访存/能耗/面积建模 | `doc/paper/tex/7.evaluation.tex` | `main_focus.csv`、`dse_*.csv`、功耗面积 breakdown |
| `rtl/` | Focus 与基线硬件模块 | `doc/paper/tex/4.overview.tex`、`5.semantic.tex`、`6.vector.tex` | Verilog/SystemVerilog 源文件 |
| `evaluation_scripts/` | 图表与表格复现 | `doc/paper/tex/7.evaluation.tex` | notebook、图表输出 |
| `doc/paper/` | 原始论文源码 | 全文 | TeX、论文插图、参考文献 |

## 有意义的目录树

```text
.
├── algorithm/
│   ├── run_eval.py
│   ├── run_focus.sh / run_dse.sh / run_*baseline*.sh
│   ├── focus/
│   │   ├── interface.py
│   │   ├── main.py
│   │   ├── baseline_adaptiv.py
│   │   ├── baseline_CMC.py
│   │   ├── configs/*.csv
│   │   └── models/
│   └── lmms-eval/
├── simulator/
│   ├── main.py
│   ├── arch/accelerator.py
│   ├── models/models.py
│   ├── models/sparse_info.py
│   ├── core/simulator.py
│   ├── core/simulator_comp.py
│   └── core/simulator_mem.py
├── rtl/
├── evaluation_scripts/plot_scripts/ipynb_src/
└── doc/paper/
```

## 关键连接关系

### 论文方法 -> 算法代码

- **直接观察**：SEC 在论文中由 `doc/paper/tex/5.semantic.tex` 描述，在代码中主要落在 `algorithm/focus/main.py` 和 `algorithm/focus/models/qwen2/modeling_qwen2.py`。
- **直接观察**：SIC 在论文中由 `doc/paper/tex/6.vector.tex` 描述，在代码中也主要落在 `algorithm/focus/main.py` 与 Qwen2 补丁前向。
- **关键推断**：论文说有 semantic / block / vector 三层；但代码里并不是三个完全独立的软件模块。  
  block level 主要体现在 `algorithm/focus/main.py` 里 `block_size`、`frame_block_size` 和局部候选构造；vector level 体现在同一个 `focus_similarity_concentration(...)` 过程里按 `vector_size` 分块做匹配。

### 算法代码 -> 模拟器

- **直接观察**：`algorithm/focus/main.py` 输出 Focus trace 和 `meta_data.csv`。
- **直接观察**：`simulator/models/sparse_info.py` 与 `simulator/models/models.py` 读取这些文件。
- **推断**：这个仓库最值得学习的工程方法，就是把“模型执行”和“硬件评估”之间的接口做成稳定的文件产物，而不是耦合成一个大进程。

### 模拟器 -> 论文图表

- **直接观察**：`evaluation_scripts/plot_scripts/ipynb_src/*.ipynb` 读取 simulator CSV 和 algorithm CSV。
- **直接观察**：`simulator/utils/analysis.py` 单独负责最坏情况分析图，对应 `doc/paper/tex/7.evaluation.tex` 里的 worst-case 分析。

### RTL -> 模拟器

- **直接观察**：`simulator/arch/accelerator.py` 读取 `simulator/arch/focus_rtl.csv`、`adaptiv_rtl.csv`、`cmc_rtl.csv`。
- **不明确**：`rtl/` 中没有综合脚本，所以无法从仓库内部完全复现这些 CSV 的生成流程。

## 仓库结构是否“镜像”论文结构

- **直接观察**：不是。论文是“动机 -> 总览 -> SEC -> SIC -> 评测”，定义在 `doc/paper/tex/*.tex`。
- **直接观察**：仓库是“算法运行 -> trace 导出 -> 模拟器 -> RTL 支撑 -> 图表复现”。
- **推断**：这说明仓库优先服务“研究工程执行”，而不是“论文章节展示”。

## 哪些是核心科学贡献，哪些更偏基础设施

| 类别 | 关键文件 | 说明 |
| --- | --- | --- |
| 科学贡献核心 | `algorithm/focus/main.py`、`algorithm/focus/interface.py`、`algorithm/focus/models/qwen2/modeling_qwen2.py` | 真正实现 SEC、SIC 和模型注入路径 |
| 论文方法到硬件结论的桥 | `simulator/models/sparse_info.py`、`simulator/core/simulator.py`、`simulator/core/simulator_comp.py`、`simulator/core/simulator_mem.py` | 把 trace 变成延迟/能耗/访存结论 |
| 硬件实现证据 | `rtl/*.v`、`rtl/*.sv`、`simulator/arch/*_rtl.csv` | 支撑面积/功耗声明 |
| 基础设施/胶水 | `algorithm/lmms-eval/`、shell 脚本、notebook | 重要，但不是方法本体 |

## 这个项目为什么值得学习

- **直接观察**：算法端和模拟器端通过 `.pth` / CSV 文件接口解耦。
- **直接观察**：模型补丁集中在 `algorithm/focus/interface.py`，而不是散落在很多脚本里。
- **推断**：如果你以后要做自己的“算法-硬件协同设计”项目，这种组织方式非常值得复用：
  - 先把方法注入真实模型
  - 再把行为序列化成稳定 artifact
  - 再用模拟器/RTL/图表分层消费

## 结论

- **直接观察**：这个仓库的真实主线是“论文方法 -> 模型前向中的稀疏行为 -> trace 文件 -> 加速器模拟 -> 论文图表”。
- **推断**：学习它最好的方式不是按目录看，而是按论文概念看，同时跟踪每个 artifact 在哪里生成、被哪里消费。
