# 14 Glossary

| Term | Meaning in this repository |
| --- | --- |
| Focus | The main method of the paper, implemented in `algorithm/focus/main.py` and modeled in `simulator/`. |
| SIC | Similarity concentrator. In code, this is the vector/block similarity path implemented by `Focus.focus_similarity_concentration(...)` in `algorithm/focus/main.py`. |
| SEC | Semantic concentrator. In code, this is the attention-guided token-pruning path implemented by `Focus.semantic_concentration(...)`. |
| VLM | Vision-Language Model. Supported families include LLaVA-Video, LLaVA-OneVision, MiniCPM-V, and Qwen2.5-VL. |
| `run_eval.py` | Main algorithm-side driver for evaluation, trace export, and accuracy CSV generation. |
| `lmms-eval` | Vendored evaluation framework in `algorithm/lmms-eval/` used to run tasks and collect metrics. |
| Trace | Serialized sparse behavior emitted by Focus to a `.pth` file plus metadata in `meta_data.csv`. |
| `mask_zero` | Focus trace field marking vectors that are already zero. |
| `mask_similar` | Focus trace field marking vectors that were merged away as similar. |
| `group_idx` | Focus trace field storing cluster/group assignments. |
| `meta_data.csv` | CSV that stores sequence length, frame count, patch count, and median trace index per model/dataset. |
| Adaptiv | Baseline method implemented in `algorithm/focus/baseline_adaptiv.py` and simulated from a coarse sparsity CSV. |
| CMC | Baseline method implemented in `algorithm/focus/baseline_CMC.py` and simulated from coarse sparsity terms. |
| FrameFusion | Baseline integrated via `algorithm/lmms-eval/lmms_eval/framefusion/interface.py`. |
| `ModelConfig` | Simulator object in `simulator/models/models.py` that defines the modeled layer shapes and sequence statistics. |
| `SparseInfo` | Simulator object in `simulator/models/sparse_info.py` that loads trace or sparsity artifacts. |
| `Accelerator` | Simulator object in `simulator/arch/accelerator.py` describing buffers, array shape, components, and power/area sources. |
| `Simulator` | Aggregation layer in `simulator/core/simulator.py` that drives compute, memory, and energy accounting. |
| ScaleSim | External GEMM performance simulator used in `simulator/core/simulator_comp.py`. |
| CACTI | External SRAM modeling tool used in `simulator/memory/cacti.py`. |
| DRAMsim3 | Third-party DRAM simulator included as a submodule; currently represented indirectly through constants rather than direct invocation in normal simulator runs. |
| `*_rtl.csv` | Precomputed component area/power summaries used by the simulator. |
| `main_focus.csv` | Main Focus simulator output table. Equivalent `main_dense.csv`, `main_adaptiv.csv`, and `main_cmc.csv` exist for comparisons. |
| DSE | Design Space Exploration. In this repo it covers tile size, vector size, block size, and scatter-resource sweeps. |
| INT8 Focus | Focus run with quantized model execution on the algorithm side and a separate `focus_int8` trace path for the simulator. |
