# 04 Algorithm Deep Dive

## Scope

- **Direct observation**: the algorithm side lives in `algorithm/`.
- **Direct observation**: the actual Focus implementation is in `algorithm/focus/main.py`, but it only becomes active when `algorithm/focus/interface.py` rewires model forwards inside the `lmms-eval` execution path.
- **Reasoned inference**: this subsystem is best understood as “evaluation harness + model surgery + Focus runtime state machine,” not as a standalone library call.

## Core Files

| File | Role |
| --- | --- |
| `algorithm/run_eval.py` | User-facing Python entry for evaluation, trace export, quantization flags, and accuracy CSV writing. |
| `algorithm/lmms-eval/lmms_eval/evaluator.py` | Underlying evaluation loop; triggers `apply_focus` / `apply_framefusion`. |
| `algorithm/focus/interface.py` | Model-family detection and forward-function replacement. |
| `algorithm/focus/main.py` | `Focus` class implementing SIC, SEC, trace buffering, and export. |
| `algorithm/focus/baseline_adaptiv.py` | Adaptiv baseline implementation and sparsity export. |
| `algorithm/focus/baseline_CMC.py` | CMC baseline implementation and sparsity export. |
| `algorithm/focus/utils.py` | shared helpers such as `AverageMeter`, CSV overwrite helpers, and attribute lookup. |
| `algorithm/focus/models/qwen2/modeling_qwen2.py` | Main patched Qwen2 decoder, attention, and MLP logic used by several supported models. |
| `algorithm/focus/models/llava_video/modeling_llava_video.py` | LLaVA-Video metadata extraction for Focus. |
| `algorithm/focus/models/minicpmv/modeling_minicpmv.py` | MiniCPM-V metadata extraction for Focus. |
| `algorithm/focus/models/qwen2_5_vl/modeling_qwen2_5_vl.py` | Qwen2.5-VL-specific patch path and metadata logic. |
| `algorithm/focus/configs/focus.csv` | Default Focus thresholds, selected SEC layers, and alpha schedule. |
| `algorithm/focus/configs/adaptiv.csv` | Default Adaptiv thresholds. |
| `algorithm/focus/configs/cmc.csv` | Default CMC thresholds. |

## The Core Abstraction

### What Focus Actually Is In Code

- **Direct observation**: `algorithm/focus/main.py` defines `class Focus(nn.Module)`.
- **Direct observation**: `Focus` is stateful. It stores:
  - similarity threshold
  - vector size
  - spatial and temporal block sizes
  - selected SEC layers
  - alpha schedule
  - model dimensions
  - trace export settings
  - sample-level sparse metrics and cached trace tensors
- **Reasoned inference**: the class is designed as a runtime controller that follows one sample through the model, rather than a pure functional sparsifier.

### Why The Statefulness Exists

- **Direct observation**: `Focus.prepare(...)` receives patch layout, frame count, token boundaries, and sequence length before decoder execution.
- **Direct observation**: `Focus.set_token_importance(...)` stores attention-derived token importance for later SEC pruning.
- **Direct observation**: `Focus.post_process(...)` finalizes per-sample statistics and writes trace artifacts only after full-sequence inference completes.
- **Reasoned inference**: Focus spans multiple layers and multiple moments in the forward pass, so it cannot be implemented as one isolated tensor transform.

## How Model Execution Is Modified

## 1. Evaluation Harness Entry

- **Direct observation**: `algorithm/run_eval.py` calls `lmms_eval.evaluator.simple_evaluate(...)`.
- **Direct observation**: `algorithm/lmms-eval/lmms_eval/evaluator.py::evaluate(...)` checks the CLI flags and calls:
  - `apply_framefusion(...)` for FrameFusion
  - `apply_focus(...)` for Focus, CMC, or Adaptiv

## 2. Model-Family Detection And Surgery

`algorithm/focus/interface.py::apply_focus(...)` branches on model family:

| Family | Detection | Metadata hook | Patched decoder stack |
| --- | --- | --- | --- |
| LLaVA-Video / LLaVA-OneVision | `isinstance(model, LlavaQwenForCausalLM)` | replace `prepare_inputs_labels_for_multimodal` | Qwen2 patched forwards |
| MiniCPM-V | `model.config.architectures[0] == "MiniCPMV"` | replace `get_vllm_embedding` | Qwen2 patched forwards |
| Qwen2.5-VL | `isinstance(model, Qwen2_5_VLForConditionalGeneration)` | replace top-level forward | Qwen2.5-VL patched forwards |

### Important Engineering Detail

- **Direct observation**: `replace_focus_forward(...)` in `algorithm/focus/interface.py` uses `MethodType(...)` to replace forward methods dynamically.
- **Direct observation**: it also preserves Hugging Face Accelerate hooks by checking `decoder_layer._hf_hook` and calling `add_hook_to_module(...)`.
- **Reasoned inference**: this is a careful monkeypatching pattern that avoids forking entire upstream model codebases unnecessarily.

## 3. Default Hyperparameter Injection

If the user leaves thresholds at sentinel values, `algorithm/focus/interface.py` pulls defaults from CSV config files:

- Focus defaults from `algorithm/focus/configs/focus.csv`
- Adaptiv defaults from `algorithm/focus/configs/adaptiv.csv`
- CMC defaults from `algorithm/focus/configs/cmc.csv`

For Focus, the CSV provides:

- `selected_layers`
- `alpha_list`
- `similarity_threshold`

### Example

- `llava_vid` on `videomme` defaults to `selected_layers=[3, 6, 9, 18, 26]` and `alpha_list=[0.4, 0.3, 0.2, 0.15, 0.1]` in `algorithm/focus/configs/focus.csv`.

## Where Sparse Trace Generation Happens

### Metadata Initialization

Metadata is collected before the decoder stack begins:

- `algorithm/focus/models/llava_video/modeling_llava_video.py`
- `algorithm/focus/models/minicpmv/modeling_minicpmv.py`
- `algorithm/focus/models/qwen2_5_vl/modeling_qwen2_5_vl.py`

These files compute:

- number of frames
- patch height / width
- token start and end positions
- frame, height, and width strides
- image-token length
- original sequence length

They then call `self.focus.prepare(...)`.

### Trace Tensor Allocation

- **Direct observation**: `Focus.prepare(...)` allocates `mask_zero`, `mask_similar`, and `group_idx` tensors when `export_focus_trace` is enabled.
- **Direct observation**: trace tensors are allocated for these operation names:
  - `q_proj`
  - `o_proj`
  - `query`
  - `gate_proj`
  - `down_proj`
- **Reasoned inference**: these are the operations the simulator models explicitly for Focus.

## Where The Model Execution Is Modified

### SIC: Similarity Concentration Around GEMMs

In `algorithm/focus/models/qwen2/modeling_qwen2.py`:

- `Qwen2SdpaAttention_focus_forward(...)` calls `self.focus(hidden_states, name="q_proj")` before Q/K/V projections.
- The same function calls `self.focus(query_states, is_attention=True, name="query")` on the attention query tensor.
- It calls `self.focus(attn_output, name="o_proj")` before the output projection.
- `Qwen2MLP_focus_forward(...)` calls `self.focus(x, name="gate_proj")` before the MLP expansion path and `self.focus(down_input, name="down_proj")` before the MLP reduction.

These calls route to `algorithm/focus/main.py::Focus.forward(...)`.

### What `Focus.forward(...)` Does

- **Direct observation**: it extracts the image-token slice using indices recorded in `prepare(...)`.
- **Direct observation**: for 4D attention inputs it reshapes heads into a 3D sequence-by-vector layout before concentration.
- **Direct observation**: it pads the hidden dimension if needed so it is divisible by `vector_size`.
- **Direct observation**: it calls `focus_similarity_concentration(...)`.
- **Direct observation**: it writes the updated tokens back into the original hidden-state tensor.

### SIC Core Logic

`algorithm/focus/main.py::focus_similarity_concentration(...)` does the heavy lifting:

- forms vector groups of width `vector_size`
- treats zeros as already-pruned vectors
- restricts matching candidates using `block_size`, `frame_block_size`, and optional `gemm_m_size`
- computes cosine similarity between candidate vectors
- keeps the best candidate above `similarity_threshold`
- merges vectors using a pointer-chasing / union-find style grouping scheme
- replaces each group with its average
- computes a sparsity metric
- writes `mask_zero`, `mask_similar`, and `group_idx` into the trace dict

### SEC: Semantic Concentration In Selected Decoder Layers

In `algorithm/focus/models/qwen2/modeling_qwen2.py::Qwen2DecoderLayer_focus_forward(...)`:

- after self-attention and residual connection, if the current layer index is in `self.focus.selected_layer`, the code:
  - updates `alpha` with `self.focus.update_alpha(...)`
  - recovers tokens if a previous SEC step had already dropped them
  - prunes the sequence using `self.focus.semantic_concentration(...)`
  - drops unselected tokens using `self.focus.drop_tokens(...)`

### How SEC Decides What To Keep

- **Direct observation**: `Qwen2SdpaAttention_focus_forward(...)` computes attention weights and calls `self.focus.set_token_importance(attn_weights)` when the current layer participates in SEC.
- **Direct observation**: `Focus.set_token_importance(...)` takes the max text-to-image attention over heads and over query tokens.
- **Direct observation**: `Focus.semantic_concentration(...)` keeps the top `k = int(image_token_length * alpha)` image tokens, while preserving text/query tokens and updating position embeddings and attention masks consistently.
- **Reasoned inference**: SEC is the code-level implementation of “importance-driven visual-token pruning.”

## Where Accuracy Evaluation Is Performed

- **Direct observation**: model task execution and metrics come from `algorithm/lmms-eval/`.
- **Direct observation**: `algorithm/run_eval.py::get_score_from_results(...)` converts task-specific result structures into one scalar score used by the paper tables.
- **Direct observation**: `algorithm/run_eval.py::save_score_to_main_csv(...)` writes the comparison table columns `Dense`, `FF`, `Adaptiv`, `CMC`, `Focus`, `INT8_Dense`, and `INT8_Focus`.
- **Reasoned inference**: `lmms-eval` remains the metric engine, while `run_eval.py` is the paper-facing summarizer.

## Important Data Structures And File Formats

### In-Memory Focus Trace

`Focus.info_dict` in `algorithm/focus/main.py` contains nested dicts:

- `mask_zero[name][layer]`
- `mask_similar[name][layer]`
- `group_idx[name][layer]`

### Serialized Focus Trace

- Written by `torch.save(...)` in `Focus.post_process(...)`
- Saved to `trace_dir/trace_name.pth`
- Consumed by `simulator/models/sparse_info.py`

### Metadata CSV

Written by `Focus.post_process(...)` to `meta_data.csv` with columns:

- `Model`
- `Dataset`
- `Sequence length`
- `Num frames`
- `Num patches`
- `Median index`

### Accuracy CSV

Written by `algorithm/run_eval.py::save_score_to_main_csv(...)` with columns:

- `Models`
- `Dataset`
- `Metric`
- `Dense`
- `FF`
- `Adaptiv`
- `CMC`
- `Focus`
- `INT8_Dense`
- `INT8_Focus`

### Baseline Sparsity CSVs

- `adaptiv_sparsity.csv` from `algorithm/focus/baseline_adaptiv.py`
- `cmc_sparsity.csv` from `algorithm/focus/baseline_CMC.py`

## Baseline Implementations

### Adaptiv

- **Direct observation**: `algorithm/focus/baseline_adaptiv.py` defines `class Adaptiv`.
- **Direct observation**: it stores spatial/token metadata in `prepare(...)`.
- **Direct observation**: its `forward(...)` compares each image token to its top and left neighbors using sign similarity and merges/prunes tokens accordingly.
- **Direct observation**: `post_process(...)` writes coarse sparsity to `adaptiv_sparsity.csv`.
- **Reasoned inference**: the implementation is adapted to the same patching framework as Focus, which makes simulator comparison easier.

### CMC

- **Direct observation**: `algorithm/focus/baseline_CMC.py` defines `class CMC`.
- **Direct observation**: it uses `detect_non_informative(...)` during `prepare(...)` and `approximate_non_informative(...)` during `forward(...)`.
- **Direct observation**: it tracks separate sparsity terms for linear layers, query computation, and attention scores, then writes them to `cmc_sparsity.csv`.
- **Reasoned inference**: CMC is modeled at a coarser abstraction than Focus because the simulator only consumes its aggregate sparsity rates.

### FrameFusion

- **Direct observation**: FrameFusion is integrated through `algorithm/lmms-eval/lmms_eval/framefusion/interface.py`.
- **Direct observation**: it follows the same monkeypatch pattern as Focus, but lives inside the `lmms-eval` subtree rather than `algorithm/focus/`.
- **Unclear**: FrameFusion internals were not the main focus of this repository; the repo primarily exposes it as a comparison method.

## Paper Method To Code Mapping

### Semantic concentration

- **Direct observation**: implemented by `Focus.semantic_concentration(...)` in `algorithm/focus/main.py`.
- **Direct observation**: activated only in selected decoder layers by the patched decoder-layer forward in `algorithm/focus/models/qwen2/modeling_qwen2.py`.

### Similarity concentration

- **Direct observation**: implemented by `Focus.focus_similarity_concentration(...)` in `algorithm/focus/main.py`.
- **Direct observation**: injected before major GEMM sites in `algorithm/focus/models/qwen2/modeling_qwen2.py`.

### Hardware-friendly sparse trace

- **Direct observation**: implemented by the trace tensor allocation in `Focus.prepare(...)` and the writeout path in `Focus.post_process(...)`.
- **Reasoned inference**: this is the code-level bridge that makes the paper’s hardware claims testable without rerunning the VLM.

## Reusable Engineering Patterns

- **Direct observation**: model surgery is centralized in `algorithm/focus/interface.py`, not spread across scripts.
- **Direct observation**: algorithm hyperparameters are separated into CSV lookup tables in `algorithm/focus/configs/`.
- **Direct observation**: paper-facing metrics are emitted in stable CSV formats from `algorithm/run_eval.py`.
- **Reasoned inference**: this is a good pattern for future research code. Keep model instrumentation, evaluation, and downstream artifact contracts separate.

## Honest Caveats

- **Direct observation**: only certain model families are explicitly supported in `algorithm/focus/interface.py`.
- **Direct observation**: Qwen2.5-VL support is more version-sensitive than the main video-model path.
- **Unclear**: the baseline implementations are integrated and runnable, but exact equivalence to the original baseline repositories is not fully auditable from this repo alone.
