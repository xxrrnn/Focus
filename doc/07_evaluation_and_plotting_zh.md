# 07 评测与绘图

## 论文锚点

几乎所有实验叙事都集中在 `doc/paper/tex/7.evaluation.tex`。

## 论文图表与真实生成路径映射

| 论文产物 | 论文来源 | 真实生成来源 | 状态 |
| --- | --- | --- | --- |
| 精度 + 计算稀疏度表 | `doc/paper/tex/7.evaluation.tex` 中 `tab:accuracy` | `algorithm/run_eval.py` + `table_2.ipynb` + simulator CSV | 强 |
| 主性能 / 能耗 / 面积功耗图 | `fig:main` | `figure_9.ipynb` + `detailed_power_area_breakdown.csv` + `jetson_stats/figure9_gpu.csv` | 强 |
| 架构默认配置表 | `tab:hardware-config` | 主要来自 `simulator/arch/accelerator.py` 的默认配置和论文文本整理 | 部分 |
| 架构对比表 | `tab:setup_compare` | `simulator/arch/accelerator.py` + `accelerator_area_power_buffer.csv` | 数字强，最终排版路径较弱 |
| DSE 图 | `fig:DSE` | `algorithm/run_dse.sh` + `simulator/run_dse_sim.sh` + `figure_10.ipynb` | 强 |
| 消融图 | `fig:ablation` | `simulator/main.py --SEC_only` + `figure_11.ipynb` | 强 |
| 访存分析图 | `fig:mem_access` | `figure_12.ipynb` | 强 |
| INT8 表 | `tab:int8_degrade` | `algorithm/run_focus.sh int8` + `simulator/main.py --quantization` + `table_4.ipynb` | 强 |
| 图像 VLM 表 | `tab:image` | image 脚本 + `run_image_sim.sh` + `table_5.ipynb` | 强 |
| worst-case 图 | `fig:worst_case` | `simulator/utils/analysis.py` | 强 |

## 最容易追踪的论文产物

- `fig:DSE`
- `fig:ablation`
- `fig:mem_access`
- `tab:int8_degrade`
- `tab:image`

这些项在脚本和 notebook 中的路径非常直接。

## 相对更弱或更绕的项

- `tab:hardware-config`：更像论文作者整理出的默认硬件参数摘要，而不是一个单独脚本直接吐出的表
- `fig:main` 右侧 breakdown：可以从 `detailed_power_area_breakdown.csv` 重建，但最终论文呈现仍依赖 notebook 风格加工
- `tab:setup_compare`：数字来源可追踪，但最终论文表格不是由单一脚本直接生成

## 真实评测工作流

1. 先在 `algorithm/` 跑脚本，生成 traces 与 accuracy CSV。
2. 再在 `simulator/` 跑脚本，生成性能/能耗 CSV。
3. 最后在 `evaluation_scripts/plot_scripts/ipynb_src/` 运行 notebook。
4. notebook 通过路径变量指向生成目录或 example 目录。

## notebook 与论文图表的直接关系

- `figure_9.ipynb` 读取 `main_dense.csv`、`main_adaptiv.csv`、`main_cmc.csv`、`main_focus.csv`、`figure9_gpu.csv`
- `figure_10.ipynb` 读取 DSE simulator CSV 与 DSE accuracy CSV
- `figure_11.ipynb` 读取 `main_dense.csv`、`main_cmc.csv`、`main_focus_SEC_only.csv`、`main_focus.csv`
- `figure_12.ipynb` 读取 simulator 中的 DRAM/access 字段
- `table_2.ipynb` 读取 `accuracy.csv` 和 simulator 输出
- `table_4.ipynb` 读取 `accuracy.csv`、`main_focus.csv`、`int8_focus.csv`
- `table_5.ipynb` 读取图像任务的 accuracy 与 simulator 输出

## 这层实现为什么值得学

- **直接观察**：绘图层完全是下游消费者，不会重新执行算法逻辑。
- **推断**：这是非常好的研究工程实践。它把“论文呈现”与“方法实现核心”分开，方便反复改图、复用历史 CSV、减少不必要重跑。
