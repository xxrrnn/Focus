# 16 Implementation-vs-Paper Gaps

This document explains where the repository mirrors the paper closely, where it compresses the paper into fewer code structures, and where evidence is weaker than the paper narrative.

## Core Gap Table

| Topic | Paper side | Repository side | Assessment |
| --- | --- | --- | --- |
| Three concentration levels | `doc/paper/tex/0.abstract.tex` and `doc/paper/tex/3.motivation.tex` present semantic-, block-, and vector-level concentration separately | `algorithm/focus/main.py` has one SEC path and one SIC path; block and vector logic are fused in `focus_similarity_concentration(...)` | same core behavior, different software decomposition |
| SEC hardware sub-blocks | `doc/paper/tex/5.semantic.tex` names analyzer, top-k sorter, and offset encoder | software expresses these through attention reduction, `torch.topk`, and retained-index bookkeeping in `algorithm/focus/main.py`; no equally explicit RTL block set was found | algorithmic effect is present; sub-block embodiment is only partial |
| SIC hardware sub-blocks | `doc/paper/tex/6.vector.tex` names gather, layouter, and scatter | gather-like behavior appears in algorithm code; layouter and scatter are mainly simulator concepts in `simulator/core/simulator_comp.py` and `simulator/core/simulator_mem.py` | paper is cleaner than code here |
| Architecture overview | `doc/paper/tex/4.overview.tex` presents a single Focus Unit near memory | repo splits this across `algorithm/`, `simulator/`, and `rtl/` | sensible for engineering, but not visually one-to-one |
| Baseline comparison | paper discusses AdapTiV and CMC as architecture baselines in `doc/paper/tex/7.evaluation.tex` | algorithm baselines exist in `algorithm/focus/baseline_*.py`, but simulator uses aggregate sparsity CSVs in `simulator/models/sparse_info.py` | baseline path is less detailed than Focus path |
| DRAM modeling | paper says DRAMsim3 is used in `doc/paper/tex/7.evaluation.tex` | checked-in runtime mainly uses constants in `simulator/arch/accelerator.py` | evidence is weaker than wording |
| RTL support | paper describes SystemVerilog implementation and synthesis in `doc/paper/tex/7.evaluation.tex` | repo has RTL blocks and CSV summaries, but no full synthesis/testbench flow | partially represented |
| Model coverage | paper evaluates three video VLMs and two image VLMs in `doc/paper/tex/7.evaluation.tex` | algorithm supports these models, but simulator uses a single Qwen2-like architecture template in `simulator/models/models.py` | execution support is broader than simulator fidelity |

## Naming Mismatches

| Paper term | Code term(s) | Why the mismatch exists |
| --- | --- | --- |
| Semantic Concentrator (SEC) | `semantic_concentration`, `set_token_importance`, retained token bookkeeping | the code names actions; the paper names hardware blocks |
| Similarity Gather / Scatter | `focus_similarity_concentration`, `group_idx`, simulator scatter/gather helpers | the runtime records metadata, while the simulator interprets it architecturally |
| Convolution-style Layouter | `layouter` buffer in `simulator/arch/accelerator.py`, memory-side assumptions in `simulator/core/simulator_mem.py` | the paper foregrounds hardware dataflow; the code foregrounds performance modeling |

## Does The Repo Mirror The Paper Structure?

- Direct observation: no. The paper is `motivation -> overview -> SEC -> SIC -> evaluation`, while the repo is `algorithm -> simulator -> rtl -> plotting`.
- Reasoned inference: the repository organization is better for experiment execution and artifact production. It keeps each execution boundary explicit.
- Reusable lesson: if you are building your own co-design project, organizing by pipeline boundary can be more maintainable than organizing by paper section.

## What Is Central vs What Is Glue

### Central to the scientific contribution

- `algorithm/focus/main.py`
- `algorithm/focus/interface.py`
- `algorithm/focus/models/qwen2/modeling_qwen2.py`
- `simulator/core/simulator.py`
- `simulator/core/simulator_comp.py`
- `simulator/core/simulator_mem.py`

### Mostly infrastructure or glue

- shell scripts in `algorithm/run_*.sh` and `simulator/run_*.sh`
- CSV-writing helpers in `algorithm/run_eval.py`
- notebooks in `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`
- vendored frameworks in `algorithm/lmms-eval/` and `3rd_party/`

## Why This Organization Is Still Worth Learning

- It isolates model-patching concerns in `algorithm/focus/interface.py`.
- It turns the algorithm/simulator boundary into a stable file format contract in `algorithm/focus/main.py` and `simulator/models/sparse_info.py`.
- It keeps the paper presentation layer downstream, so figure generation does not interfere with method execution.

## Bottom Line

- Direct observation: the biggest gaps are not about whether Focus exists in code, but about whether the paper’s hardware decomposition is exposed with equal clarity in the repo.
- Reasoned inference: this is normal for research code. The paper optimizes for persuasion; the repo optimizes for execution.
