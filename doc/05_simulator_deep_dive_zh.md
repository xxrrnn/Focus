# 05 模拟器深读

## 论文锚点

模拟器是 `doc/paper/tex/7.evaluation.tex` 中方法学、性能、能耗、面积结论的实际承载层。

## 核心文件

- `simulator/main.py`
- `simulator/models/models.py`
- `simulator/models/sparse_info.py`
- `simulator/arch/accelerator.py`
- `simulator/core/simulator.py`
- `simulator/core/simulator_comp.py`
- `simulator/core/simulator_mem.py`

## 主对象流

```text
simulator/main.py
  -> ModelConfig
  -> SparseInfo
  -> Accelerator
  -> Simulator
```

## 模拟器默认接受了哪些假设

| 假设 | 文件路径 | 为什么重要 |
| --- | --- | --- |
| 用一个 Qwen2 风格 decoder 模板代表多种支持模型 | `simulator/models/models.py` | 模型结构大多是硬编码的 |
| 每个 model/dataset 用一条中位 workload profile 表示 | `simulator/models/models.py`, `meta_data.csv` | 工作负载被压缩成单行统计 |
| Focus trace 提供 per-layer 稀疏结构 | `simulator/models/sparse_info.py` | Focus 比基线建模更细 |
| 基线稀疏性可以用粗粒度统计概括 | `simulator/models/sparse_info.py` | Adaptiv / CMC 只用 CSV 汇总值 |

## 它到底建模了什么

### 计算

- `simulator/core/simulator_comp.py` 中基于 ScaleSim 的 GEMM 周期估计
- 同文件中的 Focus scatter/gather 开销公式

### 访存

- `simulator/core/simulator_mem.py` 中的 SRAM / DRAM 流量公式
- Focus 特有的 `layouter`、`similarity_map`、`similarity_table` 命名空间

### 能耗 / 面积

- buffer 面积/能量来自 `simulator/memory/buffer.py` 或 `simulator/memory/cacti.py`
- 核心组件面积/功耗来自 `simulator/arch/*_rtl.csv`

## 论文声明与模拟器证据

| 论文声明 | 论文来源 | 模拟器证据 | 状态 |
| --- | --- | --- | --- |
| cycle-accurate simulation | `doc/paper/tex/7.evaluation.tex` | `simulator/core/simulator.py`、`simulator/core/simulator_comp.py`、`simulator/core/simulator_mem.py` | 只能部分验证；模拟器存在，但“cycle-accurate”仍带有建模性质 |
| 基于 ScaleSim-v2 | `doc/paper/tex/7.evaluation.tex` | `simulator/core/simulator_comp.py` 导入 ScaleSim | 直接支持 |
| 接受来自 PyTorch 实现的 sparse trace | `doc/paper/tex/7.evaluation.tex` | `simulator/models/sparse_info.py` 读取 `.pth` 和基线 CSV | 直接支持 |
| 用 DRAMsim3 建模 DRAM 能耗 | `doc/paper/tex/7.evaluation.tex` | 代码里主要体现为 DRAM 常数 | 只部分可见 |
| 在 500MHz 下评估 | `doc/paper/tex/7.evaluation.tex` | `simulator/arch/accelerator.py` 中的频率设置 | 直接支持 |

## 最大的模拟器侧近似

- **直接观察**：`simulator/models/models.py` 不是为每个支持模型动态抽取结构，而是固定了一套 Qwen2-7B 风格模板。
- **关键推断**：这是整个模拟器里最需要你在阅读时保持警惕的近似。

## DSE 覆盖情况

| 论文 DSE 项 | 脚本路径 | 模拟器模式 | 可追踪性 |
| --- | --- | --- | --- |
| GEMM `m` tile size | `algorithm/run_dse.sh`, `simulator/run_dse_sim.sh` | `dse_m_tile_size(...)` | 强 |
| vector size | 同上 | `dse_vector_size(...)` | 部分，因为它只做 layer-wise 分析 |
| SIC block size | 同上 | `dse_block_size(...)` | 强 |
| scatter accumulators | `simulator/run_dse_sim.sh` | `dse_num_scatter(...)` | 强 |

## 哪些论文结论主要依赖模拟器

- `doc/paper/tex/7.evaluation.tex` 里的性能 speedup
- 同节里的能效结论
- DRAM access / activation size 结论
- 面积 / 功耗开销结论（结合 RTL 派生统计）

## 哪些结论不那么依赖模拟器

- 由 `algorithm/run_eval.py` 支撑的精度和计算稀疏度
- SEC / SIC 的真实注入位置与 trace 导出行为

## 这个模拟器值得学的点

- **直接观察**：它把 compute、memory、energy/area 拆成不同文件和类。
- **直接观察**：它把算法和硬件之间的接口收敛到 trace 文件。
- **推断**：如果你以后做自己的架构模拟器，这种“对象流清晰、artifact 边界明确”的组织方式非常值得借鉴。

## 结论

- **直接观察**：模拟器是论文硬件结论的主要证据引擎。
- **推断**：它足够用来理解方法学，但有些 headline claim 仍然依赖模型假设和近似建模，而不是完全闭合的 RTL 级执行流。
