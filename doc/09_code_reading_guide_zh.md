# 09 代码阅读指南

这份指南面向“研究者式读代码”，不是面向普通使用者。最有效的阅读顺序不是按目录从上到下扫，而是先抓住论文的论证结构，再顺着真实执行路径进入代码，最后再看 RTL 和作图层。

## 推荐阅读顺序

1. 先读 `doc/paper/main.tex`，并重点看 `doc/paper/tex/1.introduction.tex`、`doc/paper/tex/4.overview.tex`、`doc/paper/tex/5.semantic.tex`、`doc/paper/tex/6.vector.tex`、`doc/paper/tex/7.evaluation.tex`。
2. 再读 `algorithm/run_eval.py` 和 `algorithm/lmms-eval/lmms_eval/evaluator.py`，确认真正的运行入口。
3. 再读 `algorithm/focus/interface.py`，理解 Focus 是如何被注入模型的。
4. 接着读 `algorithm/focus/main.py` 与 `algorithm/focus/models/qwen2/modeling_qwen2.py`，把 SEC、SIC 和真实 forward 路径连起来。
5. 然后读 `simulator/main.py`、`simulator/models/sparse_info.py`、`simulator/core/simulator.py`、`simulator/core/simulator_comp.py`、`simulator/core/simulator_mem.py`。
6. 只有在理解 CSV/trace 是如何产生之后，再去看 `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`。
7. 最后看 `rtl/README.md` 和选定的 RTL 文件，把它们当作“硬件实现证据”，而不是最先理解系统的入口。

## 为什么这样读是最好的

- 论文 `doc/paper/tex/*.tex` 是按“贡献叙事”组织的。
- 仓库则是按“执行流水线”组织的：`algorithm/` -> `simulator/` -> `evaluation_scripts/`。
- 如果只按仓库目录读，你会很难感受到论文里 SEC 与 SIC 的概念分工。
- 如果只按论文顺序读，你又会忽略真正重要的工程边界，尤其是 `algorithm/focus/main.py` 与 `simulator/models/sparse_info.py` 之间的 trace 接口。

## 最值得先读的 16 个文件

| 顺序 | 文件 | 重点看什么 | 能学到什么工程方法 |
| --- | --- | --- | --- |
| 1 | `doc/paper/main.tex` | include 顺序与整体叙事骨架 | 论文主文件本身就是作者意图的索引。 |
| 2 | `doc/paper/tex/4.overview.tex` | Focus Unit、SEC、SIC 的角色划分 | 论文层面先把系统分块，再落到实现。 |
| 3 | `doc/paper/tex/5.semantic.tex` | SEC 的三个子模块与假设 | 论文中的硬件子块不一定在软件里一一出现。 |
| 4 | `doc/paper/tex/6.vector.tex` | gather、layouter、scatter | 一个论文概念可能会在多个代码文件中被“拆开实现”。 |
| 5 | `doc/paper/tex/7.evaluation.tex` | 论文到底声称了哪些证据 | 先看 claim，再看脚本，能避免被结果文件牵着走。 |
| 6 | `algorithm/run_eval.py` | CLI 参数、CSV 写出、实验入口 | 研究代码最好把实验入口与结果格式集中管理。 |
| 7 | `algorithm/lmms-eval/lmms_eval/evaluator.py` | Focus 真正在哪被激活 | 把方法挂到统一 evaluation 框架上，比散落在脚本里更稳。 |
| 8 | `algorithm/focus/interface.py` | monkeypatch 策略与配置加载 | 薄接口层是这类研究代码非常值得学的组织方式。 |
| 9 | `algorithm/focus/main.py` | 方法主体 | 科学贡献的核心逻辑应该集中，别分散在太多文件里。 |
| 10 | `algorithm/focus/models/qwen2/modeling_qwen2.py` | 精确注入点 | patch 最小、最稳定的模型函数集合，是高价值工程经验。 |
| 11 | `algorithm/focus/models/llava_video/modeling_llava_video.py` | 多模态元数据准备 | 模型特定的 patch / 几何推导应该与核心算法解耦。 |
| 12 | `simulator/main.py` | normal run、DSE、量化、ablation 分派 | 一个入口统一调度多种实验模式，便于复现与维护。 |
| 13 | `simulator/models/sparse_info.py` | trace 加载契约 | trace 驱动的跨层解耦，是本仓库最值得借鉴的设计之一。 |
| 14 | `simulator/core/simulator.py` | layer 级 orchestration | 让 orchestration 和 micro-model 分层，有利于检查假设。 |
| 15 | `simulator/core/simulator_comp.py` | ScaleSim 耦合、scatter/gather 计算模型 | 通用组件交给第三方工具，方法特定逻辑自己维护。 |
| 16 | `simulator/core/simulator_mem.py` | memory traffic 公式与计数逻辑 | 把 memory model 单独拆出来，后续审计会轻松很多。 |

## 第一遍阅读时不建议深挖的部分

- 不要一开始就看 `evaluation_scripts/plot_scripts/ipynb_src/` 的 notebook；它们只是下游展示层。
- 不要一开始就看 `rtl/` 的原始 Verilog/SystemVerilog；它们是硬件证据，不是最快理解系统的入口。
- 不要在没理解仓库自有修改前就钻进 `algorithm/lmms-eval/` 或 `3rd_party/` 的内部实现。

## 从工程方法角度最值得学习的文件

- `algorithm/focus/interface.py`：在外部模型代码与新方法逻辑之间建立一个干净的 patch 层。
- `algorithm/focus/main.py`：让一个类同时掌握运行时行为与 trace 序列化，核心逻辑集中。
- `simulator/models/sparse_info.py`：用显式序列化边界把 ML 执行与硬件模拟解耦。
- `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`：把论文展示层严格放在下游，不污染核心运行路径。

## 直接观察 / 推断 / 不清楚

- 直接观察：当论文概念能落到 `algorithm/focus/main.py` 或 trace 驱动的 simulator 路径时，这个仓库最清晰。
- 推断：仓库的组织优先服务于实验执行效率和 artifact 产出，而不是优先服务于教学式可读性。
- 不清楚：`simulator/arch/*_rtl.csv` 的完整生成流程，无法仅从当前仓库直接审计。
