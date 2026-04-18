# 20 术语表

| 术语 | 在本仓库中的含义 | 主要路径 |
| --- | --- | --- |
| Focus | 论文提出的整体方法与架构 | `doc/paper/main.tex`、`algorithm/focus/`、`simulator/`、`rtl/` |
| Focus Unit | 论文层面的硬件总模块，内部包含 SEC 和 SIC | `doc/paper/tex/4.overview.tex`、`simulator/arch/accelerator.py`、`rtl/README.md` |
| SEC | Semantic Concentrator，即 token pruning 路径 | `doc/paper/tex/5.semantic.tex`、`algorithm/focus/main.py` |
| SIC | Similarity Concentrator，即向量/块级相似性路径 | `doc/paper/tex/6.vector.tex`、`algorithm/focus/main.py`、`simulator/core/` |
| VLM | Vision-Language Model；本仓库主要支持 LLaVA-Video、LLaVA-OneVision、MiniCPM-V、Qwen2.5-VL | `algorithm/README.md`、`doc/paper/tex/7.evaluation.tex` |
| Similarity Gather | 论文术语，表示去除冗余向量并形成紧凑输出 | `doc/paper/tex/6.vector.tex`、`algorithm/focus/main.py` |
| Similarity Scatter | 论文术语，表示利用 similarity map 把紧凑向量重建回完整输出 | `doc/paper/tex/6.vector.tex`、`simulator/core/simulator_comp.py`、`simulator/core/simulator_mem.py` |
| Convolution-style Layouter | 为局部块访问提供 conflict-free 数据布局的硬件思想 | `doc/paper/tex/6.vector.tex`、`simulator/arch/accelerator.py` |
| `run_eval.py` | 算法侧主驱动，负责评测、trace 导出与 accuracy CSV 写出 | `algorithm/run_eval.py` |
| `lmms-eval` | 被 vendor 进仓库的多模态评测框架，用于跑任务与汇总指标 | `algorithm/lmms-eval/` |
| `selected_layers` | 启用 SEC 的层索引 | `algorithm/focus/configs/focus.csv` |
| `alpha_list` | SEC 各层的 token 保留比例 | `algorithm/focus/configs/focus.csv` |
| `vector_size` | SIC 的相似性粒度，也是 DSE 的一个扫描轴 | `algorithm/focus/main.py`、`algorithm/run_dse.sh`、`simulator/main.py` |
| `block_size` | 局部相似性匹配的空间块大小 | `algorithm/focus/main.py`、`algorithm/run_dse.sh`、`simulator/main.py` |
| `frame_block_size` | 局部相似性匹配的时间维块大小 | `algorithm/focus/main.py`、`algorithm/run_dse.sh` |
| `gemm_m_size` | Focus 限制比较范围时使用的 GEMM tile 高度 | `algorithm/focus/main.py`、`algorithm/run_dse.sh`、`simulator/arch/accelerator.py` |
| `mask_zero` | Focus trace 中标记零向量的 tensor | `algorithm/focus/main.py`、`simulator/models/sparse_info.py` |
| `mask_similar` | Focus trace 中标记冗余相似向量的 tensor | `algorithm/focus/main.py`、`simulator/models/sparse_info.py` |
| `group_idx` | Focus trace 中存储代表向量/分组映射的 tensor | `algorithm/focus/main.py`、`simulator/models/sparse_info.py` |
| `meta_data.csv` | simulator 在加载 trace 之前读取的序列元数据文件 | `algorithm/focus/main.py`、`simulator/models/models.py` |
| Trace | 由 `.pth` 与配套 metadata CSV 组成的稀疏行为序列化结果，是算法与模拟器之间的桥梁 | `algorithm/focus/main.py`、`simulator/models/sparse_info.py` |
| Adaptiv | baseline 方法；算法实现位于 `algorithm/focus/baseline_adaptiv.py`，模拟器侧从粗粒度 sparsity CSV 读取 | `algorithm/focus/baseline_adaptiv.py`、`simulator/models/sparse_info.py` |
| CMC | baseline 方法；算法实现位于 `algorithm/focus/baseline_CMC.py`，模拟器侧从粗粒度 sparsity 项读取 | `algorithm/focus/baseline_CMC.py`、`simulator/models/sparse_info.py` |
| FrameFusion | 通过 vendor 评测栈中的包装接口集成进来的 baseline | `algorithm/lmms-eval/lmms_eval/framefusion/interface.py` |
| `ModelConfig` | 定义模拟层尺寸与序列统计信息的 simulator 对象 | `simulator/models/models.py` |
| `SparseInfo` | 负责加载 Focus trace 或 baseline sparsity CSV 的 simulator 对象 | `simulator/models/sparse_info.py` |
| `Accelerator` | 描述 buffer、阵列形状、组件与功耗/面积来源的 simulator 对象 | `simulator/arch/accelerator.py` |
| `Simulator` | 负责调度 compute、memory、energy 统计的上层模拟对象 | `simulator/core/simulator.py` |
| ScaleSim | 被 Focus 模拟器调用的外部 GEMM 性能模拟工具 | `3rd_party/scalesim`、`simulator/core/simulator_comp.py` |
| CACTI | 当 compiler 表不足时，用于 SRAM 建模的外部工具 | `3rd_party/cacti`、`simulator/memory/cacti.py` |
| DRAMsim3 | 作为子模块包含的第三方 DRAM 模拟器；常规运行中更偏“间接代表”而非直接调用 | `3rd_party/DRAMsim3`、`simulator/arch/accelerator.py` |
| `*_rtl.csv` | simulator 消费的预计算组件面积/功耗汇总表 | `simulator/arch/focus_rtl.csv`、`simulator/arch/adaptiv_rtl.csv`、`simulator/arch/cmc_rtl.csv` |
| `main_focus.csv` | Focus 的主模拟结果表 | `simulator/run_main_sim.sh`、`evaluation_scripts/plot_scripts/ipynb_src/figure_9.ipynb` |
| DSE | 设计空间探索 | `doc/paper/tex/7.evaluation.tex`、`algorithm/run_dse.sh`、`simulator/run_dse_sim.sh` |
| `SEC_only` | 只保留 semantic pruning、关闭 SIC 的 ablation 模式 | `algorithm/run_eval.py`、`simulator/main.py`、`figure_11.ipynb` |
| INT8 Focus | 量化模型执行下的 Focus 路径，并配有单独的 `focus_int8` trace 输出 | `algorithm/run_focus.sh int8`、`simulator/main.py --quantization` |
