# The AI Scientist v1 — 工程避坑与排错指南

> 基于真实源码与文件契约核验；每项给出**症状 → 源码根因 → 定位方法 → 修复**。
> 行号锚点可用 `grep -n` 复核。配套：[`code_trace_and_prompts.md`](code_trace_and_prompts.md)（调用链/Prompt/契约）、[`execution_guide.md`](execution_guide.md)（环境与命令）、[`architecture.md`](architecture.md)。

---

## 0. 先建立"证据层级"（排错前必读）

v1 是**文件状态机**：所有中间结果落盘，进程无内存态。排错时按以下顺序找证据，多数问题无需读源码：

| 证据 | 位置 | 能回答 |
| --- | --- | --- |
| 想法与标记 | `templates/<exp>/ideas.json` | 想法是否生成、`novel` 是否为 true |
| 跨阶段记忆 | `results/<exp>/<run>/notes.txt` | 基线、每轮结果、图说明是否被写进去 |
| 代码快照 | `results/<run>/run_<n>.py` | Aider 第 n 轮把 experiment.py 改成了什么 |
| 结果 | `results/<run>/run_<n>/final_info.json` | 指标是否符合契约 |
| 图 | `results/<run>/*.png` | plot.py 是否产出图 |
| Aider 对话 | `results/<run>/<run名>_aider.txt` | Aider 是否理解指令、补丁是否应用 |
| LaTeX | `results/<run>/latex/template.tex` | 章节/引用/图是否被填充 |
| 评审 | `results/<run>/review.txt` | 评审是否生成 |
| 并行日志 | `results/<run>/log.txt` | 并行模式下 stdout/stderr（顺序模式只在控制台） |

控制台阶段标记（源码 `print`）：`*Starting idea* → *Starting Experiments* → *Starting Writeup* → *Starting Review* → FINISHED IDEA`。卡在哪一步就看哪一步的产物。

---

## 1. Aider 无法定位/应用补丁（SEARCH/REPLACE 失败）

### 症状
- 控制台出现 `Apply update failed`, `No matching search block`, `Failed to apply edit` 之类 Aider 报错；
- `template.tex`/`experiment.py` 内容没变，后续 run 一直报同样的错；
- `*_aider.txt` 里能看到 Aider 反复尝试同一段补丁。

### 源码根因
1. `edit_format="diff"`（`launch_scientist.py:215,251`）：Aider 依赖精确的上下文匹配来定位 SEARCH/REPLACE 块。LaTeX 里 `{{`/`}}` 转义、重复章节、特殊符号都会让匹配失败。
2. 写作 Prompt 要求 "Be sure to first name the file and use *SEARCH/REPLACE* blocks"（`perform_writeup.py:416,444`）；模型若不写文件名或上下文太短，Aider 找不到目标。
3. `refinement_prompt` 调用前做 `.replace(r"{{","{").replace(r"}}","}")`（`perform_writeup.py:519-520`）：若你自定义 Prompt 时忘了这个还原，`error_list` 里的 `{{` 会以双花括号进入 Aider，破坏匹配。
4. `use_git=False`：没有 git 兜底，补丁失败无法 `git checkout` 回退，只能靠 `*_aider.txt` 人工判断。

### 定位
```bash
# 看 Aider 到底想改哪段、失败几次
grep -n -i "failed\|search\|replace\|update" results/<exp>/<run>/<run名>_aider.txt | tail -40
# 看文件是否真的被改
diff <模板原文件> results/<exp>/<run>/<对应文件>
```

### 修复
- **Prompt 层**：在自定义 Prompt 里明确 `filename + exact surrounding lines`；避免让模型一次性改过大区块。
- **转义**：保持 `error_list`（`perform_writeup.py:182-195`）中的 `\\section{{...}}` 形态，调用时确保 `.replace(r"{{","{")` 被执行。
- **重复内容**：`generate_latex`（`:66-81`）会因重复 section/图触发修复指令；若 Aider 反复失败，手动编辑 `template.tex` 去重后再重跑写作（`python ai_scientist/perform_writeup.py --folder <run> --model ...`，见 `execution_guide.md`）。
- **核弹选项**：删掉 `*_aider.txt` 让 Aider 对话历史清零（脚本 `__main__` 也提示 "Make sure you cleaned the Aider logs if re-generating the writeup!"，`perform_writeup.py:540`），再单独重跑写作阶段。

---

## 2. Semantic Scholar API 限流 / 检索失败

### 症状
- 控制台打印 `Response Status Code: 429`（或 400/500），伴随 `Backing off X.X seconds after N tries ... calling function search_for_papers`；
- 新颖性检查卡在查询循环，或所有想法 `novel=False`；
- 写作补引用阶段 `No papers found.`。

### 源码根因
- `generate_ideas.search_for_papers`（`:285-320`）用 `@backoff.on_exception(backoff.expo, requests.exceptions.HTTPError)`，指数退避但**无限重试**（无 `max_tries`）；每次成功 `time.sleep(1.0)`。
- 无 `S2_API_KEY` 时请求不带 `X-API-KEY`（`:291-292`），公共配额很严（尤其并行模式多 worker 同时查）。
- v1 另有 `openalex` 引擎（`:323-353`），`pyalex` + 可选 `OPENALEX_MAIL_ADDRESS`，不走 S2 配额。

### 修复（按优先级）
1. **申请并设置 key**：`export S2_API_KEY="..."`，让 header 带 `X-API-KEY`（提高速率）。
2. **换引擎**：`--engine openalex`，先 `pip install pyalex` + `export OPENALEX_MAIL_ADDRESS=you@uni.edu`。写作补引用与新颖性检查共用同一个 `--engine`（`launch_scientist.py:362,248`）。
3. **降低并发**：`--parallel` 减到 1，或利用代码已有的 `time.sleep(150)` 错峰（`:390`）——不要同时开多个主进程跑同一模板。
4. **跳过新颖性**：`--skip-novelty-check`，但**必须手动保证 `ideas.json` 每条有 `"novel": true`**（否则 `novel_ideas=[i for i in ideas if i["novel"]]` 在 `launch_scientist.py:363` KeyError）。
5. **根治**：给 `search_for_papers` 的 backoff 加 `max_tries`，失败改走 `openalex` 兜底（二次开发点）。

---

## 3. LaTeX 缺包 / pdflatex 编译失败

### 症状
- 启动即 `Error: Required LaTeX dependencies not found: ['pdflatex','chktex']` 然后 `sys.exit(1)`；
- 控制台 `GENERATING LATEX` 后 `Latex timed out after 30 seconds` / `/usr/bin/pdflatex: ... ! LaTeX Error: File 'iclr2024_conference.sty' not found.`；
- PDF 未生成，`Failed to rename PDF.`。

### 源码根因
1. `check_latex_dependencies()`（`launch_scientist.py:101-119`）用 `shutil.which` 检查 `pdflatex` 和 `chktex`，缺任一就退出（只在 `--writeup latex` 时检查，`:340`）。
2. `compile_latex()`（`perform_writeup.py:94-127`）在 `latex/` 目录跑 4 条命令（pdflatex→bibtex→pdflatex→pdflatex），每条 `timeout=30`；**超时/异常只打印不抛**，所以"编译失败"常表现为"PDF 没生成"而非崩溃。
3. 模板依赖本地 `.sty/.bst`：每个模板 `latex/` 目录下自带 `iclr2024_conference.sty`, `fancyhdr.sty`, `natbib.sty`, `iclr2024_conference.bst`；缺这些文件会 `File ... not found`。
4. 模板 `\usepackage{cleveref,algorithm,algorithmicx,algpseudocode,subcaption,...}`（如 nanoGPT `template.tex` 头部）需完整 TeX 发行版。
5. `generate_latex`（`:22-29`）若 `references.bib` 正则不匹配，直接 `print("No references.bib found in template.tex"); return`——**连 compile_latex 都不调**。

### 修复
```bash
# 1) 安装完整 TeX（含 chktex、cleveref、algorithm2e 等）
sudo apt-get install texlive-full        # 或 texlive + texlive-latex-extra + texlive-fonts-recommended
which pdflatex chktex                    # 必须都有输出

# 2) 手动复现编译，看到完整报错
cd results/<exp>/<run>/latex
pdflatex -interaction=nonstopmode template.tex
bibtex template && pdflatex -interaction=nonstopmode template.tex && pdflatex -interaction=nonstopmode template.tex

# 3) 只重编译（不重写），路径指向已有 run
python ai_scientist/perform_writeup.py --folder results/<exp>/<run> --no-writing --model gpt-4o-2024-05-13
```
- **缺包**：把报错中的 `.sty`（如 `iclr2024_conference.sty`）补齐到 `latex/` 目录（可从未改动的模板目录复制）。
- **bib 缺失**：确认 `template.tex` 顶部有 `\begin{filecontents}{references.bib}...\end{filecontents}` 且 `\bibliography{references}` 存在。
- **chktex 假错**：`chktex ... -q -n2 -n24 -n13 -n1` 已过滤 4 类噪声；若仍循环修不好，手动改 `template.tex` 后用 `--no-writing`。
- **超时**：把 `compile_latex` 的 `timeout=30`（`perform_writeup.py:94`）调大；或先手动编译成功再让脚本跳过。

---

## 4. CUDA OOM / 显存崩溃

### 症状
- `torch.cuda.OutOfMemoryError: CUDA out of memory. Tried to allocate ...`；
- `run_experiment` 返回非零 → 控制台 `Run <n> failed ...` → stderr 尾部进下一轮 prompt；连续 4 次 → `Max iterations reached` / `Not all experiments completed.`。

### 源码根因
- 框架**不做显存管理**：`run_experiment`（`perform_experiments.py:30-85`）只是 `subprocess.run(["python","experiment.py","--out_dir=run_n"])`，子进程 OOM 后 `returncode!=0` → 删 `run_n` 目录 → 把 stderr 尾部（`MAX_STDERR_OUTPUT=1500`）交给 Aider 修。
- Aider 无法改 GPU 参数（除 2d_diffusion 外实验脚本只接受 `--out_dir`，且框架禁止加额外参数）——只能改脚本内的 batch size/模型规模常量。
- 并行模式每个 worker 独占一张 GPU（`worker` 设 `CUDA_VISIBLE_DEVICES`，`launch_scientist.py:123`）；`--parallel` 超过显存可承受数会互相抢显存。

### 修复
- **看清楚 OOM 发生在哪一轮**：`results/<run>/run_<n>.py` 是当时的代码快照，`log.txt`（并行）或控制台有 stderr。
- **让 Aider 自动修**：stderr 已进下一轮 prompt；通常它能把 batch size 调小或加 `torch.cuda.empty_cache()`。若连续失败：手动在模板 `experiment.py` 减小 `batch_size`/`block_size`/模型层数，或 `num_seeds`/数据集数量。
- **降低并行度**：`--parallel 1` 或 `--gpus 0`（单卡）。
- **清残留进程**：OOM 的子进程可能残留占显存，`nvidia-smi` 查到后 `kill -9 <pid>`。
- **根治**：`experiment.py` 内加 `try/except torch.cuda.OutOfMemoryError` 回退 CPU/小 batch；或用 `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`。

---

## 5. 断点续跑 / 重跑单个阶段

v1 **没有内建断点续跑**（无 checkpoint 文件、无状态机），但可利用"文件即状态"的特性手动续跑：

### 5.1 跳过想法生成 / 新颖性（复用 `ideas.json`）
```bash
# 编辑后复用已有 ideas.json（确保每条有 "novel": true/false）
python launch_scientist.py --experiment <exp> --skip-idea-generation --skip-novelty-check ...
```
⚠ 注意：`launch_scientist.py:363` 仍要 `idea["novel"]`；若 `ideas.json` 缺该键会 KeyError。续跑前用脚本批量补 `"novel": true`。

### 5.2 单独重跑写作（最常见）
`perform_writeup.py` 的 `__main__`（`:532-580`）支持对已有结果目录重跑：
```bash
python ai_scientist/perform_writeup.py \
  --folder results/<exp>/<run名> \
  --model gpt-4o-2024-05-13 \
  --engine semanticscholar
# 仅重新编译，不重写
python ai_scientist/perform_writeup.py --folder <run> --no-writing --model gpt-4o-2024-05-13
```
- 它会从 `<folder>/ideas.json` 按 `idea["Name"] in basename(folder)` 找回对应想法（`:552-557`）；找不到 → `ValueError`。
- **先删 `<folder>/<run名>_aider.txt`** 清对话历史（脚本也提示），否则 Aider 带着旧上下文。

### 5.3 单独重跑评审
评审无 CLI 入口，但可按根 README 的 Python 片段：
```python
from ai_scientist.perform_review import load_paper, perform_review
import openai, json
text = load_paper("results/<exp>/<run>/<Name>.pdf")
review = perform_review(text, model="gpt-4o-2024-05-13",
    client=openai.OpenAI(), num_reflections=5, num_fs_examples=1,
    num_reviews_ensemble=5, temperature=0.1)
open("results/<exp>/<run>/review.txt","w").write(json.dumps(review, indent=4))
```

### 5.4 避免"目录已存在"中断
`do_idea` 用 `assert not osp.exists(folder_name)`（`launch_scientist.py:169`）；同一秒同名 idea 或残留目录会 assert 失败。续跑前确认 `results/<exp>/` 下没有同名 `<时间戳>_<Name>`，或删掉旧目录。

### 5.5 并行中断后的清理
并行 worker 被杀后队列里剩余想法可能丢失；`log.txt` 会停留在最后处理的想法。重启用 `--skip-idea-generation` + 手动把已完成的想法从 `ideas.json` 移除，或靠时间戳区分新旧结果目录。

---

## 6. 其他高频坑（速查表）

| 现象 | 源码根因 | 处理 |
| --- | --- | --- |
| `KeyError: 'novel'` | `--skip-novelty-check` 且 ideas.json 无 novel 字段（`:363`） | 给 ideas.json 每条补 `"novel": true` |
| `ZeroDivisionError: integer modulo by zero` | 无 GPU 却 `--parallel>0`（`:374` `available_gpus[i % len]`） | 去掉 `--parallel` 或提供 GPU |
| `open run_0/final_info.json 失败` | 未做模板基线准备 | 进模板跑 `python experiment.py --out_dir run_0 && python plot.py` |
| 2d_diffusion 实验改参数无效 | 框架只传 `--out_dir`（`perform_experiments.py:39-43`），额外 CLI 参数被忽略 | 改脚本内默认值/常量，别依赖命令行 |
| 绘图反复失败 | `plot.py` 的 `labels` 没填或读的 npy/pkl 不存在 | 看 `log.txt`；确认 `labels` 含需绘制的 run；`all_results.npy/pkl` 存在 |
| 评审报错无 review.txt | 评审硬编码 `openai.OpenAI()` + gpt-4o（`:268-275`），缺 `OPENAI_API_KEY` | 设置该 key（即使主模型是 Claude） |
| `Failed to extract JSON` / assert JSON None | 模型不守 ```json 格式；`extract_json_between_markers` 返回 None（`llm.py:289-315`） | 换更强模型；必要时改写解析函数加更多兜底 |
| 所有想法很"水"/重复 | 生成 archive 去重失效或模型太弱 | 增大 `--num-ideas`/反思轮，换 Claude 3.5 Sonnet；检查 ideas.json |
| LaTeX 表 `\cite` 找不到 key | `generate_latex:31-38` 静态检查触发 Aider 修；修不好 | 手动对齐 bib key，注意 `\cite` 名与 bibtex 完全一致 |
| 中文/特殊字符乱码 | `inputenc`/`fontenc` 已配但模型写入未转义字符 | 检查 `error_list` 的转义规则（`perform_writeup.py:193-194`） |

---

## 7. 一键自检脚本（开发时快速体检）

```bash
# 1) 环境
which python pdflatex bibtex chktex
python -c "import torch,aider,anthropic,openai,pymupdf,pymupdf4llm; print(torch.cuda.device_count())"

# 2) 模板契约
python - <<'PY'
import json, os
for t in ["nanoGPT","2d_diffusion","grokking"]:
    d = f"templates/{t}"
    need = ["experiment.py","plot.py","prompt.json","seed_ideas.json",
            "run_0/final_info.json","latex/template.tex"]
    print(t, {n: os.path.exists(os.path.join(d,n)) for n in need})
    p = json.load(open(f"{d}/prompt.json"))
    print("  system:", p["system"][:40], "...")
    fi = json.load(open(f"{d}/run_0/final_info.json"))
    print("  datasets:", list(fi.keys()))
PY

# 3) 行号锚点复核
grep -n '^def ' launch_scientist.py ai_scientist/*.py
```

更多运行参数与模型配置见 [`execution_guide.md`](execution_guide.md)；完整调用链见 [`code_trace_and_prompts.md`](code_trace_and_prompts.md)。
