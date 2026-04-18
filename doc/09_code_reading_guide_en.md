# 09 Code Reading Guide

This guide is for reading the repository as a researcher, not as a package user. The most effective order is paper-first, then execution path, then subsystem deep dives.

## Best Reading Order

1. Read `doc/paper/main.tex` together with `doc/paper/tex/1.introduction.tex`, `doc/paper/tex/4.overview.tex`, `doc/paper/tex/5.semantic.tex`, `doc/paper/tex/6.vector.tex`, and `doc/paper/tex/7.evaluation.tex`.
2. Read `algorithm/run_eval.py` and `algorithm/lmms-eval/lmms_eval/evaluator.py` to understand the real entry point.
3. Read `algorithm/focus/interface.py` to see how Focus is injected.
4. Read `algorithm/focus/main.py` and `algorithm/focus/models/qwen2/modeling_qwen2.py` to understand SEC, SIC, and where they sit in the model forward.
5. Read `simulator/main.py`, `simulator/models/sparse_info.py`, `simulator/core/simulator.py`, `simulator/core/simulator_comp.py`, and `simulator/core/simulator_mem.py`.
6. Read `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb` only after you understand the CSV producers.
7. Read `rtl/README.md` and selected RTL files last, as architectural evidence rather than the primary execution path.

## Why This Order Works

- The paper in `doc/paper/tex/*.tex` is contribution-oriented.
- The repo is execution-oriented: `algorithm/` -> `simulator/` -> `evaluation_scripts/`.
- Reading in repository order alone hides the paper’s conceptual split between SEC and SIC.
- Reading in paper order alone hides the actual artifact boundaries, especially the trace interface between `algorithm/focus/main.py` and `simulator/models/sparse_info.py`.

## First 16 Files To Read

| Order | File | What to focus on | Reusable lesson |
| --- | --- | --- | --- |
| 1 | `doc/paper/main.tex` | include order and narrative spine | Keep one top-level paper source that reveals author intent. |
| 2 | `doc/paper/tex/4.overview.tex` | Focus Unit, SEC, SIC roles | Good paper-level decomposition, even if code organization differs. |
| 3 | `doc/paper/tex/5.semantic.tex` | SEC sub-blocks and assumptions | Separate algorithm effect from hardware embodiment. |
| 4 | `doc/paper/tex/6.vector.tex` | gather, layouter, scatter | Hardware concepts may map to multiple software files. |
| 5 | `doc/paper/tex/7.evaluation.tex` | what evidence the paper claims | Always read claimed evaluation methodology before trusting scripts. |
| 6 | `algorithm/run_eval.py` | CLI, CSV writers, flags | Centralize experiment entry and output formatting. |
| 7 | `algorithm/lmms-eval/lmms_eval/evaluator.py` | where Focus is actually activated | Put method injection in the evaluation stack, not in ad hoc scripts. |
| 8 | `algorithm/focus/interface.py` | monkeypatch strategy and config loading | A thin integration layer keeps core method code model-agnostic. |
| 9 | `algorithm/focus/main.py` | the actual method | Keep scientific logic concentrated in one class. |
| 10 | `algorithm/focus/models/qwen2/modeling_qwen2.py` | exact injection sites | Patch the smallest stable set of model functions. |
| 11 | `algorithm/focus/models/llava_video/modeling_llava_video.py` | metadata preparation | Model-specific multimodal geometry should be isolated. |
| 12 | `simulator/main.py` | simulator entry modes | Keep normal run, DSE, quantization, and ablation in one dispatcher. |
| 13 | `simulator/models/sparse_info.py` | trace loading contract | Use an explicit serialized boundary between ML runtime and simulator. |
| 14 | `simulator/core/simulator.py` | layer orchestration | Separate orchestration from compute/memory micro-models. |
| 15 | `simulator/core/simulator_comp.py` | ScaleSim coupling and scatter/gather compute | Delegate commodity modeling to external tools; keep method-specific logic local. |
| 16 | `simulator/core/simulator_mem.py` | memory traffic formulas | Put memory accounting in a separate module so assumptions are auditable. |

## What To Ignore On First Pass

- Do not start with notebooks in `evaluation_scripts/plot_scripts/ipynb_src/`; they are downstream-only.
- Do not start with raw RTL files in `rtl/`; they support hardware claims, but they are not the fastest way to understand execution.
- Do not over-read vendored code in `algorithm/lmms-eval/` or `3rd_party/` before you understand what this repo changed.

## What To Study For Engineering Methodology

- `algorithm/focus/interface.py`: a clean patch layer between external model code and new research logic.
- `algorithm/focus/main.py`: one concentrated class that owns both runtime behavior and trace serialization.
- `simulator/models/sparse_info.py`: a strong example of trace-based decoupling between ML execution and hardware simulation.
- `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`: a downstream-only presentation layer that does not contaminate the core runtime.

## Direct Observation / Inference / Unclear

- Direct observation: the repo is strongest when the paper concept maps to either `algorithm/focus/main.py` or the trace-driven simulator path.
- Reasoned inference: the codebase was organized for experiment throughput and artifact production, not for pedagogical alignment with the paper.
- Unclear: the full internal process that generated `simulator/arch/*_rtl.csv` is not readable from the checked-in repo alone.
