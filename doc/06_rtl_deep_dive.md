# 06 RTL Deep Dive

## Scope

- **Direct observation**: `rtl/` contains individual Verilog/SystemVerilog modules and a descriptive `rtl/README.md`.
- **Direct observation**: there is no checked-in top-level accelerator wrapper, no synthesis script, and no testbench directory.
- **Reasoned inference**: the RTL layer is best treated as a library of synthesized building blocks used to back the simulator’s area/power assumptions.

## What Exists In `rtl/`

### Focus-Related Blocks

| File | Main module | Role |
| --- | --- | --- |
| `rtl/traditional_systolic.v` | `traditional_systolic` | 32x32 systolic array used as the dense GEMM engine. |
| `rtl/traditional_mac.v` | `traditional_mac` | MAC processing element built from FP16 multiply and FP32 accumulation. |
| `rtl/cosine_similarity_unit.sv` | `cosine_similarity_unit` | SIC cosine-similarity block for vector comparison. |
| `rtl/inv_magnitude_unit.v` | `inv_magnitude_unit` | SIC norm / inverse-magnitude block. |
| `rtl/max_unit_fp16.v` | `max_unit_fp16` | Max-selection block used in SIC and SEC. |
| `rtl/average_update_unit.sv` | `average_update_unit` | SIC averaging / cluster-update block. |
| `rtl/fastinvsqrt_fp32.v` | `fast_inv_sqrt_fp32` | SFU for approximate reciprocal square root. |
| `rtl/fp16_exp.v` | `fp16_exp` | SFU exponential approximation. |
| `rtl/fp16_recip.v` | `fp16_recip` | SFU reciprocal approximation. |
| `rtl/fp16_mult.v` | `fp16_mult` | scalar FP16 multiply primitive. |
| `rtl/fp16_add.v` | `fp16_add` | scalar FP16 add primitive. |
| `rtl/fp32_add.v` | `fp32_add` | scalar FP32 add primitive. |

### Baseline Blocks

| File | Main module | Role |
| --- | --- | --- |
| `rtl/adaptiv_array.v` | `adaptiv_array` | Adaptiv MAC-array structure. |
| `rtl/cmc_codec_pe.v` | `cmc_codec_pe` | CMC codec processing element. |
| `rtl/cmc_addertree_4to1.v` | `cmc_addertree_4to1` | Helper adder tree for CMC data path. |

## RTL Hierarchy As Declared By The Repo

`rtl/README.md` explicitly groups the modules as:

- Focus:
  - systolic array
  - SIC cosine similarity, L2 norm, max unit, average update, accumulator
  - SEC max unit
  - SFUs
- Baselines:
  - dense systolic array
  - Adaptiv PE array
  - CMC codec unit and adder tree

### Why This Matters

- **Direct observation**: the component names in `rtl/README.md` line up with the component names used in `simulator/arch/focus_rtl.csv`, `adaptiv_rtl.csv`, and `cmc_rtl.csv`.
- **Reasoned inference**: the simulator’s component-level area/power accounting was built to mirror this hierarchy.

## Top-Level RTL Hierarchy

### Systolic Array

`rtl/traditional_systolic.v`:

- instantiates a `ROWS x COLS` grid of `traditional_mac`
- propagates left-to-right and top-to-bottom interconnect buses
- exposes control bits for stationary-dataflow behavior

`rtl/traditional_mac.v`:

- multiplies FP16 inputs
- converts into FP32 accumulation
- converts accumulated output back to FP16

### SIC Block Family

`rtl/cosine_similarity_unit.sv`:

- instantiates 32 `fp16_mult` units
- accumulates dot-product terms
- multiplies by inverse magnitudes to produce cosine similarity

`rtl/inv_magnitude_unit.v`:

- squares vector elements
- accumulates them
- applies `fast_inv_sqrt` to approximate reciprocal norm

`rtl/max_unit_fp16.v`:

- tracks an FP16 maximum using internal comparison logic

`rtl/average_update_unit.sv`:

- computes an update of the form `alpha * new + (1-alpha) * old`

### SEC Block Family

- **Direct observation**: the same `max_unit_fp16.v` module is listed in `rtl/README.md` as the SEC max unit.
- **Reasoned inference**: SEC ranking/reduction is represented as a simple reusable comparator block rather than a separate large datapath.

### SFU Family

`rtl/fp16_exp.v`, `rtl/fp16_recip.v`, `rtl/fp16_mult.v`, `rtl/fp16_add.v`, and `rtl/fastinvsqrt_fp32.v` provide scalar arithmetic primitives that the README groups under SFUs.

## How Data Likely Flows Through The RTL

### Focus Path

1. Input activations and weights enter the systolic array in `rtl/traditional_systolic.v`.
2. SIC-related vectors are analyzed by:
   - `rtl/inv_magnitude_unit.v`
   - `rtl/cosine_similarity_unit.sv`
   - `rtl/max_unit_fp16.v`
   - `rtl/average_update_unit.sv`
3. **Reasoned inference**: SEC uses `rtl/max_unit_fp16.v` to rank token-importance values.
4. SFU modules provide scalar arithmetic support for nonlinear / normalization style helper computations.

### Baseline Paths

1. Dense uses `rtl/traditional_systolic.v`.
2. Adaptiv uses `rtl/adaptiv_array.v`.
3. CMC uses `rtl/cmc_codec_pe.v` plus `rtl/cmc_addertree_4to1.v`.

### Observation And Inference Boundary

- **Direct observation**: the individual module roles are explicit in filenames, module names, and `rtl/README.md`.
- **Reasoned inference**: the exact global dataflow between buffers, controller logic, SIC, SEC, and the array is not fully reconstructable because no integrated top-level Focus accelerator module is present.

## Module-Level Notes

### `rtl/traditional_systolic.v`

- **Direct observation**: this is a true 2D systolic mesh with horizontal and vertical interconnect arrays.
- **Direct observation**: the file header comments reference a Scale-Sim example source.
- **Reasoned inference**: the dense compute core was likely derived from or adapted from known systolic reference RTL rather than written from scratch.

### `rtl/adaptiv_array.v`

- **Direct observation**: this file also references a Scale-Sim example in its header comment.
- **Direct observation**: each PE is instantiated independently and driven from flattened row/column buses.
- **Reasoned inference**: this looks more like an area-model MAC fabric for Adaptiv than a deeply integrated, cycle-accurate system-level array wrapper.

### `rtl/cosine_similarity_unit.sv`

- **Direct observation**: the module declares `logic [15:0] add_result;`.
- **Direct observation**: the instantiated `fp16_add u_add` drives `.result(accum)` directly.
- **Direct observation**: the sequential block later does `accum <= add_result;`.
- **Reasoned inference**: this is internally inconsistent and may indicate an unfinished or incorrect connection in the checked-in RTL.

### `rtl/cmc_codec_pe.v`

- **Direct observation**: the module computes `sum_temp` via `adder_tree_64` and registers it into `sum`.
- **Direct observation**: the output ports include `distance` and `done`.
- **Direct observation**: the file as checked in does not assign either `distance` or `done`.
- **Reasoned inference**: this module appears incomplete as a standalone, externally usable block.

## Relation To Simulator And Paper Architecture

### Simulator Coupling

- **Direct observation**: `simulator/arch/focus_rtl.csv`, `adaptiv_rtl.csv`, and `cmc_rtl.csv` use component names that match the RTL README grouping.
- **Direct observation**: `simulator/arch/accelerator.py` multiplies those component power/area values by configured instance counts.
- **Reasoned inference**: the simulator is using synthesized RTL block statistics as component library entries.

### Paper Architecture Coupling

- **Direct observation**: the top-level README and `rtl/README.md` both refer to SIC, SEC, systolic array, and SFUs.
- **Reasoned inference**: the RTL is the hardware embodiment of the architectural block diagram, but only at block granularity, not as a full tapeout-ready integrated accelerator in this repo.

## Testbench / Synthesis / Build Clues

- **Direct observation**: `rtl/README.md` says the modules were synthesized with Synopsys Design Compiler.
- **Direct observation**: no synthesis TCL, no simulation scripts, and no testbench files were found under `rtl/`.
- **Unclear**: how exactly the authors generated the checked-in `simulator/arch/*_rtl.csv` tables from these RTL files is not reproducible from the repo alone.

## What Is Clean Vs Brittle

### Clean

- **Direct observation**: the module set maps clearly onto the named architectural blocks.
- **Direct observation**: arithmetic primitives are factored out cleanly.
- **Reasoned inference**: as a block library for area/power estimation, the RTL layer is conceptually clear.

### Brittle

- **Direct observation**: there is no integrated top-level module showing full accelerator control and buffering.
- **Direct observation**: at least `rtl/cosine_similarity_unit.sv` and `rtl/cmc_codec_pe.v` have suspicious signal/output issues in the checked-in code.
- **Unclear**: whether the exact synthesized versions were identical to the current checked-in files.

## Bottom Line

- **Direct observation**: the RTL layer contains meaningful hardware blocks for Focus and the baselines.
- **Reasoned inference**: it is sufficient to understand how the simulator’s area/power tables map to hardware concepts.
- **Unclear**: it is not sufficient, by itself, to reproduce an integrated hardware flow without external missing scripts and possibly corrected versions of some modules.
