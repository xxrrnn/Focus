# 01 Mandatory Entry Points

This file lists the real runnable entry points found in the repository. It distinguishes primary entry points from helper scripts and notebook-only entry points.

## Primary Entry Points

| Entry point | Example command from repo | Actual file entered | Why it matters |
| --- | --- | --- | --- |
| Algorithm evaluation / trace export | `cd algorithm && python -m run_eval --model llava_vid --tasks videomme --focus ...` | `algorithm/run_eval.py` | This is the core Python entry for accuracy evaluation, trace generation, and baseline runs. |
| Architecture simulation | `cd simulator && python main.py --model llava_vid --dataset videomme --accelerator focus --trace_dir ../algorithm/output --output_dir results` | `simulator/main.py` | This is the core Python entry for Focus, dense, Adaptiv, CMC, DSE, INT8, and SEC-only simulations. |
| Accelerator area/power comparison | `cd simulator && python arch/accelerator.py --output_dir results` | `simulator/arch/accelerator.py` | This generates the area/power comparison table from RTL-derived CSV statistics and memory models. |
| Worst-case tile-utilization analysis | `cd simulator && python utils/analysis.py --trace_dir ../algorithm/output --output_dir results` | `simulator/utils/analysis.py` | This generates Figure 13 style analysis from Focus traces. |
| Plot generation | `cd evaluation_scripts/plot_scripts/ipynb_src` and run notebooks | `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb` | This is how paper figures/tables are actually assembled. |

## Algorithm-Side Batch Scripts

These are user-facing wrappers around `algorithm/run_eval.py`.

| Script | Underlying entry | Mode | Primary output |
| --- | --- | --- | --- |
| `algorithm/run_focus.sh` | `algorithm/run_eval.py` | Focus video tasks; trace export by default; full accuracy when passed `full`; INT8 when passed `int8` | `focus_main/*.pth`, `meta_data.csv`, or `accuracy.csv` |
| `algorithm/run_original.sh` | `algorithm/run_eval.py` | Dense baseline accuracy | `accuracy.csv` |
| `algorithm/run_framefusion.sh` | `algorithm/run_eval.py` | FrameFusion baseline accuracy | `accuracy.csv` |
| `algorithm/run_adaptiv.sh` | `algorithm/run_eval.py` | Adaptiv sparsity by default; full accuracy with `full` | `adaptiv_sparsity.csv` or `accuracy.csv` |
| `algorithm/run_cmc.sh` | `algorithm/run_eval.py` | CMC sparsity by default; full accuracy with `full` | `cmc_sparsity.csv` or `accuracy.csv` |
| `algorithm/run_dse.sh` | `algorithm/run_eval.py` | Focus design-space sweeps | DSE trace directories or `dse_*_accuracy.csv` |
| `algorithm/run_focus_image.sh` | `algorithm/run_eval.py` | Focus on image-input tasks | `focus_main/*.pth`, `meta_data.csv`, or `accuracy.csv` |
| `algorithm/run_adaptiv_image.sh` | `algorithm/run_eval.py` | Adaptiv on image-input tasks | `adaptiv_sparsity.csv` or `accuracy.csv` |
| `algorithm/run_original_image.sh` | `algorithm/run_eval.py` | Dense image-task accuracy | `accuracy.csv` |

### Why `algorithm/run_eval.py` Is The Real Algorithm Entry

- **Direct observation**: all shell scripts above call `python -m run_eval ...`, which enters `algorithm/run_eval.py`.
- **Direct observation**: `algorithm/run_eval.py` parses Focus-specific flags such as `--focus`, `--export_focus_trace`, `--trace_dir`, `--trace_name`, `--use_median`, `--CMC`, `--adaptiv`, `--frame_fusion`, `--write_accuracy`, `--write_sparsity`, and quantization flags.
- **Direct observation**: `algorithm/run_eval.py` calls `lmms_eval.evaluator.simple_evaluate(...)`, then writes accuracy summaries in `print_results`.
- **Reasoned inference**: the shell scripts are convenience presets; the real control plane is `run_eval.py`.

## Simulator-Side Batch Scripts

These are wrappers around `simulator/main.py`.

| Script | Underlying entry | Mode | Primary output |
| --- | --- | --- | --- |
| `simulator/run_main_sim.sh` | `simulator/main.py` | Main dense / Adaptiv / CMC / Focus comparison | `main_dense.csv`, `main_adaptiv.csv`, `main_cmc.csv`, `main_focus.csv` |
| `simulator/run_dse_sim.sh` | `simulator/main.py` | Focus DSE runs | `dse_a_m_tile_size.csv`, `dse_b_vector_size.csv`, `dse_c_block_size.csv`, `dse_d_num_scatter_accumulator.csv` |
| `simulator/run_image_sim.sh` | `simulator/main.py` | Image-task simulation | image-task rows appended to `main_*.csv` |

### Why `simulator/main.py` Is The Real Simulator Entry

- **Direct observation**: `simulator/main.py` contains the normal `main(args)` path plus DSE paths `dse_m_tile_size`, `dse_block_size`, `dse_vector_size`, `dse_num_scatter`, and `run_quantization`.
- **Direct observation**: it constructs `ModelConfig`, `SparseInfo`, `Accelerator`, and `Simulator`, then calls `simulator.run()`.
- **Reasoned inference**: the batch scripts are just prefilled experiment lists. All actual simulation modes converge in `simulator/main.py`.

## Auxiliary But Important Python Entry Points

| Entry point | Role | Notes |
| --- | --- | --- |
| `algorithm/lmms-eval/lmms_eval/evaluator.py` | Underlying evaluation harness | `algorithm/run_eval.py` delegates here for model loading, task execution, and injection of Focus / CMC / Adaptiv / FrameFusion. |
| `algorithm/focus/interface.py` | Model patching boundary | Not directly user-invoked, but it is the runtime switchboard that rewires supported model families. |
| `simulator/core/simulator.py` | Main simulation engine | Not directly user-invoked, but `simulator/main.py` passes all normal runs through it. |

## Notebook Entry Points

The notebooks are not imported by the simulator. They are the final reporting layer.

| Notebook | Paper artifact type | Main dependencies |
| --- | --- | --- |
| `evaluation_scripts/plot_scripts/ipynb_src/figure_9.ipynb` | Speedup and energy comparison | `main_dense.csv`, `main_adaptiv.csv`, `main_cmc.csv`, `main_focus.csv`, `jetson_stats/figure9_gpu.csv` |
| `evaluation_scripts/plot_scripts/ipynb_src/figure_10.ipynb` | DSE plots | DSE simulator CSVs and DSE accuracy CSVs |
| `evaluation_scripts/plot_scripts/ipynb_src/figure_11.ipynb` | Ablation study | `main_dense.csv`, `main_cmc.csv`, `main_focus_SEC_only.csv`, `main_focus.csv` |
| `evaluation_scripts/plot_scripts/ipynb_src/figure_12.ipynb` | Memory-access analysis | `main_dense.csv`, `main_adaptiv.csv`, `main_cmc.csv`, `main_focus.csv` |
| `evaluation_scripts/plot_scripts/ipynb_src/table_2.ipynb` | Accuracy and sparsity table | `algorithm/example_output/accuracy.csv` and simulator result CSVs |
| `evaluation_scripts/plot_scripts/ipynb_src/table_4.ipynb` | INT8 table | `accuracy.csv`, `main_focus.csv`, `int8_focus.csv` |
| `evaluation_scripts/plot_scripts/ipynb_src/table_5.ipynb` | Image-task table | image-task `accuracy.csv` and simulator result CSVs |

## Command Examples That Matter Most

### Generate Focus traces for simulator input

```bash
cd algorithm
python -m run_eval \
  --model llava_vid \
  --model_args pretrained=lmms-lab/LLaVA-Video-7B-Qwen2,conv_template=qwen_1_5,max_frames_num=64,mm_spatial_pool_mode=average \
  --tasks videomme \
  --focus \
  --batch_size 1 \
  --log_samples \
  --output_path ./logs_traces/ \
  --limit 10 \
  --export_focus_trace \
  --trace_dir ./output/focus_main/ \
  --trace_name llava_vid_videomme \
  --use_median \
  --trace_meta_dir ./output/
```

Actual files involved: `algorithm/run_eval.py`, `algorithm/lmms-eval/lmms_eval/evaluator.py`, `algorithm/focus/interface.py`, `algorithm/focus/main.py`, and the relevant model-specific patch files such as `algorithm/focus/models/qwen2/modeling_qwen2.py` and `algorithm/focus/models/llava_video/modeling_llava_video.py`.

### Run simulator on generated traces

```bash
cd simulator
python main.py \
  --model llava_vid \
  --dataset videomme \
  --accelerator focus \
  --trace_dir ../algorithm/output \
  --output_dir results
```

Actual files involved: `simulator/main.py`, `simulator/models/models.py`, `simulator/models/sparse_info.py`, `simulator/arch/accelerator.py`, `simulator/core/simulator.py`, `simulator/core/simulator_comp.py`, and `simulator/core/simulator_mem.py`.

## Primary Vs Auxiliary

### Primary

- `algorithm/run_eval.py`
- `simulator/main.py`
- `simulator/arch/accelerator.py`
- `simulator/utils/analysis.py`
- `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`

### Auxiliary

- All `algorithm/run_*.sh` scripts
- All `simulator/run_*.sh` scripts
- `algorithm/lmms-eval/lmms_eval/evaluator.py`
- `algorithm/focus/interface.py`
- `simulator/core/simulator.py`

## Important Caveats

- **Direct observation**: the repo’s public examples emphasize the shell scripts, but those scripts are thin wrappers.
- **Direct observation**: `algorithm/lmms-eval/pyproject.toml` also exposes a console script `lmms-eval = lmms_eval.__main__:cli_evaluate`, but this repository’s own experiment scripts do not use that CLI directly.
- **Unclear**: there is no checked-in top-level orchestration script that runs the full paper workflow across algorithm, simulator, and notebooks in one command.
