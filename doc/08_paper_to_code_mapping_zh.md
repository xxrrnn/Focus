# 08 论文到代码映射

这是整套文档里最核心的“论文 -> 仓库”对照表。本文档使用以下状态标签：

- `fully implemented`：论文机制在仓库中有明确、可执行的实现。
- `partially implemented`：核心思想存在，但某些子模块被简化、合并，或缺少独立实现。
- `approximated in code`：论文中的硬件/结构性机制，在代码里只保留了算法效果或近似表达。
- `represented mainly by scripts/configs`：更多体现为脚本、配置、实验组织，而不是独立核心模块。
- `only used in evaluation`：主要出现在结果整理和作图阶段。
- `not clearly found`：论文描述了该机制，但仓库里没有找到足够直接的实现证据。

| 论文概念 | 论文来源 | 实现状态 | 主要代码路径 | 实际执行路径 | 说明 |
| --- | --- | --- | --- | --- | --- |
| 语义级、块级、向量级三级 concentration | `doc/paper/tex/0.abstract.tex`、`doc/paper/tex/1.introduction.tex`、`doc/paper/tex/3.motivation.tex` | fully implemented，但软件结构与论文结构不完全同构 | `algorithm/focus/main.py`、`algorithm/focus/interface.py`、`algorithm/focus/models/qwen2/modeling_qwen2.py` | `algorithm/run_eval.py` -> `lmms_eval/evaluator.py` -> `apply_focus(...)` -> patched model forward | 直接观察：SEC 和 SIC 都在真实推理路径中执行。推断：块级与向量级并没有拆成两个软件模块，而是合并在 `Focus.focus_similarity_concentration(...)` 中。 |
| Semantic Concentrator (SEC) | `doc/paper/tex/5.semantic.tex` | 算法层 fully implemented | `algorithm/focus/main.py::set_token_importance(...)`、`algorithm/focus/main.py::semantic_concentration(...)`、`algorithm/focus/models/qwen2/modeling_qwen2.py` | 同上 | SEC 的触发层来自 `algorithm/focus/configs/focus.csv`。 |
| Streaming Importance Analyzer | `doc/paper/tex/5.semantic.tex` | 算法层 fully implemented | `algorithm/focus/main.py::set_token_importance(...)` | 由 patched attention 路径调用 | 直接观察：代码确实从 cross-modal attention 中提取 text-to-image 最大值。 |
| Top-k Bubble Sorter | `doc/paper/tex/5.semantic.tex` | approximated in code | `algorithm/focus/main.py::semantic_concentration(...)` | 同上 | 直接观察：软件里使用的是张量操作与 `topk` 类选择，而不是独立的流式 sorter 模块。 |
| Offset Encoder | `doc/paper/tex/5.semantic.tex` | partially implemented | `algorithm/focus/main.py`、`algorithm/focus/models/qwen2/utils.py` | SEC 后续恢复位置信息，再供 SIC / simulator 使用 | 直接观察：保留 token 的索引与位置恢复逻辑存在。尚不清楚：是否有与论文 encoder 结构一一对应的独立软件/RTL 文件。 |
| Similarity Gather | `doc/paper/tex/6.vector.tex` | 算法层 fully implemented | `algorithm/focus/main.py::focus_similarity_concentration(...)`、`algorithm/focus/models/qwen2/modeling_qwen2.py` | patched linear/attention/MLP projection -> trace export | 算法端会形成局部分组，并导出 `mask_zero`、`mask_similar`、`group_idx`。 |
| Convolution-style Layouter | `doc/paper/tex/6.vector.tex` | 主要体现在 simulator 假设与论文公式中 | `simulator/core/simulator_mem.py`、`simulator/arch/accelerator.py`、`rtl/README.md` | simulator 的 memory accounting 与 RTL 解释路径 | 推断：layouter 更像硬件侧概念，算法代码里没有等价的独立模块。 |
| Similarity Scatter | `doc/paper/tex/6.vector.tex` | partially implemented，分散在 simulator 与 trace metadata 中 | `simulator/core/simulator_comp.py`、`simulator/core/simulator_mem.py`、`algorithm/focus/main.py` 导出的 trace 字段 | `simulator/main.py` -> `Simulator.run_focus(...)` -> scatter/gather compute/memory model | 直接观察：scatter 主要由 simulator 建模，而不是在 VLM 推理软件里真实执行。 |
| Focus Unit 插入在 memory interface 附近 | `doc/paper/tex/4.overview.tex` | 主要由 simulator/RTL 组织表达 | `simulator/arch/accelerator.py`、`rtl/README.md`、`rtl/*.v`、`rtl/*.sv` | simulator area/power 路径与 RTL 模块清单 | 推断：仓库里没有一个完整的 top-level RTL wrapper 来完全对应论文中的整体框图。 |
| Focus 注入真实 VLM | `doc/paper/tex/7.evaluation.tex` | fully implemented | `algorithm/focus/interface.py`、`algorithm/lmms-eval/lmms_eval/evaluator.py`、`algorithm/lmms-eval/lmms_eval/models/*.py` | `algorithm/run_eval.py` -> `simple_evaluate(...)` -> `apply_focus(...)` | 这是论文方法真正落到模型推理路径上的桥梁。 |
| 稀疏 trace 导出供硬件模拟使用 | `doc/paper/tex/7.evaluation.tex`、`doc/paper/tex/artifact_appendix.tex` | fully implemented | `algorithm/focus/main.py::prepare(...)`、`algorithm/focus/main.py::post_process(...)`、`algorithm/focus/utils.py` | Focus 推理 -> `.pth` trace + `meta_data.csv` | 这是算法层与模拟器之间最重要的接口。 |
| 中位样本 trace 选择 | 不是论文主要方法点，而是 artifact workflow 细节 | fully implemented | `algorithm/focus/main.py::post_process(...)` | 带 `--use_median` 的 trace 生成脚本 | 直接观察：`run_focus.sh` 默认用 `--limit 10 --use_median` 生成主实验 trace。 |
| 真实 VLM benchmark 的 accuracy evaluation | `doc/paper/tex/7.evaluation.tex` | fully implemented | `algorithm/run_eval.py`、`algorithm/lmms-eval/lmms_eval/evaluator.py`、`algorithm/lmms-eval/lmms_eval/tasks/*` | shell script -> `run_eval.py` -> `accuracy.csv` | 全量评测耗时很长，但代码路径是真实的。 |
| AdapTiV baseline | `doc/paper/tex/7.evaluation.tex` | partially implemented / approximated | `algorithm/focus/baseline_adaptiv.py`、`algorithm/focus/interface.py`、`simulator/core/simulator.py` | 算法端 `--adaptiv`，模拟器端读取 CSV | 直接观察：算法基线存在。推断：模拟器只使用聚合 sparsity，而不是细粒度 trace。 |
| CMC baseline | `doc/paper/tex/7.evaluation.tex` | partially implemented / approximated | `algorithm/focus/baseline_CMC.py`、`algorithm/focus/interface.py`、`simulator/core/simulator.py` | 算法端 `--CMC`，模拟器端读取 CSV | 与 AdapTiV 类似，模拟器侧是近似建模。 |
| FrameFusion baseline | `doc/paper/tex/7.evaluation.tex` | partially implemented | `algorithm/lmms-eval/lmms_eval/framefusion/interface.py` | `algorithm/run_eval.py` 中的 `--frame_fusion` | 直接观察：这里是对官方实现的包装，而非深度重写。 |
| 基于 sparse trace 的 cycle-accurate simulator | `doc/paper/tex/7.evaluation.tex`、`doc/paper/tex/artifact_appendix.tex` | partially implemented | `simulator/main.py`、`simulator/core/simulator.py`、`simulator/core/simulator_comp.py`、`simulator/core/simulator_mem.py`、`3rd_party/scalesim` | trace 文件 -> `SparseInfo` -> `Simulator` -> result CSV | 直接观察：模拟器确实吃 trace，并借助 ScaleSim。尚不清楚：部分 memory/DRAM 细节更像公式化建模，而不是完整第三方联动。 |
| 由 RTL 综合结果支撑 area/power | `doc/paper/tex/7.evaluation.tex` | partially implemented | `simulator/arch/focus_rtl.csv`、`simulator/arch/adaptiv_rtl.csv`、`simulator/arch/cmc_rtl.csv`、`simulator/arch/accelerator.py` | `python arch/accelerator.py` -> `accelerator_area_power_buffer.csv` | 直接观察：仓库消费的是预计算 RTL 汇总。尚不清楚：生成这些 CSV 的 synthesis 脚本未随仓库提供。 |
| DRAMsim3 建模 DRAM 能耗 | `doc/paper/tex/7.evaluation.tex` | not clearly found in active runtime path | `3rd_party/DRAMsim3`、`simulator/arch/accelerator.py` | 正常 simulator 路径使用 `Accelerator.dram_config` 常量 | 直接观察：DRAMsim3 子模块存在。尚不清楚：常规运行并不会直接调用它。 |
| tile size / vector size / block size / scatter accumulator 的 DSE | `doc/paper/tex/7.evaluation.tex` | simulator 侧 fully implemented，accuracy 侧 partially implemented | `algorithm/run_dse.sh`、`simulator/run_dse_sim.sh`、`simulator/main.py` | DSE trace generation -> DSE simulation -> `figure_10.ipynb` | accuracy 只在部分轴上用 500 样本粗测。 |
| 与量化的协同 | `doc/paper/tex/7.evaluation.tex` | fully implemented | `algorithm/run_focus.sh int8`、`simulator/main.py --quantization`、`evaluation_scripts/plot_scripts/ipynb_src/table_4.ipynb` | INT8 eval 路径 -> `accuracy.csv` + `int8_focus.csv` | 直接观察：支持基于 bitsandbytes 的 INT8 流程。 |
| 推广到 image VLM | `doc/paper/tex/7.evaluation.tex` | fully implemented | `algorithm/run_focus_image.sh`、`algorithm/run_adaptiv_image.sh`、`algorithm/run_original_image.sh`、`simulator/run_image_sim.sh`、`table_5.ipynb` | image-task trace/accuracy -> image simulation -> table notebook | Qwen2.5-VL 需要 `qwen25_vl` 依赖分支。 |
| Worst-/Best-case 分析 | `doc/paper/tex/7.evaluation.tex` | fully implemented | `simulator/utils/analysis.py` | `python utils/analysis.py --trace_dir ... --output_dir ...` | 产出 `figure_13.svg`，对应 `fig:worst_case`。 |

## 对应关系最强的部分

- `doc/paper/tex/5.semantic.tex` 中的 SEC，与 `algorithm/focus/main.py` 及 `algorithm/focus/models/qwen2/modeling_qwen2.py` 的对应关系最清晰。
- `doc/paper/tex/7.evaluation.tex` 中“算法导出 trace，模拟器消费 trace”的叙述，与 `algorithm/focus/main.py`、`simulator/models/sparse_info.py`、`simulator/main.py` 的对应关系最清晰。
- DSE 从 `doc/paper/tex/7.evaluation.tex` 到 `algorithm/run_dse.sh`、`simulator/run_dse_sim.sh`、`figure_10.ipynb` 的链路也很完整。

## 对应关系最弱的部分

- 论文把 SEC 拆成 analyzer / top-k sorter / offset encoder，但代码并没有一一对应的独立软件模块。
- 论文中的 layouter 与 scatter 是非常鲜明的硬件概念，但仓库中实现证据分散在 simulator 与局部 RTL 文件里，而不是一个集成式工件。
- `doc/paper/tex/7.evaluation.tex` 中关于 DRAMsim3 的说法，比当前仓库里实际可见的运行路径更强。
