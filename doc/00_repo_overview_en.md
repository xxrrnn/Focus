# 00 Repo Overview

## Purpose

This repository is a full-stack implementation of *Focus*, a VLM acceleration project whose paper narrative is defined in `doc/paper/main.tex` and especially `doc/paper/tex/4.overview.tex`, `doc/paper/tex/5.semantic.tex`, `doc/paper/tex/6.vector.tex`, and `doc/paper/tex/7.evaluation.tex`.

- **Paper claim**: *Focus* is a streaming concentration architecture with semantic-, block-, and vector-level redundancy elimination, introduced in `doc/paper/tex/0.abstract.tex`, `doc/paper/tex/1.introduction.tex`, and `doc/paper/tex/3.motivation.tex`.
- **Code evidence**: the repository splits that story across `algorithm/`, `simulator/`, `rtl/`, and `evaluation_scripts/`.
- **Reasoned inference**: the repo is organized by execution pipeline, not by paper section order.

## What Problem It Solves

- **Direct observation**: the paper frames the problem as efficient VLM inference under heavy visual-token redundancy in `doc/paper/tex/1.introduction.tex` and `doc/paper/tex/2.background.tex`.
- **Direct observation**: the repo implements:
  - algorithm-side redundancy elimination in `algorithm/`
  - hardware/performance modeling in `simulator/`
  - hardware blocks in `rtl/`
  - result reproduction in `evaluation_scripts/`

## Repository Structure With Paper Alignment

| Directory | Main role | Paper anchor | Typical outputs |
| --- | --- | --- | --- |
| `algorithm/` | Real VLM execution, Focus injection, trace export, accuracy evaluation | `doc/paper/tex/5.semantic.tex`, `doc/paper/tex/6.vector.tex`, `doc/paper/tex/7.evaluation.tex` | `.pth` traces, `meta_data.csv`, `accuracy.csv`, baseline sparsity CSVs |
| `simulator/` | Cycle / memory / energy / area modeling | `doc/paper/tex/7.evaluation.tex` | `main_focus.csv`, `dse_*.csv`, power/area breakdown CSVs |
| `rtl/` | RTL blocks corresponding to Focus and baseline hardware components | `doc/paper/tex/4.overview.tex`, `doc/paper/tex/5.semantic.tex`, `doc/paper/tex/6.vector.tex` | Verilog/SystemVerilog sources; indirect support for `*_rtl.csv` |
| `evaluation_scripts/` | Figure/table assembly | `doc/paper/tex/7.evaluation.tex` | notebooks, plotted figures/tables |
| `doc/paper/` | Original paper source | all paper sections | TeX, figures, references |

## Meaningful Tree

```text
.
├── algorithm/
│   ├── run_eval.py
│   ├── run_focus.sh / run_dse.sh / run_*baseline*.sh
│   ├── focus/
│   │   ├── interface.py
│   │   ├── main.py
│   │   ├── baseline_adaptiv.py
│   │   ├── baseline_CMC.py
│   │   ├── configs/*.csv
│   │   └── models/
│   └── lmms-eval/
├── simulator/
│   ├── main.py
│   ├── arch/accelerator.py
│   ├── models/models.py
│   ├── models/sparse_info.py
│   ├── core/simulator.py
│   ├── core/simulator_comp.py
│   └── core/simulator_mem.py
├── rtl/
│   ├── traditional_systolic.v
│   ├── cosine_similarity_unit.sv
│   ├── inv_magnitude_unit.v
│   ├── average_update_unit.sv
│   ├── adaptiv_array.v
│   └── cmc_codec_pe.v
├── evaluation_scripts/plot_scripts/ipynb_src/
└── doc/paper/
    ├── main.tex
    └── tex/*.tex
```

## How The Major Pieces Connect

### Paper -> Algorithm

- **Direct observation**: SEC is described in `doc/paper/tex/5.semantic.tex` and maps mainly to `algorithm/focus/main.py` plus patched decoder logic in `algorithm/focus/models/qwen2/modeling_qwen2.py`.
- **Direct observation**: SIC is described in `doc/paper/tex/6.vector.tex` and maps mainly to `algorithm/focus/main.py` plus the same patched forward path.
- **Reasoned inference**: the paper’s “block level” is not a standalone software module; in code it is expressed through `block_size`, `frame_block_size`, and local candidate construction inside `Focus.focus_similarity_concentration(...)` in `algorithm/focus/main.py`.

### Algorithm -> Simulator

- **Direct observation**: `algorithm/focus/main.py` writes Focus traces and metadata.
- **Direct observation**: `simulator/models/sparse_info.py` and `simulator/models/models.py` load those artifacts.
- **Reasoned inference**: the real interface between ML execution and hardware evaluation is the trace file format, not shared runtime code.

### Simulator -> Paper Figures

- **Direct observation**: notebooks in `evaluation_scripts/plot_scripts/ipynb_src/` read simulator CSVs and algorithm accuracy CSVs.
- **Direct observation**: `simulator/utils/analysis.py` produces the worst-case histogram figure corresponding to `doc/paper/tex/7.evaluation.tex`.

### RTL -> Simulator

- **Direct observation**: `simulator/arch/accelerator.py` consumes `simulator/arch/focus_rtl.csv`, `adaptiv_rtl.csv`, and `cmc_rtl.csv`.
- **Unclear**: the synthesis scripts that produced those CSVs are not present in `rtl/`.

## Does The Repo Mirror The Paper?

- **Direct observation**: no. The paper is organized as motivation -> overview -> SEC -> SIC -> evaluation in `doc/paper/tex/*.tex`.
- **Direct observation**: the repo is organized as execution stages: algorithm runtime, simulator, RTL, plotting.
- **Reasoned inference**: this is a stronger engineering layout for reproducible experiments, even though it is less paper-shaped.

## What Is Central vs Infrastructure

| Category | Files | Why they matter |
| --- | --- | --- |
| Core scientific contribution | `algorithm/focus/main.py`, `algorithm/focus/interface.py`, `algorithm/focus/models/qwen2/modeling_qwen2.py` | These files implement SEC, SIC, and the real model injection path. |
| Hardware-method bridge | `simulator/models/sparse_info.py`, `simulator/core/simulator.py`, `simulator/core/simulator_comp.py`, `simulator/core/simulator_mem.py` | These files turn Focus traces into hardware claims. |
| Hardware realization evidence | `rtl/*.v`, `rtl/*.sv`, `simulator/arch/*_rtl.csv` | These files support area/power claims. |
| Infrastructure / glue | `algorithm/lmms-eval/`, shell scripts, notebook code | Necessary for execution and reporting, but not the core method itself. |

## Bottom Line

- **Direct observation**: the repository is a pipeline from paper method -> patched VLM execution -> serialized sparsity -> accelerator simulation -> figures.
- **Reasoned inference**: the best way to study it is not folder-by-folder, but paper concept by paper concept while tracking where artifacts are created and consumed.
