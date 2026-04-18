# 12 Paper Structure and Claims

This document reads the paper as source code for author intent. The ground truth is `doc/paper/main.tex` plus the included files under `doc/paper/tex/`.

## Paper Structure

| Section | TeX file | Main purpose | Main labels / artifacts | Closest code anchor |
| --- | --- | --- | --- | --- |
| Abstract | `doc/paper/tex/0.abstract.tex` | compress the whole thesis into contribution claims | none | entire repo, especially `algorithm/focus/` and `simulator/` |
| Introduction | `doc/paper/tex/1.introduction.tex` | problem framing, contribution summary, headline numbers | `fig:intro` | `README.md`, `algorithm/`, `simulator/` |
| Background | `doc/paper/tex/2.background.tex` | VLM and prior-efficiency context | none | context only |
| Motivation | `doc/paper/tex/3.motivation.tex` | why multilevel concentration and locality-aware hardware are needed | `fig:motivation1`, `fig:motivation2` | `algorithm/focus/main.py`, `simulator/core/simulator_mem.py` |
| Architecture Overview | `doc/paper/tex/4.overview.tex` | define Focus Unit, SEC, SIC | `fig:focus_arch` | `algorithm/focus/main.py`, `simulator/arch/accelerator.py`, `rtl/README.md` |
| Semantic Concentrator | `doc/paper/tex/5.semantic.tex` | explain SEC analyzer, sorter, encoder | `fig:semantic` | `algorithm/focus/main.py`, `algorithm/focus/models/qwen2/modeling_qwen2.py` |
| Similarity Concentrator | `doc/paper/tex/6.vector.tex` | explain gather, layouter, scatter | `fig:sic_gather`, `fig:sic_layouter`, `fig:sic_scatter` | `algorithm/focus/main.py`, `simulator/core/simulator_comp.py`, `simulator/core/simulator_mem.py`, `rtl/*.v`, `rtl/*.sv` |
| Evaluation | `doc/paper/tex/7.evaluation.tex` | methodology, main results, DSE, ablations, discussion | `tab:hardware-config`, `tab:accuracy`, `fig:main`, `tab:setup_compare`, `fig:DSE`, `fig:ablation`, `fig:mem_access`, `tab:int8_degrade`, `tab:image`, `fig:worst_case` | `algorithm/run_*.sh`, `simulator/run_*.sh`, `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`, `simulator/utils/analysis.py` |
| Conclusion | `doc/paper/tex/8.conclusion.tex` | restate contributions and headline metrics | none | summary only |
| Artifact Appendix | `doc/paper/tex/artifact_appendix.tex` | reproducibility framing, environment, workflow | none | `README.md`, `algorithm/README.md`, `simulator/README.md` |

## Top Paper Claims

### Claim 1: Focus is a streaming multilevel concentration architecture for VLMs

- Paper source: `doc/paper/tex/0.abstract.tex`, `doc/paper/tex/1.introduction.tex`, `doc/paper/tex/4.overview.tex`.
- Code evidence: `algorithm/focus/main.py` implements SEC and SIC behavior; `simulator/arch/accelerator.py` and `rtl/README.md` carry the architecture-side interpretation.
- Validation status: partially direct, partially simulator-backed.

### Claim 2: Focus removes redundancy at semantic, block, and vector levels

- Paper source: `doc/paper/tex/0.abstract.tex`, `doc/paper/tex/3.motivation.tex`, `doc/paper/tex/6.vector.tex`.
- Code evidence: semantic level is explicit in `algorithm/focus/main.py::semantic_concentration(...)`; block and vector effects are fused in `algorithm/focus/main.py::focus_similarity_concentration(...)`.
- Validation status: direct for algorithm effect; structural separation is weaker than in the paper narrative.

### Claim 3: Focus preserves accuracy while achieving higher sparsity than baselines

- Paper source: `doc/paper/tex/7.evaluation.tex`, `tab:accuracy`.
- Code evidence: `algorithm/run_eval.py`, `algorithm/example_output/accuracy.csv`, `algorithm/example_output/adaptiv_sparsity.csv`, `algorithm/example_output/cmc_sparsity.csv`, `evaluation_scripts/plot_scripts/ipynb_src/table_2.ipynb`.
- Validation status: directly testable from the repo, though full runs are expensive.

### Claim 4: Focus yields speedup, energy gains, and low hardware overhead

- Paper source: `doc/paper/tex/7.evaluation.tex`, `fig:main`, `tab:setup_compare`.
- Code evidence: `simulator/main.py`, `simulator/core/*.py`, `simulator/arch/accelerator.py`, `simulator/example_sim_results/*.csv`, `figure_9.ipynb`.
- Validation status: simulator-backed, not directly observable from algorithm runtime alone.

### Claim 5: Focus supports design-space exploration

- Paper source: `doc/paper/tex/7.evaluation.tex`, `fig:DSE`.
- Code evidence: `algorithm/run_dse.sh`, `simulator/run_dse_sim.sh`, `simulator/main.py`, `figure_10.ipynb`.
- Validation status: strong.

### Claim 6: Focus composes with INT8 quantization and generalizes to image VLMs

- Paper source: `doc/paper/tex/7.evaluation.tex`, `tab:int8_degrade`, `tab:image`.
- Code evidence: `algorithm/run_focus.sh int8`, `simulator/main.py --quantization`, `algorithm/run_focus_image.sh`, `simulator/run_image_sim.sh`, `table_4.ipynb`, `table_5.ipynb`.
- Validation status: strong.

## What The Paper Emphasizes Most

- Direct observation: the paper is contribution-centric. It spends separate sections on SEC and SIC instead of following the code’s folder boundaries.
- Direct observation: `doc/paper/tex/7.evaluation.tex` spends significant space on hardware efficiency, area/power, and DSE, which explains why the simulator is as important as the algorithm code.
- Reasoned inference: the repository is best understood as a co-design artifact. The paper is not “algorithm first, hardware second”; the two are intertwined from the start.

## What The Paper Leaves To The Repository

- Exact CLI shapes and output filenames are not in the paper, but are in `algorithm/run_*.sh`, `simulator/run_*.sh`, and `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`.
- Exact trace serialization details are not in the paper, but are in `algorithm/focus/main.py` and `simulator/models/sparse_info.py`.
- Baseline implementation details are only lightly described in the paper and need code inspection in `algorithm/focus/baseline_*.py` and `simulator/core/simulator.py`.

## Unclear Or Harder-To-Validate Claims

- “First architecture tailored for VLMs” in `doc/paper/tex/1.introduction.tex` is a literature-positioning claim, not a code-verifiable one.
- The exact synthesis flow behind `simulator/arch/*_rtl.csv` is not exposed in the repo.
- The DRAMsim3 wording in `doc/paper/tex/7.evaluation.tex` is stronger than the visible runtime coupling in `simulator/arch/accelerator.py`.
