# 10 未决问题与风险

这份文档故意保持批判性。目的不是“找毛病”，而是把论文源码 `doc/paper/*.tex` 与仓库代码都看过之后，哪些地方最扎实、哪些地方最脆弱、哪些地方仍然不够可审计，明确写出来。

## 主要未决问题

| 主题 | 证据 | 为什么重要 | 状态 |
| --- | --- | --- | --- |
| 多个模型共用一个硬件模板 | `simulator/models/models.py` 中硬编码了 `get_QWen2_7B_architecture()`，只通过 `meta_data.csv` 改 sequence length | 不同模型的隐藏维度、层结构如果不完全一致，性能与操作数估计可能存在偏差 | 直接观察 |
| 论文里的 DRAMsim3 叙述与实际运行路径不完全一致 | `doc/paper/tex/7.evaluation.tex` 提到 DRAMsim3，但 `simulator/arch/accelerator.py` 使用固定 DRAM 带宽与 energy-per-byte 常量 | 某些能耗结论依赖的假设，比论文文字表述更弱 | 直接观察 |
| RTL 汇总 CSV 的来源链缺失 | `simulator/arch/focus_rtl.csv`、`adaptiv_rtl.csv`、`cmc_rtl.csv` 被 `simulator/arch/accelerator.py` 消费，但 `rtl/` 里没有配套综合脚本 | area/power 结果难以做端到端审计 | 直接观察 |
| 缺少集成式 Focus 顶层 RTL | `rtl/README.md` 只列出模块，没有找到完整 top module 或 testbench flow | 论文中的总体硬件结构图，比仓库里现成 RTL 证据更完整 | 直接观察 |
| baseline 硬件模拟的保真度 | `simulator/models/sparse_info.py` 对 AdapTiV 和 CMC 只读取聚合 sparsity | baseline 对比能说明方向，但与 Focus 的细粒度 trace 驱动并不对等 | 直接观察 |
| vector-size DSE 的范围 | `simulator/main.py::dse_vector_size(...)` 更像单层配置扫描，而不是完整 end-to-end 全模型 sweep | 结论有价值，但它覆盖的范围比论文总体表述更窄 | 直接观察 |
| notebook 对文件名的硬编码依赖 | `evaluation_scripts/plot_scripts/ipynb_src/` 里的 notebook 直接读取 `main_focus.csv`、`dse_a_m_tile_size.csv`、`accuracy.csv` 等固定文件名 | 复现实验很方便，但目录规范较脆弱 | 直接观察 |
| median-trace 方法学 | `algorithm/run_focus.sh` 默认 `--limit 10 --use_median`，`algorithm/focus/main.py::post_process(...)` 选择中位 sparsity 样本 | 主实验的 simulator 输入是“代表样本”，不是完整分布 | 直接观察 |

## 代码层面的脆弱点

| 文件 | 观察 | 风险 |
| --- | --- | --- |
| `algorithm/focus/interface.py` | 依赖 monkeypatch 精确匹配模型类与 forward 名称 | 对 `transformers` 版本与模型包装方式非常敏感 |
| `algorithm/focus/models/qwen2/modeling_qwen2.py` | Focus 的注入方式深度绑定 Qwen2 系列执行形态 | 迁移到新模型族时，需要人工精细改写 |
| `simulator/core/simulator_comp.py` | 会改写共享的 ScaleSim 配置文件 `simulator/core/scalesim_cfg/` | 并发运行可能互相踩踏，`simulator/README.md` 已明确提示这一点 |
| `rtl/cosine_similarity_unit.sv` | `add_result` 与 `accum` 相关连线看起来不一致 | 在把 RTL 当成“可直接投产的模块”前，需要人工复审 |
| `rtl/cmc_codec_pe.v` | 当前文件里看不出所有输出端口都被完整驱动，如 `distance`、`done` | baseline RTL 证据强度低于成熟 IP 模块应有水平 |

## 论文命名与代码命名的不一致

- 论文在 `doc/paper/tex/5.semantic.tex` 与 `doc/paper/tex/6.vector.tex` 中使用 `SEC`、`SIC`、`Similarity Gather`、`Similarity Scatter`、`Convolution-style Layouter`。
- 代码更常见的命名是 `Focus`、`semantic_concentration`、`focus_similarity_concentration`、`mask_zero`、`mask_similar`、`group_idx`，见 `algorithm/focus/main.py`。
- 推断：这不是简单命名随意，而是视角不同。论文在讲硬件模块与机制，代码在讲运行时变换与序列化产物。

## 更依赖 simulator 而不是算法直接执行的结论

- `doc/paper/tex/7.evaluation.tex` 里的 speedup 结论
- `doc/paper/tex/7.evaluation.tex` 里的 energy-efficiency 结论
- `doc/paper/tex/7.evaluation.tex` 里的 `2.7%` area overhead
- `doc/paper/tex/7.evaluation.tex` 里的细粒度 memory-traffic 结论

这些结论并不是无效，但其证据链是 `algorithm trace -> simulator -> notebook`，而不是“只跑算法代码就能直接证明”。

## 可复现性风险

- 全量 accuracy 评测开销极大；`algorithm/README.md` 与 `doc/paper/tex/artifact_appendix.tex` 都表明需要多天 GPU 时间。
- 环境对版本比较敏感，因为 `algorithm/focus/pyproject.toml` 对 main 路径固定 `transformers==4.48.2`，对 `qwen25_vl` 则切换到 `4.49.0`。
- 仓库依赖 HuggingFace 上可能需要权限的模型与数据集，这一点在 `README.md` 与 `doc/paper/tex/artifact_appendix.tex` 都有体现。
- shell 脚本默认写死了 `./output` 与 `results` 等路径，虽然便于复现，但对目录结构有隐含假设。

## 特别值得学习的工程模式

- Trace 驱动的子系统解耦：`algorithm/focus/main.py` 负责写 trace，`simulator/models/sparse_info.py` 负责消费 trace，不需要把模型运行时直接拖进模拟器。
- 薄接口层：`algorithm/focus/interface.py` 把模型 patch 与核心方法 `algorithm/focus/main.py` 隔开。
- 下游作图层不污染核心逻辑：`evaluation_scripts/plot_scripts/ipynb_src/*.ipynb` 只消费 CSV，不重新运行方法。
- 显式实验模式：`simulator/main.py` 把主实验、DSE、量化、ablation 分成清晰入口。

## 小结

- 直接观察：这个仓库在“方法执行、trace 导出、CSV 驱动实验”这三件事上做得最扎实。
- 推断：最薄弱的一环，是从论文硬件叙事到可完全审计的 RTL/综合流程之间，仍然存在缺口。
- 不清楚：仅凭当前仓库，无法完全确认所有 area/power 数字是否都能在没有额外私有脚本的情况下重新生成。
