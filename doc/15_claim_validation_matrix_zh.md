# 15 主张验证矩阵

这份矩阵比“实现在哪个文件里”更严格。它要回答的是：仓库到底为每个论文主张提供了哪一种证据？

| 主张 | 论文来源 | 证据类型 | 关键仓库路径 | 验证状态 | 说明 |
| --- | --- | --- | --- | --- | --- |
| Focus 实现了 multilevel concentration | `doc/paper/tex/0.abstract.tex`、`doc/paper/tex/1.introduction.tex` | 直接代码证据 | `algorithm/focus/main.py`、`algorithm/focus/models/qwen2/modeling_qwen2.py` | supported | SEC 与 SIC 都是实际运行时行为。 |
| Semantic pruning 由 cross-modal attention 驱动 | `doc/paper/tex/5.semantic.tex` | 直接代码证据 | `algorithm/focus/main.py::set_token_importance(...)` | supported | 这是最清晰的一对一算法映射。 |
| block-level 与 vector-level similarity 都被利用 | `doc/paper/tex/6.vector.tex` | 直接代码证据加解释 | `algorithm/focus/main.py::focus_similarity_concentration(...)` | supported，但结构上被合并 | 块级与向量级逻辑被融合在一个函数里。 |
| Focus 在小精度损失下保持准确率 | `doc/paper/tex/7.evaluation.tex`、`tab:accuracy` | 直接实验路径 | `algorithm/run_eval.py`、`algorithm/example_output/accuracy.csv`、`table_2.ipynb` | supported | 全量重跑代价高，但证据链存在。 |
| Focus 的 sparsity 高于 AdapTiV、CMC、FrameFusion | `doc/paper/tex/7.evaluation.tex`、`tab:accuracy` | 直接实验路径加 notebook 汇总 | `algorithm/example_output/accuracy.csv`、`algorithm/example_output/adaptiv_sparsity.csv`、`algorithm/example_output/cmc_sparsity.csv`、`table_2.ipynb` | supported | `table_2.ipynb` 中 FrameFusion sparsity 被直接写死为 70。 |
| Focus 相比 dense SA、AdapTiV、CMC 有更高 speedup | `doc/paper/tex/7.evaluation.tex`、`fig:main` | simulator 支撑 | `simulator/main.py`、`simulator/core/*.py`、`simulator/example_sim_results/main_*.csv`、`figure_9.ipynb` | supported by simulator | 不是只靠 PyTorch runtime 就能直接验证。 |
| Focus 提升 energy efficiency | `doc/paper/tex/7.evaluation.tex`、`fig:main` | simulator 支撑 | `simulator/core/*.py`、`simulator/arch/accelerator.py`、`simulator/example_sim_results/main_*.csv`、`figure_9.ipynb` | supported by simulator | 依赖 compute、memory、area/power 假设。 |
| Focus 相比 systolic array 只有 2.7% 面积开销 | `doc/paper/tex/7.evaluation.tex`、`tab:setup_compare` | simulator 加预计算 RTL 汇总 | `simulator/arch/focus_rtl.csv`、`simulator/arch/accelerator.py`、`accelerator_area_power_buffer.csv` | partially supported | 可以从 CSV 重算，但综合来源链不完整。 |
| SEC 与 SIC 仅占总面积/功耗很小比例 | `doc/paper/tex/7.evaluation.tex`、`fig:main` | simulator 加预计算 RTL 汇总 | `simulator/example_sim_results/detailed_power_area_breakdown.csv`、`figure_9.ipynb` | partially supported | 与上面同样存在 provenance 缺口。 |
| Focus 支持 tile size、vector size、block size、scatter accumulator 的 DSE | `doc/paper/tex/7.evaluation.tex`、`fig:DSE` | 直接脚本链 | `algorithm/run_dse.sh`、`simulator/run_dse_sim.sh`、`figure_10.ipynb` | supported | 从脚本到图的对应关系很强。 |
| Focus 在 INT8 下仍然有效 | `doc/paper/tex/7.evaluation.tex`、`tab:int8_degrade` | 直接脚本链加 simulator | `algorithm/run_focus.sh int8`、`simulator/main.py --quantization`、`table_4.ipynb` | supported | 需要 bitsandbytes 与可用的 INT8 模型加载路径。 |
| Focus 可推广到 image VLM | `doc/paper/tex/7.evaluation.tex`、`tab:image` | 直接脚本链加 simulator | `algorithm/run_focus_image.sh`、`algorithm/run_adaptiv_image.sh`、`simulator/run_image_sim.sh`、`table_5.ipynb` | supported | Qwen2.5-VL 需要不同版本的 transformers。 |
| Focus 在 worst/best case 下仍保持较稳健利用率 | `doc/paper/tex/7.evaluation.tex`、`fig:worst_case` | 直接脚本链 | `simulator/utils/analysis.py` | supported | 图可以直接由 trace 生成。 |
| Focus 是首个专门面向 VLM 的架构 | `doc/paper/tex/1.introduction.tex`、`doc/paper/tex/7.evaluation.tex` | 文献定位型主张 | 仓库内部无充分证据 | not directly testable | 这需要外部文献审阅，不是代码 inspection 能证明的。 |
| DRAM 能耗通过 DRAMsim3 建模 | `doc/paper/tex/7.evaluation.tex` | 代码证据较弱 | `3rd_party/DRAMsim3`、`simulator/arch/accelerator.py` | unclear / partially supported | 子模块存在，但正常运行路径主要使用固定常量。 |

## 如何理解这个矩阵

- `supported` 表示仓库里存在真实可运行或可检查的证据链。
- `supported by simulator` 表示主张有意义，但它依赖架构假设，而不是只依赖 PyTorch 算法路径。
- `partially supported` 表示结果可以从现有产物重建，但最深层来源链仍不完整。
- `not directly testable` 表示该主张超出了仓库本身能证明的范围。
