# 10 Open Questions And Risks

## Unresolved Ambiguities

- **Unclear**: `simulator/models/models.py` hardcodes one Qwen2-7B-like decoder architecture for all supported models. It is not obvious how closely this matches `llava_onevision`, `minicpm_v`, or `qwen2_5_vl` in every modeled layer.
- **Unclear**: `simulator/README.md` calls the simulator cycle-accurate, but the implementation in `simulator/core/simulator_comp.py` and `simulator/core/simulator_mem.py` is a hybrid of ScaleSim calls and closed-form overhead formulas rather than direct RTL execution.
- **Unclear**: `3rd_party/DRAMsim3` is cloned and built per `README.md`, but the simulator currently uses only a DRAM constant in `simulator/arch/accelerator.py`; direct DRAMsim3 execution is not visible in normal runs.
- **Unclear**: the checked-in `rtl/` files do not reveal the exact synthesis flow that produced `simulator/arch/focus_rtl.csv`, `adaptiv_rtl.csv`, and `cmc_rtl.csv`.
- **Unclear**: the exact baseline fidelity for Adaptiv, CMC, and FrameFusion relative to their source papers/repos is not fully auditable from this repo alone.

## Brittle Spots

- **Direct observation**: `algorithm/focus/pyproject.toml` and `algorithm/README.md` require switching between `transformers==4.48.2` and `transformers==4.49.0` depending on whether Qwen2.5-VL is used.
- **Direct observation**: `simulator/core/simulator_comp.py` rewrites `simulator/core/scalesim_cfg/config.cfg` and `gemm.csv` in place. This matches the warning in `simulator/README.md` that parallel runs may break.
- **Direct observation**: `simulator/main.py::dse_vector_size(...)` simulates only `o_proj` at block 9 using `run_layer_wise_focus(...)`, so its output is not directly comparable to full-model DSE runs without context.
- **Direct observation**: `algorithm/focus/main.py::post_process(...)` chooses one median sample from a fixed limited sample set, usually 10. That is a small sample on which to anchor the simulator’s workload.
- **Direct observation**: `rtl/cosine_similarity_unit.sv` and `rtl/cmc_codec_pe.v` contain suspicious incomplete signal wiring/output behavior in the checked-in code.

## Reproducibility Risks

- **Direct observation**: `algorithm/README.md` estimates very large GPU-hour costs for full accuracy runs, especially CMC. Many users will rely on checked-in example outputs instead of reproducing from scratch.
- **Direct observation**: model checkpoints and datasets are downloaded dynamically through Hugging Face and external task loaders in `algorithm/lmms-eval/`.
- **Direct observation**: notebooks in `evaluation_scripts/plot_scripts/ipynb_src/` use directory-relative path variables that may require manual editing when outputs are regenerated elsewhere.
- **Reasoned inference**: the easiest part to reproduce is the plotting layer; the hardest part is the full algorithm environment and model download stack.
- **Reasoned inference**: long-term reproducibility depends on freezing external upstreams more tightly than the repository currently does.

## Hidden Assumptions

- **Direct observation**: `algorithm/focus/interface.py` assumes supported models can be represented by a Qwen2-like decoder patching strategy, even when the outer multimodal wrappers differ.
- **Direct observation**: Focus trace export records only selected operation types in `algorithm/focus/main.py`, and the simulator remaps missing layer types in `simulator/core/simulator.py`.
- **Direct observation**: `simulator/core/simulator.py` assumes one sequence-length row per model/dataset in `meta_data.csv`.
- **Reasoned inference**: the simulator assumes that one median sparse pattern is representative enough to stand in for a dataset/model pair.
- **Reasoned inference**: the repository is optimized for controlled paper comparisons, not for arbitrary model families or arbitrary workload diversity.

## Missing Documentation

- **Direct observation**: there is no `evaluation_scripts/README.md`.
- **Direct observation**: there is no integrated explanation in-source of why `ModelConfig` hardcodes one decoder template.
- **Direct observation**: there are no synthesis scripts in `rtl/`.
- **Direct observation**: there is no top-level orchestration script that chains algorithm generation, simulator runs, and notebook execution.

## Especially Elegant Engineering Patterns Worth Learning

- **Direct observation**: `algorithm/focus/interface.py` centralizes all model surgery in one place. That is a strong pattern for research repos that patch third-party models.
- **Direct observation**: `algorithm/focus/main.py` exports hardware-facing sparse traces as stable files. That decouples expensive ML execution from hardware iteration.
- **Direct observation**: `simulator/arch/accelerator.py` consumes precomputed RTL summaries from CSV instead of requiring the simulator to know EDA details.
- **Direct observation**: `algorithm/run_eval.py` keeps paper-facing score extraction and CSV formatting in one file, making the comparison contract easy to inspect.
- **Reasoned inference**: the repo’s best reusable idea is the artifact-centered pipeline: generate once, replay many times, plot later.

## What I Would Treat Carefully If Continuing This Project

- **Reasoned inference**: validate the hardcoded simulator architecture against each supported model before extending the paper claims.
- **Reasoned inference**: replace the layer-wise vector-size DSE shortcut with a full-model version if vector-size conclusions need stronger backing.
- **Reasoned inference**: add tests or at least minimal simulation harnesses for RTL modules with suspicious wiring.
- **Reasoned inference**: freeze a container or lockfile-based environment for the algorithm stack if long-term reproducibility matters.
