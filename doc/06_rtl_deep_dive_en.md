# 06 RTL Deep Dive

## Paper Anchor

RTL-relevant hardware terms appear in:

- `doc/paper/tex/4.overview.tex`
- `doc/paper/tex/5.semantic.tex`
- `doc/paper/tex/6.vector.tex`
- synthesis/evaluation claims in `doc/paper/tex/7.evaluation.tex`

## What Exists

| Hardware concept | RTL files | Simulator linkage |
| --- | --- | --- |
| systolic array | `rtl/traditional_systolic.v`, `rtl/traditional_mac.v` | `simulator/arch/focus_rtl.csv`, `cmc_rtl.csv`, dense config in `simulator/arch/accelerator.py` |
| SIC cosine / norm / max / update | `rtl/cosine_similarity_unit.sv`, `rtl/inv_magnitude_unit.v`, `rtl/max_unit_fp16.v`, `rtl/average_update_unit.sv` | Focus component table in `simulator/arch/focus_rtl.csv` |
| Adaptiv array | `rtl/adaptiv_array.v` | `simulator/arch/adaptiv_rtl.csv` |
| CMC codec | `rtl/cmc_codec_pe.v`, `rtl/cmc_addertree_4to1.v` | `simulator/arch/cmc_rtl.csv` |
| SFUs | `rtl/fp16_exp.v`, `rtl/fp16_recip.v`, `rtl/fp16_mult.v`, `rtl/fp16_add.v`, `rtl/fp32_add.v`, `rtl/fastinvsqrt_fp32.v` | Focus / baseline SFU counts in `simulator/arch/accelerator.py` |

## Strong Correspondences

- **Direct observation**: the component names in `rtl/README.md` match the component names in `simulator/arch/*_rtl.csv`.
- **Direct observation**: `simulator/arch/accelerator.py` composes accelerator totals by multiplying those component stats by instance counts.

## Weak Correspondences

- **Direct observation**: there is no integrated `focus_top`-style RTL module.
- **Direct observation**: there are no synthesis scripts or testbenches.
- **Reasoned inference**: the paper’s hardware block diagram is represented as a component library, not as a fully packaged hardware project.

## Paper Hardware Terms vs Repository Reality

| Paper term | Paper source | Repository reality |
| --- | --- | --- |
| SEC importance analyzer / top-k sorter / offset encoder | `doc/paper/tex/5.semantic.tex` | no explicit separate RTL files found for all three; closest evidence is `max_unit_fp16.v` plus simulator component categories |
| Similarity Gather / Layouter / Scatter | `doc/paper/tex/6.vector.tex` | no full integrated RTL pipeline; ideas are split across RTL blocks plus simulator-side logic and buffers |
| standalone Focus unit beside systolic array | `doc/paper/tex/4.overview.tex` | represented conceptually by component tables and block RTL, but not as one checked-in top module |

## What Is Directly Testable

- module hierarchy and internal datapaths in the checked-in RTL files
- component naming consistency with simulator CSV summaries

## What Is Not Fully Testable From Repo Alone

- the exact synthesis flow claimed in `doc/paper/tex/7.evaluation.tex`
- whether the current checked-in RTL is exactly the same revision used for paper numbers
- full accelerator integration timing/functional behavior

## Code Quality Notes

- **Direct observation**: `rtl/cosine_similarity_unit.sv` has inconsistent `add_result` wiring versus `accum` usage.
- **Direct observation**: `rtl/cmc_codec_pe.v` does not assign the `distance` and `done` outputs in the checked-in file.
- **Reasoned inference**: the RTL should be treated as strong architectural evidence, but not as a fully polished standalone hardware release.

## Bottom Line

- **Direct observation**: the RTL layer supports the paper’s hardware vocabulary and simulator power/area accounting.
- **Reasoned inference**: it is better understood as “evidence for component implementation” than as a complete turnkey RTL project.
