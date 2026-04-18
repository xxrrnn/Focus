# 13 Paper-Guided Code Reading

This guide is for reading the paper and repository side by side. For each paper section, it tells you which code to open next, what to verify, and how strong the mapping is.

## Section-by-Section Reading Map

| Paper section | Paper file | Read next in code | What to verify | Mapping strength |
| --- | --- | --- | --- | --- |
| Abstract and Introduction | `doc/paper/tex/0.abstract.tex`, `doc/paper/tex/1.introduction.tex` | `README.md`, `algorithm/README.md`, `simulator/README.md` | confirm the repo really has algorithm, simulator, RTL, and evaluation layers | strong |
| Motivation | `doc/paper/tex/3.motivation.tex` | `algorithm/focus/main.py`, `simulator/core/simulator_mem.py` | verify that the method targets semantic redundancy plus local vector similarity, not only token pruning | strong |
| Architecture Overview | `doc/paper/tex/4.overview.tex` | `algorithm/focus/main.py`, `simulator/arch/accelerator.py`, `rtl/README.md` | check how the Focus Unit is split between software behavior, architecture model, and RTL blocks | medium |
| Semantic Concentrator | `doc/paper/tex/5.semantic.tex` | `algorithm/focus/main.py`, `algorithm/focus/models/qwen2/modeling_qwen2.py`, `algorithm/focus/models/*/modeling_*.py` | verify importance extraction, top-k selection, retained-token bookkeeping, and where SEC is triggered | strong for algorithm effect, medium for hardware sub-block names |
| Similarity Concentrator | `doc/paper/tex/6.vector.tex` | `algorithm/focus/main.py`, `simulator/core/simulator_comp.py`, `simulator/core/simulator_mem.py`, `rtl/*.v`, `rtl/*.sv` | verify how local candidate blocks, vector grouping, layouter assumptions, and scatter/gather modeling are split across the repo | medium |
| Evaluation Methodology | `doc/paper/tex/7.evaluation.tex` | `algorithm/run_*.sh`, `simulator/run_*.sh`, `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb` | verify that every major figure/table has a real script chain | strong |
| Artifact Appendix | `doc/paper/tex/artifact_appendix.tex` | `README.md`, `algorithm/README.md`, `simulator/README.md`, `algorithm/focus/pyproject.toml`, `algorithm/lmms-eval/requirements.txt` | verify environment, models, datasets, and workflow claims | strong |

## What To Check In Each Pairing

### When reading `doc/paper/tex/5.semantic.tex`

Open:

- `algorithm/focus/main.py`
- `algorithm/focus/models/qwen2/modeling_qwen2.py`
- `algorithm/focus/models/llava_video/modeling_llava_video.py`
- `algorithm/focus/models/minicpmv/modeling_minicpmv.py`
- `algorithm/focus/models/qwen2_5_vl/modeling_qwen2_5_vl.py`

Check:

- `set_token_importance(...)` really uses text-to-image attention.
- `semantic_concentration(...)` really performs top-k retention and updates position/attention metadata.
- `prepare(...)` in model-specific files really computes image-token ranges and frame/patch geometry.

Interpretation:

- Direct observation: the algorithmic effect of SEC is clearly present.
- Reasoned inference: the paper’s analyzer/sorter/encoder names are hardware-facing decompositions, not one-to-one software modules.

### When reading `doc/paper/tex/6.vector.tex`

Open:

- `algorithm/focus/main.py`
- `simulator/core/simulator_comp.py`
- `simulator/core/simulator_mem.py`
- `rtl/cosine_similarity_unit.sv`
- `rtl/inv_magnitude_unit.v`
- `rtl/average_update_unit.sv`

Check:

- `focus_similarity_concentration(...)` really constructs local candidate windows using `block_size`, `frame_block_size`, and `gemm_m_size`.
- the trace fields `mask_zero`, `mask_similar`, and `group_idx` are enough for later simulator-side gather/scatter modeling.
- simulator code has explicit scatter/gather methods such as `run_linear_scatter_focus(...)`, `run_qk_scatter_focus(...)`, and `run_gather_linear_focus(...)`.

Interpretation:

- Direct observation: vector-level grouping is algorithmically real.
- Reasoned inference: layouter and scatter are mainly hardware/simulator concepts in this repo.

### When reading `doc/paper/tex/7.evaluation.tex`

Open:

- `algorithm/run_focus.sh`, `algorithm/run_dse.sh`, `algorithm/run_original.sh`, `algorithm/run_adaptiv.sh`, `algorithm/run_cmc.sh`
- `simulator/run_main_sim.sh`, `simulator/run_dse_sim.sh`, `simulator/run_image_sim.sh`
- `evaluation_scripts/plot_scripts/ipynb_src/figure_9.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/figure_10.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/figure_11.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/figure_12.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_2.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_4.ipynb`
- `evaluation_scripts/plot_scripts/ipynb_src/table_5.ipynb`

Check:

- which results are produced directly by scripts, and which are assembled only in notebooks.
- which claims depend on algorithm runtime, and which depend on simulator outputs.
- where external inputs such as `evaluation_scripts/jetson_stats/figure9_gpu.csv` enter the story.

## Best “Paper Beside Code” Workflow

1. Read the paper section.
2. Open the mapped code files from the table above.
3. Classify each mechanism as `direct observation`, `reasoned inference`, or `unclear`.
4. Follow the actual execution path into scripts and produced files.
5. Only then decide whether the paper claim is fully supported, partially supported, or only simulator-backed.

## Most Educational Pairings

- `doc/paper/tex/5.semantic.tex` with `algorithm/focus/main.py`
- `doc/paper/tex/6.vector.tex` with `algorithm/focus/main.py` plus `simulator/core/simulator_comp.py`
- `doc/paper/tex/7.evaluation.tex` with `algorithm/run_dse.sh`, `simulator/run_dse_sim.sh`, and `figure_10.ipynb`

## Bottom Line

- Direct observation: the strongest paper-to-code reading experience comes from the algorithm files and the DSE/evaluation script chain.
- Reasoned inference: the weakest experience comes from the hardware narrative, because the paper presents a cleaner architectural decomposition than the checked-in RTL artifacts do.
