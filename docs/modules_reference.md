# The AI Scientist v1 — 核心模块与函数参考

> 基于源码核验。每个条目给出：**签名 → 入参 → 返回 → 副作用/关键点 → 源码位置**。
> 行号对应当前仓库版本（可用 `grep -n '^def ' ai_scientist/*.py` 复核）。

---

## 1. `ai_scientist/llm.py`（模型访问层）

常量：`MAX_NUM_TOKENS = 4096`；`AVAILABLE_LLMS` 是 CLI `--model` 的白名单，包含 Anthropic（claude-3.5-sonnet 两种）、OpenAI（gpt-4o/4.1/o1/o3 等）、OpenRouter（llama3.1-405b）、Bedrock/Vertex Claude、DeepSeek、Gemini 等。

### `create_client(model) -> (client, model_name)`  `llm.py:317`
- 入参：`model`（CLI 原字符串）。
- 返回：二元组 `(SDK client 实例, 实际用于 API 的 model 名)`。
- 路由：
  | model 前缀/条件 | 客户端 | 环境变量 |
  | --- | --- | --- |
  | `claude-` | `anthropic.Anthropic()` | `ANTHROPIC_API_KEY` |
  | `bedrock`+claude | `anthropic.AnthropicBedrock()`，`model_name=model.split("/")[-1]` | AWS 三件套 |
  | `vertex_ai`+claude | `anthropic.AnthropicVertex()` | GCP 认证 |
  | `gpt`/`o1`/`o3` | `openai.OpenAI()` | `OPENAI_API_KEY` |
  | `deepseek-*` | `openai.OpenAI(base_url="https://api.deepseek.com")` | `DEEPSEEK_API_KEY` |
  | `llama3.1-405b` | `openai.OpenAI(base_url="https://openrouter.ai/api/v1")`，model_name=`meta-llama/llama-3.1-405b-instruct` | `OPENROUTER_API_KEY` |
  | `gemini` | `openai.OpenAI(base_url="https://generativelanguage.googleapis.com/v1beta/openai/")` | `GEMINI_API_KEY` |
- 不支持则 `raise ValueError`。**加新模型从改这里 + `AVAILABLE_LLMS` 开始。**

### `get_response_from_llm(msg, client, model, system_message, print_debug=False, msg_history=None, temperature=0.75) -> (str, list)`  `llm.py:143`
- 入参：`msg` 用户消息（字符串）；`msg_history` OpenAI 风格消息列表（Claude 用 content block 格式）；`temperature` 采样温度。
- 返回：`(content 文本, 新的消息历史)`。
- 分支要点：
  - claude → `client.messages.create`（content 为 `[{type:"text",text:...}]`，`system` 单列）；
  - gpt → `chat.completions.create`，`system` 作为首条消息，`seed=0`，`n=1`；
  - o1/o3 → `max_completion_tokens`、`temperature=1`，system 作为 user 消息（`MAX_NUM_TOKENS` 通过 completion token 参数）；
  - llama/deepseek/gemini 走各自 OpenAI 兼容路径。
- 用 `@backoff.on_exception(backoff.expo, (openai.RateLimitError, openai.APITimeoutError))` 自动退避。
- **改动风险**：返回类型固定为 `(str, list)`；所有调用方都依赖 `extract_json_between_markers` 解析 content。

### `get_batch_responses_from_llm(...)`  `llm.py:67`
- 与上面类似但 `n_responses=N`，返回 `(list[str], list[list[msg_history]])`；用于评审集成（一次生成 5 份）。gpt/llama 用单次 API 的 `n` 参数，其它模型循环调用 `get_response_from_llm` N 次。

### `extract_json_between_markers(llm_output) -> dict | None`  `llm.py:289`
- 先正则匹配 ` ```json ... ``` `，无匹配则退化为任意 `{...}`；逐段 `json.loads`，失败时去掉 `[\x00-\x1F\x7F]` 控制字符再试；全部失败返回 `None`。
- **所有 prompt 都要求 LLM 用 ` ```json ` 包裹输出**，这是解析契约。

---

## 2. `ai_scientist/generate_ideas.py`（想法与文献）

### `generate_ideas(base_dir, client, model, skip_generation=False, max_num_generations=20, num_reflections=5) -> list[dict]`  `generate_ideas.py:76`
- 入参：`base_dir` 模板目录；`skip_generation` 为 True 时直接读 `ideas.json`；`max_num_generations` 由 CLI `--num-ideas` 传入（默认 50）。
- 返回：想法 dict 列表（字段 `Name/Title/Experiment/Interestingness/Feasibility/Novelty`）。
- 副作用：读 `seed_ideas.json`/`experiment.py`/`prompt.json`；**覆盖写 `ideas.json`**。
- 提示词：`idea_first_prompt`、`idea_reflection_prompt`（模块级字符串）；反思中检测 `"I am done"` 早停。

### `generate_next_idea(base_dir, client, model, prev_idea_archive=[], num_reflections=5, max_attempts=10) -> list[dict]`  `generate_ideas.py:178`
- 开放式"生成一个新想法"：第一次取 `seed_ideas.json[:1]` 作种，之后基于已有想法的 `Score` 字段生成；写回 `ideas.json`。不在主入口调用。

### `search_for_papers(query, result_limit=10, engine="semanticscholar") -> list[dict] | None`  `generate_ideas.py:285`
- `@backoff.on_exception(backoff.expo, requests.exceptions.HTTPError)`。
- `semanticscholar`：`GET graph/v1/paper/search`，header `X-API-KEY: S2_API_KEY`（可选），fields=`title,authors,venue,year,abstract,citationStyles,citationCount`；`raise_for_status()`；`time.sleep(1.0)`。
- `openalex`：`pyalex.Works().search(query).get(per_page=result_limit)`，抽 `extract_info_from_work`（abstract 截断到 1000 字符防超上下文）。
- 返回论文 dict 列表（含 `citationStyles` 供写作生成 bibtex）；`query` 为空返回 `None`。

### `check_idea_novelty(ideas, base_dir, client, model, max_num_iterations=10, engine="semanticscholar") -> list[dict]`  `generate_ideas.py:405`
- 入参：ideas 列表；对每条无 `novel` 字段的想法做最多 10 轮"思考→查询→文献→再思考"。
- 返回：每条追加 `novel: bool` 的 ideas；写回 `ideas.json`。
- 判定：LLM 文本含 `decision made: novel` / `decision made: not novel`（大小写不敏感）。

---

## 3. `ai_scientist/perform_experiments.py`（实验执行）

常量：`MAX_ITERS=4`、`MAX_RUNS=5`、`MAX_STDERR_OUTPUT=1500`；`coder_prompt` 为初始 prompt 模板。

### `run_experiment(folder_name, run_num, timeout=7200) -> (int, str)`  `perform_experiments.py:30`
- 动作：复制 `experiment.py` → `run_<n>.py`；`subprocess.run(["python","experiment.py","--out_dir=run_<n>"], cwd=abspath(folder_name), stderr=PIPE, text=True, timeout=timeout)`。
- 返回：`(returncode, next_prompt)`。
  - 成功：`next_prompt` 含 `run_<n>/final_info.json` 的 means + "写进 notes.txt + 实现清单下一项"；
  - 失败：删除 `run_<n>` 目录，`next_prompt` 含 stderr 尾部 1500 字符；
  - 超时：删除目录，`next_prompt` 提示 timeout。

### `run_plotting(folder_name, timeout=600) -> (int, str)`  `perform_experiments.py:88`
- `subprocess.run(["python","plot.py"], cwd=..., timeout=600)`；成功 `next_prompt=""`，失败返回含 stderr 的 prompt。

### `perform_experiments(idea, folder_name, coder, baseline_results) -> bool`  `perform_experiments.py:116`
- 入参：`idea` dict；`coder` 为 Aider `Coder` 实例；`baseline_results` 是 `{k: means}`。
- 流程：实验循环（检测 `ALL_COMPLETED`、连续失败 MAX_ITERS 保护）→ 绘图循环 → 补 notes.txt。
- 返回：实验+绘图都走通为 `True`，否则 `False`。

---

## 4. `ai_scientist/perform_writeup.py`（论文写作与编译）

### `generate_latex(coder, folder_name, pdf_file, timeout=30, num_error_corrections=5)`  `perform_writeup.py:15`
- 前置静态检查：`\cite{}` vs `references.bib`、 `\includegraphics{}` vs 目录内 png、重复图、重复 `\section` → 发现问题让 Aider 修。
- `chktex template.tex -q -n2 -n24 -n13 -n1`（过滤若干噪声规则）→ 最多 5 轮让 Aider 修 LaTeX 语法。
- 调 `compile_latex()`；最后 `shutil.move(cwd/template.pdf, pdf_file)`。

### `compile_latex(cwd, pdf_file, timeout=30)`  `perform_writeup.py:94`
- 顺序跑 4 条命令：`pdflatex -interaction=nonstopmode template.tex` → `bibtex template` → 两次 pdflatex；cwd 为 `latex/` 目录。超时/异常只打印不抛。

### `get_citation_aider_prompt(client, model, draft, current_round, total_rounds, engine="semanticscholar") -> (str | None, bool)`  `perform_writeup.py:296`
- 返回 `(给 Aider 的 prompt 或 None, done)`。`done=True` 表示 LLM 说 "No more citations needed"。
- 内部两轮 LLM：选 query → `search_for_papers` → 选 `Selected` 索引 → 拼 bibtex 与 "ABSOLUTELY DO NOT ADD IT AGAIN" 提示。

### `perform_writeup(idea, folder_name, coder, cite_client, cite_model, num_cite_rounds=20, engine="semanticscholar")`  `perform_writeup.py:401`
- 模块级常量：`per_section_tips`（每节写作要点）、`error_list`、`refinement_prompt`、`second_refinement_prompt`、`citation_system_msg` 等提示词。
- 流程见 pipeline 文档第 5 节。副作用：改 `latex/template.tex`、生成 `<Name>.pdf`。无显式返回值（`None`）。
- `__main__` 子命令支持对已有结果目录单独重跑写作（`--folder`, `--no-writing`, `--model`, `--engine`）。

---

## 5. `ai_scientist/perform_review.py`（论文评审）

提示词常量：`reviewer_system_prompt_neg/pos`、`neurips_form`（评审表+说明）、`template_instructions`、`reviewer_reflection_prompt`、`meta_reviewer_system_prompt`；few-shot 文件在 `fewshot_examples/{132_automated_relational,attention,2_carpe_diem}.{pdf,json,txt}`。

### `perform_review(text, model, client, num_reflections=1, num_fs_examples=1, num_reviews_ensemble=1, temperature=0.75, msg_history=None, return_msg_history=False, reviewer_system_prompt=reviewer_system_prompt_neg, review_instruction_form=neurips_form) -> dict | (dict, list)`  `perform_review.py:126`
- 入参：`text` 论文 Markdown 文本；`num_reviews_ensemble>1` 时走集成（batch 生成 + `get_meta_review` + 数值字段取均值）；`num_reflections>1` 时再迭代反思。
- 返回：review dict（字段见 architecture 文档第 4.3 节）；`return_msg_history=True` 时返回 `(review, msg_history)`。
- 主入口调用参数固定：`num_reflections=5, num_fs_examples=1, num_reviews_ensemble=5, temperature=0.1, model="gpt-4o-2024-05-13"`。

### `load_paper(pdf_path, num_pages=None, min_size=100) -> str`  `perform_review.py:267`
- 三级回退：`pymupdf4llm.to_markdown` → `pymupdf` 逐页 `get_text()` → `pypdf` `extract_text()`；文本长度 `< min_size` 视为失败并进入下一级。

### `load_review(path) -> dict` / `get_review_fewshot_examples(num_fs_examples=1) -> str` / `get_meta_review(model, client, temperature, reviews) -> dict`  `perform_review.py:301/323/361`
- `load_review` 取 json 的 `["review"]` 键；few-shot 优先读同名 `.txt`，否则 `load_paper(pdf)`；meta-review 让 "Area Chair" 聚合 N 份评审。

### `perform_improvement(review, coder)`  `perform_review.py:386`
- 把 `json.dumps(review)` 塞进 `improvement_prompt`，`coder.run(...)` 修改论文（无返回值，靠副作用改 template.tex）。

---

## 6. 入口脚本 `launch_scientist.py`（函数清单）

| 函数 | 位置 | 作用 |
| --- | --- | --- |
| `print_time()` | `:25` | 打印时间戳日志 |
| `parse_arguments()` | `:29` | 定义 CLI 参数（见 execution_guide 第 3 节） |
| `get_available_gpus(gpu_ids=None)` | `:95` | `--gpus` 逗号串→int 列表；未指定→`range(torch.cuda.device_count())` |
| `check_latex_dependencies()` | `:101` | `shutil.which` 检查 `pdflatex`/`chktex` 是否存在；缺则 False |
| `worker(queue, base_dir, results_dir, model, client, client_model, writeup, improvement, gpu_id)` | `:122` | 并行 worker：设 CUDA_VISIBLE_DEVICES，循环从队列取 idea 调 `do_idea` |
| `do_idea(base_dir, results_dir, idea, model, client, client_model, writeup, improvement, log_file=False) -> bool` | `:154` | 单想法全生命周期（准备/实验/写作/评审/改进），失败返回 False |
| `__main__` | `:321` | 入口：GPU 检查→LaTeX 依赖检查→生成/筛想法→顺序或并行执行 |

常量：`NUM_REFLECTIONS = 3`（想法生成反思轮数）。

> 复核提示：`grep -n '^def ' launch_scientist.py ai_scientist/*.py` 可得到所有函数定义行号；本文行号基于当前仓库版本，改动后请以 grep 结果为准。
