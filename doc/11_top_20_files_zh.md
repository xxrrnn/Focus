# 11 最重要的 20 个文件

下面这 20 个文件，是理解 Focus 这个“论文驱动研究仓库”时信息密度最高的部分。

1. `doc/paper/main.tex` 是论文的真正根文件。它直接告诉你作者把哪些章节当成主干，也为后续所有“论文到代码”的对照提供了基准。

2. `doc/paper/tex/4.overview.tex` 是最清晰的架构总览。Focus Unit、SEC、SIC 的角色都在这里被最紧凑地定义出来，后面整个仓库都在支撑这张图。

3. `doc/paper/tex/5.semantic.tex` 很重要，因为它把 SEC 的硬件叙事讲得比代码布局更清楚。当论文术语与代码命名对不上时，这个文件能解释作者原本想表达什么。

4. `doc/paper/tex/6.vector.tex` 是理解 block size、vector size、gather、layouter、scatter 的最佳入口。很多 simulator 和 trace 设计，读完它才会“突然变得合理”。

5. `doc/paper/tex/7.evaluation.tex` 是整仓库的“证据合同”。所有脚本、CSV、notebook 是否真的支撑论文结论，都应该回到这个文件检查。

6. `algorithm/run_eval.py` 是算法侧的主入口。CLI 参数、日志、accuracy 表写出、sparsity 表写出、trace 导出，全都在这里接起来。

7. `algorithm/lmms-eval/lmms_eval/evaluator.py` 是 evaluation 框架真正激活 Focus、CMC、AdapTiV、FrameFusion 的地方。没有这个文件，`run_eval.py` 只是参数胶水。

8. `algorithm/focus/interface.py` 是论文方法与真实 VLM 实现之间的接缝。它也是本仓库最值得借鉴的工程模式之一：把方法逻辑和模型族 patch 逻辑分开。

9. `algorithm/focus/main.py` 是方法核心。SEC、SIC、trace 分配、sparsity 统计、trace 序列化都收敛到这里，是全仓库最重要的代码文件。

10. `algorithm/focus/models/qwen2/modeling_qwen2.py` 的价值在于它展示了真正的注入点：Focus 究竟在哪些 attention、MLP、projection、模型收尾路径上生效。抽象方法在这里变成了真实 forward。

11. `algorithm/focus/models/llava_video/modeling_llava_video.py` 很重要，因为 Focus 运行前需要先准备好多模态 token 几何信息。它说明了方法并不只是“改 attention”，还依赖正确的 frame/patch 元数据。

12. `algorithm/focus/configs/focus.csv` 很小，但很关键。论文里的 pruning schedule 与默认超参，最后都是在这里变成可执行配置的。

13. `simulator/main.py` 是模拟器总入口。主实验、DSE、INT8、image run、SEC-only ablation 都从这里分流。

14. `simulator/models/sparse_info.py` 是算法输出与架构评估之间的协议边界。它精确定义了 simulator 期待的 Focus trace 和 baseline CSV 长什么样。

15. `simulator/models/models.py` 的重要性在于它暴露了一个隐藏假设：模拟器复用了一个 Qwen2 风格的架构模板。判断硬件结果对模型是否足够忠实，必须看这个文件。

16. `simulator/core/simulator.py` 是 layer-by-layer 模拟调度中心。dense、Focus、CMC、AdapTiV 都在同一个框架下被组织起来。

17. `simulator/core/simulator_comp.py` 是 ScaleSim 与 Focus 特定 scatter/gather 逻辑耦合的地方。它是“算法 sparsity 如何变成 cycle estimate”的最强证据。

18. `simulator/core/simulator_mem.py` 特别重要，因为论文中的很多收益本质上是 memory-driven。这个文件编码了 DRAM/SRAM 与 activation traffic 的关键假设。

19. `simulator/arch/accelerator.py` 是 simulator 的 area/power/buffer 权威来源。它也能让你看清：哪些硬件结论来自常量，哪些来自 CACTI / memory compiler，哪些来自预计算 RTL CSV。

20. `evaluation_scripts/plot_scripts/ipynb_src/figure_10.ipynb` 是最值得看的 notebook。它把 DSE trace、simulator 输出和论文图表连接得最干净，如果你想在核心代码之后只看一个下游产物，就看它。
