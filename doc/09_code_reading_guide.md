# 09 Code Reading Guide

This guide is optimized for a new researcher who wants to understand the repository quickly without getting buried in vendored code too early.

## Recommended Reading Strategy

### Pass 1: Orient Yourself

1. Read the repository story and locate the real entry points.
2. Understand the artifact handoff between `algorithm/` and `simulator/`.
3. Only then go into subsystem internals.

### Pass 2: Trace One Real Flow

1. Start with a Focus trace-generation command.
2. Follow it through model patching and trace export.
3. Then follow the resulting trace into the simulator.

### Pass 3: Study The Architectural Assumptions

1. Read how the simulator interprets traces.
2. Read how area/power data comes from RTL summaries.
3. Read the notebooks only after you understand the CSV contracts.

## First 15 Files To Read

| Order | File | Why this is early | What to focus on | Implementation lesson |
| --- | --- | --- | --- | --- |
| 1 | `README.md` | Gives the official subsystem boundaries and example commands. | Ignore the marketing; extract the pipeline stages and expected outputs. | A good top-level README should advertise the workflow skeleton, not every detail. |
| 2 | `algorithm/README.md` | Explains the intended algorithm experiments and output files. | Which scripts produce traces, sparsity CSVs, and accuracy CSVs. | Research repos benefit from separating “trace generation” from “full accuracy” workflows. |
| 3 | `simulator/README.md` | Explains what the simulator expects and what it produces. | Main simulation, DSE, SEC-only, INT8, and worst-case analysis modes. | Simulator docs should name artifact contracts explicitly. |
| 4 | `algorithm/run_eval.py` | This is the true algorithm-side entry point. | CLI flags, `simple_evaluate` call, score extraction, CSV write paths. | Keep the paper-facing CLI thin and explicit. |
| 5 | `algorithm/lmms-eval/lmms_eval/evaluator.py` | This is where Focus is actually activated. | `simple_evaluate`, `evaluate`, and the `apply_focus` / `apply_framefusion` calls. | Hooking a research idea into an evaluation harness is often the real integration point. |
| 6 | `algorithm/focus/interface.py` | Central switchboard for all Focus/baseline patching. | Model-family detection, CSV-config lookup, forward replacement, Accelerate hook preservation. | Centralize model surgery in one file. |
| 7 | `algorithm/focus/main.py` | Core Focus implementation. | `prepare`, `forward`, `focus_similarity_concentration`, `semantic_concentration`, `post_process`. | Make the runtime state machine explicit when a method spans layers and phases. |
| 8 | `algorithm/focus/models/qwen2/modeling_qwen2.py` | Shows exactly where Focus enters decoder execution. | Attention, MLP, decoder-layer, and model-level patches. | If a method changes model semantics, read the patched forward before reading helper code. |
| 9 | `algorithm/focus/models/llava_video/modeling_llava_video.py` | Shows how multimodal metadata is derived for a real supported model. | The `self.focus.prepare(...)` call and surrounding patch-geometry logic. | Hardware-aware model work often starts in preprocessing, not in the math kernel. |
| 10 | `algorithm/focus/baseline_adaptiv.py` | One baseline path in the same framework. | How coarse sparsity is measured and exported. | Keeping baselines inside the same runtime framework reduces comparison bias. |
| 11 | `algorithm/focus/baseline_CMC.py` | The other main baseline path. | Separate sparsity metrics for linear, query, and attention-score behavior. | Baselines often need different abstraction levels than the main method. |
| 12 | `simulator/main.py` | True simulator entry point. | How it builds `ModelConfig`, `SparseInfo`, `Accelerator`, and `Simulator`. | Entry files should be readable orchestration, not a dump of formulas. |
| 13 | `simulator/models/models.py` | Exposes one of the simulator’s strongest assumptions. | Hardcoded Qwen2-7B-like architecture plus `meta_data.csv` loading. | Always identify where a simulator hardcodes the workload model. |
| 14 | `simulator/models/sparse_info.py` | Shows the algorithm-to-simulator bridge. | File naming conventions and Focus/baseline loading logic. | Stable artifact naming is as important as stable APIs. |
| 15 | `simulator/core/simulator.py` | Shows the top-level hardware accounting loop. | `run_focus`, `run_dense`, `run_adaptiv`, `run_cmc`, and result aggregation. | Keep aggregation separate from low-level formulas. |

## Next 5 Files After That

| Order | File | Why it matters | What to learn |
| --- | --- | --- | --- |
| 16 | `simulator/core/simulator_comp.py` | Compute-cycle formulas and ScaleSim integration. | How sparse traces turn into compute and stall cycles. |
| 17 | `simulator/core/simulator_mem.py` | Memory accounting. | How the repository models SRAM/DRAM traffic for Focus and baselines. |
| 18 | `simulator/arch/accelerator.py` | Accelerator composition, area, and power. | How hardware blocks and SRAM models are combined. |
| 19 | `rtl/README.md` | Fast way to map hardware block names to RTL files. | Which RTL modules correspond to which architectural blocks. |
| 20 | `evaluation_scripts/plot_scripts/ipynb_src/figure_10.ipynb` | Best notebook for understanding end-to-end DSE reporting. | How algorithm accuracy and simulator performance are joined in the paper. |

## Why This Reading Order Works

- **Reasoned inference**: it postpones vendored framework details until after the repo’s own glue code is clear.
- **Reasoned inference**: it makes you understand the artifact handoff before reading the formulas that consume those artifacts.
- **Reasoned inference**: it also surfaces the biggest hidden assumption early: the simulator’s hardcoded architecture template in `simulator/models/models.py`.

## What To Focus On In Each Phase

### While Reading Algorithm Files

- How `run_eval.py` delegates into `lmms-eval`
- How `interface.py` chooses a model family
- Where `self.focus.prepare(...)` is called
- Where SIC and SEC are injected into decoder execution
- When traces are written

### While Reading Simulator Files

- How `meta_data.csv` and `.pth` traces are loaded
- Where layer-name remapping happens
- Which parts come from ScaleSim vs closed-form formulas
- Where area/power comes from `*_rtl.csv`

### While Reading RTL Files

- Which modules are real datapath blocks vs arithmetic primitives
- Which simulator component names map to which RTL files
- Which pieces are missing for a full integrated hardware flow

## Implementation Lessons Worth Reusing

- **Direct observation**: a strong research repo keeps entry points small and pushes complexity into a few clear switchboards.
- **Direct observation**: artifact contracts such as `.pth` traces and CSV tables are first-class interfaces here.
- **Reasoned inference**: if you build your own research codebase, imitate the separation between algorithm runtime, simulator, and plotting. That is one of the best engineering decisions in this repo.
