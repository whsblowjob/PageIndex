# 03 · LLM 在 PageIndex 中的 6 大角色（逐个代码拆解）

项目对 LLM 的定位：**"能读懂文档但不可靠的外包工人"**。LLM 只负责语义判断，所有能用普通代码做的事（切分、对齐、建树、校验）一律不交给 LLM。本篇按角色逐个看代码；防御侧（"被防御的对象"）单独见 04 篇。

---

## 角色 1：分类器（Classifier）—— 回答 yes/no

| 函数 | 位置 | 问 LLM 的问题 |
|---|---|---|
| `toc_detector_single_page` | `pageindex/page_index.py:172-190` | 这一页是不是目录页？ |
| `detect_page_index` | `pageindex/page_index.py:271-289` | 这份目录里印了页码吗？ |
| `check_if_toc_transformation_is_complete` | `pageindex/page_index.py:216-235` | 你转换出的目录完整吗？ |
| `check_title_appearance_in_start` | `pageindex/page_index.py:115-139` | 这个标题是不是在页面**开头**？ |

以最后一个为例（它的结果直接影响页码区间计算，见 02 篇 `post_processing`）：

```python
# pageindex/page_index.py:115-139
async def check_title_appearance_in_start(title, page_text, model=None, logger=None):    
    prompt = _SYSTEM_HARDENING + f"""
    You will be given the current section title and the current page_text.
    Your job is to check if the current section starts in the beginning of the given page_text.
    If there are other contents before the current section title, then the current section does not start in the beginning of the given page_text.
    If the current section title is the first content in the given page_text, then the current section starts in the beginning of the given page_text.

    Note: do fuzzy matching, ignore any space inconsistency in the page_text.

    The given section title is {title}.
    The given page_text is:
    {_secure_doc_text(page_text)}
    
    reply format:
    {{
        "thinking": <why do you think the section appears or starts in the page_text>
        "start_begin": "yes or no" (yes if the section starts in the beginning of the page_text, no otherwise)
    }}
    Directly return the final JSON structure. Do not output anything else."""

    response = await llm_acompletion(model=model, prompt=prompt)
    response = extract_json(response)
    return response.get("start_begin", "no")
```

**分类器类 prompt 的共同模式**：

1. 把判断标准写成两个互斥的 if 句式（"If there are other contents before... / If ... is the first content..."），不给模型自由发挥空间；
2. 要求 fuzzy matching，因为 PDF 抽文本常有空格/换行噪声；
3. `thinking` 字段在结论前（廉价 CoT）；
4. `.get("start_begin", "no")` 兜底取保守默认值。

并发包装器 `check_title_appearance_in_start_concurrent`（`pageindex/page_index.py:142-169`）用 `asyncio.gather(*tasks, return_exceptions=True)` 对全部条目并发提问，单个异常不会拖垮整批，失败项降级为 `'no'`。

## 角色 2：抽取器（Extractor）—— 从脏文本里抠结构

`extract_toc_content`（`pageindex/page_index.py:237-269`）从目录页原文抽出干净的目录文本；`toc_transformer`（`:364-425`）再把它转成 JSON。两者都内置**截断续写循环**，这是处理长目录的关键：

```python
# pageindex/page_index.py:399-421（toc_transformer 的续写循环，节选）
    continue_prompt = "Please continue the table of contents JSON structure from where you left off. Directly output only the remaining part."

    position = last_complete.rfind('}')
    if position != -1:
        last_complete = last_complete[:position+2]      # 剪到最后一个完整条目，丢弃半截的残片

    max_attempts = 5
    for attempt in range(max_attempts):
        new_complete, finish_reason = llm_completion(model=model, prompt=continue_prompt, chat_history=chat_history, return_finish_reason=True)
        if new_complete.startswith('```json'):
            new_complete = get_json_content(new_complete)
        last_complete = last_complete + new_complete    # 字符串级拼接
        chat_history.append({"role": "user", "content": continue_prompt})
        chat_history.append({"role": "assistant", "content": new_complete})
        if_complete = check_if_toc_transformation_is_complete(toc_content, last_complete, model)
        if if_complete == "yes" and finish_reason == "finished":
            break
    else:
        raise Exception('Failed to complete TOC transformation after maximum retries')
```

非显易见的点：

- `rfind('}')` 剪尾：截断可能发生在某条目中间，先剪到最后一个完整 `}` 再拼接，避免拼出非法 JSON；
- 终止条件是**双重的**：`finish_reason == "finished"`（机器信号：没被截断）**且** 另一次 LLM 调用确认内容完整（语义信号）；
- `for...else`：5 轮都没完成就抛异常，交给上层降级。

## 角色 3：定位器（Locator）—— 把标题对到物理页码

这是建树的核心工作，也是 `<physical_index_X>` 标签存在的意义：

| 函数 | 位置 | 场景 |
|---|---|---|
| `toc_index_extractor` | `pageindex/page_index.py:332-362` | 路线A：在样本页里定位少量标题（用于算 offset） |
| `add_page_number_to_toc` | `pageindex/page_index.py:550-586` | 路线B：已知目录结构，逐 chunk 填页码 |
| `generate_toc_init` / `generate_toc_continue` | `pageindex/page_index.py:642-674` / `:602-639` | 路线C：识别标题+判层级+定页码一步到位 |
| `single_toc_item_index_fixer` | `pageindex/page_index.py:897-921` | 修复阶段：在夹逼区间里重新定位单个标题 |

看修复场景的定位 prompt（搜索空间已被代码压缩到几页，准确率自然提升）：

```python
# pageindex/page_index.py:897-921
async def single_toc_item_index_fixer(section_title, content, model=None):
    toc_extractor_prompt = """
    You are given a section title and several pages of a document, your job is to find the physical index of the start page of the section in the partial document.

    The provided pages contains tags like <physical_index_X> and <physical_index_X> to indicate the physical location of the page X.

    Reply in a JSON format:
    {
        "thinking": <explain which page, started and closed by <physical_index_X>, contains the start of this section>,
        "physical_index": "<physical_index_X>" (keep the format)
    }
    Directly return the final JSON structure. Do not output anything else."""

    prompt = (
        _SYSTEM_HARDENING + toc_extractor_prompt
        + '\nSection Title:\n' + _secure_doc_text(str(section_title))
        + '\nDocument pages:\n' + _secure_doc_text(content)
    )
    response = await llm_acompletion(model=model, prompt=prompt)
    json_content = extract_json(response)    
    physical_index = json_content.get('physical_index')
    if physical_index is None:
        return None
    return convert_physical_index_to_int(physical_index)
```

注意所有定位类 prompt 都强调 *"keep the format"*：输出必须是 `<physical_index_X>` 字面量而不是裸数字，这样才能用正则（`_PHYSICAL_INDEX_MARKER_RE`）严格验证"这个标签确实在你看过的文本里出现过"（见 04 篇）。

## 角色 4：审计员（Verifier）—— LLM 检查 LLM

用一次**独立的** LLM 调用去审计另一次 LLM 的产出：

```python
# pageindex/page_index.py:79-112（check_title_appearance，节选）
async def check_title_appearance(item, page_list, start_index=1, model=None):    
    title=item['title']
    if 'physical_index' not in item or item['physical_index'] is None:
        return {'list_index': item.get('list_index'), 'answer': 'no', 'title':title, 'page_number': None}

    page_number = item['physical_index']
    page_text = page_list[page_number-start_index][0]   # 只取声称的那一页

    prompt = _SYSTEM_HARDENING + f"""
    Your job is to check if the given section appears or starts in the given page_text.
    Note: do fuzzy matching, ignore any space inconsistency in the page_text.
    The given section title is {title}.
    The given page_text is:
    {_secure_doc_text(page_text)}
    ..."""
```

重点：审计时**只把声称的那一页文本喂进去**（`page_list[page_number-start_index][0]`）——如果页码错了，标题就不在这页里，LLM 很容易判 no。把"验证页码对不对"转化成了更简单可靠的"文本包含判断"。

上层 `verify_toc`（`pageindex/page_index.py:1065-1117`）的两个细节：

```python
    # 早退：最后一个有效页码连文档一半都不到 → 直接判 0 分，触发降级
    if last_physical_index is None or last_physical_index < len(page_list)/2:
        return 0, []
    ...
    tasks = [check_title_appearance(item, page_list, start_index, model)
             for item in indexed_sample_list]
    results = await asyncio.gather(*tasks)      # 全量并发抽查
    ...
    accuracy = correct_count / checked_count if checked_count > 0 else 0
```

- **廉价的结构性早退**：目录只覆盖前半本书，说明生成严重不完整，不用花 LLM 调用就能判死；
- `accuracy` 和 `incorrect_results`（含每个错误条目的 `list_index`）驱动 `meta_processor` 的"通过/修复/降级"三分支（见 04 篇）。

## 角色 5：摘要生成器（Summarizer）—— 唯一的"创作型"任务

```python
# pageindex/utils.py:579-587
async def generate_node_summary(node, model=None):
    prompt = f"""You are given a part of a document, your task is to generate a description of the partial document about what are main points covered in the partial document.

    Partial Document Text: {node['text']}
    
    Directly return the description, do not include any other text.
    """
    response = await llm_acompletion(model, prompt)
    return response
```

这些 summary 不是给人看的，是**给检索期 LLM 看的导航信息**。Markdown 路线还有个省钱技巧：

```python
# pageindex/page_index_md.py:10-16
async def get_node_summary(node, summary_token_threshold=200, model=None):
    node_text = node.get('text')
    num_tokens = count_tokens(node_text, model=model)
    if num_tokens < summary_token_threshold:
        return node_text          # 短节点直接拿原文当 summary，不调 LLM
    else:
        return await generate_node_summary(node, model=model)
```

另外 `generate_summaries_for_structure_md`（`pageindex/page_index_md.py:19-29`）区分叶子和父节点：叶子写 `summary`，父节点写 `prefix_summary`（只概括子节点之前的"引言段"，避免和子节点 summary 重复）。

## 角色 6：检索推理引擎（Retrieval Reasoner）—— 替代 vector search 的那个 LLM

索引建完后，检索阶段的 LLM 扮演完全不同的角色：**它本身就是检索算法**。没有 embedding、没有相似度计算，"算法"写在 system prompt 里：

```python
# examples/agentic_vectorless_rag_demo.py:44-52
AGENT_SYSTEM_PROMPT = """
You are PageIndex, a document QA assistant.
TOOL USE:
- Call get_document() first to confirm status and page/line count.
- Call get_document_structure() to identify relevant page ranges.
- Call get_page_content(pages=\"5-7\") with tight ranges; never fetch the whole document.
- Before each tool call, output one short sentence explaining the reason.
Answer based only on tool output. Be concise.
"""
```

三个 tool 是对 `pageindex/retrieve.py` 纯函数的轻包装：

```python
# examples/agentic_vectorless_rag_demo.py:67-79
    @function_tool
    def get_document_structure() -> str:
        """Get the document's full tree structure (without text) to find relevant sections."""
        return client.get_document_structure(doc_id)

    @function_tool
    def get_page_content(pages: str) -> str:
        """
        Get the text content of specific pages or line numbers.
        Use tight ranges: e.g. '5-7' for pages 5 to 7, '3,8' for pages 3 and 8, '12' for page 12.
        For Markdown documents, use line numbers from the structure's line_num field.
        """
        return client.get_page_content(doc_id, pages)
```

检索侧为 LLM 省 token 的关键设计——给 agent 的树**去掉 text 字段**，只留 title/summary/页码区间：

```python
# pageindex/retrieve.py:100-107
def get_document_structure(documents: dict, doc_id: str) -> str:
    """Return tree structure JSON with text fields removed (saves tokens)."""
    doc_info = documents.get(doc_id)
    if not doc_info:
        return json.dumps({'error': f'Document {doc_id} not found'})
    structure = doc_info.get('structure', [])
    structure_no_text = remove_fields(structure, fields=['text'])
    return json.dumps(structure_no_text, ensure_ascii=False)
```

完整检索循环：

```
 用户问题
    │
    ▼
 agent 读树（title + summary + 页码区间）
    │   推理：哪些 node 可能包含答案？
    ▼
 get_page_content(pages=\"21-22\")    ← 精确取回原文
    │
    ▼
 基于原文作答，天然带页码引用（traceable）
```

## 反证：Markdown 路线几乎不用 LLM

**LLM 的介入范围严格随"语义模糊度"伸缩**。Markdown 的标题层级在 `#` 里是显式的，所以建树全程用正则，一次 LLM 都不调：

```python
# pageindex/page_index_md.py:32-68（extract_nodes_from_markdown，节选）
def extract_nodes_from_markdown(markdown_content):
    header_pattern = r'^(#{1,6})\\s+(.+)$'
    bold_heading_pattern = r'^\\*\\*(.+?)\\*\\*\\s*$'
    code_block_pattern = r'^```'
    ...
    for line_num, line in enumerate(lines, 1):
        if re.match(code_block_pattern, stripped_line):
            in_code_block = not in_code_block      # 跟踪代码块，避免把代码里的 # 误认为标题
            continue
        if not in_code_block:
            match = re.match(header_pattern, stripped_line)
            if match:
                title = match.group(2).strip()
                level = len(match.group(1))        # 层级 = # 的个数，不需要 LLM 判断
                node_list.append({'node_title': title, 'line_num': line_num, 'level': level})
```

PDF 路线里 LLM 干的活（识别标题、判层级、定页码），在 MD 路线里全部退化成正则和计数；LLM 只在 summary 生成时登场（且短节点连这一步都省了）。这是理解本项目"LLM 只做代码做不了的事"设计哲学最好的对照样本。
