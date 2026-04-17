# 15 Architecture ASCII Map

## Repository-Wide Flow

```text
User Command
  |
  +--> algorithm/run_eval.py
         |
         +--> lmms_eval.evaluator.simple_evaluate(...)
                |
                +--> model adapter in algorithm/lmms-eval/lmms_eval/models/
                |
                +--> focus.interface.apply_focus(...)
                       |
                       +--> model-specific metadata hook
                       |      |
                       |      +--> Focus.prepare(...)
                       |
                       +--> patched decoder / attention / MLP forwards
                              |
                              +--> Focus.forward(...)  [SIC]
                              +--> Focus.set_token_importance(...)
                              +--> Focus.semantic_concentration(...) [SEC]
                              +--> Focus.post_process(...)
                                       |
                                       +--> focus_main/*.pth
                                       +--> meta_data.csv
                                       +--> accuracy.csv / DSE accuracy CSVs
```

## Algorithm To Simulator Bridge

```text
algorithm output/
  |
  +--> focus_main/<model>_<dataset>.pth
  +--> focus_int8/<model>_<dataset>.pth
  +--> adaptiv_sparsity.csv
  +--> cmc_sparsity.csv
  +--> meta_data.csv
         |
         v
simulator/main.py
  |
  +--> ModelConfig(meta_data.csv)
  +--> SparseInfo(trace or sparsity CSV)
  +--> Accelerator(hardware config)
  +--> Simulator(run_focus / run_dense / run_adaptiv / run_cmc)
         |
         +--> main_focus.csv / main_dense.csv / ...
         +--> dse_*.csv
         +--> detailed_power_area_breakdown.csv
```

## Plotting Flow

```text
algorithm output CSVs     simulator output CSVs     jetson_stats/figure9_gpu.csv
          |                         |                             |
          +-------------------------+-----------------------------+
                                    |
                                    v
               evaluation_scripts/plot_scripts/ipynb_src/*.ipynb
                                    |
                                    +--> paper figures / tables
```

## Focus Runtime Structure

```text
Supported VLM
  |
  +--> multimodal preprocessing hook
         |
         +--> Focus.prepare(patch/frame/token metadata)
  |
  +--> decoder layers
         |
         +--> self attention
         |      |
         |      +--> Focus(q_proj input)
         |      +--> Focus(query tensor)
         |      +--> attention weights -> token importance
         |      +--> Focus(o_proj input)
         |
         +--> selected SEC layers
         |      |
         |      +--> Focus.semantic_concentration(...)
         |
         +--> MLP
                |
                +--> Focus(gate_proj input)
                +--> Focus(down_proj input)
  |
  +--> Focus.post_process()
```

## Focus Accelerator Structure As Reflected In Simulator + RTL

```text
                 +--------------------+
                 |   Input / Weight   |
                 |      Buffers       |
                 +----------+---------+
                            |
                            v
                 +--------------------+
                 |   Systolic Array   |
                 | traditional_* RTL  |
                 +----------+---------+
                            |
                            +-------------------+
                            |                   |
                            v                   v
                 +--------------------+   +--------------------+
                 |   SIC detection    |   |   SEC ranking      |
                 | cosine_similarity  |   |   max_unit_fp16    |
                 | inv_magnitude      |   | importance buffer  |
                 | average_update     |   +--------------------+
                 | max / accumulator  |
                 +----------+---------+
                            |
                            v
                 +--------------------+
                 | layouter / sim map |
                 | similarity table   |
                 | concentrate_out    |
                 +----------+---------+
                            |
                            v
                 +--------------------+
                 |    output buffer    |
                 +--------------------+
```

## Baseline Hardware Map

```text
Dense:
  traditional_systolic.v + traditional_mac.v

Adaptiv:
  adaptiv_array.v + shared SFU blocks

CMC:
  traditional_systolic.v + cmc_codec_pe.v + cmc_addertree_4to1.v + shared SFU blocks
```

## Key Boundary Reminder

- **Direct observation**: the simulator consumes traces and RTL-derived CSV summaries.
- **Direct observation**: the simulator does not directly run the RTL.
- **Reasoned inference**: the repository’s methodology is “software traces drive architectural simulation; synthesized RTL calibrates area/power,” not “one integrated hardware/software co-simulation stack.”
