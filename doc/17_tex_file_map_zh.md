# 17 TeX 文件地图

这份地图把论文源码本身也视作仓库架构的一部分，而不是单纯的附录文件。

## 顶层文件

| 文件 | 角色 | 应该关注什么 | 相关仓库路径 |
| --- | --- | --- | --- |
| `doc/paper/main.tex` | 论文总入口 | include 顺序、章节顺序、参考文献挂接方式 | 所有文档与代码映射 |
| `doc/paper/macros.tex` | 全局宏定义 | `\proj` 被定义为 `Focus`，以及图表章节缩写宏 | 有助于解释正文表达 |
| `doc/paper/hpca-template.tex` | 会议排版模板 | 主要是格式，不是方法内容 | 优先级低 |
| `doc/paper/refs.bib` | 参考文献 | AdapTiV、CMC、FrameFusion、SCALEsim、DRAMsim3 等术语来源 | 主要用于外部背景 |
| `doc/paper/00README.json` | artifact 元信息 | 确认 `main.tex` 是入口，也提供构建工具信息 | 环境与 artifact 上下文 |

## 章节文件

| 文件 | 主要内容 | 主要标签 / 图表 | 最接近的代码或脚本锚点 |
| --- | --- | --- | --- |
| `doc/paper/tex/0.abstract.tex` | 用极短篇幅概括贡献与 headline 指标 | 无 | `algorithm/focus/main.py`、`simulator/main.py` |
| `doc/paper/tex/1.introduction.tex` | 问题定义、贡献列表、引导图 | `fig:intro` | `README.md`、`algorithm/`、`simulator/` |
| `doc/paper/tex/2.background.tex` | VLM 与效率优化背景 | 无 | 主要是背景说明 |
| `doc/paper/tex/3.motivation.tex` | 为什么要同时抓 semantic redundancy 与局部向量冗余 | `fig:motivation1`、`fig:motivation2` | `algorithm/focus/main.py`、`simulator/core/simulator_mem.py` |
| `doc/paper/tex/4.overview.tex` | Focus Unit 总览 | `fig:focus_arch` | `algorithm/focus/main.py`、`simulator/arch/accelerator.py`、`rtl/README.md` |
| `doc/paper/tex/5.semantic.tex` | SEC 设计与硬件叙事 | `fig:semantic` | `algorithm/focus/main.py`、`algorithm/focus/models/qwen2/modeling_qwen2.py` |
| `doc/paper/tex/6.vector.tex` | SIC 的 gather、layouter、scatter | `fig:sic_gather`、`fig:sic_layouter`、`fig:sic_scatter` | `algorithm/focus/main.py`、`simulator/core/simulator_comp.py`、`simulator/core/simulator_mem.py`、`rtl/*.v`、`rtl/*.sv` |
| `doc/paper/tex/7.evaluation.tex` | methodology 与所有主实验结果 | `tab:hardware-config`、`tab:accuracy`、`fig:main`、`tab:setup_compare`、`fig:DSE`、`fig:ablation`、`fig:mem_access`、`tab:int8_degrade`、`tab:image`、`fig:worst_case` | `algorithm/run_*.sh`、`simulator/run_*.sh`、`evaluation_scripts/plot_scripts/ipynb_src/` 下的 notebook、`simulator/utils/analysis.py` |
| `doc/paper/tex/8.conclusion.tex` | 对贡献的最终重述 | 无 | 主要是总结 |
| `doc/paper/tex/acknowledgement.tex` | 致谢 | 无 | 技术上不重要 |
| `doc/paper/tex/artifact_appendix.tex` | artifact 范围、环境、工作流、预期结果 | 无 | `README.md`、`algorithm/README.md`、`simulator/README.md`、依赖文件 |

## 为什么这份地图重要

- 直接观察：论文源码本身是模块化的，而且比代码目录树更能体现概念贡献的切分。
- 推断：如果你想一直保持“论文与代码脑内对齐”，最值得反复来回看的四个 TeX 文件是 `doc/paper/tex/4.overview.tex`、`doc/paper/tex/5.semantic.tex`、`doc/paper/tex/6.vector.tex`、`doc/paper/tex/7.evaluation.tex`。
- 可复用经验：把论文源码放进同一个仓库非常有价值，因为作者意图、术语系统、artifact 预期都会被版本化保留下来。
