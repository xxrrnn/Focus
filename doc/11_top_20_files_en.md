# 11 Top 20 Files

These are the 20 files that carry the most explanatory weight for understanding Focus as a paper-backed research artifact.

1. `doc/paper/main.tex` is the paper’s real root. It tells you which sections the authors consider first-class, and it anchors every later paper-to-code comparison.

2. `doc/paper/tex/4.overview.tex` is the cleanest high-level statement of the architecture. It defines the Focus Unit, SEC, and SIC in the form the rest of the repo is trying to support.

3. `doc/paper/tex/5.semantic.tex` matters because it explains the SEC hardware story in more detail than the code layout itself does. When code and paper terminology diverge, this file explains why.

4. `doc/paper/tex/6.vector.tex` is the best source for understanding why the repository talks about block size, vector size, gather, layouter, and scatter. The simulator and trace format make far more sense after reading it.

5. `doc/paper/tex/7.evaluation.tex` is the contract for what the repository is supposed to prove. Every script, CSV, and notebook should be judged against this file.

6. `algorithm/run_eval.py` is the primary algorithm-side entry point. It is where CLI flags, logging, accuracy writing, sparsity writing, and trace export are stitched together.

7. `algorithm/lmms-eval/lmms_eval/evaluator.py` is where the evaluation framework actually activates Focus, CMC, AdapTiV, or FrameFusion. Without this file, `run_eval.py` is just argument plumbing.

8. `algorithm/focus/interface.py` is the integration seam between the paper method and multiple real VLM implementations. It is also one of the best reusable patterns in the repo: keep method logic separate from model-family patch logic.

9. `algorithm/focus/main.py` is the method core. SEC, SIC, trace allocation, sparsity accounting, and trace serialization all converge here, making it the single most important code file.

10. `algorithm/focus/models/qwen2/modeling_qwen2.py` matters because it shows the real injection points: where Focus touches attention, MLP, projections, and end-of-model cleanup. It converts the abstract method into an actual forward path.

11. `algorithm/focus/models/llava_video/modeling_llava_video.py` is important because it prepares multimodal token geometry before Focus runs. It shows how the method depends on correct frame/patch metadata.

12. `algorithm/focus/configs/focus.csv` is where paper-level pruning schedules become executable defaults. It is small, but it encodes a lot of the paper’s practical hyperparameter story.

13. `simulator/main.py` is the simulator dispatcher. Main results, DSE, INT8, image runs, and SEC-only ablations all branch from here.

14. `simulator/models/sparse_info.py` is the bridge contract between algorithm outputs and architecture evaluation. It shows exactly what the simulator expects Focus traces and baseline CSVs to look like.

15. `simulator/models/models.py` matters because it exposes a hidden assumption: the simulator reuses one Qwen2-like architecture template. This file is crucial for judging how model-faithful the hardware results really are.

16. `simulator/core/simulator.py` is the top-level orchestration of layer-by-layer simulation. It tells you how dense, Focus, CMC, and AdapTiV are compared under one framework.

17. `simulator/core/simulator_comp.py` is where ScaleSim is coupled to Focus-specific scatter/gather logic. It is the strongest evidence for how algorithmic sparsity becomes cycle estimates.

18. `simulator/core/simulator_mem.py` matters because many of the paper’s gains are memory-driven. This file encodes the DRAM/SRAM and activation-traffic assumptions behind those claims.

19. `simulator/arch/accelerator.py` is the area/power/buffer authority for the simulator. It is also the file that reveals which hardware claims come from constants, which come from CACTI/compiler tables, and which come from precomputed RTL CSVs.

20. `evaluation_scripts/plot_scripts/ipynb_src/figure_10.ipynb` is the single most educational notebook because it cleanly connects DSE traces, simulator outputs, and paper figures. If you want one downstream artifact to read after the core code, read this one.
