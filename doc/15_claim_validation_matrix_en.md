# 15 Claim Validation Matrix

This matrix answers a stricter question than “where is this implemented?” It asks: what kind of evidence does the repository actually provide for each paper claim?

| Claim | Paper source | Evidence type | Key repo paths | Validation status | Notes |
| --- | --- | --- | --- | --- | --- |
| Focus implements multilevel concentration | `doc/paper/tex/0.abstract.tex`, `doc/paper/tex/1.introduction.tex` | direct code evidence | `algorithm/focus/main.py`, `algorithm/focus/models/qwen2/modeling_qwen2.py` | supported | SEC and SIC are real runtime behaviors. |
| Semantic pruning is driven by cross-modal attention | `doc/paper/tex/5.semantic.tex` | direct code evidence | `algorithm/focus/main.py::set_token_importance(...)` | supported | Clear one-to-one algorithmic mapping. |
| Block-level and vector-level similarity are both exploited | `doc/paper/tex/6.vector.tex` | direct code evidence plus interpretation | `algorithm/focus/main.py::focus_similarity_concentration(...)` | supported, but structurally merged | Block and vector logic are fused in one function. |
| Focus preserves accuracy with small degradation | `doc/paper/tex/7.evaluation.tex`, `tab:accuracy` | direct experiment path | `algorithm/run_eval.py`, `algorithm/example_output/accuracy.csv`, `table_2.ipynb` | supported | Full reruns are expensive, but the path is present. |
| Focus reaches higher sparsity than AdapTiV, CMC, and FrameFusion | `doc/paper/tex/7.evaluation.tex`, `tab:accuracy` | direct experiment path plus notebook assembly | `algorithm/example_output/accuracy.csv`, `algorithm/example_output/adaptiv_sparsity.csv`, `algorithm/example_output/cmc_sparsity.csv`, `table_2.ipynb` | supported | FrameFusion sparsity is hardcoded as 70 in `table_2.ipynb`. |
| Focus speeds up inference over dense SA, AdapTiV, and CMC | `doc/paper/tex/7.evaluation.tex`, `fig:main` | simulator-backed | `simulator/main.py`, `simulator/core/*.py`, `simulator/example_sim_results/main_*.csv`, `figure_9.ipynb` | supported by simulator | Not directly testable from PyTorch runtime alone. |
| Focus improves energy efficiency | `doc/paper/tex/7.evaluation.tex`, `fig:main` | simulator-backed | `simulator/core/*.py`, `simulator/arch/accelerator.py`, `simulator/example_sim_results/main_*.csv`, `figure_9.ipynb` | supported by simulator | Depends on compute, memory, and area/power assumptions. |
| Focus has only 2.7% area overhead vs systolic array | `doc/paper/tex/7.evaluation.tex`, `tab:setup_compare` | simulator plus precomputed RTL summaries | `simulator/arch/focus_rtl.csv`, `simulator/arch/accelerator.py`, `accelerator_area_power_buffer.csv` | partially supported | The arithmetic is reproducible from the CSVs, but synthesis provenance is incomplete. |
| SEC and SIC contribute only small fractions of total area/power | `doc/paper/tex/7.evaluation.tex`, `fig:main` | simulator plus precomputed RTL summaries | `simulator/example_sim_results/detailed_power_area_breakdown.csv`, `figure_9.ipynb` | partially supported | Same provenance caveat as above. |
| Focus supports DSE over tile size, vector size, block size, and scatter accumulators | `doc/paper/tex/7.evaluation.tex`, `fig:DSE` | direct script chain | `algorithm/run_dse.sh`, `simulator/run_dse_sim.sh`, `figure_10.ipynb` | supported | Strong script-to-figure correspondence. |
| Focus remains effective under INT8 quantization | `doc/paper/tex/7.evaluation.tex`, `tab:int8_degrade` | direct script chain plus simulator | `algorithm/run_focus.sh int8`, `simulator/main.py --quantization`, `table_4.ipynb` | supported | Requires bitsandbytes and INT8-compatible model loading. |
| Focus generalizes to image VLMs | `doc/paper/tex/7.evaluation.tex`, `tab:image` | direct script chain plus simulator | `algorithm/run_focus_image.sh`, `algorithm/run_adaptiv_image.sh`, `simulator/run_image_sim.sh`, `table_5.ipynb` | supported | Qwen2.5-VL path requires a different transformers version. |
| Focus maintains robust utilization in worst/best cases | `doc/paper/tex/7.evaluation.tex`, `fig:worst_case` | direct script chain | `simulator/utils/analysis.py` | supported | The figure is directly generated from traces. |
| Focus is the first architecture tailored for VLMs | `doc/paper/tex/1.introduction.tex`, `doc/paper/tex/7.evaluation.tex` | literature-positioning claim | none inside repo | not directly testable | Requires external literature review, not code inspection. |
| DRAM energy is modeled with DRAMsim3 | `doc/paper/tex/7.evaluation.tex` | weak code evidence | `3rd_party/DRAMsim3`, `simulator/arch/accelerator.py` | unclear / partially supported | The submodule exists, but the normal runtime path uses fixed constants. |

## Reading The Matrix

- `supported` means the repo contains a real runnable or inspectable evidence chain.
- `supported by simulator` means the claim is meaningful but depends on architectural assumptions, not only the PyTorch algorithm path.
- `partially supported` means the numbers are reproducible from checked-in artifacts, but the deepest provenance is incomplete.
- `not directly testable` means the claim is outside what the repository alone can prove.
