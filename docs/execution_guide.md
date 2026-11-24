# The AI Scientist v1 — 环境搭建、运行与排错

> 基于根 `README.md`、`requirements.txt`、`launch_scientist.py`、`ai_scientist/*.py` 核验。

---

## 1. 硬件与系统要求

- **Linux + NVIDIA GPU + CUDA + PyTorch**（源码用 `torch.cuda.device_count()`；README 明确 CPU-only 不可行、其它 OS 需大量调整）。
- 需要本地安装 **LaTeX**（`pdflatex`、`bibtex`、`chktex`），因为论文用 `pdflatex` 编译、`chktex` 做语法检查。
- 多 GPU 可用 `--parallel` 并行；GPU 不足会自动下调并行度。**无 GPU 时不要加 `--parallel`。**

---

## 2. 环境与依赖安装

```bash
conda create -n ai_scientist python=3.11
conda activate ai_scientist
sudo apt-get install texlive-full          # 很慢，可能需要长按 Enter
pip install -r requirements.txt
```

`requirements.txt` 内容（无版本锁定）：
```
anthropic, aider-chat, backoff, openai, google-generativeai,   # LLM/Aider
matplotlib, pypdf, pymupdf4llm,                                # PDF/绘图
torch, numpy, transformers, datasets, tiktoken, wandb, tqdm   # 通用
```

按所用模型/引擎额外安装：
- Bedrock Claude：`pip install anthropic[bedrock]` + `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_REGION_NAME`
- Vertex Claude：`pip install google-cloud-aiplatform anthropic[vertex]` + `CLOUD_ML_REGION`/`ANTHROPIC_VERTEX_PROJECT_ID`/`VERTEXAI_LOCATION`/`VERTEXAI_PROJECT`
- 文献检索走 OpenAlex：`pip install pyalex` + `export OPENALEX_MAIL_ADDRESS=...`，运行时加 `--engine openalex`
- DeepSeek/OpenRouter/Gemini：分别用 `DEEPSEEK_API_KEY`/`OPENROUTER_API_KEY`/`GEMINI_API_KEY`

### API Key 汇总
| 变量 | 用途 |
| --- | --- |
| `OPENAI_API_KEY` | GPT-4o / o1 / o3；**评审固定用 GPT-4o，因此即使主模型不是 OpenAI 也需要它** |
| `ANTHROPIC_API_KEY` | Claude 主模型 |
| `DEEPSEEK_API_KEY` / `OPENROUTER_API_KEY` / `GEMINI_API_KEY` | 对应供应商 |
| `S2_API_KEY` | Semantic Scholar（可选，提高文献检索速率） |
| `OPENALEX_MAIL_ADDRESS` | OpenAlex（可选，`--engine openalex`） |

```bash
export OPENAI_API_KEY="..."
export ANTHROPIC_API_KEY="..."
export S2_API_KEY="..."     # 可选
```

> 安全：系统会执行 LLM 生成的代码。根 README 提供 `experimental/Dockerfile`（社区贡献），建议容器化并限制网络。

---

## 3. 命令行参数（`launch_scientist.py --help`）

| 参数 | 类型/默认 | 说明 |
| --- | --- | --- |
| `--experiment` | str, `nanoGPT` | 用 `templates/<experiment>` 作为模板 |
| `--model` | str, `claude-3-5-sonnet-20240620`，choices=`AVAILABLE_LLMS` | 想法/实验/写作模型（**评审不受此参数控制**） |
| `--writeup` | str, `latex`（唯一合法值） | 写作格式 |
| `--num-ideas` | int, `50` | 生成想法数量 |
| `--parallel` | int, `0` | 并行进程数；0=顺序；>0 按 GPU 缩放 |
| `--gpus` | str, `None` | 逗号分隔 GPU id，如 `0,1,2`；未指定用全部可见 GPU |
| `--improvement` | flag | 评审后再改进论文并重评 |
| `--skip-idea-generation` | flag | 直接加载已有 `ideas.json` |
| `--skip-novelty-check` | flag | 跳过新颖性检查（注意 ideas.json 需已有 novel 字段，见下） |
| `--engine` | str, `semanticscholar`，choices=`semanticscholar/openalex` | 文献检索引擎 |

---

## 4. 运行前准备（每个模板）

主仓库论文用的是 nanoGPT / 2d_diffusion / grokking 三个模板（其余为社区贡献）。**必须先生成基线 `run_0/final_info.json`**，否则 `do_idea` 读基线会失败：

```bash
# nanoGPT
python data/enwik8/prepare.py
python data/shakespeare_char/prepare.py
python data/text8/prepare.py
cd templates/nanoGPT
python experiment.py --out_dir run_0 && python plot.py

# 2d_diffusion:  pip install NPEET scikit-learn 后同上
# grokking:      pip install einops 后同上
```

---

## 5. 运行命令

```bash
conda activate ai_scientist

# 最小验证：2 个想法，快速模型
python launch_scientist.py --model "gpt-4o-2024-05-13" --experiment nanoGPT_lite --num-ideas 2

# 论文常用：Claude Sonnet
python launch_scientist.py --model "claude-3-5-sonnet-20241022" --experiment nanoGPT_lite --num-ideas 50

# 多 GPU 并行 + 指定卡 + 改进 + OpenAlex 检索
python launch_scientist.py --model claude-3-5-sonnet-20241022 \
  --experiment grokking --num-ideas 20 --parallel 2 --gpus 0,1 \
  --improvement --engine openalex
```

其它可用入口（非主流程）：
```bash
# 只生成想法 / 只检查新颖性
python ai_scientist/generate_ideas.py --experiment nanoGPT --model gpt-4o-2024-05-13 --check-novelty

# 对已有结果目录重跑写作
python ai_scientist/perform_writeup.py --folder results/nanoGPT/<时间戳_名字> --model gpt-4o-2024-05-13
# 仅重新编译（不写）
python ai_scientist/perform_writeup.py --folder <...> --no-writing

# 独立跑 LLM 评审（见根 README 的 Python 片段）与 ICLR 基准批处理
cd review_iclr_bench && python iclr_analysis.py --num_reviews 500 --batch_size 100 ...
```

---

## 6. 输出与监控

- 结果目录：`results/<experiment>/<YYYYmmdd_HHMMSS>_<Name>/`。
- 并行模式：每个想法的 stdout/stderr 被重定向到该目录的 `log.txt`。
- 进度日志：控制台打印 `*Starting idea* / *Starting Experiments* / *Starting Writeup* / *Starting Review* / FINISHED IDEA` 及时间戳。
- 关键中间产物：`notes.txt`（实验记忆）、`ideas.json`（含 novel）、`review.txt`、`<Name>.pdf`。

---

## 7. 排错手册

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| 启动即 `SystemExit` 且提示缺 LaTeX 依赖 | `check_latex_dependencies()` 找不到 `pdflatex`/`chktex` | 安装 `texlive-full` 与 chktex，确保 `which pdflatex` 有输出 |
| `Folder ... already exists` assert | 同一秒内同名 idea 目录（极少）或残留 | 清理 `results/<exp>/` 下同名目录 |
| `open run_0/final_info.json` 失败 | 未做模板基线准备 | 按第 4 节跑 `python experiment.py --out_dir run_0` |
| 所有想法 `novel` 判定异常 / `KeyError: 'novel'` | 用了 `--skip-novelty-check` 但 ideas.json 无 novel 字段，或检索 API 限流 | 去掉该 flag 重跑新颖性检查；或手动给 ideas.json 每条加 `"novel": true`；设置 S2_API_KEY / 换 `--engine openalex` |
| 实验阶段反复 "Run failed" 直到放弃 | Aider 改的代码跑不通；stderr 已写进下一轮 prompt | 查看 `log.txt`、`run_<n>.py` 快照与 `notes.txt`；确认模板依赖已装；`MAX_ITERS/MAX_RUNS` 可调 |
| "Max iterations reached / Not all experiments completed" | 连续 4 次 run 无进展 | 降 idea 复杂度、换更强模型、调 `perform_experiments.py` 常量 |
| PDF 未生成 / 为空 | `compile_latex` 超时（30s/命令）或 chktex 未收敛 | 看控制台 LaTeX stdout/stderr；手动进 `latex/` 跑 pdflatex；调 `generate_latex` 的 `timeout`/`num_error_corrections` |
| `No references.bib found in template.tex` | 模板缺 `filecontents{references.bib}` | 在模板 `latex/template.tex` 顶部补上 |
| 评审报错、无 review.txt | 评审固定用 `openai.OpenAI()`，缺 `OPENAI_API_KEY` 或模型不可达 | 设置 OPENAI_API_KEY（评审硬编码 gpt-4o，见 `do_idea`） |
| 并行模式 `ZeroDivisionError: integer modulo by zero` | 无 GPU 却设了 `--parallel>0` | 去掉 `--parallel` 或提供 GPU / 用 `--gpus` |
| 并行同时启动互相冲突 | 代码已 `time.sleep(150)` 错峰，但资源仍可能不够 | 减小 `--parallel`，检查 GPU 显存 |
| LLM 输出解析失败 "Failed to extract JSON" | 模型不遵守 ```json 格式 | 换更强模型；`extract_json_between_markers` 已有控制字符修复与 `{}` 兜底，必要时自定义解析 |

---

## 8. 成本与成功率（源自根 README）

- Claude Sonnet 3.5 每篇论文通常 < $15；DeepSeek Coder V2 更便宜。
- 成功率取决于模板、基础模型与想法复杂度；**Claude Sonnet 3.5 成功率最高，评审建议用 GPT-4o**（其它模型有"过于正面"或不遵守格式的问题）。
- 修改模型支持：改 `ai_scientist/llm.py` 的 `AVAILABLE_LLMS` 与 `create_client`；不建议用明显弱于 GPT-4 的模型。
