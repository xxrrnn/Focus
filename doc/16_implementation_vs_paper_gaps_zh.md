# 16 实现与论文之间的差距

这份文档专门解释：哪些地方仓库与论文贴得很紧，哪些地方论文被压缩成更少的代码结构，哪些地方仓库证据强度低于论文叙事。

## 核心差距表

| 主题 | 论文侧 | 仓库侧 | 评价 |
| --- | --- | --- | --- |
| 三级 concentration | `doc/paper/tex/0.abstract.tex` 与 `doc/paper/tex/3.motivation.tex` 把 semantic、block、vector 三层明确分开 | `algorithm/focus/main.py` 只有一条 SEC 路径和一条 SIC 路径，block 与 vector 逻辑融合在 `focus_similarity_concentration(...)` 中 | 核心行为一致，但软件分解方式不同 |
| SEC 硬件子模块 | `doc/paper/tex/5.semantic.tex` 明确给出 analyzer、top-k sorter、offset encoder | 软件主要通过 attention reduction、`torch.topk`、以及 `algorithm/focus/main.py` 中的 retained-index bookkeeping 来表达；没有找到同样清晰的一组 RTL 模块 | 算法效果是有的，但子模块层面只算部分对应 |
| SIC 硬件子模块 | `doc/paper/tex/6.vector.tex` 明确给出 gather、layouter、scatter | gather 类行为在算法代码中存在；layouter 与 scatter 更主要体现在 `simulator/core/simulator_comp.py` 与 `simulator/core/simulator_mem.py` | 这里论文结构比代码更干净 |
| 架构总览 | `doc/paper/tex/4.overview.tex` 把 Focus Unit 画成 memory 附近的一个整体单元 | 仓库把它拆散到 `algorithm/`、`simulator/`、`rtl/` 三层 | 工程上合理，但不是视觉上一一对应 |
| baseline 对比 | 论文在 `doc/paper/tex/7.evaluation.tex` 中把 AdapTiV 与 CMC 作为架构 baseline | 算法基线在 `algorithm/focus/baseline_*.py` 里有实现，但模拟器端在 `simulator/models/sparse_info.py` 主要读取聚合 sparsity CSV | baseline 路径比 Focus 路径更粗糙 |
| DRAM 建模 | 论文在 `doc/paper/tex/7.evaluation.tex` 中写到 DRAMsim3 | 当前可见运行路径主要使用 `simulator/arch/accelerator.py` 中的常量 | 仓库证据强度弱于论文表述 |
| RTL 支撑 | 论文在 `doc/paper/tex/7.evaluation.tex` 中说有 SystemVerilog 实现与综合 | 仓库里确实有 RTL 模块与 CSV 汇总，但没有完整 synthesis/testbench flow | 只能算部分体现 |
| 模型覆盖 | 论文在 `doc/paper/tex/7.evaluation.tex` 中评测三类视频 VLM 与两类图像 VLM | 算法侧支持这些模型，但 simulator 在 `simulator/models/models.py` 中复用了单一 Qwen2 风格模板 | 执行支持更广，模拟保真度更窄 |

## 命名不一致

| 论文术语 | 代码术语 | 为什么会不一致 |
| --- | --- | --- |
| Semantic Concentrator (SEC) | `semantic_concentration`、`set_token_importance`、retained token bookkeeping | 代码倾向于给“动作”命名，论文倾向于给“硬件模块”命名 |
| Similarity Gather / Scatter | `focus_similarity_concentration`、`group_idx`、simulator 里的 scatter/gather helper | 运行时侧在记录元数据，simulator 侧才把这些元数据解释为架构行为 |
| Convolution-style Layouter | `simulator/arch/accelerator.py` 中的 `layouter` buffer，`simulator/core/simulator_mem.py` 中的 memory 假设 | 论文强调硬件数据流，代码更强调性能建模 |

## 仓库结构是否镜像论文结构

- 直接观察：不是。论文是 `motivation -> overview -> SEC -> SIC -> evaluation`，仓库则是 `algorithm -> simulator -> rtl -> plotting`。
- 推断：这种组织方式更适合执行实验和管理 artifact，因为每个执行边界都被显式保留下来。
- 可复用经验：如果你自己做软硬件协同设计项目，按 pipeline boundary 组织代码，往往比按论文章节组织更稳定、更容易维护。

## 哪些是科学贡献核心，哪些更像胶水

### 科学贡献核心

- `algorithm/focus/main.py`
- `algorithm/focus/interface.py`
- `algorithm/focus/models/qwen2/modeling_qwen2.py`
- `simulator/core/simulator.py`
- `simulator/core/simulator_comp.py`
- `simulator/core/simulator_mem.py`

### 更像基础设施 / 胶水

- `algorithm/run_*.sh` 与 `simulator/run_*.sh`
- `algorithm/run_eval.py` 中的 CSV 写出辅助逻辑
- `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`
- `algorithm/lmms-eval/` 与 `3rd_party/` 中的 vendored 框架

## 为什么这种组织方式仍然值得学习

- `algorithm/focus/interface.py` 把模型 patch 问题与核心方法问题隔离开。
- `algorithm/focus/main.py` 与 `simulator/models/sparse_info.py` 通过 trace 文件格式建立稳定边界。
- 论文展示层被严格放在下游，作图不会反过来污染方法执行路径。

## 小结

- 直接观察：最大的差距并不在于“Focus 有没有实现”，而在于“论文里的硬件分解是否在仓库里同样清晰可见”。
- 推断：这在研究代码里很常见。论文追求说服力，仓库追求可执行性。
