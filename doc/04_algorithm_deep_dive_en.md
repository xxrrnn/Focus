# 04 Algorithm Deep Dive

## Paper Anchor

The algorithmic method is spread across:

- `doc/paper/tex/3.motivation.tex`
- `doc/paper/tex/5.semantic.tex`
- `doc/paper/tex/6.vector.tex`

## Core Files

- `algorithm/run_eval.py`
- `algorithm/lmms-eval/lmms_eval/evaluator.py`
- `algorithm/focus/interface.py`
- `algorithm/focus/main.py`
- `algorithm/focus/models/qwen2/modeling_qwen2.py`
- model-specific metadata hooks in `algorithm/focus/models/*/`

## Paper Mechanism -> Code Mechanism

| Paper mechanism | Code evidence | Status |
| --- | --- | --- |
| semantic-guided token pruning | `Focus.set_token_importance(...)`, `Focus.semantic_concentration(...)`, SEC trigger in `algorithm/focus/models/qwen2/modeling_qwen2.py` | fully implemented |
| block-level local similarity scope | `block_size`, `frame_block_size`, local candidate generation in `Focus.focus_similarity_concentration(...)` | implemented, but not as a separate standalone module |
| vector-level similarity removal | vectorization by `vector_size` in `Focus.focus_similarity_concentration(...)` | fully implemented |
| multilevel concentration | combination of SEC + block-scoped vector matching | fully implemented, but software structure differs from the paper’s conceptual separation |

## How Focus Is Injected

- **Direct observation**: `algorithm/lmms-eval/lmms_eval/evaluator.py` calls `apply_focus(...)` when `--focus`, `--CMC`, or `--adaptiv` is enabled.
- **Direct observation**: `algorithm/focus/interface.py` detects model families and replaces:
  - multimodal metadata hooks
  - decoder forward
  - attention forward
  - MLP forward
- **Reasoned inference**: this is the real code-level embodiment of “algorithm-architecture co-design” on the software side.

## SEC In Code

### Paper claim

`doc/paper/tex/5.semantic.tex` describes:

- streaming importance analyzer
- top-k sorter
- offset encoder

### Code evidence

- text-to-image importance extraction: `algorithm/focus/main.py::set_token_importance(...)`
- top-k retain operation: `algorithm/focus/main.py::semantic_concentration(...)`
- retained index bookkeeping / recovery: `retained_ids`, `recover_PE_and_AM(...)`, `recover_tokens(...)`, `drop_tokens(...)`

### Interpretation

- **Direct observation**: the software code implements the algorithmic effect of SEC.
- **Reasoned inference**: paper sub-blocks such as “bubble sorter” and “offset encoder” are not represented as separate software modules; they are folded into tensor operations and retained-index bookkeeping.

## SIC In Code

### Paper claim

`doc/paper/tex/6.vector.tex` describes:

- Similarity Gather
- convolution-style layouter
- Similarity Scatter

### Code evidence

- similarity matching and grouping: `algorithm/focus/main.py::focus_similarity_concentration(...)`
- per-op invocation sites: `algorithm/focus/models/qwen2/modeling_qwen2.py`
- trace outputs needed for later gather/scatter modeling: `mask_zero`, `mask_similar`, `group_idx`

### Interpretation

- **Direct observation**: the algorithm code directly implements localized vector matching and group formation.
- **Reasoned inference**: paper hardware concepts like “layouter” and “scatter” are not separate algorithm-side software modules; they are mainly represented later in the simulator and hardware model.

## Model-Specific Metadata Preparation

This is where the code computes patch geometry and image-token ranges:

- `algorithm/focus/models/llava_video/modeling_llava_video.py`
- `algorithm/focus/models/minicpmv/modeling_minicpmv.py`
- `algorithm/focus/models/qwen2_5_vl/modeling_qwen2_5_vl.py`

These files eventually call `self.focus.prepare(...)`.

## Trace Export

- **Direct observation**: `algorithm/focus/main.py::prepare(...)` allocates trace tensors.
- **Direct observation**: `algorithm/focus/main.py::post_process(...)` saves:
  - `meta_data.csv`
  - one median `.pth` trace per model/dataset
- **Paper alignment**: this is the concrete implementation behind the evaluation methodology described in `doc/paper/tex/7.evaluation.tex`.

## Baselines

| Baseline | Files | Notes |
| --- | --- | --- |
| Adaptiv | `algorithm/focus/baseline_adaptiv.py` | token-level sparsity export, same patching framework |
| CMC | `algorithm/focus/baseline_CMC.py` | separate sparsity terms for linear/query/attention-score |
| FrameFusion | `algorithm/lmms-eval/lmms_eval/framefusion/interface.py` | wrapped official implementation |

## What Is Scientifically Central

- `algorithm/focus/main.py`
- `algorithm/focus/interface.py`
- `algorithm/focus/models/qwen2/modeling_qwen2.py`

These files define the actual method behavior.

## What Is Mostly Infrastructure

- shell wrappers in `algorithm/run_*.sh`
- `lmms-eval` task machinery
- score-to-CSV formatting in `algorithm/run_eval.py`

## Bottom Line

- **Direct observation**: the algorithm side fully implements the paper’s high-level SEC + SIC idea.
- **Reasoned inference**: the biggest paper-to-code mismatch is structural, not functional: the paper separates more named submodules than the software does.
