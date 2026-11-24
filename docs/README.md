# The AI Scientist (v1) — 技术文档地图

> 本目录的文档全部基于**当前仓库真实源码**逐项阅读后编写，目标是让一个对本项目零上下文的 AI/工程师能直接在此代码库上修改功能、定位并排查运行故障。
>
> 所有函数、类、文件名、命令行参数、配置项均来自源码核验；文中用 `文件:行号` 标注关键定义位置，便于跳转。

---

## 1. 项目定位（一句话）

**The AI Scientist v1 是一个「模板驱动」的全自动科研流水线**：给定一个人工编写好的实验模板目录（代码 + 数据 + LaTeX 模板 + 种子想法），LLM 批量提出研究想法、经新颖性检查筛选后，用 **Aider** 反复修改实验代码并真实运行实验，最后自动撰写 LaTeX 论文、调用 LLM 评审、并可选地根据评审意见二次改进。

其核心卖点是 **端到端自动化**（idea → experiment → paper → review），核心约束是 **强依赖人工模板**（每个研究领域都要先提供一个可运行的 `experiment.py` + `plot.py` + `prompt.json` + `seed_ideas.json` + `latex/template.tex`）。

安全提示（源自根 `README.md`）：本系统会**真实执行 LLM 编写的代码**，存在危险依赖、联网、派生进程等风险，务必在容器/受限网络环境中运行。

---

## 2. 与 v2 的关键差别（帮助选错仓库时快速纠正）

| 维度 | v1（本仓库） | v2（`AI-Scientist-v2`） |
| --- | --- | --- |
| 想法来源 | 基于模板的 `experiment.py` + `seed_ideas.json` 生成想法 | 无模板、基于 workshop 描述 + 工具调用生成想法 |
| 实验组织 | 每个想法线性执行（最多 5 次 run） | AIDE 风格的 **BFTS best-first 树搜索** + 4 阶段推进 |
| 代码修改 | Aider（`edit_format="diff"`） | 自建 `ParallelAgent`/`MinimalAgent` 生成整段代码 + 子进程解释器执行 |
| 写作 | 逐章节 `SEARCH/REPLACE` + 引用补全，ICLR2024 模板 | 8 页 normal / 4 页 ICBINB 双模式，多模型分工 |
| 评审 | LLM 文本评审（可选改进） | LLM 文本评审 + **VLM 图像/标题/正文一致性评审** |
| 入口 | `launch_scientist.py` | `launch_scientist_bfts.py` + `bfts_config.yaml` |

若你需要的是树搜索、无模板、VLM 审稿，请转到 v2 文档。

---

## 3. 文档地图（阅读顺序建议）

| 文档 | 内容 | 什么时候读 |
| --- | --- | --- |
| [`architecture.md`](architecture.md) | 系统分层、进程/模型分工、核心数据结构与 Mermaid 架构图 | 想理解系统全貌、模块边界与调用关系时 |
| [`pipeline_and_dataflow.md`](pipeline_and_dataflow.md) | 从模板依赖 → 想法生成 → 新颖性检查 → Aider 改代码执行 → 写作 → 评审 → 改进的完整数据流与每步的输入输出文件 | 想改流程、加阶段、理解中间产物时 |
| [`modules_reference.md`](modules_reference.md) | 每个核心模块的类/函数清单、入参出参、副作用、行号 | 要动手改某个函数、接新模型/新模板时 |
| [`execution_guide.md`](execution_guide.md) | GPU/环境/依赖、API Key、各模板准备、运行命令、常见故障排查 | 部署、复现实验、排错时 |
| [`code_trace_and_prompts.md`](code_trace_and_prompts.md) ★深度 | `launch_scientist.py` 入口逐行追踪、**全部 LLM/Aider Prompt 原文**、3 个内置模板（nanoGPT/2d_diffusion/grokking）的 `experiment.py`/`plot.py` 接口契约、LaTeX 变量注入格式 | 要**改 Prompt、写新模板、逐行调试调用链、核对真实数据契约**时（零上下文二次开发首选） |
| [`troubleshooting.md`](troubleshooting.md) ★深度 | 工程避坑：Aider 补丁定位失败、Semantic Scholar 限流、LaTeX 缺包/pdflatex 失败、CUDA OOM、断点续跑、逐项「症状→根因→定位→修复」 | 运行报错、想按证据层级快速定位故障时 |

---

## 4. 顶层目录速览（与流水线相关部分）

```
AI-Scientist/
├── launch_scientist.py          # 主入口：想法→实验→写作→评审→改进的全流程编排
├── ai_scientist/
│   ├── llm.py                   # 统一 LLM 客户端/调用/JSON 解析/model 白名单
│   ├── generate_ideas.py        # 想法生成 + 新颖性检查（Semantic Scholar/OpenAlex）
│   ├── perform_experiments.py   # Aider 改代码 + 真实运行 experiment.py / plot.py
│   ├── perform_review.py        # PDF 解析 + LLM 评审 + 改进
│   └── perform_writeup.py       # 逐章节 LaTeX 写作 + 引用补全 + 编译
├── templates/<experiment>/      # 每个研究领域的模板（核心依赖，见 pipeline 文档）
│   ├── experiment.py            # 基线实验脚本，接受 --out_dir，产出 run_i/final_info.json
│   ├── plot.py                  # 读 run_i 结果画图，产出 *.png
│   ├── prompt.json              # {"system": ..., "task_description": ...}
│   ├── seed_ideas.json          # 种子想法列表
│   ├── ideas.json               # 运行时生成/覆盖：所有想法 + novel 标记
│   └── latex/template.tex       # ICLR2024 论文模板（含 references.bib）
├── results/<experiment>/        # 每个想法展开为一个时间戳目录，存实验+论文+评审产物
├── data/                        # 模板所需原始数据集（如 enwik8/shakespeare_char）
├── review_iclr_bench/           # 评审基准数据与批处理脚本（iclr_analysis.py 等）
├── review_ai_scientist/         # 已生成的 AI 论文评审样例与 notebook
├── example_papers/              # 论文样例 PDF
├── experimental/Dockerfile      # 社区贡献的容器化文件
└── requirements.txt             # Python 依赖（无版本锁定）
```

> 注意：根 `README.md` 中提到的 `review_iclr_bench/iclr_analysis.py`、`data/enwik8/prepare.py` 等文件用于基准评测与数据准备，不在 `ai_scientist/` 核心包内；本套件聚焦于 `launch_scientist.py` + `ai_scientist/` 主流水线。

---

## 5. 一句话理解一次运行

```
python launch_scientist.py --model <LLM> --experiment <模板名> --num-ideas N
   ↓
generate_ideas() 生成 N 个想法 → check_idea_novelty() 打 novel 标记
   ↓
对每个 novel 想法 do_idea()：复制模板到 results/<exp>/<时间戳>_<Name>/
   ↓
perform_experiments()：Aider 改 experiment.py，跑最多 5 轮 python experiment.py --out_dir=run_i
   ↓
perform_writeup()：Aider 逐节写 latex/template.tex → generate_latex() 编译出 <Name>.pdf
   ↓
perform_review()：读 PDF 文本 → LLM 多评审集成 → review.txt
   ↓
（可选 --improvement）perform_improvement() + 重新编译 + 二次评审 → review_improved.txt
```

想深入任一环节，请按上表进入对应文档。

> **零上下文 AI / 二次开发推荐路线**：先读 [`code_trace_and_prompts.md`](code_trace_and_prompts.md) 建立源码级调用链与真实 Prompt/契约认知 → 用 [`troubleshooting.md`](troubleshooting.md) 按证据层级排错 → 需要函数签名时查 [`modules_reference.md`](modules_reference.md)。这两份深度文档与既有文档交叉引用，行号锚点均可用 `grep -n` 复核。
