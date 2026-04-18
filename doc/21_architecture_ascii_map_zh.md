# 21 架构 ASCII 地图

这份文件保留了旧版无后缀 ASCII 地图“一屏看到全局结构”的价值，但把它并入当前维护的双语文档体系。

## 全仓库执行总览

```text
用户命令
  |
  +--> algorithm/run_eval.py
         |
         +--> lmms_eval.evaluator.simple_evaluate(...)
                |
                +--> algorithm/lmms-eval/lmms_eval/models/ 中的模型适配器
                |
                +--> algorithm/focus/interface.py::apply_focus(...)
                       |
                       +--> 模型特定的 metadata hook
                       |      |
                       |      +--> algorithm/focus/main.py::prepare(...)
                       |
                       +--> patched decoder / attention / MLP forward
                              |
                              +--> Focus.forward(...)                 [SIC 路径]
                              +--> Focus.set_token_importance(...)
                              +--> Focus.semantic_concentration(...)  [SEC 路径]
                              +--> Focus.post_process(...)
                                       |
                                       +--> focus_main/*.pth
                                       +--> focus_int8/*.pth
                                       +--> meta_data.csv
                                       +--> accuracy.csv / DSE accuracy CSV
```

## 算法到模拟器的桥接

```text
algorithm output/
  |
  +--> focus_main/<model>_<dataset>.pth
  +--> focus_int8/<model>_<dataset>.pth
  +--> adaptiv_sparsity.csv
  +--> cmc_sparsity.csv
  +--> meta_data.csv
         |
         v
simulator/main.py
  |
  +--> simulator/models/models.py::ModelConfig(meta_data.csv)
  +--> simulator/models/sparse_info.py::SparseInfo(trace 或 sparsity CSV)
  +--> simulator/arch/accelerator.py::Accelerator(硬件配置)
  +--> simulator/core/simulator.py::Simulator(...)
         |
         +--> main_focus.csv / main_dense.csv / main_adaptiv.csv / main_cmc.csv
         +--> dse_*.csv
         +--> detailed_power_area_breakdown.csv
```

## 作图流程

```text
算法输出 CSV            模拟器输出 CSV            evaluation_scripts/jetson_stats/figure9_gpu.csv
      |                        |                                         |
      +------------------------+-----------------------------------------+
                               |
                               v
           evaluation_scripts/plot_scripts/ipynb_src/*.ipynb
                               |
                               +--> 论文图表 / 表格
```

## Focus 运行时结构

```text
Supported VLM
  |
  +--> 多模态预处理 hook
         |
         +--> Focus.prepare(patch/frame/token 元数据)
  |
  +--> decoder layers
         |
         +--> self attention
         |      |
         |      +--> Focus(q_proj input)
         |      +--> Focus(query tensor)
         |      +--> attention weights -> token importance
         |      +--> Focus(o_proj input)
         |
         +--> 被选中的 SEC 层
         |      |
         |      +--> Focus.semantic_concentration(...)
         |
         +--> MLP
                |
                +--> Focus(gate_proj input)
                +--> Focus(down_proj input)
  |
  +--> Focus.post_process()
```

## 从 Simulator + RTL 视角看 Focus Accelerator

```text
                 +--------------------+
                 |   Input / Weight   |
                 |      Buffers       |
                 +----------+---------+
                            |
                            v
                 +--------------------+
                 |   Systolic Array   |
                 | traditional_* RTL  |
                 +----------+---------+
                            |
                            +-------------------+
                            |                   |
                            v                   v
                 +--------------------+   +--------------------+
                 |   SIC detection    |   |   SEC ranking      |
                 | cosine_similarity  |   |   max_unit_fp16    |
                 | inv_magnitude      |   | importance buffer  |
                 | average_update     |   +--------------------+
                 | max / accumulator  |
                 +----------+---------+
                            |
                            v
                 +--------------------+
                 | layouter / sim map |
                 | similarity table   |
                 | concentrate_out    |
                 +----------+---------+
                            |
                            v
                 +--------------------+
                 |    output buffer   |
                 +--------------------+
```

## Baseline 硬件地图

```text
Dense:
  traditional_systolic.v + traditional_mac.v

Adaptiv:
  adaptiv_array.v + 共享 SFU 模块

CMC:
  traditional_systolic.v + cmc_codec_pe.v + cmc_addertree_4to1.v + 共享 SFU 模块
```

## 最重要的边界提醒

- 直接观察：simulator 消费的是 trace 与 RTL 派生的 CSV 汇总。
- 直接观察：simulator 并不会直接运行 RTL。
- 推断：这个仓库的方法学是“软件 trace 驱动架构模拟，综合结果校准 area/power”，而不是“一体化软硬件协同仿真栈”。
