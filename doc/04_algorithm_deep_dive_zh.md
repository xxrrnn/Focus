# 04 算法实现深读

## 论文锚点

算法方法主要来自：

- `doc/paper/tex/3.motivation.tex`
- `doc/paper/tex/5.semantic.tex`
- `doc/paper/tex/6.vector.tex`

## 核心文件

- `algorithm/run_eval.py`
- `algorithm/lmms-eval/lmms_eval/evaluator.py`
- `algorithm/focus/interface.py`
- `algorithm/focus/main.py`
- `algorithm/focus/models/qwen2/modeling_qwen2.py`
- `algorithm/focus/models/*/` 下的模型特定元数据准备逻辑

## 论文机制到代码机制

| 论文机制 | 代码证据 | 状态 |
| --- | --- | --- |
| 语义引导 token 剪枝 | `Focus.set_token_importance(...)`、`Focus.semantic_concentration(...)`、`algorithm/focus/models/qwen2/modeling_qwen2.py` 中的 SEC 触发逻辑 | fully implemented |
| block-level 局部相似范围 | `Focus.focus_similarity_concentration(...)` 中的 `block_size`、`frame_block_size` 和局部候选构造 | 已实现，但不是独立的软件模块 |
| vector-level 冗余移除 | `Focus.focus_similarity_concentration(...)` 中的 `vector_size` 分块 | fully implemented |
| multilevel concentration | SEC + 带 block 范围约束的 vector 匹配组合 | fully implemented，但软件结构与论文叙事不同 |

## Focus 是怎么注入模型的

- **直接观察**：`algorithm/lmms-eval/lmms_eval/evaluator.py` 在 `--focus`、`--CMC`、`--adaptiv` 时都会调用 `apply_focus(...)`。
- **直接观察**：`algorithm/focus/interface.py` 会根据模型家族替换：
  - 多模态元数据准备函数
  - decoder forward
  - attention forward
  - MLP forward
- **推断**：这其实就是论文里“算法-架构协同设计”在软件实现侧的真实边界。

## SEC 在代码里长什么样

### 论文里的说法

`doc/paper/tex/5.semantic.tex` 把 SEC 分成：

- streaming importance analyzer
- top-k sorter
- offset encoder

### 代码证据

- 文本到图像的重要度提取：`algorithm/focus/main.py::set_token_importance(...)`
- top-k 保留：`algorithm/focus/main.py::semantic_concentration(...)`
- 保留索引的维护与恢复：`retained_ids`、`recover_PE_and_AM(...)`、`recover_tokens(...)`、`drop_tokens(...)`

### 如何理解

- **直接观察**：软件代码实现了 SEC 的算法效果。
- **关键推断**：论文里的 bubble sorter、offset encoder 等硬件子模块，在软件中并没有一一对应成独立类/文件，而是折叠成张量运算和索引维护逻辑。

## SIC 在代码里长什么样

### 论文里的说法

`doc/paper/tex/6.vector.tex` 描述了：

- Similarity Gather
- convolution-style layouter
- Similarity Scatter

### 代码证据

- 相似向量匹配与分组：`algorithm/focus/main.py::focus_similarity_concentration(...)`
- 各算子调用位置：`algorithm/focus/models/qwen2/modeling_qwen2.py`
- 为后续 gather/scatter 建模导出的 trace：`mask_zero`、`mask_similar`、`group_idx`

### 如何理解

- **直接观察**：算法代码确实实现了局部向量匹配和 group 构造。
- **关键推断**：论文里的 layouter / scatter 更偏硬件实现概念，在算法代码侧不是单独软件模块，而主要在后续模拟器和硬件模型里体现。

## 模型特定的元数据准备

论文里的 block / vector 机制只有在知道 patch、frame、token 范围之后才成立。对应代码在：

- `algorithm/focus/models/llava_video/modeling_llava_video.py`
- `algorithm/focus/models/minicpmv/modeling_minicpmv.py`
- `algorithm/focus/models/qwen2_5_vl/modeling_qwen2_5_vl.py`

这些文件最终都会调用 `self.focus.prepare(...)`。

## trace 导出

- **直接观察**：`algorithm/focus/main.py::prepare(...)` 负责分配 trace tensor。
- **直接观察**：`algorithm/focus/main.py::post_process(...)` 负责写出：
  - `meta_data.csv`
  - 每个模型/数据集对应的中位样本 `.pth` trace
- **论文对应**：这正是 `doc/paper/tex/7.evaluation.tex` 里“simulator accepts layer-wise sparse traces”那句话的代码落点。

## 基线实现

| 基线 | 文件 | 说明 |
| --- | --- | --- |
| Adaptiv | `algorithm/focus/baseline_adaptiv.py` | token-level 稀疏度导出，走同一套 patch 框架 |
| CMC | `algorithm/focus/baseline_CMC.py` | 输出 linear/query/attention-score 三类稀疏度 |
| FrameFusion | `algorithm/lmms-eval/lmms_eval/framefusion/interface.py` | 包装官方实现 |

## 哪些文件最体现科学贡献

- `algorithm/focus/main.py`
- `algorithm/focus/interface.py`
- `algorithm/focus/models/qwen2/modeling_qwen2.py`

这三类文件定义了 Focus 方法本身的行为。

## 哪些更偏基础设施

- `algorithm/run_*.sh`
- `lmms-eval` 的任务与日志框架
- `algorithm/run_eval.py` 里把分数写成 CSV 的逻辑

## 这一部分最值得学的工程方法

- **直接观察**：方法注入点集中在 `algorithm/focus/interface.py`，没有散在很多地方。
- **直接观察**：trace 导出在 `algorithm/focus/main.py` 内部完成，天然与真实模型执行绑定。
- **推断**：这是很强的研究工程模式。你以后做类似工作时，最好也把“模型补丁逻辑”“方法状态机”“artifact 导出”明确分层。

## 结论

- **直接观察**：算法侧完整实现了论文的 SEC + SIC 主线。
- **推断**：最大的论文-代码差异不在功能缺失，而在结构表达方式不同：论文把更多硬件子模块单独命名，而软件实现把它们折叠成更紧凑的张量逻辑。
