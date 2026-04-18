# 05 Simulator Deep Dive

## Paper Anchor

The simulator is the concrete execution layer behind the methodology and performance claims in `doc/paper/tex/7.evaluation.tex`.

## Core Files

- `simulator/main.py`
- `simulator/models/models.py`
- `simulator/models/sparse_info.py`
- `simulator/arch/accelerator.py`
- `simulator/core/simulator.py`
- `simulator/core/simulator_comp.py`
- `simulator/core/simulator_mem.py`

## Main Object Flow

```text
simulator/main.py
  -> ModelConfig
  -> SparseInfo
  -> Accelerator
  -> Simulator
```

## What The Simulator Assumes

| Assumption | File path | Why it matters |
| --- | --- | --- |
| one Qwen2-like decoder template | `simulator/models/models.py` | model shapes are mostly hardcoded |
| one median sequence profile per model/dataset | `simulator/models/models.py`, `meta_data.csv` | workload is represented by one row |
| Focus traces provide per-layer sparse structure | `simulator/models/sparse_info.py` | Focus is modeled in more detail than baselines |
| baseline sparsity can be summarized coarsely | `simulator/models/sparse_info.py` | Adaptiv and CMC use CSV summary values, not rich traces |

## What Is Directly Modeled

### Compute

- ScaleSim-based GEMM cycle estimates in `simulator/core/simulator_comp.py`
- Focus scatter/gather overhead formulas in the same file

### Memory

- SRAM and DRAM traffic formulas in `simulator/core/simulator_mem.py`
- Focus-specific namespaces such as `layouter`, `similarity_map`, and `similarity_table`

### Energy / Area

- buffer area/energy from `simulator/memory/buffer.py` or `simulator/memory/cacti.py`
- core component area/power from `simulator/arch/*_rtl.csv`

## Paper Claim -> Simulator Evidence

| Paper claim | Paper source | Simulator evidence | Status |
| --- | --- | --- | --- |
| cycle-accurate simulation | `doc/paper/tex/7.evaluation.tex` | `simulator/core/simulator.py`, `simulator/core/simulator_comp.py`, `simulator/core/simulator_mem.py` | partially verifiable; simulator exists, but “cycle-accurate” is still a modeling claim |
| ScaleSim-v2 based | `doc/paper/tex/7.evaluation.tex` | `simulator/core/simulator_comp.py` imports ScaleSim | directly supported |
| sparse traces from PyTorch implementation | `doc/paper/tex/7.evaluation.tex` | `simulator/models/sparse_info.py` loads Focus `.pth` and baseline CSVs | directly supported |
| DRAMsim3 for DRAM energy | `doc/paper/tex/7.evaluation.tex` | only a DRAM constant in `simulator/arch/accelerator.py` | partially visible |
| 500 MHz evaluation point | `doc/paper/tex/7.evaluation.tex` | `simulator/arch/accelerator.py` frequency field | directly supported |

## Most Important Caveat

- **Direct observation**: `simulator/models/models.py` hardcodes a Qwen2-7B-like architecture instead of extracting every supported model dynamically.
- **Reasoned inference**: this is the biggest simulator-side approximation in the repository.

## DSE Coverage

| DSE item in paper | Script path | Simulator mode | Traceability |
| --- | --- | --- | --- |
| GEMM `m` tile size | `algorithm/run_dse.sh`, `simulator/run_dse_sim.sh` | `dse_m_tile_size(...)` | strong |
| vector size | same | `dse_vector_size(...)` | partial, because it is layer-wise only |
| SIC block size | same | `dse_block_size(...)` | strong |
| scatter accumulators | `simulator/run_dse_sim.sh` | `dse_num_scatter(...)` | strong |

## Which Paper Claims Depend Mostly On The Simulator

- performance speedup claims in `doc/paper/tex/7.evaluation.tex`
- energy-efficiency claims in the same section
- DRAM-access and activation-size claims
- area/power overhead claims when combined with RTL-derived component stats

## Which Claims Are Less Dependent On The Simulator

- algorithmic accuracy and sparsity claims backed by `algorithm/run_eval.py`
- qualitative method behavior such as SEC/SIC insertion and trace export

## Bottom Line

- **Direct observation**: the simulator is the main evidence engine for the paper’s hardware claims.
- **Reasoned inference**: it is strong enough to study the methodology, but some headline claims still rely on modeling assumptions that are not independently closed by executable RTL flow.
