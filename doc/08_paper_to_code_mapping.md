# 08 Paper To Code Mapping

The table below maps major paper concepts to concrete implementation locations.

| Paper concept | Implementation status | Primary file path(s) | Key function / class / module | Notes / caveats |
| --- | --- | --- | --- | --- |
| Evaluation on real VLM tasks | fully implemented | `algorithm/run_eval.py`, `algorithm/lmms-eval/lmms_eval/evaluator.py`, `algorithm/lmms-eval/lmms_eval/models/*.py` | `cli_evaluate`, `simple_evaluate`, model adapter classes | Runs real VLM inference through `lmms-eval`; not a toy driver. |
| Focus model injection into supported VLMs | fully implemented | `algorithm/focus/interface.py` | `apply_focus`, `replace_focus_forward` | Central monkeypatch layer that rewires model forwards. |
| Focus default hyperparameter presets | fully implemented | `algorithm/focus/configs/focus.csv`, `algorithm/focus/interface.py` | CSV lookup in `replace_focus_forward` | `selected_layers`, `alpha_list`, and `similarity_threshold` are dataset/model specific. |
| SIC vector/block similarity concentration | fully implemented | `algorithm/focus/main.py`, `algorithm/focus/models/qwen2/modeling_qwen2.py` | `Focus.focus_similarity_concentration`, `Focus.forward` | Inserted around `q_proj`, `query`, `o_proj`, `gate_proj`, and `down_proj`. |
| GEMM tile locality restriction for concentration | fully implemented | `algorithm/focus/main.py` | `_compute_gemm_m_tile_indices`, `_get_gemm_m_tile_mask_vectorized` | Matching can be restricted by `gemm_m_size`. |
| SEC attention-guided token pruning | fully implemented | `algorithm/focus/main.py`, `algorithm/focus/models/qwen2/modeling_qwen2.py` | `set_token_importance`, `semantic_concentration`, `Qwen2DecoderLayer_focus_forward` | Activated only on configured decoder layers. |
| Model-specific multimodal metadata extraction | fully implemented | `algorithm/focus/models/llava_video/modeling_llava_video.py`, `algorithm/focus/models/minicpmv/modeling_minicpmv.py`, `algorithm/focus/models/qwen2_5_vl/modeling_qwen2_5_vl.py` | `self.focus.prepare(...)` call sites | This is how Focus learns frame/patch/token geometry. |
| Focus sparse-trace export for hardware simulation | fully implemented | `algorithm/focus/main.py`, `algorithm/focus/utils.py` | `prepare`, `post_process`, `save_result_to_csv` | Writes `.pth` trace plus `meta_data.csv`. |
| Median-sample trace selection | fully implemented | `algorithm/focus/main.py` | `post_process` | Uses the median weighted sparsity among a limited sample set, default limit 10. |
| Accuracy summary tables for paper | fully implemented | `algorithm/run_eval.py` | `get_score_from_results`, `save_score_to_main_csv`, `save_score_to_dse_csv` | Paper-facing CSV schema is hand-coded here. |
| Adaptiv baseline | approximated | `algorithm/focus/baseline_adaptiv.py`, `algorithm/focus/interface.py` | `Adaptiv`, Adaptiv patch branch | Integrated into the same patching framework; simulator uses only aggregate sparsity. |
| CMC baseline | approximated | `algorithm/focus/baseline_CMC.py`, `algorithm/focus/interface.py` | `CMC`, CMC patch branch | Also reduced to aggregate sparsity for simulation. |
| FrameFusion baseline | partially implemented | `algorithm/lmms-eval/lmms_eval/framefusion/interface.py`, `algorithm/lmms-eval/lmms_eval/evaluator.py` | `apply_framefusion` | Wrapped and exposed for comparison, but lives in vendored `lmms-eval` subtree. |
| Sparse-trace driven Focus accelerator simulation | fully implemented | `simulator/main.py`, `simulator/models/sparse_info.py`, `simulator/core/simulator.py` | `SparseInfo`, `Simulator.run_focus` | The main bridge from algorithm output to hardware metrics. |
| Dense / Adaptiv / CMC accelerator comparisons | approximated | `simulator/core/simulator.py`, `simulator/core/simulator_comp.py`, `simulator/core/simulator_mem.py` | `run_dense`, `run_adaptiv`, `run_cmc` | Baselines are modeled from formulaic sparsity/throughput assumptions rather than detailed traces. |
| Systolic-array compute modeling | fully implemented | `simulator/core/simulator_comp.py`, `simulator/core/scalesim_cfg/*`, `3rd_party/scalesim` | `call_scalesim` | Delegates GEMM cycle estimates to ScaleSim. |
| Focus scatter/gather / concentration overhead modeling | fully implemented | `simulator/core/simulator_comp.py`, `simulator/core/simulator_mem.py` | `run_linear_scatter_focus`, `run_gather_linear_focus`, detect-memory methods | This is the main Focus-specific hardware model. |
| SRAM buffer sizing / area / energy modeling | fully implemented | `simulator/arch/accelerator.py`, `simulator/memory/buffer.py`, `simulator/memory/cacti.py` | `evaluate_buffer`, `get_buffer_area_power_energy` | Uses compiler tables by default and CACTI when needed. |
| DRAM modeling | partially implemented | `simulator/arch/accelerator.py` | `dram_config` constant | Uses a fixed bandwidth and energy-per-byte constant; no live DRAMsim3 invocation. |
| RTL block library for Focus and baselines | partially implemented | `rtl/README.md`, `rtl/*.v`, `rtl/*.sv` | individual RTL modules | Major blocks are present, but no integrated top, testbench, or synthesis scripts. |
| Area/power linkage from RTL to simulator | fully implemented | `simulator/arch/accelerator.py`, `simulator/arch/focus_rtl.csv`, `adaptiv_rtl.csv`, `cmc_rtl.csv` | `get_components_area_power` | Uses precomputed synthesis summaries. |
| End-to-end figure/table generation | only exposed via scripts / configs / notebooks | `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`, `simulator/utils/analysis.py` | notebook cells, `worst_case_analysis` | Final paper artifacts are generated downstream from CSVs. |
| Integrated RTL execution or co-simulation | not obviously present | `rtl/`, `simulator/` | none found | RTL is not driven directly by simulator in the checked-in repo. |

## Summary

- **Direct observation**: the strongest implementation coverage is on Focus runtime instrumentation, sparse-trace export, and simulator consumption of those traces.
- **Reasoned inference**: baseline and RTL support are good enough for paper comparison, but not equally deep or equally reproducible in all layers.
- **Unclear**: anything requiring full DRAM simulation, integrated RTL co-simulation, or exact baseline equivalence to external repositories is outside what this repo alone makes fully auditable.
