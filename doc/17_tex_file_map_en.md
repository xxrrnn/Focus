# 17 TeX File Map

This map treats the paper source itself as part of the repository architecture.

## Top-Level Files

| File | Role | What to look for | Related repo paths |
| --- | --- | --- | --- |
| `doc/paper/main.tex` | top-level paper entry | include order, section order, bibliography hook | all documentation and code mapping |
| `doc/paper/macros.tex` | project-wide macros | `\proj` is defined as `Focus`; shorthand for figures/tables/sections | useful for interpreting section text |
| `doc/paper/hpca-template.tex` | conference formatting template | mostly formatting, not method content | low priority |
| `doc/paper/refs.bib` | bibliography | terminology and baseline references such as AdapTiV, CMC, FrameFusion, SCALEsim, DRAMsim3 | external-context support only |
| `doc/paper/00README.json` | artifact metadata | confirms `main.tex` is the entry and gives build tool info | environment and artifact context |

## Section Files

| File | Main content | Main labels / figures / tables | Closest code or script anchors |
| --- | --- | --- | --- |
| `doc/paper/tex/0.abstract.tex` | compressed statement of contributions and headline metrics | none | `algorithm/focus/main.py`, `simulator/main.py` |
| `doc/paper/tex/1.introduction.tex` | problem framing, contributions, intro figure | `fig:intro` | `README.md`, `algorithm/`, `simulator/` |
| `doc/paper/tex/2.background.tex` | VLM and efficiency background | none | context only |
| `doc/paper/tex/3.motivation.tex` | why semantic and local vector redundancy matter | `fig:motivation1`, `fig:motivation2` | `algorithm/focus/main.py`, `simulator/core/simulator_mem.py` |
| `doc/paper/tex/4.overview.tex` | Focus Unit overview | `fig:focus_arch` | `algorithm/focus/main.py`, `simulator/arch/accelerator.py`, `rtl/README.md` |
| `doc/paper/tex/5.semantic.tex` | SEC design and hardware narrative | `fig:semantic` | `algorithm/focus/main.py`, `algorithm/focus/models/qwen2/modeling_qwen2.py` |
| `doc/paper/tex/6.vector.tex` | SIC gather, layouter, scatter | `fig:sic_gather`, `fig:sic_layouter`, `fig:sic_scatter` | `algorithm/focus/main.py`, `simulator/core/simulator_comp.py`, `simulator/core/simulator_mem.py`, `rtl/*.v`, `rtl/*.sv` |
| `doc/paper/tex/7.evaluation.tex` | methodology and all main results | `tab:hardware-config`, `tab:accuracy`, `fig:main`, `tab:setup_compare`, `fig:DSE`, `fig:ablation`, `fig:mem_access`, `tab:int8_degrade`, `tab:image`, `fig:worst_case` | `algorithm/run_*.sh`, `simulator/run_*.sh`, notebooks under `evaluation_scripts/plot_scripts/ipynb_src/`, `simulator/utils/analysis.py` |
| `doc/paper/tex/8.conclusion.tex` | final restatement of contributions | none | summary only |
| `doc/paper/tex/acknowledgement.tex` | acknowledgements | none | not technically relevant |
| `doc/paper/tex/artifact_appendix.tex` | artifact scope, environment, workflow, expected results | none | `README.md`, `algorithm/README.md`, `simulator/README.md`, dependency files |

## Why This File Map Matters

- Direct observation: the paper source is modular and mirrors the conceptual contribution split much better than the code tree does.
- Reasoned inference: if you want to keep paper and code mentally aligned, `doc/paper/tex/4.overview.tex`, `doc/paper/tex/5.semantic.tex`, `doc/paper/tex/6.vector.tex`, and `doc/paper/tex/7.evaluation.tex` are the four most important TeX files.
- Reusable lesson: storing paper source in the same repo is valuable because it exposes author intent, terminology, and artifact expectations in a version-controlled form.
