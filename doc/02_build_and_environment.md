# 02 Build And Environment

## Dependency Structure

### Layered View

| Layer | Files to inspect | Purpose |
| --- | --- | --- |
| Top-level setup guidance | `README.md`, `.gitmodules` | Describes clone, submodule init, editable installs, and native builds. |
| Focus package | `algorithm/focus/pyproject.toml` | Defines the `focus` package and pins transformer versions through extras. |
| Evaluation framework | `algorithm/lmms-eval/pyproject.toml`, `algorithm/lmms-eval/requirements.txt`, `algorithm/lmms-eval/setup.py` | Defines the evaluation harness and its heavy multimodal dependencies. |
| Native SRAM model | `3rd_party/cacti`, `simulator/memory/cacti.py`, `simulator/memory/sram_config.json` | Used for adjustable SRAM area/energy evaluation. |
| GEMM performance model | `3rd_party/scalesim`, `simulator/core/simulator_comp.py`, `simulator/core/scalesim_cfg/*` | Used for systolic-array cycle estimation. |
| External VLM implementation | `3rd_party/LLaVA-NeXT`, `algorithm/lmms-eval/lmms_eval/models/llava_vid.py`, `algorithm/lmms-eval/lmms_eval/models/llava_onevision.py` | Provides the `llava.*` code imported by the evaluation adapters. |

### Submodules

`.gitmodules` registers the following:

| Submodule | Used by this repo | Notes |
| --- | --- | --- |
| `3rd_party/scalesim` | Yes | Directly imported in `simulator/core/simulator_comp.py`. |
| `3rd_party/cacti` | Yes | Binary is expected by `simulator/memory/cacti.py`. |
| `3rd_party/DRAMsim3` | Indirectly only | Current simulator uses a DRAM energy constant in `simulator/arch/accelerator.py`; no normal run path calls DRAMsim3 directly. |
| `3rd_party/LLaVA-NeXT` | Yes | Required by `llava_vid` and `llava_onevision` model adapters in `algorithm/lmms-eval/lmms_eval/models/`. |

## Baseline Environment Setup

The repository’s intended installation flow is in `README.md`:

```bash
git submodule init
git submodule update

conda create -n focus python=3.11 -y
conda activate focus

cd 3rd_party/LLaVA-NeXT && pip install -e .
cd ../scalesim && pip install -e .
cd ../cacti && make
cd ../DRAMsim3 && make
cd ../../algorithm/lmms-eval && pip install -e .
cd ../focus && pip install -e '.[main]'
```

### What Each Install Step Enables

- **Direct observation**: `pip install -e .` in `3rd_party/LLaVA-NeXT` is necessary because `algorithm/lmms-eval/lmms_eval/models/llava_vid.py` and `algorithm/lmms-eval/lmms_eval/models/llava_onevision.py` import `llava.*`.
- **Direct observation**: `pip install -e .` in `3rd_party/scalesim` is necessary because `simulator/core/simulator_comp.py` imports `scalesim.scale_sim.scalesim`.
- **Direct observation**: `make` in `3rd_party/cacti` is necessary because `simulator/memory/cacti.py` looks for the compiled binary `3rd_party/cacti/cacti`.
- **Direct observation**: `pip install -e .` in `algorithm/lmms-eval` is necessary because `algorithm/run_eval.py` imports `lmms_eval`.
- **Direct observation**: `pip install -e '.[main]'` in `algorithm/focus` is necessary because `algorithm/run_eval.py` and `algorithm/focus/interface.py` import the local `focus` package.
- **Reasoned inference**: `make` in `3rd_party/DRAMsim3` is mainly for completeness or future extension. The checked-in simulator does not directly invoke the DRAMsim3 executable during the normal experiment scripts.

## Version-Sensitive Components

### Python And Core Libraries

- **Direct observation**: `README.md` recommends Python 3.11.
- **Direct observation**: `algorithm/focus/pyproject.toml` requires Python `>=3.11`.
- **Direct observation**: `algorithm/lmms-eval/pyproject.toml` allows Python `>=3.8`, but the top-level repo guidance and Focus package still push the effective environment to Python 3.11.

### Transformers

- **Direct observation**: `algorithm/focus/pyproject.toml` depends on `transformers>=4.48.2,<4.50.0`.
- **Direct observation**: the `main` extra in `algorithm/focus/pyproject.toml` pins `transformers==4.48.2`.
- **Direct observation**: the `qwen25_vl` extra pins `transformers==4.49.0`.
- **Direct observation**: `algorithm/README.md` explicitly tells the user to switch to `pip install -e '.[qwen25_vl]'` before Qwen2.5-VL evaluation, then switch back to `.[main]` afterward.
- **Reasoned inference**: this is one of the most fragile parts of the environment. Video-model support and Qwen2.5-VL support are not meant to coexist under one stable pinned transformers version.

### Heavy Multimodal Dependencies

- **Direct observation**: `algorithm/lmms-eval/requirements.txt` includes `decord`, `av`, `datasets`, `bitsandbytes`, `flash_attn`, and many CUDA packages.
- **Direct observation**: the model adapters in `algorithm/lmms-eval/lmms_eval/models/` use `decord`, `PIL`, Hugging Face model loading, and optional BitsAndBytes quantization wrappers.
- **Reasoned inference**: algorithm evaluation is GPU-heavy and environment-heavy, while the simulator can run with far fewer dependencies once artifacts already exist.

## What Is Needed For Each Subsystem

### Algorithm

- Python 3.11 environment.
- Editable installs for `3rd_party/LLaVA-NeXT`, `algorithm/lmms-eval`, and `algorithm/focus`.
- Hugging Face model access and dataset downloads.
- CUDA GPU. `README.md` recommends a GPU with at least 80 GB HBM.
- `decord`, `av`, `datasets`, and related multimodal dependencies from `algorithm/lmms-eval`.
- Optional BitsAndBytes support for `--load_in_8bit` or `--load_in_4bit`, as wired in `algorithm/run_eval.py` and the model adapters.

### Simulator

- Python environment with the local `scalesim` package installed.
- Compiled CACTI binary if you want DSE or any run path that falls back to CACTI, via `simulator/memory/cacti.py`.
- Trace artifacts from `algorithm/` unless using the checked-in examples under `algorithm/example_output` and `simulator/example_sim_results`.
- No GPU requirement is obvious in the simulator code paths themselves.

### RTL

- **Direct observation**: `rtl/README.md` says the modules were synthesized using Synopsys Design Compiler.
- **Unclear**: no synthesis scripts, no Makefile, and no testbench flow are checked into `rtl/`.
- **Reasoned inference**: to reproduce the exact area/power extraction process from scratch, you would need external EDA infrastructure and missing synthesis glue not provided here.

### Plotting And Paper Reproduction

- Jupyter environment with pandas and matplotlib.
- CSV outputs from `algorithm/` and `simulator/`.
- Optional use of the example outputs already stored in `algorithm/example_output` and `simulator/example_sim_results`.

## Likely Failure Points

| Risk | Where it comes from | Why it matters |
| --- | --- | --- |
| Missing submodules | `.gitmodules`, `README.md` | Model loading, ScaleSim, and CACTI all depend on them. |
| Wrong transformers version | `algorithm/focus/pyproject.toml`, `algorithm/README.md` | Qwen2.5-VL support needs a different pin than the main video setup. |
| CACTI not built | `simulator/memory/cacti.py` | DSE and any non-default buffer evaluation path can fail or fall back unexpectedly. |
| Running simulator jobs in parallel | `simulator/README.md`, `simulator/core/simulator_comp.py` | ScaleSim config files and logs are shared and mutated in place. |
| Huge accuracy runtime | `algorithm/README.md` | Full accuracy runs are measured in tens to hundreds of GPU hours. |
| Hugging Face access/download issues | `README.md`, model adapter files | Models and datasets are fetched dynamically. |
| Baseline environment switching | `algorithm/README.md` | Qwen2.5-VL image-task setup changes package versions mid-workflow. |
| Inconsistent output locations | shell scripts and notebooks | The notebooks assume specific CSV filenames and directory layouts. |

## Components That Are Cleanly Separated

- **Direct observation**: model evaluation and hardware simulation are separated by file artifacts, not by shared in-memory objects. See `algorithm/focus/main.py`, `algorithm/focus/baseline_*.py`, `simulator/models/models.py`, and `simulator/models/sparse_info.py`.
- **Direct observation**: RTL power/area data is imported as CSV summaries in `simulator/arch/*.csv`, so the simulator does not require RTL simulation tools for ordinary use.
- **Reasoned inference**: this is a strong research-engineering pattern. It keeps the expensive and proprietary parts of the workflow loosely coupled.

## Components That Are Environment Fragile

- **Direct observation**: `algorithm/lmms-eval/requirements.txt` is large and includes many pinned CUDA-sensitive packages.
- **Direct observation**: the repository asks the user to swap Focus extras when switching between main models and Qwen2.5-VL.
- **Direct observation**: `simulator/core/simulator_comp.py` rewrites `simulator/core/scalesim_cfg/config.cfg` and `simulator/core/scalesim_cfg/gemm.csv` in place before each ScaleSim invocation.
- **Reasoned inference**: the algorithm stack is the fragile part; the simulator is simpler once its Python and CACTI/ScaleSim dependencies are working.

## Honest Bottom Line

- **Direct observation**: the repository documents installation reasonably well in `README.md`, but the real dependency picture is split across top-level instructions, `algorithm/focus/pyproject.toml`, `algorithm/lmms-eval/pyproject.toml`, and `algorithm/lmms-eval/requirements.txt`.
- **Reasoned inference**: the easiest stable workflow is to generate traces once in a carefully prepared algorithm environment, then do most iteration in the simulator and plotting layers.
- **Unclear**: exact versions used for the published paper beyond the checked-in pins are not fully frozen in one place, especially for the LLaVA and Hugging Face upstreams.
