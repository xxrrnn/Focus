# 02 构建与环境

## 论文里怎么说环境，仓库里怎么落地

环境与实验流程在两个地方都有描述：

- 仓库文档：`README.md`
- 论文 artifact 附录：`doc/paper/tex/artifact_appendix.tex`

前者更偏工程安装步骤，后者更偏论文复现预期。

## 依赖分层

| 层次 | 关键文件 | 作用 |
| --- | --- | --- |
| 顶层流程 | `README.md`, `.gitmodules` | clone、submodule、安装顺序 |
| Focus 包本身 | `algorithm/focus/pyproject.toml` | Focus 运行依赖与 transformers 版本 |
| 评测框架 | `algorithm/lmms-eval/pyproject.toml`, `algorithm/lmms-eval/requirements.txt` | 多模态评测栈 |
| LLaVA 依赖 | `3rd_party/LLaVA-NeXT`, `algorithm/lmms-eval/lmms_eval/models/llava_vid.py` 等 | LLaVA 系模型适配器需要它 |
| ScaleSim | `3rd_party/scalesim`, `simulator/core/simulator_comp.py` | GEMM 周期建模 |
| CACTI | `3rd_party/cacti`, `simulator/memory/cacti.py` | SRAM 面积/能耗建模 |

## 子模块

`.gitmodules` 中注册了：

- `3rd_party/scalesim`
- `3rd_party/cacti`
- `3rd_party/DRAMsim3`
- `3rd_party/LLaVA-NeXT`

## 每个子模块在仓库中到底怎么用

- **直接观察**：`3rd_party/LLaVA-NeXT` 被 `algorithm/lmms-eval/lmms_eval/models/llava_vid.py` 与 `llava_onevision.py` 的 `llava.*` 导入路径依赖。
- **直接观察**：`3rd_party/scalesim` 被 `simulator/core/simulator_comp.py` 直接导入。
- **直接观察**：`simulator/memory/cacti.py` 会寻找 `3rd_party/cacti/cacti` 可执行文件。
- **不明确**：`3rd_party/DRAMsim3` 虽然被要求构建，但当前主模拟流程并没有直接调用它；代码里只在 `simulator/arch/accelerator.py` 里用了一个 DRAM 常数。

## 版本敏感点

### Python

- `README.md` 推荐 Python 3.11
- `algorithm/focus/pyproject.toml` 要求 `>=3.11`
- `doc/paper/tex/artifact_appendix.tex` 也写了 Python 3.11+

### transformers

- `algorithm/focus/pyproject.toml` 约束为 `>=4.48.2,<4.50.0`
- `main` extra 固定 `4.48.2`
- `qwen25_vl` extra 固定 `4.49.0`
- `algorithm/README.md` 明确要求在 Qwen2.5-VL 与主视频 VLM 流程之间切换环境

### 这说明什么

- **直接观察**：视频主路径和 Qwen2.5-VL 路径并不是在同一套完全稳定的包版本上运行。
- **推断**：这是仓库最明显的复现风险之一。

## 各子系统分别需要什么

| 子系统 | 需要的东西 |
| --- | --- |
| `algorithm/` | Python 3.11、HuggingFace 访问、GPU、`LLaVA-NeXT`、`lmms-eval`、`focus`、`decord`/`av` 等多模态依赖 |
| `simulator/` | Python、`scalesim`、在非常规 buffer 配置下需要 CACTI、以及 algorithm 产出的 traces 或 example outputs |
| `rtl/` | 仓库没有给出完整可运行的综合/仿真流，只提供源文件 |
| `evaluation_scripts/` | Jupyter、pandas、matplotlib、以及前面阶段生成的 CSV |

## 原生 / 外部构建步骤

按 `README.md` 的建议顺序：

1. 初始化 submodule
2. 创建 Python 环境
3. 在 `3rd_party/LLaVA-NeXT` 里 `pip install -e .`
4. 在 `3rd_party/scalesim` 里 `pip install -e .`
5. 在 `3rd_party/cacti` 里 `make`
6. 在 `3rd_party/DRAMsim3` 里 `make`
7. 在 `algorithm/lmms-eval` 里 `pip install -e .`
8. 在 `algorithm/focus` 里 `pip install -e '.[main]'`

## 常见失败点

| 风险 | 证据 |
| --- | --- |
| transformers 版本不对 | `algorithm/focus/pyproject.toml`, `algorithm/README.md` |
| CACTI 没编译 | `simulator/memory/cacti.py` |
| ScaleSim 没安装 | `simulator/core/simulator_comp.py` |
| LLaVA 没安装 | `algorithm/lmms-eval/lmms_eval/models/llava_vid.py` 等 |
| 模拟器并行跑出问题 | `simulator/README.md`，以及 `simulator/core/simulator_comp.py` 会覆盖共享配置文件 |
| 全量精度评估时间极长 | `algorithm/README.md`, `doc/paper/tex/artifact_appendix.tex` |

## 论文说法与仓库实现的一致性

| 主题 | 论文来源 | 仓库证据 | 结论 |
| --- | --- | --- | --- |
| 需要 A100 级 GPU | `doc/paper/tex/artifact_appendix.tex` | `README.md` 与模型适配器的重型依赖 | 一致 |
| PyTorch 实现算法 | `doc/paper/tex/7.evaluation.tex` | `algorithm/focus/`, `algorithm/run_eval.py` | 一致 |
| 基于 ScaleSim 的模拟器 | `doc/paper/tex/7.evaluation.tex` | `simulator/core/simulator_comp.py` | 直接可见 |
| DRAMsim3 建模 DRAM 能耗 | `doc/paper/tex/7.evaluation.tex` | 代码里主要表现为 DRAM 常数 | 只部分可见 |
| RTL 用 DC 综合 | `doc/paper/tex/7.evaluation.tex` | `rtl/README.md`, `simulator/arch/*_rtl.csv` | 有证据，但流程缺失 |

## 工程上值得借鉴的点

- **直接观察**：一旦 traces 生成完成，后续大部分硬件研究工作都可以脱离大模型环境，在 `simulator/` 与 notebook 层进行。
- **推断**：这是一种非常值得复用的研究工程模式：把最脆弱的依赖集中在“trace 生成”阶段，而不是让整个项目都依赖同样重的环境。
