# 03 Execution Map

## Why This Map Matters

The paper narrative in `doc/paper/tex/5.semantic.tex`, `doc/paper/tex/6.vector.tex`, and `doc/paper/tex/7.evaluation.tex` becomes concrete only when followed through the actual entry paths.

## Flow 1: Focus Trace Generation

| Step | File path | What happens |
| --- | --- | --- |
| 1 | `algorithm/run_eval.py` | parse Focus CLI flags |
| 2 | `algorithm/lmms-eval/lmms_eval/evaluator.py` | load model and task |
| 3 | `algorithm/focus/interface.py` | patch supported model with Focus-aware forwards |
| 4 | model-specific files such as `algorithm/focus/models/llava_video/modeling_llava_video.py` | compute patch/frame/token metadata and call `self.focus.prepare(...)` |
| 5 | `algorithm/focus/models/qwen2/modeling_qwen2.py` | invoke SIC and SEC during decoder execution |
| 6 | `algorithm/focus/main.py` | collect sparsity stats and save median trace |

### Artifacts

- `focus_main/<model>_<dataset>.pth`
- `meta_data.csv`

### Paper alignment

- SEC: `doc/paper/tex/5.semantic.tex`
- SIC: `doc/paper/tex/6.vector.tex`

## Flow 2: Accuracy Evaluation

| Step | File path | What happens |
| --- | --- | --- |
| 1 | `algorithm/run_*.sh` or direct CLI | launch task/model combinations |
| 2 | `algorithm/run_eval.py` | run eval through `lmms-eval` |
| 3 | `algorithm/run_eval.py::get_score_from_results(...)` | normalize task-specific metrics |
| 4 | `algorithm/run_eval.py::save_score_to_main_csv(...)` or `save_score_to_dse_csv(...)` | write paper-facing CSV tables |

### Artifacts

- `accuracy.csv`
- `dse_a_m_tile_accuracy.csv`
- `dse_b_vector_accuracy.csv`
- `dse_c_block_accuracy.csv`

### Paper alignment

- `doc/paper/tex/7.evaluation.tex` accuracy, sparsity, INT8, and image-model tables

## Flow 3: Simulator

| Step | File path | What happens |
| --- | --- | --- |
| 1 | `simulator/main.py` | parse simulation mode |
| 2 | `simulator/models/models.py` | load sequence statistics from `meta_data.csv` |
| 3 | `simulator/models/sparse_info.py` | load Focus trace or baseline sparsity CSV |
| 4 | `simulator/arch/accelerator.py` | configure hardware model |
| 5 | `simulator/core/simulator.py` | aggregate cycles, memory, and energy |
| 6 | `simulator/utils/utils.py` | append result row to CSV |

### Artifacts

- `main_focus.csv`
- `main_dense.csv`
- `main_adaptiv.csv`
- `main_cmc.csv`
- `main_focus_SEC_only.csv`
- `int8_focus.csv`
- `dse_*.csv`

### Paper alignment

- `doc/paper/tex/7.evaluation.tex` performance, energy, DSE, memory access, SEC-only ablation, INT8, image-VLM discussion

## Flow 4: Plotting / Tables

| Paper artifact | Real producer |
| --- | --- |
| Main performance and energy figure | `evaluation_scripts/plot_scripts/ipynb_src/figure_9.ipynb` + simulator CSVs + `jetson_stats/figure9_gpu.csv` |
| DSE figure | `figure_10.ipynb` + DSE CSVs |
| Ablation figure | `figure_11.ipynb` + `main_focus_SEC_only.csv` |
| Memory-access figure | `figure_12.ipynb` |
| Accuracy/sparsity table | `table_2.ipynb` |
| INT8 table | `table_4.ipynb` |
| Image-VLM table | `table_5.ipynb` |
| Worst-case histogram | `simulator/utils/analysis.py` |

## Flow 5: RTL-Derived Area / Power

| Step | File path | What happens |
| --- | --- | --- |
| 1 | `rtl/*.v`, `rtl/*.sv` | hardware blocks exist as source |
| 2 | `simulator/arch/*_rtl.csv` | precomputed component area/power tables |
| 3 | `simulator/arch/accelerator.py` | load component stats and compose accelerator totals |

### Important caveat

- **Direct observation**: the runtime simulator does not execute RTL.
- **Reasoned inference**: the repo uses “RTL-derived statistics in simulator,” not integrated RTL co-simulation.

## Central Artifact Handoffs

| Producer | Artifact | Consumer |
| --- | --- | --- |
| `algorithm/focus/main.py` | Focus `.pth` trace | `simulator/models/sparse_info.py` |
| `algorithm/focus/main.py` | `meta_data.csv` | `simulator/models/models.py` |
| `algorithm/focus/baseline_adaptiv.py` | `adaptiv_sparsity.csv` | `simulator/models/sparse_info.py` |
| `algorithm/focus/baseline_CMC.py` | `cmc_sparsity.csv` | `simulator/models/sparse_info.py` |
| `simulator/main.py` | `main_*.csv`, `dse_*.csv` | plotting notebooks |

## Bottom Line

- **Direct observation**: the real project pipeline is `run_eval.py -> trace/CSV artifacts -> simulator/main.py -> plotting notebooks`.
- **Reasoned inference**: this is the correct mental model for reading the repo alongside the paper.
