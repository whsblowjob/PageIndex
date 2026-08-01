# 01 · LLM 调用基础设施

所有 LLM 调用都收口在 `pageindex/utils.py` 的两个函数里，其余上千行业务代码从不直接碰 `litellm`。这一层决定了整个项目对 LLM 的"使用姿势"。

---

## 1. 同步调用：`llm_completion`

> 源码：`pageindex/utils.py:33-59`

```python
def llm_completion(model, prompt, chat_history=None, return_finish_reason=False):
    if model:
        model = model.removeprefix("litellm/")
    max_retries = 10
    messages = list(chat_history) + [{"role": "user", "content": prompt}] if chat_history else [{"role": "user", "content": prompt}]
    for i in range(max_retries):
        try:
            response = litellm.completion(
                model=model,
                messages=messages,
                temperature=0,
            )
            content = response.choices[0].message.content
            if return_finish_reason:
                finish_reason = "max_output_reached" if response.choices[0].finish_reason == "length" else "finished"
                return content, finish_reason
            return content
        except Exception as e:
            print('************* Retrying *************')
            logging.error(f"Error: {e}")
            if i < max_retries - 1:
                time.sleep(1)
            else:
                logging.error('Max retries reached for prompt: ' + prompt)
                if return_finish_reason:
                    return "", "error"
                return ""
```

### 逐点解释

| 代码 | 为什么这么写 |
|---|---|
| `model.removeprefix("litellm/")` | 用户可能配置 `litellm/deepseek/deepseek-chat` 这样的带前缀模型名，统一剥掉前缀后交给 `litellm.completion`。**换模型只需要改 `pageindex/config.yaml` 的 `model` 字段，不用改任何代码**。 |
| `temperature=0` | 这里 LLM 是**解析器不是创作者**。同样的 PDF 每次跑出来的树应该尽量一致，所以取贪心解码。 |
| `max_retries = 10` + `time.sleep(1)` | 网络抖动 / 限流场景下暴力重试。注意兜底行为：重试耗尽后**返回空字符串而不是抛异常**，让上层的校验逻辑（见 04 篇）把空结果当"不可信"处理。 |
| `chat_history` 参数 | 服务于**截断续写机制**：当上一轮输出被 token 上限截断时，把历史对话带上，再发一句 "please continue"（见 `toc_transformer`，02 篇）。 |
| `finish_reason` 映射 | litellm 的 `finish_reason == "length"` 表示输出被截断，项目把它翻译成自己的语义 `"max_output_reached"`。调用方靠这个信号决定要不要续写。 |

## 2. 异步调用：`llm_acompletion`

> 源码：`pageindex/utils.py:63-83`

```python
async def llm_acompletion(model, prompt):
    if model:
        model = model.removeprefix("litellm/")
    max_retries = 10
    messages = [{"role": "user", "content": prompt}]
    for i in range(max_retries):
        try:
            response = await litellm.acompletion(
                model=model,
                messages=messages,
                temperature=0,
            )
            return response.choices[0].message.content
        except Exception as e:
            print('************* Retrying *************')
            logging.error(f"Error: {e}")
            if i < max_retries - 1:
                await asyncio.sleep(1)
            else:
                logging.error('Max retries reached for prompt: ' + prompt)
                return ""
```

**什么时候用异步版？** 所有"对 N 个对象各发一次独立请求"的场景，配合 `asyncio.gather` 大规模并发：

- `verify_toc` 并发验证每个 TOC 条目（`pageindex/page_index.py:1099-1103`）
- `check_title_appearance_in_start_concurrent` 并发判断每个标题是否在页首（`pageindex/page_index.py:142-169`）
- `generate_summaries_for_structure` 并发生成全树 summary（`pageindex/utils.py:590-597`）
- `fix_incorrect_toc` 并发修复所有错误条目（`pageindex/page_index.py:1002-1006`）

注意它比同步版**少了 `chat_history` 和 `finish_reason`**——并发场景都是单轮短输出任务，不需要续写。

## 3. 容错解析：`extract_json`

LLM 被要求 "Directly return the final JSON structure"，但现实中它可能包 markdown 围栏、写 Python 的 `None`、留尾逗号。`extract_json` 就是这层"消毒"：

> 源码：`pageindex/utils.py:100-131`

```python
def extract_json(content):
    try:
        # First, try to extract JSON enclosed within ```json and ```
        start_idx = content.find("```json")
        if start_idx != -1:
            start_idx += 7  # Adjust index to start after the delimiter
            end_idx = content.rfind("```")
            json_content = content[start_idx:end_idx].strip()
        else:
            # If no delimiters, assume entire content could be JSON
            json_content = content.strip()

        # Clean up common issues that might cause parsing errors
        json_content = json_content.replace('None', 'null')  # Replace Python None with JSON null
        json_content = json_content.replace('\n', ' ').replace('\r', ' ')  # Remove newlines
        json_content = ' '.join(json_content.split())  # Normalize whitespace

        # Attempt to parse and return the JSON object
        return json.loads(json_content)
    except json.JSONDecodeError as e:
        logging.error(f"Failed to extract JSON: {e}")
        # Try to clean up the content further if initial parsing fails
        try:
            # Remove any trailing commas before closing brackets/braces
            json_content = json_content.replace(',]', ']').replace(',}', '}')
            return json.loads(json_content)
        except:
            logging.error("Failed to parse JSON even after cleanup")
            return {}
    except Exception as e:
        logging.error(f"Unexpected error while extracting JSON: {e}")
        return {}
```

### 三层清洗策略

1. **围栏剥离**：`find("```json")` + `rfind("```")` 抠出围栏内部；没有围栏就假定整段是 JSON。
2. **常见错误替换**：`None → null`（LLM 常把 Python 语法混进 JSON）、压掉换行和多余空白。
3. **二次抢救**：首次 `json.loads` 失败后，删掉 `,]` / `,}` 尾逗号再试一次。

**最重要的一点**：失败时返回 `{}` 而不是抛异常——和 `llm_completion` 返回空字符串一致，**"坏输出"被统一降级为"空输出"**，由上层校验/降级逻辑兜底，pipeline 不会因为一次坏 JSON 直接崩掉。

## 4. prompt 的统一风格

浏览 `pageindex/page_index.py` 里所有 prompt，会发现三个固定模式：

1. **强制 JSON 输出**：每个 prompt 都以 `Directly return the final JSON structure. Do not output anything else.` 结尾。
2. **廉价 chain-of-thought**：判断类任务的返回格式里都有一个 `"thinking"` 字段放在结论字段**之前**，例如（`pageindex/page_index.py:179-183`）：

   ```
   {
       "thinking": <why do you think there is a table of content in the given text>
       "toc_detected": "<yes or no>",
   }
   ```

   让模型先写理由再给结论（自回归生成中结论会被前面的推理 token 约束），代码里只取结论字段：`json_content.get('toc_detected', 'no')`。
3. **`.get(key, 保守默认值)`**：所有结果读取都带默认值，且默认值是保守方向（`'no'`）——解析失败时宁可当"没检测到"，触发后备路线。

## 5. 模型配置

> 源码：`pageindex/config.yaml`

```yaml
model: "gpt-4o-2024-11-20"   # 索引期使用（本篇所有函数）
retrieve_model: null          # 检索期 agent 使用，默认回落到 model
```

索引期和检索期可以用不同模型：建树是结构化解析任务，检索是推理任务，后者往往值得配更强的 reasoning 模型。`pageindex/client.py` 的 `_normalize_retrieve_model` 会为非 OpenAI provider 自动补 `litellm/` 前缀。
