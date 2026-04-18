# 13 论文引导式代码阅读

这份指南不是“按目录读代码”，而是“拿着论文逐节对照代码”。每个论文部分后面都告诉你应该接着看哪些代码、重点核对什么、以及映射关系到底有多强。

## 按论文章节配对阅读

| 论文部分 | 论文文件 | 接着看哪些代码 | 核对重点 | 映射强度 |
| --- | --- | --- | --- | --- |
| 摘要与引言 | `doc/paper/tex/0.abstract.tex`、`doc/paper/tex/1.introduction.tex` | `README.md`、`algorithm/README.md`、`simulator/README.md` | 确认仓库确实由 algorithm、simulator、RTL、evaluation 四层组成 | strong |
| 动机 | `doc/paper/tex/3.motivation.tex` | `algorithm/focus/main.py`、`simulator/core/simulator_mem.py` | 确认方法不仅是 token pruning，还真的针对 semantic redundancy 和局部 vector similarity | strong |
| 架构总览 | `doc/paper/tex/4.overview.tex` | `algorithm/focus/main.py`、`simulator/arch/accelerator.py`、`rtl/README.md` | 看 Focus Unit 在软件行为、架构模型、RTL 模块之间如何被拆开表达 | medium |
| Semantic Concentrator | `doc/paper/tex/5.semantic.tex` | `algorithm/focus/main.py`、`algorithm/focus/models/qwen2/modeling_qwen2.py`、`algorithm/focus/models/*/modeling_*.py` | 核对 importance extraction、top-k 选择、保留 token 的 bookkeeping，以及 SEC 在哪些层触发 | 算法效果 strong，硬件子模块命名对应 medium |
| Similarity Concentrator | `doc/paper/tex/6.vector.tex` | `algorithm/focus/main.py`、`simulator/core/simulator_comp.py`、`simulator/core/simulator_mem.py`、`rtl/*.v`、`rtl/*.sv` | 核对局部候选块、向量分组、layouter 假设、scatter/gather 建模如何分散在仓库中 | medium |
| 评估方法 | `doc/paper/tex/7.evaluation.tex` | `algorithm/run_*.sh`、`simulator/run_*.sh`、`evaluation_scripts/plot_scripts/ipynb_src/*.ipynb` | 核对每个主要图表是否真有对应脚本链路 | strong |
| Artifact Appendix | `doc/paper/tex/artifact_appendix.tex` | `README.md`、`algorithm/README.md`、`simulator/README.md`、`algorithm/focus/pyproject.toml`、`algorithm/lmms-eval/requirements.txt` | 核对环境、模型、数据集、工作流描述是否与仓库一致 | strong |

## 每组配对时应该重点看什么

### 读 `doc/paper/tex/5.semantic.tex` 时

请同时打开：

- `algorithm/focus/main.py`
- `algorithm/focus/models/qwen2/modeling_qwen2.py`
- `algorithm/focus/models/llava_video/modeling_llava_video.py`
- `algorithm/focus/models/minicpmv/modeling_minicpmv.py`
- `algorithm/focus/models/qwen2_5_vl/modeling_qwen2_5_vl.py`

要核对：

- `set_token_importance(...)` 是否真的从 text-to-image attention 中提取重要性。
- `semantic_concentration(...)` 是否真的执行 top-k 保留，并更新 position/attention 相关元数据。
- 各模型专属文件中的 `prepare(...)` 调用，是否真的建立了 image token 范围、frame/patch 几何信息。

解释：

- 直接观察：SEC 的算法效果非常明确。
- 推断：论文里的 analyzer / sorter / encoder 更像硬件分块，不是软件里的三个独立模块。

### 读 `doc/paper/tex/6.vector.tex` 时

请同时打开：

- `algorithm/focus/main.py`
- `simulator/core/simulator_comp.py`
- `simulator/core/simulator_mem.py`
- `rtl/cosine_similarity_unit.sv`
- `rtl/inv_magnitude_unit.v`
- `rtl/average_update_unit.sv`

要核对：

- `focus_similarity_concentration(...)` 是否真的利用 `block_size`、`frame_block_size`、`gemm_m_size` 构造局部候选窗口。
- trace 字段 `mask_zero`、`mask_similar`、`group_idx` 是否足以让后续 simulator 建模 gather/scatter。
- simulator 是否确实存在 `run_linear_scatter_focus(...)`、`run_qk_scatter_focus(...)`、`run_gather_linear_focus(...)` 等显式路径。

解释：

- 直接观察：vector-level grouping 是真实的算法行为。
- 推断：layouter 与 scatter 在这个仓库里主要属于 hardware/simulator 语境，而不是算法软件语境。

### 读 `doc/paper/tex/7.evaluation.tex` 时

请同时打开：

- `algorithm/run_focus.sh`、`algorithm/run_dse.sh`、`algorithm/run_original.sh`、`algorithm/run_adaptiv.sh`、`algorithm/run_cmc.sh`
- `simulator/run_main_sim.sh`、`simulator/run_dse_sim.sh`、`simulator/run_image_sim.sh`
- `evaluation_scripts/plot_scripts/ipynb_src/figure_9.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/figure_10.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/figure_11.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/figure_12.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_2.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_4.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_5.ipynb`

要核对：

- 哪些结果是脚本直接产出，哪些结果只是 notebook 里的汇总与可视化。
- 哪些论文结论主要来自算法运行，哪些结论本质上来自 simulator 输出。
- 像 `evaluation_scripts/jetson_stats/figure9_gpu.csv` 这样的外部数据，是在哪一步进入证据链的。

## 最好的“论文放旁边”阅读流程

1. 先读论文该节。
2. 按上表打开对应代码。
3. 把每个机制标记成 `直接观察`、`推断`、或 `不清楚`。
4. 顺着实际执行路径追到脚本和结果文件。
5. 最后再判断论文主张是 fully supported、partially supported，还是 only simulator-backed。

## 最有教育意义的配对

- `doc/paper/tex/5.semantic.tex` 配 `algorithm/focus/main.py`
- `doc/paper/tex/6.vector.tex` 配 `algorithm/focus/main.py` 与 `simulator/core/simulator_comp.py`
- `doc/paper/tex/7.evaluation.tex` 配 `algorithm/run_dse.sh`、`simulator/run_dse_sim.sh`、`figure_10.ipynb`

## 小结

- 直接观察：论文与代码对照体验最好的部分，是算法核心文件以及 DSE / evaluation 脚本链。
- 推断：对照体验最弱的部分，是硬件叙事，因为论文里的架构切分比当前仓库中可见的 RTL 集成形态更完整、更干净。
