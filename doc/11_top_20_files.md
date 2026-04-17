# 11 Top 20 Files

This list is ordered by “how much understanding you gain per file” rather than by execution order alone.

## 1. `README.md`

This is the best single-file orientation point. It names the intended four-layer structure of the repository, gives canonical example commands, and tells you what the authors think the project contains. It is not sufficient by itself, but it is the fastest way to align terminology before reading code.

## 2. `algorithm/run_eval.py`

This is the true algorithm-side entry point. It is where Focus-specific CLI flags, quantization flags, trace-export flags, accuracy-table writing, and task-specific score extraction all meet. If you only read one algorithm file first, read this one.

## 3. `algorithm/focus/interface.py`

This file is the runtime switchboard for Focus, CMC, and Adaptiv. It detects model families, loads default thresholds from CSV files, replaces multimodal preprocessing hooks, patches decoder/attention/MLP forwards, and preserves Accelerate hooks. It explains how the repository avoids forking entire model implementations.

## 4. `algorithm/focus/main.py`

This is the core Focus implementation. It contains the state machine for SIC, SEC, trace buffering, token-importance tracking, median-sample selection, and artifact export. Most of the paper’s method-level novelty lands here.

## 5. `algorithm/focus/models/qwen2/modeling_qwen2.py`

This is where the abstract Focus logic touches the real decoder stack. It shows the exact injection points for `q_proj`, `query`, `o_proj`, `gate_proj`, `down_proj`, and semantic pruning inside selected decoder layers. Reading this file makes the methodology concrete.

## 6. `algorithm/focus/models/llava_video/modeling_llava_video.py`

This file shows how Focus obtains frame and patch metadata from a real multimodal model path. It is one of the clearest places to see how the algorithm side bridges high-level VLM inputs to hardware-relevant token geometry.

## 7. `algorithm/focus/baseline_adaptiv.py`

This is the Adaptiv baseline implementation. It matters because it shows how the repository normalizes baseline comparison by running everything through the same patched-forward infrastructure and emitting comparable sparsity artifacts.

## 8. `algorithm/focus/baseline_CMC.py`

This is the CMC baseline implementation. It is important because it exposes the coarser abstraction used for CMC in the simulator: separate sparsity summaries for linear layers, query computation, and attention scores rather than full fine-grained traces.

## 9. `algorithm/focus/configs/focus.csv`

This small file is deceptively important. It holds dataset/model-specific defaults for selected SEC layers, alpha schedules, and similarity thresholds. It tells you that Focus is not one universal setting; it is tuned per benchmark/model pair.

## 10. `algorithm/lmms-eval/lmms_eval/evaluator.py`

This file is the integration boundary with the evaluation harness. It is where models are instantiated, tasks are built, and Focus/CMC/Adaptiv/FrameFusion are activated before inference. Without this file, the repository would be a collection of helpers rather than a runnable evaluation system.

## 11. `simulator/main.py`

This is the true simulator entry point. It shows every supported run mode: main simulation, DSE, INT8, SEC-only, and image-task paths. It is also the best place to see the object model of the simulator at a glance.

## 12. `simulator/models/models.py`

This file encodes the simulator’s biggest hidden assumption: a hardcoded Qwen2-7B-like decoder template shared across supported models, with only sequence length and patch/frame metadata varying by trace. That assumption strongly shapes how you should interpret simulator results.

## 13. `simulator/models/sparse_info.py`

This file is the actual file-format bridge between algorithm and simulator. It shows exactly how Focus `.pth` traces and baseline sparsity CSVs are loaded, named, and validated. If you ever change artifact formats, this file is one of the first places that must change.

## 14. `simulator/arch/accelerator.py`

This file defines the accelerator configurations, buffer structures, DRAM assumptions, and component counts for Focus and the baselines. It is also where RTL-derived area/power numbers and SRAM models are brought into the simulator.

## 15. `simulator/core/simulator.py`

This is the aggregation engine. It loops over blocks and layer types, maps trace names to modeled operations, calls compute and memory submodels, accumulates totals, and produces the final result dictionary. It is the most important simulator file after the entry point.

## 16. `simulator/core/simulator_comp.py`

This file contains the compute-side modeling formulas and the ScaleSim integration. It tells you what “cycle” means in this simulator: a mixture of ScaleSim GEMM estimates and custom formulas for Focus-specific scatter/gather overheads and baseline preprocess steps.

## 17. `simulator/core/simulator_mem.py`

This file contains the memory-side modeling formulas. It is where Focus-specific buffers such as `layouter`, `similarity_map`, and `similarity_table` become explicit. If you want to understand why Focus changes DRAM or SRAM behavior, this file matters.

## 18. `rtl/README.md`

This is the fastest way to map architectural block names to RTL modules. It shows how the repository authors mentally group the RTL into Focus, dense, Adaptiv, CMC, and SFU pieces, which in turn aligns with the simulator’s component CSVs.

## 19. `rtl/traditional_systolic.v`

This is the core dense compute fabric that anchors both Focus and dense baseline hardware organization. It is also one of the easiest RTL files to interpret because its module hierarchy and interconnect structure are explicit.

## 20. `rtl/cosine_similarity_unit.sv`

This file matters because it represents one of the most Focus-specific hardware blocks: the SIC cosine-similarity engine. It also reveals that the RTL layer is not equally polished everywhere, since the checked-in code has suspicious signal wiring that should be reviewed carefully.
