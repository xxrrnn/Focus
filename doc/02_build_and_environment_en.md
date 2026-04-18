# 02 Build And Environment

## Paper Sources For Environment Claims

Environment and workflow expectations are described both in the repository README and in `doc/paper/tex/artifact_appendix.tex`.

## Dependency Layers

| Layer | Files | Why it matters |
| --- | --- | --- |
| Top-level workflow | `README.md`, `.gitmodules` | Clone, submodules, basic install order |
| Focus package | `algorithm/focus/pyproject.toml` | Focus runtime dependencies and transformer version pinning |
| Evaluation framework | `algorithm/lmms-eval/pyproject.toml`, `algorithm/lmms-eval/requirements.txt` | heavy multimodal runtime stack |
| LLaVA dependency | `3rd_party/LLaVA-NeXT`, `algorithm/lmms-eval/lmms_eval/models/llava_vid.py`, `llava_onevision.py` | required for LLaVA-based model adapters |
| ScaleSim dependency | `3rd_party/scalesim`, `simulator/core/simulator_comp.py` | required for compute-cycle modeling |
| CACTI dependency | `3rd_party/cacti`, `simulator/memory/cacti.py` | required for flexible SRAM modeling |

## Submodules

From `.gitmodules`:

- `3rd_party/scalesim`
- `3rd_party/cacti`
- `3rd_party/DRAMsim3`
- `3rd_party/LLaVA-NeXT`

## What Each Submodule Is Used For

- **Direct observation**: `3rd_party/LLaVA-NeXT` is imported by `algorithm/lmms-eval/lmms_eval/models/llava_vid.py` and `llava_onevision.py`.
- **Direct observation**: `3rd_party/scalesim` is imported by `simulator/core/simulator_comp.py`.
- **Direct observation**: `simulator/memory/cacti.py` expects a compiled CACTI binary inside `3rd_party/cacti`.
- **Unclear**: `3rd_party/DRAMsim3` is present, but normal simulator execution does not directly call it. The repo uses a DRAM constant in `simulator/arch/accelerator.py`.

## Version-Sensitive Parts

### Python

- `README.md` recommends Python 3.11.
- `algorithm/focus/pyproject.toml` requires Python `>=3.11`.
- `doc/paper/tex/artifact_appendix.tex` also states Python 3.11+.

### Transformers

- `algorithm/focus/pyproject.toml` requires `transformers>=4.48.2,<4.50.0`.
- The `main` extra pins `transformers==4.48.2`.
- The `qwen25_vl` extra pins `transformers==4.49.0`.
- `algorithm/README.md` explicitly tells the user to switch environments when evaluating Qwen2.5-VL.

### Why This Matters

- **Direct observation**: video-VLM support and Qwen2.5-VL support are not frozen under one identical package set.
- **Reasoned inference**: this is one of the main reproducibility risks of the repository.

## What You Need Per Subsystem

| Subsystem | Required pieces |
| --- | --- |
| `algorithm/` | Python 3.11, Hugging Face access, GPU, `LLaVA-NeXT`, `lmms-eval`, `focus`, multimodal packages such as `decord` and `av` |
| `simulator/` | Python, `scalesim`, CACTI for non-default buffer evaluation, algorithm-generated traces or example outputs |
| `rtl/` | No runnable flow is packaged; only source files are present |
| `evaluation_scripts/` | Jupyter + pandas/matplotlib + generated CSVs |

## Native / External Build Steps

The intended order from `README.md`:

1. initialize submodules
2. create Python environment
3. `pip install -e .` in `3rd_party/LLaVA-NeXT`
4. `pip install -e .` in `3rd_party/scalesim`
5. `make` in `3rd_party/cacti`
6. `make` in `3rd_party/DRAMsim3`
7. `pip install -e .` in `algorithm/lmms-eval`
8. `pip install -e '.[main]'` in `algorithm/focus`

## Common Failure Points

| Risk | Code / doc evidence |
| --- | --- |
| Wrong transformers version | `algorithm/focus/pyproject.toml`, `algorithm/README.md` |
| Missing CACTI binary | `simulator/memory/cacti.py` |
| Missing ScaleSim install | `simulator/core/simulator_comp.py` |
| Missing LLaVA install | `algorithm/lmms-eval/lmms_eval/models/llava_vid.py`, `llava_onevision.py` |
| Parallel simulator interference | `simulator/README.md`, `simulator/core/simulator_comp.py` rewrites shared config files |
| Full accuracy is expensive | `algorithm/README.md`, `doc/paper/tex/artifact_appendix.tex` |

## What The Paper Says vs What The Repo Actually Requires

| Topic | Paper source | Repo evidence | Result |
| --- | --- | --- | --- |
| A100-class GPU | `doc/paper/tex/artifact_appendix.tex` | `README.md`, heavy model adapters in `algorithm/lmms-eval/models/` | consistent |
| PyTorch implementation | `doc/paper/tex/7.evaluation.tex` | `algorithm/focus/`, `algorithm/run_eval.py` | consistent |
| ScaleSim-based simulator | `doc/paper/tex/7.evaluation.tex` | `simulator/core/simulator_comp.py` | directly supported |
| DRAMsim3-based DRAM energy | `doc/paper/tex/7.evaluation.tex` | only a DRAM constant in `simulator/arch/accelerator.py` | partially visible |
| RTL synthesized with DC | `doc/paper/tex/7.evaluation.tex` | `rtl/README.md`, `simulator/arch/*_rtl.csv` | evidence exists, scripts missing |

## Bottom Line

- **Direct observation**: the algorithm environment is the fragile part; the simulator is much lighter once traces exist.
- **Reasoned inference**: a practical workflow is to stabilize the algorithm environment once, generate traces, and then iterate mostly in `simulator/` and `evaluation_scripts/`.
