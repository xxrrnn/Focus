# 05 Simulator Deep Dive

## Scope

- **Direct observation**: the simulator lives in `simulator/`.
- **Direct observation**: it consumes sparse artifacts from `algorithm/` and produces latency, energy, memory-access, and area/power CSVs.
- **Reasoned inference**: the simulator is the methodological center of the hardware claims in the repository.

## Key Files

| File | Role |
| --- | --- |
| `simulator/main.py` | User-facing entry point for normal runs, DSE, INT8, SEC-only, and image-task simulation. |
| `simulator/models/models.py` | Model/workload abstraction, including sequence length from `meta_data.csv` and a hardcoded decoder architecture template. |
| `simulator/models/sparse_info.py` | Loads Focus traces or baseline sparsity CSVs. |
| `simulator/arch/accelerator.py` | Defines accelerator configurations, buffer structure, area/power sources, and DRAM assumptions. |
| `simulator/core/simulator.py` | Top-level simulation orchestration and aggregation. |
| `simulator/core/simulator_comp.py` | Compute-cycle model, including ScaleSim calls and Focus scatter/gather overhead formulas. |
| `simulator/core/simulator_mem.py` | SRAM/DRAM traffic accounting for Focus and baselines. |
| `simulator/memory/buffer.py` | Memory-compiler-style buffer estimation from `buffer_model_spec.csv`. |
| `simulator/memory/cacti.py` | CACTI-based buffer area/energy estimation. |
| `simulator/utils/analysis.py` | Figure 13 style utilization analysis from Focus traces. |

## Simulator Architecture

The core abstraction stack is:

```text
simulator/main.py
  -> ModelConfig
  -> SparseInfo
  -> Accelerator
  -> Simulator
       -> SimulatorComp
       -> SimulatorMem
```

### ModelConfig

Implemented in `simulator/models/models.py`.

- **Direct observation**: `ModelConfig.add_seq_len()` reads `meta_data.csv` and loads `Sequence length`, `Num frames`, and `Num patches`.
- **Direct observation**: `ModelConfig.get_QWen2_7B_architecture()` hardcodes:
  - `dim = 3584`
  - `num_heads = 28`
  - `num_blocks = 28`
  - projection sizes for `q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj`, and `attn`
- **Direct observation**: there is a dormant method `retrieve_model_architecture(...)`, but the constructor does not call it.
- **Reasoned inference**: the simulator assumes all supported models are close enough to one Qwen2-7B-style decoder template that only the sequence length and sparsity traces need to vary.
- **Unclear**: that assumption is convenient but fragile, especially for image models and non-LLaVA families.

### SparseInfo

Implemented in `simulator/models/sparse_info.py`.

- **Direct observation**: for Focus it loads `.pth` traces from directories such as `focus_main/`, `focus_int8/`, `m_tile_size_dse/`, `block_size_dse/`, and `vector_size_dse/`.
- **Direct observation**: for Adaptiv and CMC it reads `adaptiv_sparsity.csv` and `cmc_sparsity.csv`.
- **Direct observation**: it validates Focus sequence length by comparing the trace tensor shape against `model_config.seq_len`.

### Accelerator

Implemented in `simulator/arch/accelerator.py`.

- **Direct observation**: the simulator supports four accelerator types: `focus`, `dense`, `adaptiv`, and `cmc`.
- **Direct observation**: Focus includes:
  - systolic array
  - SIC components
  - SEC components
  - dedicated buffers for concentration and similarity bookkeeping
- **Direct observation**: dense reuses the Focus base config and removes components ending in `(SIC)` or `(SEC)`.
- **Direct observation**: Adaptiv and CMC each define their own buffer and component layouts.

### Simulator

Implemented in `simulator/core/simulator.py`.

- **Direct observation**: `Simulator.run()` dispatches to `run_focus`, `run_dense`, `run_adaptiv`, or `run_cmc`.
- **Direct observation**: the result dict tracks:
  - execution time
  - total cycles
  - compute and stall cycles
  - total energy
  - DRAM access
  - bandwidth proxy
  - dense ops
  - actual ops
- **Reasoned inference**: this is the layer that turns per-layer formulas into paper-ready aggregate numbers.

## How Traces Are Parsed And Consumed

## Focus

### Trace Structure

`algorithm/focus/main.py` writes three tensor families:

- `mask_zero`
- `mask_similar`
- `group_idx`

Each is keyed by operation name such as `q_proj`, `o_proj`, `query`, `gate_proj`, and `down_proj`.

### Consumption Path

`simulator/core/simulator.py::run_focus(...)`:

1. iterates over each transformer block
2. iterates over each modeled layer type
3. maps some logical layer names to recorded trace keys
4. pulls `mask_zero`, `mask_similar`, and `group_idx`
5. calls compute and memory models
6. accumulates cycles, ops, memory counters, and activation sizes

### Important Name Mapping

- `k_proj` uses `q_proj`
- `v_proj` uses `q_proj`
- `up_proj` uses `gate_proj`
- `attn` uses `query`

This mapping is implemented in `simulator/core/simulator.py`.

## Adaptiv And CMC

- **Direct observation**: Adaptiv and CMC are modeled with aggregate sparsity rates rather than per-layer cluster traces.
- **Direct observation**: `run_adaptiv(...)` and `run_cmc(...)` in `simulator/core/simulator.py` apply the same sparsity numbers across all blocks/layers as configured by their formulas.
- **Reasoned inference**: this is simpler but also less structurally rich than the Focus path.

## How Latency Is Modeled

## Focus Compute Model

Implemented in `simulator/core/simulator_comp.py`.

### Dense Array Work

- `run_linear_focus(...)` estimates the effective number of remaining vectors and tokens after Focus clustering, then calls `call_scalesim(...)` with the active matrix dimensions.
- `run_attn_focus(...)` separately models QK and SV attention work and uses causal-halving logic.

### Focus-Specific Overheads

- `run_linear_scatter_focus(...)` models scatter cycles after concentration.
- `run_qk_scatter_focus(...)` models QK scatter cycles.
- `run_gather_linear_focus(...)` models gather cycles back into dense layout.
- `get_scatted_ops(...)` estimates scatter work for vector-size DSE.

### ScaleSim Integration

`call_scalesim(...)` in `simulator/core/simulator_comp.py`:

1. rewrites `simulator/core/scalesim_cfg/gemm.csv`
2. rewrites `simulator/core/scalesim_cfg/config.cfg`
3. instantiates `scalesim.scale_sim.scalesim`
4. runs ScaleSim
5. reads `get_total_cycles()`

### Important Caveat

- **Direct observation**: `call_scalesim(...)` mutates shared config files in place.
- **Direct observation**: `simulator/README.md` explicitly warns that multiple simultaneous simulations may encounter bugs.
- **Reasoned inference**: this is why the simulator is not safe to parallelize naïvely.

## Dense / Adaptiv / CMC Compute Models

- **Dense**: `run_linear_dense(...)` and `run_attn_dense(...)` in `simulator/core/simulator_comp.py`
- **Adaptiv**: `run_linear_adaptiv(...)` and `run_attn_adaptiv(...)`
- **CMC**: `run_linear_cmc(...)` and `run_attn_cmc(...)`

These models mix array-tile arithmetic with method-specific preprocessing assumptions.

## How Memory / Bandwidth Is Modeled

Implemented in `simulator/core/simulator_mem.py`.

### Memory Namespaces

`FocusData.data_type` defines:

- `input`
- `concentrate_out`
- `wgt`
- `output`
- `layouter`
- `similarity_map`
- `similarity_table`

### Focus Memory Paths

Key methods:

- `run_linear_focus(...)`
- `run_detect_linear_focus(...)`
- `run_detect_attn_focus(...)`
- `run_attn_focus(...)`

### What They Represent

- **Direct observation**: `run_linear_focus(...)` accounts for clustered compute data movement through `input`, `concentrate_out`, `wgt`, and `output`.
- **Direct observation**: `run_detect_linear_focus(...)` accounts for SIC/SEC bookkeeping traffic through `layouter`, `similarity_map`, and `similarity_table`.
- **Reasoned inference**: the repository models “detect and concentrate” as an explicit side path, not just reduced GEMM size.

### Aggregate Energy

`simulator/core/simulator.py::get_energy_breakdown(...)` computes:

- DRAM energy from total DRAM bytes times `accelerator.dram_config['energy_per_byte']`
- SRAM dynamic energy from per-buffer read/write energies
- SRAM leakage from buffer leak power times execution time
- core energy from accelerator core power times execution time

## How Area And Power Are Modeled

## On-Chip Components

`simulator/arch/accelerator.py` loads per-component area/power from:

- `simulator/arch/focus_rtl.csv`
- `simulator/arch/adaptiv_rtl.csv`
- `simulator/arch/cmc_rtl.csv`

### Focus Example Components

From `simulator/arch/accelerator.py` and `simulator/arch/focus_rtl.csv`:

- `Systolic Array`
- `Cosine Similarity (SIC)`
- `L2 Norm (SIC)`
- `Max Unit (SIC)`
- `Average Update (SIC)`
- `Accumulator (SIC)`
- `Max Unit (SEC)`
- `Importance Vector Buffer (SEC)`
- SFU entries such as `FP16 Exp (SFU)`

## Buffers

Two paths exist:

### Compiler-Table Path

- `simulator/memory/buffer_model_spec.csv`
- `simulator/memory/buffer.py`

Used for default, supported buffer compositions.

### CACTI Path

- `simulator/memory/cacti.py`
- `simulator/memory/sram_config.json`

Used when the configuration is outside the memory-compiler table or when `force_cacti=True`.

### Important Behavior

- **Direct observation**: Focus defaults to compiler-table evaluation only when `m_tile_size == 1024` and `num_scatter_vector == 2`.
- **Direct observation**: otherwise `Accelerator.evaluate_buffer(...)` falls back to CACTI.
- **Reasoned inference**: default paper numbers were likely tuned around one “native” Focus hardware point, with DSE points using a more flexible but slower SRAM model.

## Important Parameters And What They Mean

| Parameter | Where it appears | Meaning |
| --- | --- | --- |
| `focus_m_tile_size` | `simulator/main.py`, `simulator/arch/accelerator.py` | M-dimension tile size for Focus GEMMs and buffer sizing. |
| `block_size` | `simulator/main.py`, `simulator/arch/accelerator.py` | Used in gather-cycle modeling and Focus DSE. |
| `num_scatter_vector` | `simulator/main.py`, `simulator/arch/accelerator.py` | Number of scatter accumulator vectors in Focus. |
| `SEC_only` | `simulator/main.py`, `simulator/arch/accelerator.py` | Removes SIC behavior and changes Focus assumptions. |
| `vector_size` | `algorithm/run_dse.sh`, vector-size DSE logic | Controls Focus vector granularity on the algorithm side and array geometry in one simulator DSE mode. |

## DSE Paths

### m-tile DSE

- Full simulation path in `simulator/main.py::dse_m_tile_size(...)`
- Uses CACTI for buffer evaluation

### block-size DSE

- Full simulation path in `simulator/main.py::dse_block_size(...)`
- Loads trace files from `block_size_dse/`

### vector-size DSE

- Special path in `simulator/main.py::dse_vector_size(...)`
- Adjusts array geometry if `vector_size < 32`
- Calls `run_layer_wise_focus(...)` on `o_proj`, block 9 only

### scatter-accumulator DSE

- Full simulation path in `simulator/main.py::dse_num_scatter(...)`

## Output Artifacts

Common simulator CSVs:

- `main_focus.csv`
- `main_dense.csv`
- `main_adaptiv.csv`
- `main_cmc.csv`
- `main_focus_SEC_only.csv`
- `int8_focus.csv`
- `dse_*.csv`
- `detailed_power_area_breakdown.csv`

### Result Columns

`simulator/example_sim_results/main_focus.csv` shows the common output schema:

- `model`
- `dataset`
- `accelerator`
- `execution_time`
- `total_cycles`
- `total_compute_cycles`
- `total_stall_cycles`
- `total_energy`
- `total_dram_access`
- `dram_bandwidth`
- `dense_ops`
- `num_ops`
- `mem_counter_total`
- `total_activation`
- `total_compression_ratio`
- `dram_energy`
- `sram_energy`
- `core_energy`

## What External Tools Are Used, And How

### ScaleSim

- **Direct observation**: imported directly in `simulator/core/simulator_comp.py`.
- **Role**: provides base systolic-array cycle counts for GEMM-like workloads.

### CACTI

- **Direct observation**: called by `simulator/memory/cacti.py`.
- **Role**: estimates area, leakage, and read/write energy per byte for SRAM buffers.

### DRAMsim3

- **Direct observation**: the simulator code does not instantiate DRAMsim3.
- **Direct observation**: `simulator/arch/accelerator.py` uses a constant comment “from DRAMsim3”.
- **Reasoned inference**: DRAMsim3 is represented as a calibration source rather than a live simulation dependency in current workflows.

## Reusable Engineering Patterns

- **Direct observation**: the simulator decomposes compute, memory, and power/area into separate files and classes.
- **Direct observation**: all user-facing outputs are appended through one CSV helper in `simulator/utils/utils.py`.
- **Reasoned inference**: this is a clean way to keep hardware-model experimentation modular without making the entry point unreadable.

## Honest Caveats

- **Direct observation**: `ModelConfig` hardcodes one decoder architecture template.
- **Direct observation**: vector-size DSE is layer-wise, not end to end.
- **Direct observation**: simulator parallelism is unsafe because ScaleSim configs are overwritten in place.
- **Unclear**: the simulator is called “cycle-accurate,” but exact fidelity relative to RTL is not provable from the repository because no integrated RTL co-simulation exists.
