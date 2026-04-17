# 00 Repo Overview

## What Problem This Project Solves

- **Direct observation**: `README.md` describes Focus as a hardware-algorithm co-design for Vision-Language Model inference that removes redundancy in visual tokens using multilevel concentration.
- **Direct observation**: the repository explicitly covers four implementation layers: model-side algorithm instrumentation in `algorithm/`, accelerator/performance modeling in `simulator/`, hardware blocks in `rtl/`, and result processing in `evaluation_scripts/`.
- **Reasoned inference**: the repository is structured to study one research idea at multiple abstraction levels: modify real VLM execution, capture sparse behavior, project that behavior onto an accelerator model, and then map the accelerator back to synthesized RTL blocks for area/power reporting.

## What The Repository Contains

### Trimmed Directory Tree

```text
.
├── README.md
├── .gitmodules
├── 3rd_party/
│   ├── LLaVA-NeXT/
│   ├── scalesim/
│   ├── cacti/
│   └── DRAMsim3/
├── algorithm/
│   ├── README.md
│   ├── run_eval.py
│   ├── run_focus.sh
│   ├── run_original.sh
│   ├── run_framefusion.sh
│   ├── run_adaptiv.sh
│   ├── run_cmc.sh
│   ├── run_dse.sh
│   ├── run_focus_image.sh
│   ├── run_adaptiv_image.sh
│   ├── run_original_image.sh
│   ├── example_output/
│   ├── focus/
│   │   ├── pyproject.toml
│   │   ├── interface.py
│   │   ├── main.py
│   │   ├── baseline_adaptiv.py
│   │   ├── baseline_CMC.py
│   │   ├── utils.py
│   │   ├── configs/
│   │   │   ├── focus.csv
│   │   │   ├── adaptiv.csv
│   │   │   └── cmc.csv
│   │   └── models/
│   │       ├── qwen2/modeling_qwen2.py
│   │       ├── qwen2_5_vl/modeling_qwen2_5_vl.py
│   │       ├── llava_video/modeling_llava_video.py
│   │       └── minicpmv/modeling_minicpmv.py
│   └── lmms-eval/
│       ├── pyproject.toml
│       ├── requirements.txt
│       ├── lmms_eval/evaluator.py
│       └── lmms_eval/models/
├── simulator/
│   ├── README.md
│   ├── main.py
│   ├── run_main_sim.sh
│   ├── run_dse_sim.sh
│   ├── run_image_sim.sh
│   ├── example_sim_results/
│   ├── arch/
│   │   ├── accelerator.py
│   │   ├── focus_rtl.csv
│   │   ├── adaptiv_rtl.csv
│   │   └── cmc_rtl.csv
│   ├── core/
│   │   ├── simulator.py
│   │   ├── simulator_comp.py
│   │   ├── simulator_mem.py
│   │   └── scalesim_cfg/
│   ├── memory/
│   │   ├── buffer.py
│   │   ├── cacti.py
│   │   ├── buffer_model_spec.csv
│   │   └── sram_config.json
│   ├── models/
│   │   ├── models.py
│   │   └── sparse_info.py
│   └── utils/
│       ├── utils.py
│       └── analysis.py
├── rtl/
│   ├── README.md
│   ├── traditional_systolic.v
│   ├── traditional_mac.v
│   ├── cosine_similarity_unit.sv
│   ├── inv_magnitude_unit.v
│   ├── max_unit_fp16.v
│   ├── average_update_unit.sv
│   ├── adaptiv_array.v
│   ├── cmc_codec_pe.v
│   ├── cmc_addertree_4to1.v
│   ├── fp16_exp.v
│   ├── fp16_recip.v
│   ├── fp16_mult.v
│   ├── fp16_add.v
│   ├── fp32_add.v
│   └── fastinvsqrt_fp32.v
└── evaluation_scripts/
    ├── jetson_stats/figure9_gpu.csv
    └── plot_scripts/ipynb_src/
        ├── figure_9.ipynb
        ├── figure_10.ipynb
        ├── figure_11.ipynb
        ├── figure_12.ipynb
        ├── table_2.ipynb
        ├── table_4.ipynb
        └── table_5.ipynb
```

## What Each Important Directory Does

### `algorithm/`

- **Direct observation**: `algorithm/run_eval.py` is the actual Python entry point for model evaluation, trace export, and accuracy CSV generation.
- **Direct observation**: `algorithm/focus/interface.py` patches supported VLMs so that Focus, Adaptiv, or CMC logic executes inside real model forward passes.
- **Direct observation**: `algorithm/focus/main.py` implements the Focus algorithm itself, including semantic concentration, vector/block similarity concentration, trace capture, and median-sample selection.
- **Direct observation**: `algorithm/lmms-eval/` is a vendored evaluation framework. Focus integrates into it through `algorithm/lmms-eval/lmms_eval/evaluator.py`.

### `simulator/`

- **Direct observation**: `simulator/main.py` is the user-facing simulator entry point.
- **Direct observation**: `simulator/models/sparse_info.py` is the bridge from algorithm outputs into simulator inputs.
- **Direct observation**: `simulator/core/simulator.py`, `simulator/core/simulator_comp.py`, and `simulator/core/simulator_mem.py` split top-level orchestration, compute modeling, and memory modeling.
- **Reasoned inference**: the simulator is not coupled to the Python model code itself; it consumes serialized sparsity/trace artifacts so that hardware studies can be repeated without re-running the VLM.

### `rtl/`

- **Direct observation**: `rtl/` contains module-level hardware blocks, not a full integrated accelerator top.
- **Direct observation**: `rtl/README.md` groups the modules into Focus blocks, dense systolic-array blocks, Adaptiv blocks, and CMC blocks.
- **Reasoned inference**: the RTL exists primarily to support synthesized area/power numbers, which are then fed back into the simulator via `simulator/arch/focus_rtl.csv`, `simulator/arch/adaptiv_rtl.csv`, and `simulator/arch/cmc_rtl.csv`.

### `evaluation_scripts/`

- **Direct observation**: the plotting layer is notebook-driven, not script-driven.
- **Direct observation**: the notebooks consume CSV outputs from `algorithm/` and `simulator/`, plus one external measurement CSV in `evaluation_scripts/jetson_stats/figure9_gpu.csv`.
- **Reasoned inference**: plotting is intentionally kept downstream of the core implementation so paper figures can be changed without touching algorithm or simulator logic.

### `3rd_party/`

- **Direct observation**: `.gitmodules` registers `3rd_party/scalesim`, `3rd_party/cacti`, `3rd_party/DRAMsim3`, and `3rd_party/LLaVA-NeXT`.
- **Direct observation**: `algorithm/lmms-eval/lmms_eval/models/llava_vid.py` and `algorithm/lmms-eval/lmms_eval/models/llava_onevision.py` import `llava.*`, so the code depends on `3rd_party/LLaVA-NeXT`.
- **Direct observation**: `simulator/core/simulator_comp.py` imports `scalesim.scale_sim.scalesim`, so the simulator directly depends on the `scalesim` submodule.
- **Direct observation**: `simulator/memory/cacti.py` expects a compiled binary at `3rd_party/cacti/cacti`.
- **Reasoned inference**: `3rd_party/DRAMsim3` is much looser. The current simulator uses a DRAM energy constant in `simulator/arch/accelerator.py` and does not directly invoke the DRAMsim3 binary during normal runs.

## How The Major Subsystems Connect

### Algorithm To Simulator

- **Direct observation**: Focus trace export happens in `algorithm/focus/main.py` inside `Focus.post_process`, which writes a `.pth` trace file to `trace_dir` and metadata to `trace_meta_dir/meta_data.csv`.
- **Direct observation**: baseline sparsity export happens in `algorithm/focus/baseline_adaptiv.py` and `algorithm/focus/baseline_CMC.py`, which write `adaptiv_sparsity.csv` and `cmc_sparsity.csv`.
- **Direct observation**: `simulator/models/models.py` reads `meta_data.csv`, and `simulator/models/sparse_info.py` reads either `focus_main/*.pth` or the baseline sparsity CSVs.
- **Reasoned inference**: the repository’s central handoff format is the serialized sparse behavior, not an in-memory API.

### Simulator To Plotting

- **Direct observation**: `simulator/main.py` writes `main_focus.csv`, `main_dense.csv`, `main_adaptiv.csv`, `main_cmc.csv`, DSE CSVs, and optional breakdown CSVs.
- **Direct observation**: the notebooks in `evaluation_scripts/plot_scripts/ipynb_src/` read those CSVs directly.
- **Reasoned inference**: the plotting layer assumes a stable CSV contract and does not recompute simulator internals.

### RTL To Simulator

- **Direct observation**: `simulator/arch/accelerator.py` loads component area/power numbers from `simulator/arch/focus_rtl.csv`, `simulator/arch/adaptiv_rtl.csv`, and `simulator/arch/cmc_rtl.csv`.
- **Direct observation**: `rtl/README.md` names the hardware blocks that correspond to those component categories.
- **Reasoned inference**: the simulator uses RTL-derived synthesis summaries, not direct RTL simulation.

## Why The Repository May Be Organized This Way

- **Reasoned inference**: the project separates expensive, accuracy-critical model execution from fast architectural evaluation. That is a good fit for research code where the same sparse behavior must be replayed under many hardware assumptions.
- **Reasoned inference**: monkeypatching model forwards in `algorithm/focus/interface.py` lets the authors reuse upstream model implementations while still inserting Focus at exactly the GEMM/attention sites they care about.
- **Reasoned inference**: encoding RTL results as CSV summaries in `simulator/arch/*.csv` keeps the simulator runnable without requiring proprietary EDA tools during normal use.

## What Is Still Unclear At Repo Level

- **Unclear**: there is no single script that chains `algorithm/` to `simulator/` to `evaluation_scripts/` automatically.
- **Unclear**: the repository claims a “cycle-accurate simulator” in `simulator/README.md`, but the code implements a hybrid of ScaleSim calls, closed-form scatter/gather formulas, and memory accounting rather than direct RTL co-simulation.
- **Unclear**: the precise provenance of the `*_rtl.csv` files is not reproducible from the repo alone because synthesis scripts are not included in `rtl/`.
