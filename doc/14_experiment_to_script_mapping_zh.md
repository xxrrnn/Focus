# 14 实验到脚本映射

这份文档把论文中的主要实验，逐一映射到仓库里真实存在的脚本链路。

## 主映射表

| 论文图表 | 论文来源 | 生成脚本 / 命令 | 中间产物 | 最终消费端 | 可追踪性 |
| --- | --- | --- | --- | --- | --- |
| `tab:accuracy` 准确率与计算稀疏率 | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_original.sh full`、`sh algorithm/run_focus.sh full`、`sh algorithm/run_framefusion.sh full`、`sh algorithm/run_adaptiv.sh full`、`sh algorithm/run_cmc.sh full` | `algorithm/output/accuracy.csv`、`algorithm/output/adaptiv_sparsity.csv`、`algorithm/output/cmc_sparsity.csv`、simulator `main_*.csv` | `evaluation_scripts/plot_scripts/ipynb_src/table_2.ipynb` | strong |
| `fig:main` 速度 / 能效 / 面积功耗 | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_focus.sh`、`sh algorithm/run_adaptiv.sh`、`sh algorithm/run_cmc.sh`、`sh simulator/run_main_sim.sh`、`python simulator/arch/accelerator.py --output_dir ...` | `main_dense.csv`、`main_adaptiv.csv`、`main_cmc.csv`、`main_focus.csv`、`detailed_power_area_breakdown.csv`、`accelerator_area_power_buffer.csv`、`evaluation_scripts/jetson_stats/figure9_gpu.csv` | `figure_9.ipynb` | CSV 链路 strong，但 Jetson 数据是外部输入 |
| `tab:hardware-config` 架构设置表 | `doc/paper/tex/7.evaluation.tex` | 没有找到单一生成器 | 信息分散在 `simulator/arch/accelerator.py`、`algorithm/focus/configs/focus.csv`、`simulator/core/scalesim_cfg/config.cfg` 与论文手工表格里 | 无 | partial |
| `tab:setup_compare` 架构对比表 | `doc/paper/tex/7.evaluation.tex` | `python simulator/arch/accelerator.py --output_dir ...` | `accelerator_area_power_buffer.csv` | 更像手工/论文表格整理 | 数值 strong，最终排版 partial |
| `fig:DSE` 设计空间探索 | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_dse.sh`，如需 accuracy 曲线再跑 `sh algorithm/run_dse.sh accuracy`，随后 `sh simulator/run_dse_sim.sh` | `dse_a_m_tile_accuracy.csv`、`dse_b_vector_accuracy.csv`、`dse_c_block_accuracy.csv`、`dse_a_m_tile_size.csv`、`dse_b_vector_size.csv`、`dse_c_block_size.csv`、`dse_d_num_scatter_accumulator.csv` | `figure_10.ipynb` | strong |
| `fig:ablation` 仅 SEC 的消融 | `doc/paper/tex/7.evaluation.tex` | `python simulator/main.py --all_models_datasets --accelerator focus --SEC_only --trace_dir ... --output_dir ...`，并配合主实验的 dense/focus/CMC 结果 | `main_focus_SEC_only.csv`、`main_dense.csv`、`main_cmc.csv`、`main_focus.csv` | `figure_11.ipynb` | strong |
| `fig:mem_access` 内存访问分析 | `doc/paper/tex/7.evaluation.tex` | `sh simulator/run_main_sim.sh` | 带 memory/access 字段的 `main_dense.csv`、`main_adaptiv.csv`、`main_cmc.csv`、`main_focus.csv` | `figure_12.ipynb` | strong |
| `tab:int8_degrade` 量化协同 | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_focus.sh int8 full`、`sh algorithm/run_original.sh int8 full`、`python simulator/main.py --all_models_datasets --accelerator focus --quantization --trace_dir ... --output_dir ...` | 带 `INT8_Dense` 和 `INT8_Focus` 列的 `accuracy.csv`、`int8_focus.csv` | `table_4.ipynb` | strong |
| `tab:image` image VLM 泛化 | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_original_image.sh accuracy`、`sh algorithm/run_focus_image.sh accuracy`、`sh algorithm/run_adaptiv_image.sh accuracy`，以及 `sh algorithm/run_focus_image.sh`、`sh algorithm/run_adaptiv_image.sh`、`sh simulator/run_image_sim.sh` | image 任务 `accuracy.csv`、`main_dense.csv`、`main_adaptiv.csv`、`main_focus.csv` | `table_5.ipynb` | strong |
| `fig:worst_case` 最坏/最好情况分析 | `doc/paper/tex/7.evaluation.tex` | `python simulator/utils/analysis.py --trace_dir ... --output_dir ...` | Focus trace 与统计结果 | 直接输出 SVG | strong |

## 按主题整理脚本链

### 主结果

1. 先用 `algorithm/run_focus.sh`、`algorithm/run_adaptiv.sh`、`algorithm/run_cmc.sh` 生成 Focus 与 baseline trace。
2. 再用 `simulator/run_main_sim.sh` 跑硬件模拟。
3. 用 `python simulator/arch/accelerator.py --output_dir ...` 生成 area/power 汇总。
4. 打开 `evaluation_scripts/plot_scripts/ipynb_src/figure_9.ipynb` 与 `table_2.ipynb`。

### DSE

1. 用 `algorithm/run_dse.sh` 生成 DSE trace。
2. 如果需要论文中的粗略 accuracy 曲线，再运行 `algorithm/run_dse.sh accuracy`。
3. 用 `simulator/run_dse_sim.sh` 进行模拟。
4. 打开 `figure_10.ipynb`。

### 量化与 image task

1. 使用 `algorithm/run_focus.sh int8` 或 image task 脚本。
2. 使用 `simulator/main.py --quantization` 或 `simulator/run_image_sim.sh`。
3. 打开 `table_4.ipynb` 或 `table_5.ipynb`。

## 最容易追踪的图表

- `fig:DSE`
- `fig:ablation`
- `fig:mem_access`
- `tab:int8_degrade`
- `tab:image`

这些图表从脚本到 notebook 的链路最完整、最清晰。

## 最难精确追踪的图表

- `tab:hardware-config`：没有单一生成脚本，更像是论文作者根据多个默认值手工汇总的表。
- `fig:main`：主要 CSV 链路清楚，但 GPU 对比依赖 `evaluation_scripts/jetson_stats/figure9_gpu.csv`。
- `tab:setup_compare`：数值能从 `accelerator_area_power_buffer.csv` 追到，但最终论文表格格式不是单独脚本吐出来的。

## 小结

- 直接观察：实验流水线的脚本化程度很高。
- 推断：项目很好地把“结果生成”和“论文展示样式”分离开了，但部分论文表格仍然包含 notebook 或手工排版层面的整理。
