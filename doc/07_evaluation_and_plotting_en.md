# 07 Evaluation And Plotting

## Paper Anchor

Nearly all experiment narrative lives in `doc/paper/tex/7.evaluation.tex`.

## Figure / Table Mapping

| Paper artifact | Paper source | Real producer | Status |
| --- | --- | --- | --- |
| accuracy + computation sparsity table | `doc/paper/tex/7.evaluation.tex`, `tab:accuracy` | `algorithm/run_eval.py` + `table_2.ipynb` + simulator CSVs | strong |
| main speedup / energy / area-power figure | `fig:main` in `doc/paper/tex/7.evaluation.tex` | `figure_9.ipynb` + `detailed_power_area_breakdown.csv` + `jetson_stats/figure9_gpu.csv` | strong |
| architecture setup table | `tab:hardware-config` | mostly `simulator/arch/accelerator.py` config and paper text | partial |
| architecture comparison table | `tab:setup_compare` | `simulator/arch/accelerator.py` + `accelerator_area_power_buffer.csv` | strong for numbers, partial for exact paper formatting |
| DSE figure | `fig:DSE` | `algorithm/run_dse.sh`, `simulator/run_dse_sim.sh`, `figure_10.ipynb` | strong |
| ablation figure | `fig:ablation` | `simulator/main.py --SEC_only`, `figure_11.ipynb` | strong |
| memory-access figure | `fig:mem_access` | `figure_12.ipynb` | strong |
| INT8 table | `tab:int8_degrade` | `algorithm/run_focus.sh int8`, `simulator/main.py --quantization`, `table_4.ipynb` | strong |
| image-VLM table | `tab:image` | image scripts + `run_image_sim.sh` + `table_5.ipynb` | strong |
| worst-case figure | `fig:worst_case` | `simulator/utils/analysis.py` | strong |

## Easiest Paper Artifacts To Trace

- `fig:DSE`
- `fig:ablation`
- `fig:mem_access`
- `tab:int8_degrade`
- `tab:image`

These have very direct script/notebook paths.

## Harder Or Weaker Cases

- `tab:hardware-config`: mostly a paper-authored summary of architectural defaults, not a single generated artifact
- `fig:main` right-hand breakdown: reproducible from `detailed_power_area_breakdown.csv`, but final paper styling is notebook-driven
- `tab:setup_compare`: numbers are traceable, but the final paper table layout is not emitted by one dedicated script

## Evaluation Workflow

1. Run algorithm scripts in `algorithm/` to generate traces and/or accuracy CSVs.
2. Run simulator scripts in `simulator/` to generate result CSVs.
3. Open notebooks in `evaluation_scripts/plot_scripts/ipynb_src/`.
4. Point notebooks at generated output directories or example directories.

## Direct Paper-To-Notebook Links

- `figure_9.ipynb` reads `main_dense.csv`, `main_adaptiv.csv`, `main_cmc.csv`, `main_focus.csv`, and `figure9_gpu.csv`.
- `figure_10.ipynb` reads DSE simulator CSVs and DSE accuracy CSVs.
- `figure_11.ipynb` reads `main_dense.csv`, `main_cmc.csv`, `main_focus_SEC_only.csv`, `main_focus.csv`.
- `figure_12.ipynb` reads DRAM/access-related simulator CSV fields.
- `table_2.ipynb` reads `accuracy.csv` and simulator outputs.
- `table_4.ipynb` reads `accuracy.csv`, `main_focus.csv`, and `int8_focus.csv`.
- `table_5.ipynb` reads image-task accuracy plus image-task simulator outputs.

## Bottom Line

- **Direct observation**: the plotting layer is downstream-only; it consumes CSVs and does not rerun method logic.
- **Reasoned inference**: this is a good engineering choice because it decouples paper presentation from the implementation core.
