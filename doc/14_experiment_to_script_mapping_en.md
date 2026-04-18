# 14 Experiment-to-Script Mapping

This document maps each major paper experiment to the real script chain in the repository.

## Main Mapping Table

| Paper artifact | Paper source | Producer scripts / commands | Intermediate artifacts | Final consumer | Traceability |
| --- | --- | --- | --- | --- | --- |
| `tab:accuracy` Accuracy and computation sparsity | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_original.sh full`, `sh algorithm/run_focus.sh full`, `sh algorithm/run_framefusion.sh full`, `sh algorithm/run_adaptiv.sh full`, `sh algorithm/run_cmc.sh full` | `algorithm/output/accuracy.csv`, `algorithm/output/adaptiv_sparsity.csv`, `algorithm/output/cmc_sparsity.csv`, simulator `main_*.csv` | `evaluation_scripts/plot_scripts/ipynb_src/table_2.ipynb` | strong |
| `fig:main` Speedup / energy / area-power | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_focus.sh`, `sh algorithm/run_adaptiv.sh`, `sh algorithm/run_cmc.sh`, `sh simulator/run_main_sim.sh`, `python simulator/arch/accelerator.py --output_dir ...` | `main_dense.csv`, `main_adaptiv.csv`, `main_cmc.csv`, `main_focus.csv`, `detailed_power_area_breakdown.csv`, `accelerator_area_power_buffer.csv`, `evaluation_scripts/jetson_stats/figure9_gpu.csv` | `figure_9.ipynb` | strong for CSV chain, partial for external Jetson data |
| `tab:hardware-config` Architecture setup | `doc/paper/tex/7.evaluation.tex` | no single generator found | settings are spread across `simulator/arch/accelerator.py`, `algorithm/focus/configs/focus.csv`, `simulator/core/scalesim_cfg/config.cfg`, and paper-authored table values | none | partial |
| `tab:setup_compare` Architecture comparison table | `doc/paper/tex/7.evaluation.tex` | `python simulator/arch/accelerator.py --output_dir ...` | `accelerator_area_power_buffer.csv` | notebook-free or manual table formatting | strong for values, partial for exact paper formatting |
| `fig:DSE` Design-space exploration | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_dse.sh`, optionally `sh algorithm/run_dse.sh accuracy`, then `sh simulator/run_dse_sim.sh` | `dse_a_m_tile_accuracy.csv`, `dse_b_vector_accuracy.csv`, `dse_c_block_accuracy.csv`, `dse_a_m_tile_size.csv`, `dse_b_vector_size.csv`, `dse_c_block_size.csv`, `dse_d_num_scatter_accumulator.csv` | `figure_10.ipynb` | strong |
| `fig:ablation` SEC-only ablation | `doc/paper/tex/7.evaluation.tex` | `python simulator/main.py --all_models_datasets --accelerator focus --SEC_only --trace_dir ... --output_dir ...` plus main dense/focus/CMC runs | `main_focus_SEC_only.csv`, `main_dense.csv`, `main_cmc.csv`, `main_focus.csv` | `figure_11.ipynb` | strong |
| `fig:mem_access` Memory-access analysis | `doc/paper/tex/7.evaluation.tex` | `sh simulator/run_main_sim.sh` | `main_dense.csv`, `main_adaptiv.csv`, `main_cmc.csv`, `main_focus.csv` with memory/access fields | `figure_12.ipynb` | strong |
| `tab:int8_degrade` Quantization synergy | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_focus.sh int8 full`, `sh algorithm/run_original.sh int8 full`, `python simulator/main.py --all_models_datasets --accelerator focus --quantization --trace_dir ... --output_dir ...` | `accuracy.csv` with `INT8_Dense` and `INT8_Focus`, `int8_focus.csv` | `table_4.ipynb` | strong |
| `tab:image` Image-VLM generalization | `doc/paper/tex/7.evaluation.tex` | `sh algorithm/run_original_image.sh accuracy`, `sh algorithm/run_focus_image.sh accuracy`, `sh algorithm/run_adaptiv_image.sh accuracy`, plus `sh algorithm/run_focus_image.sh`, `sh algorithm/run_adaptiv_image.sh`, `sh simulator/run_image_sim.sh` | image-task `accuracy.csv`, image-task `main_dense.csv`, `main_adaptiv.csv`, `main_focus.csv` | `table_5.ipynb` | strong |
| `fig:worst_case` Worst-/best-case analysis | `doc/paper/tex/7.evaluation.tex` | `python simulator/utils/analysis.py --trace_dir ... --output_dir ...` | Focus traces and generated histogram/utilization data | direct SVG output | strong |

## Script Chains By Theme

### Main paper results

1. Generate Focus and baseline traces with `algorithm/run_focus.sh`, `algorithm/run_adaptiv.sh`, and `algorithm/run_cmc.sh`.
2. Run hardware simulation with `simulator/run_main_sim.sh`.
3. Run area/power summarization with `python simulator/arch/accelerator.py --output_dir ...`.
4. Open `evaluation_scripts/plot_scripts/ipynb_src/figure_9.ipynb` and `table_2.ipynb`.

### DSE

1. Generate Focus DSE traces with `algorithm/run_dse.sh`.
2. If you want the paper’s rough accuracy overlays, run `algorithm/run_dse.sh accuracy`.
3. Run simulation with `simulator/run_dse_sim.sh`.
4. Open `figure_10.ipynb`.

### Quantization and image tasks

1. Use `algorithm/run_focus.sh int8` or image-task scripts in `algorithm/`.
2. Use `simulator/main.py --quantization` or `simulator/run_image_sim.sh`.
3. Open `table_4.ipynb` or `table_5.ipynb`.

## Which Figures/Tables Are Easiest To Reproduce

- `fig:DSE`
- `fig:ablation`
- `fig:mem_access`
- `tab:int8_degrade`
- `tab:image`

They have the cleanest script-to-notebook chain.

## Which Figures/Tables Are Harder To Trace Exactly

- `tab:hardware-config`: no single generator; it is a paper-authored summary over multiple code defaults.
- `fig:main`: the main CSV chain is clear, but the GPU comparison depends on `evaluation_scripts/jetson_stats/figure9_gpu.csv`.
- `tab:setup_compare`: the values are traceable through `accelerator_area_power_buffer.csv`, but the final paper table layout is not emitted by a dedicated script.

## Bottom Line

- Direct observation: the experiment pipeline is highly scriptable.
- Reasoned inference: the project separates result production from figure styling well, but some paper tables still involve manual or notebook-level formatting.
