# 10 Open Questions and Risks

This document is intentionally critical. Its purpose is to show what is strong, what is brittle, and what remains unclear after reading both `doc/paper/*.tex` and the code.

## Main Open Questions

| Topic | Evidence | Why it matters | Status |
| --- | --- | --- | --- |
| One hardware template for multiple models | `simulator/models/models.py` hardcodes `get_QWen2_7B_architecture()` and uses it for all supported models after reading `meta_data.csv` | Performance and operation counts may not fully match model-specific hidden dimensions or layer structures | Direct observation |
| DRAMsim3 claim vs checked-in runtime | `doc/paper/tex/7.evaluation.tex` cites DRAMsim3, but `simulator/arch/accelerator.py` uses fixed DRAM bandwidth and energy-per-byte constants | Some energy claims depend on assumptions that are weaker than the paper wording suggests | Direct observation |
| Provenance of RTL-derived CSVs | `simulator/arch/focus_rtl.csv`, `adaptiv_rtl.csv`, and `cmc_rtl.csv` are consumed by `simulator/arch/accelerator.py`, but synthesis scripts are not in `rtl/` | Area/power claims are harder to audit end to end | Direct observation |
| Integrated Focus top-level RTL | `rtl/README.md` lists blocks, but no integrated top module or testbench flow was found | The paper’s architectural picture is stronger than the checked-in RTL integration evidence | Direct observation |
| Fidelity of baseline hardware simulation | `simulator/models/sparse_info.py` loads only aggregate sparsity for AdapTiV and CMC | Baseline comparisons may be directionally useful but are not trace-equivalent to Focus | Direct observation |
| DSE vector-size path | `simulator/main.py::dse_vector_size(...)` simulates one layer config, not a full end-to-end model sweep | Design conclusions for vector size are meaningful but narrower than the paper’s overall phrasing may imply | Direct observation |
| Notebook hardcoding | notebooks in `evaluation_scripts/plot_scripts/ipynb_src/` read specific filenames such as `main_focus.csv`, `dse_a_m_tile_size.csv`, and `accuracy.csv` | Reproduction is convenient, but folder conventions are brittle | Direct observation |
| Median-trace methodology | `algorithm/run_focus.sh` uses `--limit 10 --use_median`, and `algorithm/focus/main.py::post_process(...)` selects a median-sparsity sample | Simulator inputs for main results are based on representative samples, not full-trace distributions | Direct observation |

## Code-Level Brittle Spots

| File | Observation | Risk |
| --- | --- | --- |
| `algorithm/focus/interface.py` | relies on monkeypatching exact model classes and forward names | highly version-sensitive to `transformers` and model wrapper changes |
| `algorithm/focus/models/qwen2/modeling_qwen2.py` | Focus is wired into a specific Qwen2-family execution shape | extending to new model families requires careful manual patching |
| `simulator/core/simulator_comp.py` | rewrites shared ScaleSim config files under `simulator/core/scalesim_cfg/` | parallel runs can race; `simulator/README.md` already warns about this |
| `rtl/cosine_similarity_unit.sv` | signal wiring around `add_result` and `accum` looks inconsistent | RTL may require manual review before treating it as production-clean |
| `rtl/cmc_codec_pe.v` | checked-in file does not clearly drive all output ports such as `distance` and `done` | baseline RTL evidence is weaker than a polished IP block would be |

## Naming Mismatches Between Paper and Code

- The paper uses `SEC`, `SIC`, `Similarity Gather`, `Similarity Scatter`, and `Convolution-style Layouter` in `doc/paper/tex/5.semantic.tex` and `doc/paper/tex/6.vector.tex`.
- The code mostly uses `Focus`, `semantic_concentration`, `focus_similarity_concentration`, `mask_zero`, `mask_similar`, and `group_idx` in `algorithm/focus/main.py`.
- Reasoned inference: this mismatch is not accidental. The paper names hardware-facing blocks, while the code names runtime transformations and serialized artifacts.

## Claims That Depend More On Simulation Than On Direct Execution

- Speedup claims in `doc/paper/tex/7.evaluation.tex`
- Energy-efficiency claims in `doc/paper/tex/7.evaluation.tex`
- The `2.7%` area overhead claim in `doc/paper/tex/7.evaluation.tex`
- Detailed memory-traffic claims in `doc/paper/tex/7.evaluation.tex`

These are still meaningful, but the supporting path is `algorithm trace -> simulator -> notebook`, not `algorithm runtime alone`.

## Reproducibility Risks

- Full accuracy runs are extremely expensive; `algorithm/README.md` and `doc/paper/tex/artifact_appendix.tex` both imply multi-day GPU usage.
- Environment sensitivity is high because `algorithm/focus/pyproject.toml` pins `transformers==4.48.2` for main runs and `4.49.0` for `qwen25_vl`.
- The repo depends on access to gated HuggingFace models and datasets, noted in `README.md` and `doc/paper/tex/artifact_appendix.tex`.
- Shell scripts assume fixed output locations such as `./output` and `results`, which is convenient but brittle.

## Especially Elegant Patterns Worth Learning

- Trace-based subsystem decoupling: `algorithm/focus/main.py` writes traces, and `simulator/models/sparse_info.py` consumes them without requiring live model code.
- Thin integration layer: `algorithm/focus/interface.py` keeps model-specific patching separate from the method logic in `algorithm/focus/main.py`.
- Downstream-only plotting: `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb` consume CSVs rather than re-running the method.
- Explicit experiment modes: `simulator/main.py` cleanly separates main runs, DSE, quantization, and ablation paths.

## Bottom Line

- Direct observation: the repository is strong on method execution, trace export, and CSV-based experiment plumbing.
- Reasoned inference: the weakest part is the final mile from paper hardware narrative to auditable, integrated RTL/synthesis flow.
- Unclear: whether the checked-in RTL files are the exact source basis for all reported power/area numbers without any missing private scripts.
