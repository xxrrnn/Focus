# 06 RTL 深读

## 论文锚点

RTL 相关硬件术语主要出现在：

- `doc/paper/tex/4.overview.tex`
- `doc/paper/tex/5.semantic.tex`
- `doc/paper/tex/6.vector.tex`
- 以及 `doc/paper/tex/7.evaluation.tex` 里的综合/评测方法描述

## 仓库里实际有什么

| 硬件概念 | RTL 文件 | 与模拟器的连接 |
| --- | --- | --- |
| systolic array | `rtl/traditional_systolic.v`, `rtl/traditional_mac.v` | `simulator/arch/focus_rtl.csv`、`cmc_rtl.csv`、以及 `simulator/arch/accelerator.py` 中 dense/focus 配置 |
| SIC 的 cosine / norm / max / update | `rtl/cosine_similarity_unit.sv`, `rtl/inv_magnitude_unit.v`, `rtl/max_unit_fp16.v`, `rtl/average_update_unit.sv` | `simulator/arch/focus_rtl.csv` |
| Adaptiv array | `rtl/adaptiv_array.v` | `simulator/arch/adaptiv_rtl.csv` |
| CMC codec | `rtl/cmc_codec_pe.v`, `rtl/cmc_addertree_4to1.v` | `simulator/arch/cmc_rtl.csv` |
| SFU | `rtl/fp16_exp.v`, `rtl/fp16_recip.v`, `rtl/fp16_mult.v`, `rtl/fp16_add.v`, `rtl/fp32_add.v`, `rtl/fastinvsqrt_fp32.v` | `simulator/arch/accelerator.py` 中的 SFU 计数 |

## 强对应关系

- **直接观察**：`rtl/README.md` 里的组件名字，与 `simulator/arch/*_rtl.csv` 中的组件名字是对得上的。
- **直接观察**：`simulator/arch/accelerator.py` 正是通过这些组件统计乘以实例数，得到整机面积/功耗。

## 弱对应关系

- **直接观察**：仓库里没有一个完整的 `focus_top` 之类集成顶层 RTL。
- **直接观察**：没有综合脚本，也没有 testbench。
- **推断**：论文里的硬件框图在仓库中更像“组件库 + 派生统计”，而不是一个完整封装好的硬件工程。

## 论文硬件术语与仓库现实的关系

| 论文术语 | 论文来源 | 仓库现实 |
| --- | --- | --- |
| SEC importance analyzer / top-k sorter / offset encoder | `doc/paper/tex/5.semantic.tex` | 没有找到三者一一独立的 RTL 文件；最接近的证据是 `max_unit_fp16.v` 和 simulator 里的组件分类 |
| Similarity Gather / Layouter / Scatter | `doc/paper/tex/6.vector.tex` | 没有完整集成 RTL 流；这些概念被拆散在若干 RTL 模块、模拟器 buffer 命名和 scatter/gather 逻辑里 |
| 独立的 Focus unit 挂在 systolic array 旁边 | `doc/paper/tex/4.overview.tex` | 在仓库中主要以组件表与模块集合存在，而不是一个单独顶层模块 |

## 哪些是可以直接验证的

- 已提交 RTL 文件中的模块层次与内部数据通路
- RTL README 与 simulator `*_rtl.csv` 的命名一致性

## 哪些无法仅靠仓库完全验证

- `doc/paper/tex/7.evaluation.tex` 中所说的完整综合流程
- 当前提交的 RTL 是否与论文出数版本完全一致
- 完整加速器级的时序 / 功能联调行为

## 代码质量观察

- **直接观察**：`rtl/cosine_similarity_unit.sv` 中 `add_result` 与 `accum` 的连线存在可疑不一致。
- **直接观察**：`rtl/cmc_codec_pe.v` 在当前提交版本中没有给 `distance` 和 `done` 明确赋值。
- **推断**：RTL 层更适合被看作“架构实现证据”，而不是“可直接交付的完整硬件 release”。

## 结论

- **直接观察**：RTL 层确实支撑了论文里的硬件术语和 simulator 的面积/功耗统计。
- **推断**：更准确的定位是“为论文硬件结论提供组件级证据”，而不是一个一键可复现的完整 RTL 项目。
