# 12 论文结构与核心主张

这份文档把论文当成“作者意图源码”来读。真正的依据是 `doc/paper/main.tex` 以及它包含的 `doc/paper/tex/*.tex` 文件，而不是仓库 README 的二手摘要。

## 论文结构

| 章节 | TeX 文件 | 主要目的 | 主要标签 / 图表 | 最接近的代码锚点 |
| --- | --- | --- | --- | --- |
| 摘要 | `doc/paper/tex/0.abstract.tex` | 用最短篇幅压缩整篇论文的论点 | 无 | 整个仓库，尤其是 `algorithm/focus/` 与 `simulator/` |
| 引言 | `doc/paper/tex/1.introduction.tex` | 问题定义、贡献总结、headline 数字 | `fig:intro` | `README.md`、`algorithm/`、`simulator/` |
| 背景 | `doc/paper/tex/2.background.tex` | VLM 与已有优化方向的背景铺垫 | 无 | 主要是背景，不直接映射到单独模块 |
| 动机 | `doc/paper/tex/3.motivation.tex` | 解释为什么需要 multilevel concentration 与 locality-aware hardware | `fig:motivation1`、`fig:motivation2` | `algorithm/focus/main.py`、`simulator/core/simulator_mem.py` |
| 架构总览 | `doc/paper/tex/4.overview.tex` | 定义 Focus Unit、SEC、SIC | `fig:focus_arch` | `algorithm/focus/main.py`、`simulator/arch/accelerator.py`、`rtl/README.md` |
| Semantic Concentrator | `doc/paper/tex/5.semantic.tex` | 说明 SEC 的 analyzer、sorter、encoder | `fig:semantic` | `algorithm/focus/main.py`、`algorithm/focus/models/qwen2/modeling_qwen2.py` |
| Similarity Concentrator | `doc/paper/tex/6.vector.tex` | 说明 gather、layouter、scatter | `fig:sic_gather`、`fig:sic_layouter`、`fig:sic_scatter` | `algorithm/focus/main.py`、`simulator/core/simulator_comp.py`、`simulator/core/simulator_mem.py`、`rtl/*.v`、`rtl/*.sv` |
| 评估 | `doc/paper/tex/7.evaluation.tex` | methodology、主结果、DSE、ablation、discussion | `tab:hardware-config`、`tab:accuracy`、`fig:main`、`tab:setup_compare`、`fig:DSE`、`fig:ablation`、`fig:mem_access`、`tab:int8_degrade`、`tab:image`、`fig:worst_case` | `algorithm/run_*.sh`、`simulator/run_*.sh`、`evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`、`simulator/utils/analysis.py` |
| 结论 | `doc/paper/tex/8.conclusion.tex` | 重申贡献与核心指标 | 无 | 主要是总结 |
| Artifact Appendix | `doc/paper/tex/artifact_appendix.tex` | 复现条件、环境、工作流说明 | 无 | `README.md`、`algorithm/README.md`、`simulator/README.md` |

## 论文的核心主张

### 主张 1：Focus 是面向 VLM 的 streaming multilevel concentration architecture

- 论文来源：`doc/paper/tex/0.abstract.tex`、`doc/paper/tex/1.introduction.tex`、`doc/paper/tex/4.overview.tex`。
- 代码证据：`algorithm/focus/main.py` 实现了 SEC 和 SIC 的行为；`simulator/arch/accelerator.py` 与 `rtl/README.md` 提供架构侧解释。
- 验证状态：一部分可直接从代码观察，一部分依赖 simulator。

### 主张 2：Focus 在 semantic、block、vector 三个层次去除冗余

- 论文来源：`doc/paper/tex/0.abstract.tex`、`doc/paper/tex/3.motivation.tex`、`doc/paper/tex/6.vector.tex`。
- 代码证据：semantic 层在 `algorithm/focus/main.py::semantic_concentration(...)` 中非常明确；block 与 vector 的效果在 `algorithm/focus/main.py::focus_similarity_concentration(...)` 中融合实现。
- 验证状态：算法效果可直接验证，但实现结构没有论文叙事那样分层清晰。

### 主张 3：Focus 在保持准确率的同时，比 baseline 获得更高 sparsity

- 论文来源：`doc/paper/tex/7.evaluation.tex`，尤其是 `tab:accuracy`。
- 代码证据：`algorithm/run_eval.py`、`algorithm/example_output/accuracy.csv`、`algorithm/example_output/adaptiv_sparsity.csv`、`algorithm/example_output/cmc_sparsity.csv`、`table_2.ipynb`。
- 验证状态：仓库中可直接测试，但完整重跑成本高。

### 主张 4：Focus 带来 speedup、energy 优势，并且硬件额外开销较小

- 论文来源：`doc/paper/tex/7.evaluation.tex` 中的 `fig:main` 与 `tab:setup_compare`。
- 代码证据：`simulator/main.py`、`simulator/core/*.py`、`simulator/arch/accelerator.py`、`simulator/example_sim_results/*.csv`、`figure_9.ipynb`。
- 验证状态：主要依赖 simulator 支撑，不是单靠算法运行时就能直接看见。

### 主张 5：Focus 支持系统性的 design-space exploration

- 论文来源：`doc/paper/tex/7.evaluation.tex` 中的 `fig:DSE`。
- 代码证据：`algorithm/run_dse.sh`、`simulator/run_dse_sim.sh`、`simulator/main.py`、`figure_10.ipynb`。
- 验证状态：较强。

### 主张 6：Focus 能与 INT8 量化协同，并可推广到 image VLM

- 论文来源：`doc/paper/tex/7.evaluation.tex` 中的 `tab:int8_degrade` 与 `tab:image`。
- 代码证据：`algorithm/run_focus.sh int8`、`simulator/main.py --quantization`、`algorithm/run_focus_image.sh`、`simulator/run_image_sim.sh`、`table_4.ipynb`、`table_5.ipynb`。
- 验证状态：较强。

## 论文最强调的是什么

- 直接观察：论文是按“贡献结构”展开的，它把 SEC 与 SIC 分成独立章节，而不是按仓库目录讲解。
- 直接观察：`doc/paper/tex/7.evaluation.tex` 对硬件效率、area/power 与 DSE 的篇幅很大，这也解释了为什么 simulator 在这个仓库里并不是附属品，而是核心组成部分。
- 推断：这个仓库本质上是一个 co-design artifact。它不是“先算法、后硬件”的结构，而是从一开始就把两者绑在一起。

## 论文没有写细，但仓库必须承担的部分

- 精确的 CLI 形式和输出文件命名，论文没有展开，但 `algorithm/run_*.sh`、`simulator/run_*.sh` 与 `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb` 给出了真实路径。
- trace 的序列化细节，论文没有展开，但 `algorithm/focus/main.py` 与 `simulator/models/sparse_info.py` 是关键。
- baseline 的实际实现细节在论文里描述较轻，需要结合 `algorithm/focus/baseline_*.py` 与 `simulator/core/simulator.py` 才能判断。

## 比较难验证或仍不清楚的主张

- `doc/paper/tex/1.introduction.tex` 里的“first architecture tailored for VLMs”更像文献定位，不是代码可验证命题。
- `simulator/arch/*_rtl.csv` 的完整综合生成流程并未在仓库中公开。
- `doc/paper/tex/7.evaluation.tex` 中关于 DRAMsim3 的表述，比 `simulator/arch/accelerator.py` 可见的运行耦合更强。
