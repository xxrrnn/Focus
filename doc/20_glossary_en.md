# 20 Glossary

| Term | Meaning in this repo | Main paths |
| --- | --- | --- |
| Focus | The overall method and architecture proposed in the paper | `doc/paper/main.tex`, `algorithm/focus/`, `simulator/`, `rtl/` |
| Focus Unit | Paper-level hardware block containing SEC and SIC | `doc/paper/tex/4.overview.tex`, `simulator/arch/accelerator.py`, `rtl/README.md` |
| SEC | Semantic Concentrator, the token-pruning path | `doc/paper/tex/5.semantic.tex`, `algorithm/focus/main.py` |
| SIC | Similarity Concentrator, the vector/block similarity path | `doc/paper/tex/6.vector.tex`, `algorithm/focus/main.py`, `simulator/core/` |
| VLM | Vision-Language Model; in this repo mainly LLaVA-Video, LLaVA-OneVision, MiniCPM-V, and Qwen2.5-VL | `algorithm/README.md`, `doc/paper/tex/7.evaluation.tex` |
| Similarity Gather | Paper term for removing redundant vectors and building compact outputs | `doc/paper/tex/6.vector.tex`, `algorithm/focus/main.py` |
| Similarity Scatter | Paper term for reconstructing full outputs from compact vectors using similarity maps | `doc/paper/tex/6.vector.tex`, `simulator/core/simulator_comp.py`, `simulator/core/simulator_mem.py` |
| Convolution-style Layouter | Hardware data-layout idea for conflict-free local block access | `doc/paper/tex/6.vector.tex`, `simulator/arch/accelerator.py` |
| `run_eval.py` | Main algorithm-side driver for evaluation, trace export, and accuracy CSV writing | `algorithm/run_eval.py` |
| `lmms-eval` | Vendored evaluation framework used to run multimodal tasks and collect metrics | `algorithm/lmms-eval/` |
| `selected_layers` | Layers where SEC is applied | `algorithm/focus/configs/focus.csv` |
| `alpha_list` | Per-layer token retention ratios for SEC | `algorithm/focus/configs/focus.csv` |
| `vector_size` | Similarity granularity for SIC and one DSE axis | `algorithm/focus/main.py`, `algorithm/run_dse.sh`, `simulator/main.py` |
| `block_size` | Spatial block extent for local similarity matching | `algorithm/focus/main.py`, `algorithm/run_dse.sh`, `simulator/main.py` |
| `frame_block_size` | Temporal extent for local similarity matching | `algorithm/focus/main.py`, `algorithm/run_dse.sh` |
| `gemm_m_size` | Height of the GEMM tile over which Focus comparisons are restricted | `algorithm/focus/main.py`, `algorithm/run_dse.sh`, `simulator/arch/accelerator.py` |
| `mask_zero` | Focus trace tensor marking zero vectors | `algorithm/focus/main.py`, `simulator/models/sparse_info.py` |
| `mask_similar` | Focus trace tensor marking vectors matched as redundant | `algorithm/focus/main.py`, `simulator/models/sparse_info.py` |
| `group_idx` | Focus trace tensor storing representative/group mapping | `algorithm/focus/main.py`, `simulator/models/sparse_info.py` |
| `meta_data.csv` | Sequence metadata file that the simulator reads before loading traces | `algorithm/focus/main.py`, `simulator/models/models.py` |
| Trace | Serialized sparse behavior saved as `.pth` plus metadata CSVs, bridging algorithm and simulator | `algorithm/focus/main.py`, `simulator/models/sparse_info.py` |
| Adaptiv | Baseline method implemented in `algorithm/focus/baseline_adaptiv.py` and simulated from a coarse sparsity CSV | `algorithm/focus/baseline_adaptiv.py`, `simulator/models/sparse_info.py` |
| CMC | Baseline method implemented in `algorithm/focus/baseline_CMC.py` and simulated from coarse sparsity terms | `algorithm/focus/baseline_CMC.py`, `simulator/models/sparse_info.py` |
| FrameFusion | Baseline integrated through a wrapped implementation in the vendored evaluation stack | `algorithm/lmms-eval/lmms_eval/framefusion/interface.py` |
| `ModelConfig` | Simulator object that defines modeled layer shapes and sequence statistics | `simulator/models/models.py` |
| `SparseInfo` | Simulator object that loads Focus traces or baseline sparsity CSVs | `simulator/models/sparse_info.py` |
| `Accelerator` | Simulator object describing buffers, array shape, components, and power/area sources | `simulator/arch/accelerator.py` |
| `Simulator` | Orchestration layer that drives compute, memory, and energy accounting | `simulator/core/simulator.py` |
| ScaleSim | External GEMM performance simulator used under the Focus simulator | `3rd_party/scalesim`, `simulator/core/simulator_comp.py` |
| CACTI | External SRAM modeling tool used when compiler tables are insufficient | `3rd_party/cacti`, `simulator/memory/cacti.py` |
| DRAMsim3 | Third-party DRAM simulator included as a submodule; in normal runs it is represented more indirectly than explicitly invoked | `3rd_party/DRAMsim3`, `simulator/arch/accelerator.py` |
| `*_rtl.csv` | Precomputed component area/power summaries consumed by the simulator | `simulator/arch/focus_rtl.csv`, `simulator/arch/adaptiv_rtl.csv`, `simulator/arch/cmc_rtl.csv` |
| `main_focus.csv` | Main simulator result table for Focus | `simulator/run_main_sim.sh`, `evaluation_scripts/plot_scripts/ipynb_src/figure_9.ipynb` |
| DSE | Design-space exploration | `doc/paper/tex/7.evaluation.tex`, `algorithm/run_dse.sh`, `simulator/run_dse_sim.sh` |
| `SEC_only` | Ablation mode that keeps semantic pruning and disables SIC | `algorithm/run_eval.py`, `simulator/main.py`, `figure_11.ipynb` |
| INT8 Focus | Focus run with quantized model execution and a separate `focus_int8` trace path | `algorithm/run_focus.sh int8`, `simulator/main.py --quantization` |
