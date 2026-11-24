# The AI Scientist v1 — 源码级调用链追踪、Prompt 模板与数据契约

> 本文基于**当前仓库真实源码**逐行核对，行号以 `文件:行号` 标注（可用 `grep -n '^def ' ai_scientist/*.py launch_scientist.py` 复核）。
> 目标：让零上下文 AI 能据此直接排错、二次开发、替换 Prompt、编写新模板。所有 Prompt 均为**逐字摘录**，占位符 `{...}` 保留原样。
> 配套阅读：[`pipeline_and_dataflow.md`](pipeline_and_dataflow.md)（阶段级 I/O）、[`modules_reference.md`](modules_reference.md)（函数签名）、[`troubleshooting.md`](troubleshooting.md)（排错）。

---

## 0. 阅读约定

- **「实际 Prompt」**指源码中作为字符串常量存在、经 `.format(...)` 后送入 LLM/Aider 的文本。本文用「原文摘录」块给出，占位符与转义（如 `\\cite`、`{{`/`}}`）保持源码形态。
- **「契约」**指框架代码与模板代码之间必须匹配的隐式协议（文件名、JSON 结构、命令行参数、字典键）。契约被破坏时不会给出友好错误，而是 `KeyError`/`FileNotFoundError`/静默失败。
- 两个"大脑"通道要分清：
  1. **LLM 通道**（`ai_scientist/llm.py`）：想法生成、新颖性、引用、评审，输出靠 ```json ... ``` 标记解析。
  2. **Aider 通道**（`aider.coders.Coder.run`）：改文件（实验代码/plot/LaTeX/notes），输出靠检测 `ALL_COMPLETED` 之类字符串，框架不解析其结构化输出。

---

## 1. 入口逐行追踪：`launch_scientist.py`

### 1.1 模块导入与常量（`launch_scientist.py:1-22`）

```python
from aider.coders import Coder              # Aider 代码编辑器
from aider.io import InputOutput
from aider.models import Model
from ai_scientist.generate_ideas import generate_ideas, check_idea_novelty
from ai_scientist.llm import create_client, AVAILABLE_LLMS
from ai_scientist.perform_experiments import perform_experiments
from ai_scientist.perform_review import perform_review, load_paper, perform_improvement
from ai_scientist.perform_writeup import perform_writeup, generate_latex
NUM_REFLECTIONS = 3                         # 想法生成反思轮数（注意：generate_ideas 默认是 5，但入口传 3）
```

`NUM_REFLECTIONS=3` 是**入口覆盖值**：`__main__` 调 `generate_ideas(..., num_reflections=NUM_REFLECTIONS)`（`:339`）。直接跑 `generate_ideas.py` 时其 `__main__` 用 5。

### 1.2 `parse_arguments()`（`:29-92`）

CLI 参数共 9 项，逐一映射到后续变量：

| 参数 | 默认 | 流向 |
| --- | --- | --- |
| `--skip-idea-generation` | False | `generate_ideas(skip_generation=...)` |
| `--skip-novelty-check` | False | 决定是否调 `check_idea_novelty` |
| `--experiment` | `nanoGPT` | `base_dir=templates/<exp>`, `results_dir=results/<exp>` |
| `--model` | `claude-3-5-sonnet-20240620`，choices=`AVAILABLE_LLMS` | `create_client(args.model)` → `(client, client_model)` |
| `--writeup` | `latex`（唯一合法值） | `do_idea` 中 `if writeup == "latex"` 分支 |
| `--parallel` | 0 | 0=顺序；>0 起 `multiprocessing.Process` |
| `--improvement` | False | `do_idea` 中评审后改进分支 |
| `--gpus` | None | `get_available_gpus(args.gpus)` |
| `--num-ideas` | 50 | `generate_ideas(max_num_generations=...)` |
| `--engine` | `semanticscholar` | 新颖性检查 + 写作补引用共用的检索后端 |

### 1.3 `__main__` 主流程（`:322-420`）——严格的执行顺序

```
① args = parse_arguments()                                     # :322
② available_gpus = get_available_gpus(args.gpus)               # :325
   若 args.parallel > len(available_gpus) → 下调并打印 Warning  # :326-331
③ if args.writeup=="latex" and not check_latex_dependencies(): sys.exit(1)  # :335-336
   —— shutil.which("pdflatex") / ("chktex") 缺一即退出
④ client, client_model = create_client(args.model)             # :339
⑤ base_dir = templates/<exp>;  results_dir = results/<exp>     # :341-342
⑥ ideas = generate_ideas(base_dir, client, client_model,
        skip_generation=args.skip_idea_generation,
        max_num_generations=args.num_ideas, num_reflections=3)  # :343-349
⑦ if not args.skip_novelty_check:
       ideas = check_idea_novelty(ideas, base_dir, client,
                client_model, engine=args.engine)               # :352-358
⑧ json.dump(ideas, <base_dir>/ideas.json, indent=4)            # :361-362
⑨ novel_ideas = [i for i in ideas if i["novel"]]               # :363
   —— ⚠ 若走了 --skip-novelty-check 且 ideas.json 无 "novel" 键 → 此处 KeyError
⑩ 调度：
   并行（:367-400）：Queue 装所有 novel_idea；启动 args.parallel 个 Process(target=worker,...)
       gpu_id = available_gpus[i % len(available_gpus)]（:374）；每启动一个 time.sleep(150) 秒错峰（:390）
       全部入队后追加 args.parallel 个 None 作退出信号（:395）；最后 join()
   顺序（:402-419）：for idea in novel_ideas: do_idea(...)，try/except 包住，打印 traceback
⑪ print("All ideas evaluated.")（:420）
```

**关键不变量**：`check_idea_novelty` 给每条 idea 追加 `"novel": bool`；`novel_ideas` 只保留 `True`。这是"想法→实验"的唯一闸门。

### 1.4 `do_idea()`（`launch_scientist.py:154-311`）——单想法全生命周期

这是最核心的函数。按下游调用逐段拆解：

**A. 目录与基线（`:167-185`）**
```python
timestamp  = datetime.now().strftime("%Y%m%d_%H%M%S")
idea_name  = f"{timestamp}_{idea['Name']}"
folder_name = osp.join(results_dir, idea_name)
assert not osp.exists(folder_name), f"Folder {folder_name} already exists."
shutil.copytree(base_dir, destination_dir, dirs_exist_ok=True)   # 复制整个模板目录
with open(osp.join(base_dir, "run_0", "final_info.json")) as f:  # 读人工基线
    baseline_results = json.load(f)
if isinstance(baseline_results, dict):
    baseline_results = {k: v["means"] for k, v in baseline_results.items()}  # 抽 means
```
- **契约①**：模板目录必须有 `run_0/final_info.json`，且为 `{dataset: {"means": {...}}}` 结构。
- `notes.txt` 初始内容（`:181-185`）：`# Title: ...`、`# Experiment description: ...`、`## Run 0: Baseline`、`Results: <baseline_results>`、`Description: Baseline results.`。`notes.txt` 是跨阶段"记忆"。

**B. Aider 实例（实验阶段，`:196-215`）**
```python
fnames = [exp_file, vis_file, notes]     # experiment.py, plot.py, notes.txt
io = InputOutput(yes=True, chat_history_file=f"{folder_name}/{idea_name}_aider.txt")
# 模型映射（:201-208，写作阶段 :237-244 重复同样一段）
if   model == "deepseek-coder-v2-0724": main_model = Model("deepseek/deepseek-coder")
elif model == "deepseek-reasoner":       main_model = Model("deepseek/deepseek-reasoner")
elif model == "llama3.1-405b":           main_model = Model("openrouter/meta-llama/llama-3.1-405b-instruct")
else:                                    main_model = Model(model)
coder = Coder.create(main_model=main_model, fnames=fnames, io=io,
                     stream=False, use_git=False, edit_format="diff")
```
- `edit_format="diff"` 意味着 Aider 用 **SEARCH/REPLACE 块**改文件；`use_git=False` 关闭 git（结果是纯文件系统快照）。
- 实验与写作**各建一个 coder**，但共用同一个 `io`（同一 `chat_history_file`）。

**C. 阶段调用（`:218-309`）**
```
perform_experiments(idea, folder_name, coder, baseline_results)   # :221 → 失败 return False
perform_writeup(idea, folder_name, coder_new, client, client_model, engine=args.engine)  # :254
paper_text = load_paper(f"{folder_name}/{idea['Name']}.pdf")      # :267
review = perform_review(paper_text, model="gpt-4o-2024-05-13",
        client=openai.OpenAI(), num_reflections=5, num_fs_examples=1,
        num_reviews_ensemble=5, temperature=0.1)                   # :268-275
json.dump(review, review.txt, indent=4)                            # :278-280
if improvement:
    perform_improvement(review, coder); generate_latex(..._improved.pdf)
    二评 → review_improved.txt                                     # :283-307
return True                                                        # :309
```
- **契约②**：评审**硬编码** `gpt-4o-2024-05-13` + `openai.OpenAI()`，与 `--model` 无关；即使主模型是 Claude 也必须有 `OPENAI_API_KEY`。
- `try/except/finally`（`:193/:310/:313`）：任一阶段异常 → 打印并返回 `False`；`finally` 打印 `FINISHED IDEA`，并行模式恢复 stdout/stderr 并关闭 `log.txt`。
- 并行模式 stdout/stderr 重定向（`:186-191`）：`sys.stdout = sys.stderr = open(folder/log.txt,"a")`。

### 1.5 `worker()`（`:122-152`）与 GPU 分配

`worker` 先设 `os.environ["CUDA_VISIBLE_DEVICES"]=str(gpu_id)`（`:133`），然后 `while True: idea=queue.get(); if idea is None: break; do_idea(..., log_file=True)`。
`get_available_gpus`（`:95-98`）：`--gpus "0,1"` → `[0,1]`；未指定 → `list(range(torch.cuda.device_count()))`。**无 GPU 且 `--parallel>0`** 时 `available_gpus=[]`，`i % len([])` → `ZeroDivisionError`。

---

## 2. LLM 通道：Prompt 原文与解析契约

### 2.1 公共解析：`extract_json_between_markers()`（`ai_scientist/llm.py:289-315`）

```python
json_pattern = r"```json(.*?)```"          # 优先抓 ```json 围栏
matches = re.findall(json_pattern, llm_output, re.DOTALL)
if not matches:
    json_pattern = r"\{.*?\}"               # 退化：抓任意 {...}
for json_string in matches:
    try: return json.loads(json_string.strip())
    except JSONDecodeError:
        clean = re.sub(r"[\x00-\x1F\x7F]", "", json_string)   # 去控制字符再试
        try: return json.loads(clean)
        except: continue
return None
```
**结论**：所有 Prompt 都被要求用 ```json 包裹；解析失败返回 `None`，调用方要么 `assert ... is not None`（想法生成）要么 `try/except`（新颖性/引用）。`get_response_from_llm`（`:143`）与 `get_batch_responses_from_llm`（`:67`）都带 `@backoff.on_exception(backoff.expo, (openai.RateLimitError, openai.APITimeoutError))`。

### 2.2 想法生成 Prompt（`generate_ideas.py`）

`idea_first_prompt`（`:14-52`，原文摘录，占位符保留）：
```
{task_description}
<experiment.py>
{code}
</experiment.py>

Here are the ideas that you have already generated:

'''
{prev_ideas_string}
'''

Come up with the next impactful and creative idea for research experiments and directions you can feasibly investigate with the code provided.
Note that you will not have access to any additional resources or datasets.
Make sure any idea is not overfit the specific training dataset or model, and has wider significance.

Respond in the following format:

THOUGHT:
<THOUGHT>

NEW IDEA JSON:
```json
<JSON>
```

In <THOUGHT>, first briefly discuss your intuitions and motivations for the idea. Detail your high-level plan, necessary design choices and ideal outcomes of the experiments. Justify how the idea is different from the existing ones.

In <JSON>, provide the new idea in JSON format with the following fields:
- "Name": A shortened descriptor of the idea. Lowercase, no spaces, underscores allowed.
- "Title": A title for the idea, will be used for the report writing.
- "Experiment": An outline of the implementation. E.g. which functions need to be added or modified, how results will be obtained, ...
- "Interestingness": A rating from 1 to 10 (lowest to highest).
- "Feasibility": A rating from 1 to 10 (lowest to highest).
- "Novelty": A rating from 1 to 10 (lowest to highest).

Be cautious and realistic on your ratings.
This JSON will be automatically parsed, so ensure the format is precise.
You will have {num_reflections} rounds to iterate on the idea, but do not need to use them all.
```

`idea_reflection_prompt`（`:54-73`，原文摘录）：
```
Round {current_round}/{num_reflections}.
In your thoughts, first carefully consider the quality, novelty, and feasibility of the idea you just created.
Include any other factors that you think are important in evaluating the idea.
Ensure the idea is clear and concise, and the JSON is the correct format.
Do not make things overly complicated.
In the next attempt, try and refine and improve your idea.
Stick to the spirit of the original idea unless there are glaring issues.

Respond in the same format as before:
THOUGHT:
<THOUGHT>

NEW IDEA JSON:
```json
<JSON>
```

If there is nothing to improve, simply repeat the previous JSON EXACTLY after the thought and include "I am done" at the end of the thoughts but before the JSON.
ONLY INCLUDE "I am done" IF YOU ARE MAKING NO MORE CHANGES.
```

**System Message 来源**：`prompt.json` 的 `system` 字段（`:110`，`idea_system_prompt = prompt["system"]`）。
**填充值**：`task_description`/`code` 来自 `prompt.json` 与 `experiment.py`；`prev_ideas_string = "\n\n".join(idea_str_archive)`，archive 初始为 `seed_ideas.json` 每条 `json.dumps`（`:99-102`），每轮生成的想法又追加进 archive（`:161`），形成"已生成想法作为下轮上下文"的去重机制。
**早停**：反思文本含 `"I am done"` → `break`（`:157-159`）。
**异常**：单想法任一轮 `Exception` → `print + continue`（`:151-153`），不中断整体。

### 2.3 新颖性检查 Prompt（`generate_ideas.py`）

`novelty_system_msg`（`:356-371`）要点（原文摘录）：
```
You are an ambitious AI PhD student who is looking to publish a paper that will contribute significantly to the field.
You have an idea and you want to check if it is novel or not. ...
Be a harsh critic for novelty ...
You will be given access to the Semantic Scholar API ... The top 10 results for any search query will be presented to you with the abstracts.

You will be given {num_rounds} to decide on the paper, but you do not need to use them all.
...
Decide a paper idea is novel if after sufficient searching, you have not found a paper that significantly overlaps with your idea.
Decide a paper idea is not novel, if you have found a paper that significantly overlaps with your idea.

{task_description}
<experiment.py>
{code}
</experiment.py>
```

`novelty_prompt`（`:373-403`）原文摘录：
```
Round {current_round}/{num_rounds}.
You have this idea:

"""
{idea}
"""

The results of the last query are (empty on first round):
"""
{last_query_results}
"""

Respond in the following format:

THOUGHT:
<THOUGHT>

RESPONSE:
```json
<JSON>
```

In <THOUGHT>, first briefly reason over the idea and identify any query that could help you make your decision.
If you have made your decision, add "Decision made: novel." or "Decision made: not novel." to your thoughts.

In <JSON>, respond in JSON format with ONLY the following field:
- "Query": An optional search query to search the literature (e.g. attention is all you need). You must make a query if you have not decided this round.
...
```

**判定逻辑**（`check_idea_novelty`，`:405-470`）：
- 最多 `max_num_iterations=10` 轮；`text.lower()` 含 `"decision made: novel"` → `novel=True` 并 break；含 `"decision made: not novel"` → break（保持 `novel=False`）。
- 否则从 JSON 取 `Query` → `search_for_papers(query, result_limit=10, engine)`；结果格式化为 `"{i}: {title}. {authors}. {venue}, {year}.\nNumber of citations: {cites}\nAbstract: {abstract}"`，累加进 `papers_str` 作下一轮上下文。
- 结束 `idea["novel"] = novel`，写回 `ideas.json`。
- `--engine semanticscholar`（`:285-320`）：`GET https://api.semanticscholar.org/graph/v1/paper/search`，可选 `X-API-KEY: S2_API_KEY`，fields=`title,authors,venue,year,abstract,citationStyles,citationCount`，`time.sleep(1.0)`；`@backoff.on_exception(backoff.expo, requests.exceptions.HTTPError)`。
- `--engine openalex`（`:323-353`）：`pyalex.Works().search(query).get(per_page=result_limit)`，abstract 截断 1000 字符防超上下文，`OPENALEX_MAIL_ADDRESS` 可选。

### 2.4 引用补全 Prompt（`perform_writeup.py`，两轮 LLM）

`citation_system_msg`（`:220-238`）要点：扮演 PhD 补引用；"DO NOT ADD A CITATION THAT ALREADY EXISTS"；`{total_rounds}` 轮。

`citation_first_prompt`（`:239-269`）原文摘录（关键段）：
```
Round {current_round}/{total_rounds}:

You have written this LaTeX draft so far:

"""
{draft}
"""

Identify the most important citation that you still need to add, and the query to find the paper.

Respond in the following format:

THOUGHT:
<THOUGHT>

RESPONSE:
```json
<JSON>
```
...
In <JSON>, ... fields:
- "Description": A precise description of the required edit, along with the proposed text and location where it should be made.
- "Query": The search query to find the paper (e.g. attention is all you need).
...
If no more citations are needed, add "No more citations needed" to your thoughts.
```

`citation_second_prompt`（`:271-294`）原文摘录（关键段）：
```
Search has recovered the following articles:

{papers}

Respond in the following format:
THOUGHT:
<THOUGHT>
RESPONSE:
```json
<JSON>
```
...
In <JSON>, respond in JSON format with the following fields:
- "Selected": A list of the indices of the selected papers to be cited, e.g. "[0, 1]". Can be "[]" if no papers are selected. This must be a string.
- "Description": Update the previous description of the required edit if needed. Ensure that any cites precisely match the name in the bibtex!!!
...
```

**`get_citation_aider_prompt()`（`:296-398`）返回给 Aider 的 prompt（`:376-395`，原文）**：
```
The following citations have just been added to the end of the `references.bib` file definition at the top of the file:
"""
{bibtex}
"""
You do not need to add them yourself.
ABSOLUTELY DO NOT ADD IT AGAIN!!!

Make the proposed change to the draft incorporating these new cites:
{description}

Use your judgment for whether these should be cited anywhere else.
Make sure that any citation precisely matches the name in `references.bib`. Change its name to the correct name in the bibtex if needed.
Ensure the citation is well-integrated into the text.
 You must use \cite or \citet to reference papers, do not manually type out author names.
```
**关键机制**：框架自己把新 bibtex 插进 `\end{filecontents}` 之前（`perform_writeup` `:456-461`：`draft.replace(r"\end{filecontents}", f"{bibtex_string}{search_str}")`），**再**让 Aider 只改正文加 `\cite`，并用 "ABSOLUTELY DO NOT ADD IT AGAIN!!!" 防止重复添加。`Selected` 必须是字符串（如 `"[0, 1]"`），由 `list(map(int, selected_papers.strip("[]").split(",")))` 解析（`:350-353`）。

### 2.5 评审 Prompt（`perform_review.py`）

`reviewer_system_prompt_base/neg/pos`（`:13-25`）：
```
base = "You are an AI researcher who is reviewing a paper that was submitted to a prestigious ML venue.Be critical and cautious in your decision."
neg  = base + "If a paper is bad or you are unsure, give it bad scores and reject it."
pos  = base + "If a paper is good or you are unsure, give it good scores and accept it."
```
主入口默认用 `neg`（`perform_review(reviewer_system_prompt=reviewer_system_prompt_neg)`）。

`template_instructions`（`:27-63`）规定 JSON 字段与顺序：`Summary, Strengths, Weaknesses, Originality(1-4), Quality(1-4), Clarity(1-4), Significance(1-4), Questions, Limitations, Ethical Concerns(bool), Soundness(1-4), Presentation(1-4), Contribution(1-4), Overall(1-10), Confidence(1-5), Decision(Accept/Reject)`，并强调 `Decision` 只能是 Accept/Reject。

`neurips_form`（`:65-125`）= NeurIPS 评审表逐项说明 + `template_instructions`。

`perform_review()` 组装（`:126-244`）：
```
base_prompt = neurips_form + get_review_fewshot_examples(num_fs_examples)
base_prompt += f"\nHere is the paper you are asked to review:\n```\n{text}\n```"
```
- 集成（`num_reviews_ensemble=5`）：`get_batch_responses_from_llm(..., temperature=0.75, n_responses=5)` 生成 5 份 → 各自 `extract_json` → `get_meta_review`（Area Chair，`meta_reviewer_system_prompt` `:355`）聚合 → 9 个数值字段（Originality/Quality/Clarity/Significance/Soundness/Presentation/Contribution/Overall/Confidence）取 `int(round(np.mean(scores)))`；`Decision` 取 meta-reviewer 的。
- 反思（`num_reflections=5`）：再跑 4 轮 `reviewer_reflection_prompt`（`:246-265`），含 `"I am done"` 早停。
- few-shot（`:323-353`）：`fewshot_papers/reviews` 三对（`132_automated_relational`、`attention`、`2_carpe_diem`），优先读同名 `.txt`，否则 `load_paper(pdf)`。

`load_paper()`（`:267-299`）三级回退：`pymupdf4llm.to_markdown` → `pymupdf` 逐页 `get_text()` → `pypdf.PdfReader.extract_text()`；`len(text) < min_size(=100)` 视为失败进入下一级。

`perform_improvement()`（`:386-395`）原文：
```
The following review has been created for your research paper:
"""
{review}
"""

Improve the text using the review.
```
（`review = json.dumps(review)`，用写作阶段的 coder 改 `template.tex`。）

---

## 3. Aider 通道：实际 Prompt 模板

Aider 不用 ```json，而是靠"自然语言指令 + SEARCH/REPLACE 块"改文件。以下均为 `coder.run(...)` 的实参原文。

### 3.1 实验初始 Prompt（`perform_experiments.py:12-27` 的 `coder_prompt`，`.format` 后）
模板原文（占位符 `{title}/{idea}/{max_runs}/{baseline_results}`）：
```
Your goal is to implement the following idea: {title}.
The proposed experiment is as follows: {idea}.
You are given a total of up to {max_runs} runs to complete the necessary experiments. You do not need to use all {max_runs}.

First, plan the list of experiments you would like to run. For example, if you are sweeping over a specific hyperparameter, plan each value you would like to test for each run.

Note that we already provide the vanilla baseline results, so you do not need to re-run it.

For reference, the baseline results are as follows:

{baseline_results}

After you complete each change, we will run the command `python experiment.py --out_dir=run_i' where i is the run number and evaluate the results.
YOUR PROPOSED CHANGE MUST USE THIS COMMAND FORMAT, DO NOT ADD ADDITIONAL COMMAND LINE ARGS.
You can then implement the next thing on your list.
```
> 注意原文中 `--out_dir=run_i'` 结尾是**单引号**（源码如此），模型一般能理解这是排版笔误。

### 3.2 每轮实验结果反馈 Prompt（`run_experiment` 成功分支，`:66-81`）
```
Run {run_num} completed. Here are the results:
{results}

Decide if you need to re-plan your experiments given the result (you often will not need to need to).

Someone else will be using `notes.txt` to perform a writeup on this in the future.
Please include *all* relevant information for the writeup on Run {run_num}, including an experiment description and the run number. Be as verbose as necessary.

Then, implement the next thing on your list.
We will then run the command `python experiment.py --out_dir=run_{run_num + 1}'.
YOUR PROPOSED CHANGE MUST USE THIS COMMAND FORMAT, DO NOT ADD ADDITIONAL COMMAND LINE ARGS.
If you are finished with experiments, respond with 'ALL_COMPLETED'.
```
失败分支（`:52-60`）：`f"Run failed with the following error {stderr_output}"`，`stderr_output` 截断到尾部 `MAX_STDERR_OUTPUT=1500` 字符（前缀 `"..."`）。超时分支（`:83`）：`"Run timed out after {timeout} seconds"`。

### 3.3 绘图 Prompt（`perform_experiments`，`:144-153`）
```
Great job! Please modify `plot.py` to generate the most relevant plots for the final writeup.

In particular, be sure to fill in the "labels" dictionary with the correct names for each run that you want to plot.

Only the runs in the `labels` dictionary will be plotted, so make sure to include all relevant runs.

We will be running the command `python plot.py` to generate the plots.
```

### 3.4 notes 补图说明 Prompt（`:159-164`）
```
Please modify `notes.txt` with a description of what each plot shows along with the filename of the figure. Please do so in-depth.

Somebody else will be using `notes.txt` to write a report on this in the future.
```

### 3.5 写作逐节 Prompt（`perform_writeup.py:401-520`）

抽象节（`:405-416`）：
```
We've provided the `latex/template.tex` file to the project. We will be filling it in section by section.

First, please fill in the "Title" and "Abstract" sections of the writeup.

Some tips are provided below:
{per_section_tips["Abstract"]}

Before every paragraph, please include a brief description of what you plan to write in that paragraph in a comment.

Be sure to first name the file and use *SEARCH/REPLACE* blocks to perform these edits.
```

正文节（`:430-444`，section ∈ Introduction/Background/Method/Experimental Setup/Results/Conclusion）：
```
Please fill in the {section} of the writeup. Some tips are provided below:
{per_section_tips[section]}

Be sure to use \cite or \citet where relevant, referring to the works provided in the file.
Do not cite anything that is not already in `references.bib`. Do not add any new entries to this.

Keep the experimental results (figures and tables) only in the Results section, and make sure that any captions are filled in.
In this pass, do not reference anything in later sections of the paper.

Before every paragraph, please include a brief description of what you plan to write in that paragraph in a comment.

Be sure to first name the file and use *SEARCH/REPLACE* blocks to perform these edits.
```

Related Work 草图（`:447-459`）：只在 LaTeX 注释（`%`）里规划，明确"Do not modify `references.bib`"。

自查/精修：`refinement_prompt`（`:197-204`）= "criticize and refine only the {section}" + `error_list`（`:182-195`，含"Unenclosed math symbols""References to papers not in .bib, DO NOT ADD ANY NEW CITATIONS!"等 13 条）；`second_refinement_prompt`（`:206-218`）= 带 `{tips}` 的二次压缩。调用前都做 `.replace(r"{{","{").replace(r"}}","}")` 把双花括号还原。

### 3.6 Aider 的"无结构化返回"契约

`perform_experiments` 只检测 `if "ALL_COMPLETED" in coder_out`（`:132`）作为"实验做完"信号；其余阶段 `coder.run(...)` 的返回值基本被丢弃。这意味着：
- 改 Prompt 时**必须保留 `ALL_COMPLETED` 字样**，否则实验循环会一直跑满 `MAX_RUNS`。
- 绘图循环靠 `run_plotting` 的 `returncode`（不是 coder 输出）判定成败（`:153-158`）。

---

## 4. 三个内置模板的接口契约（`templates/{nanoGPT,2d_diffusion,grokking}`）

### 4.1 共同契约（框架 mandates）

| 契约 | 说明 | 源码锚点 |
| --- | --- | --- |
| `experiment.py --out_dir=<dir>` | 必须接受且只依赖这一个自定义参数 | `perform_experiments.py:39-43` 写死 `["python","experiment.py",f"--out_dir=run_{n}"]` |
| `<out_dir>/final_info.json` | 成功运行后必须生成，结构 `{dataset: {"means": {...}, "stderrs": {...}, "final_info_dict": {...}}}` | 三模板 `__main__` 尾段；`launch_scientist.py:167`/`perform_experiments.py:64` 用 `{k:v["means"]}` 解析 |
| `run_0/final_info.json` | 人工基线，模板目录内必须预置 | `launch_scientist.py:172-176` |
| `plot.py` | 无参数，扫描 `./run*`，写 `*.png` 到当前目录 | `perform_experiments.py:91-94` 写死 `["python","plot.py"]` |
| `plot.py` 的 `labels` 字典 | Aider 被提示填它；只有其中的 run 会被画 | 三模板 `plot.py` + `perform_experiments.py:144-149` |
| `prompt.json` | `{"system": ..., "task_description": ...}` | `generate_ideas.py:101-103` |
| `seed_ideas.json` | 想法数组，字段同 idea dict | `generate_ideas.py:88-90` |
| `latex/template.tex` | 顶部 `\begin{filecontents}{references.bib}` + 各节 `XXX HERE` 占位 | `perform_writeup.py:22-29` 检查 bib |

> 框架解析 `final_info.json` 时**只取 `["means"]`**；`stderrs`/`final_info_dict` 供写作/绘图参考，不参与框架逻辑。若某 run 的 JSON 顶层不是 `{dataset:{...}}`（例如直接是 `{"means":...}`），`{k: v["means"]}` 会对标量 `v` 报错。

### 4.2 nanoGPT（`templates/nanoGPT`）

- **CLI**（`experiment.py:681-683`）：仅 `--out_dir`（默认 `run_0`）。
- **数据集/种子**（`:687-690`）：`num_seeds = {"shakespeare_char":3, "enwik8":1, "text8":1}`，遍历 `["shakespeare_char","enwik8","text8"]`。
- **单 run 指标**（`train()` 尾 `:602-604,672`）：`final_train_loss`, `best_val_loss`, `total_train_time`, `avg_inference_tokens_per_second`。
- **聚合**（`:703-713`）：`final_info_dict = {k:[d[k] for d in final_info_list] ...}`；`means = {f"{k}_mean": np.mean(v)}`；`stderrs = {f"{k}_stderr": np.std(v)/len(v)}`；写 `<out_dir>/final_info.json` 与 `all_results.npy`（`:716-720`）。
- **绘图**（`plot.py`）：读每个 `run*/final_info.json` + `all_results.npy`；`datasets=["shakespeare_char","enwik8","text8"]`；`labels={"run_0":"Baselines"}`（`:44-47`）；产 `train_loss_<dataset>.png`、`val_loss_<dataset>.png`（`:75,94`）。
- **prompt.json system**：`"You are an ambitious AI researcher who is looking to publish a paper that will contribute significantly to the field."`
- **task_description**：字符级多数据集语言模型训练（详见 `prompt.json:3`）。
- **基线准备**：需先跑 `data/{enwik8,shakespeare_char,text8}/prepare.py`。

### 4.3 2d_diffusion（`templates/2d_diffusion`）

- **CLI**（`experiment.py:167-178`）：**参数最多**——`--train_batch_size(256)`, `--eval_batch_size(10000)`, `--learning_rate(3e-4)`, `--num_timesteps(100)`, `--num_train_steps(10000)`, `--beta_schedule(linear|quadratic)`, `--embedding_dim(128)`, `--hidden_size(256)`, `--hidden_layers(3)`, `--out_dir(run_0)`。
  - **排错要点**：框架调用时只传 `--out_dir`，其余用默认值；Aider 若把实验设计写成"用 `--learning_rate=...` 启动"会无效（框架不加额外参数）——必须改脚本内默认值/常量。
- **单 run 指标**（`:276-283`）：按 dataset 直接写 `means`（`training_time`, `eval_loss`, `inference_time`, `kl_divergence`），**没有 stderrs/final_info_dict 顶层结构**（与另两个模板不同，但 `{k:v["means"]}` 解析仍成立）。
- **数据**：`datasets=["circle","dino","line","moons"]`；`DatasaurusDozen.tsv`, `datasets.py`, `ema_pytorch.py`；`ee.kldiv` 用 NPEET。
- **绘图**（`plot.py`）：`labels` 缺省自动补齐（`if run not in labels: labels[run]=run`，`:43-45`）；产 `train_loss.png`、`generated_images.png`。
- **依赖**：`pip install NPEET scikit-learn`。

### 4.4 grokking（`templates/grokking`）

- **CLI**（`experiment.py:374-376`）：仅 `--out_dir`。
- **数据集/种子**（`:387-390`）：`num_seeds = {"x_div_y":3,"x_minus_y":3,"x_plus_y":3,"permutation":3}`，遍历 `["x_div_y","x_minus_y","x_plus_y","permutation"]`。
- **单 run 指标**（`:359-365`）：`final_train_loss`, `final_val_loss`, `final_train_acc`, `final_val_acc`, `step_val_acc_99`。
- **聚合**：与 nanoGPT 同构（`means`/`stderrs`/`final_info_dict` + `all_results.npy`，`:398-415`）。
- **绘图**（`plot.py`）：`datasets` 同上；产 4 类图 `train_loss_/val_loss_/train_acc_/val_acc_<dataset>.png`；`labels={"run_0":"Baselines"}`（`:74-77`）。
- **依赖**：`pip install einops`。

### 4.5 三模板对照表

| 维度 | nanoGPT | 2d_diffusion | grokking |
| --- | --- | --- | --- |
| `--out_dir` 之外参数 | 无 | 9 个（都会被框架忽略） | 无 |
| 数据集数 | 3（字符级 LM） | 4（2D 分布） | 4（算术任务） |
| 每数据集种子数 | 3/1/1 | —（单模型） | 3 |
| `final_info.json` 结构 | `{ds:{means,stderrs,final_info_dict}}` | `{ds:{means}}` | `{ds:{means,stderrs,final_info_dict}}` |
| 附加产物 | `all_results.npy` | `all_results.pkl` | `all_results.npy` |
| 图文件名 | `train_loss_/val_loss_<ds>.png` | `train_loss.png`, `generated_images.png` | 4 类 `*_<ds>.png` |
| `labels` 默认 | `{"run_0":"Baselines"}` | `{"run_0":"Baseline"}` + 自动补 | `{"run_0":"Baselines"}` |
| 需预装 | `data/*/prepare.py` | `NPEET scikit-learn` | `einops` |

---

## 5. LaTeX 变量注入格式

v1 的 LaTeX 不是 Jinja 模板，而是**占位符文本 + Aider SEARCH/REPLACE**。以 `templates/nanoGPT/latex/template.tex` 为例：

### 5.1 占位符（写作阶段被逐节替换）
```
\title{TITLE HERE}
\begin{abstract}
ABSTRACT HERE
\end{abstract}
\section{Introduction}\label{sec:intro}      INTRO HERE
\section{Related Work}\label{sec:related}    RELATED WORK HERE
\section{Background}\label{sec:background}   BACKGROUND HERE
\section{Method}\label{sec:method}           METHOD HERE
\section{Experimental Setup}\label{sec:experimental}  EXPERIMENTAL SETUP HERE
\section{Results}\label{sec:results}         RESULTS HERE
\section{Conclusions and Future Work}\label{sec:conclusion} CONCLUSIONS HERE
```
（`2d_diffusion`/`grokking` 的占位符名相同，仅示例图不同：前者 `generated_images.png`，后者 `train_acc_x_div_y.png`/`train_loss_x_div_y.png`。）

### 5.2 引用块（补引用阶段写入）
```
\begin{filecontents}{references.bib}
@article{lu2024aiscientist, ...}
...
\end{filecontents}
```
- 框架 `generate_latex`（`:22-29`）用正则 `r"\\begin{filecontents}{references.bib}(.*?)\\end{filecontents}"` 定位；**缺失就 `print("No references.bib found in template.tex"); return`（直接不编译）**。
- 补引用时，新 bibtex 由框架插入 `\end{filecontents}` 之前（`perform_writeup:459-461`），正文 `\cite{key}`/`\citet{key}` 由 Aider 添加。

### 5.3 图片路径
```
\graphicspath{{../}}     % 编译 cwd=latex/，图在上一级（结果目录根）
\includegraphics[width=\textwidth]{val_loss_enwik8.png}
```
- 编译在 `latex/` 目录进行（`compile_latex:96` `cwd=cwd=latex/`），所以 `\graphicspath{{../}}` 指向结果目录；`plot.py` 产出的 `*.png` 必须在该目录。
- `generate_latex`（`:40-48`）检查 `\includegraphics` 引用的文件是否真在结果目录，否则让 Aider 修。

### 5.4 编译链（`compile_latex`，`:94-127`）
```
pdflatex -interaction=nonstopmode template.tex
bibtex template
pdflatex -interaction=nonstopmode template.tex
pdflatex -interaction=nonstopmode template.tex
```
每条 `timeout=30`（秒），超时/异常**只打印不抛**；最后 `shutil.move(latex/template.pdf, <Name>.pdf)`（失败仅打印）。之前还有 `chktex template.tex -q -n2 -n24 -n13 -n1` 驱动最多 `num_error_corrections=5` 轮修错（`:83-92`）。

### 5.5 静态一致性检查（`generate_latex`，`:15-81`）
按序检查并让 Aider 修复：① `\cite[a-z]*{...}` 中每个 key 是否在 bib；② `\includegraphics` 引用的 png 是否存在；③ 重复图；④ 重复 `\section`。这四项是"编译前闸门"，二次开发新模板时务必保留这些占位符/命令形态。

---

## 6. 二次开发锚点速查

| 想改什么 | 改哪里 |
| --- | --- |
| 想法/反思/新颖性/引用/评审 Prompt | 本文第 2 节列出的字符串常量（均在模块顶层，改后无需改调用方） |
| Aider 实验/写作指令 | 本文第 3 节（`perform_experiments.py` / `perform_writeup.py` 内 `coder.run` 实参）；保留 `ALL_COMPLETED` |
| 每想法实验轮数 | `perform_experiments.py:8-10` 的 `MAX_ITERS/MAX_RUNS/MAX_STDERR_OUTPUT` |
| 想法数量/反思轮数 | CLI `--num-ideas` / `launch_scientist.py:22` 的 `NUM_REFLECTIONS` |
| 评审模型/参数 | `launch_scientist.py:263-270`（硬编码 gpt-4o） |
| 新增模型 | `llm.py:AVAILABLE_LLMS` + `create_client`（`:317`） |
| 新增模板 | 严格照第 4.1 节契约 + 第 5 节 LaTeX 占位符 |
| 并行/GPU | `launch_scientist.py:95-98,122-152,371-400` |

**完整端到端一次运行的文件流**见 [`pipeline_and_dataflow.md`](pipeline_and_dataflow.md)；函数签名见 [`modules_reference.md`](modules_reference.md)；故障排查见 [`troubleshooting.md`](troubleshooting.md)。
