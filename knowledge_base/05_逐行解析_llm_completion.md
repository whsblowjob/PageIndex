# 05 · 逐行解析：`llm_completion`

> 源码位置：`pageindex/utils.py:33-59`
>
> 定位：整个 PageIndex 项目中**所有同步 LLM 调用的唯一入口**。上千行业务代码（`toc_transformer`、`generate_toc_init` 等）从不直接碰 `litellm`，全部经由这个函数。它的异步孪生兄弟是 `llm_acompletion`（`pageindex/utils.py:63-83`）。

---

## 完整源码

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

---

## 逐行解析

### 第 1 行：函数签名

```python
def llm_completion(model, prompt, chat_history=None, return_finish_reason=False):
```

四个参数对应四种使用需求：

| 参数 | 类型 | 作用 | 谁在用 |
|---|---|---|---|
| `model` | `str` | 模型名，来自 `config.yaml` 的 `model` 字段（默认 `gpt-4o-2024-11-20`） | 所有调用方 |
| `prompt` | `str` | 本轮用户消息 | 所有调用方 |
| `chat_history` | `list \\| None` | 历史对话，默认 `None`（单轮调用） | 只有"截断续写"场景：`toc_transformer`、`extract_toc_content` |
| `return_finish_reason` | `bool` | 是否返回"输出是否被截断"信号，默认 `False` | 需要判断完整性的场景：`generate_toc_init`、`toc_transformer` 等 |

注意这个函数的**返回类型随参数变化**：`return_finish_reason=False` 时返 `str`，`True` 时返 `(str, str)` 元组。这是简单直接的设计（更工程化的写法会拆成两个函数或返回 dataclass），读调用方代码时要留意。

### 第 2-3 行：模型名前缀归一化

```python
    if model:
        model = model.removeprefix("litellm/")
```

- 用户在配置里可能写 `litellm/deepseek/deepseek-chat` 这种带前缀的名字（`pageindex/client.py` 的 `_normalize_retrieve_model` 甚至会主动给非 OpenAI provider 补上这个前缀，因为检索侧的 OpenAI Agents SDK 需要它）。
- 但 `litellm.completion()` 自己不认这个前缀，所以进入真正的 API 调用前要剥掉。
- `removeprefix` 是 Python 3.9+ 方法：只在开头匹配时删除，不匹配则原样返回，比 `replace` 安全（不会误伤名字中间的子串）。
- `if model:` 兜住 `model=None` 的情况（`None.removeprefix` 会抛 `AttributeError`）。

**效果：换模型只需改 `pageindex/config.yaml`，不用改任何代码。**

### 第 4 行：重试预算

```python
    max_retries = 10
```

硬编码 10 次——对一个要发出成百上千次 LLM 请求的索引 pipeline 来说，单次请求因限流/网络抖动失败几乎必然会发生，宽重试预算换整体稳定性。

### 第 5 行：拼装 messages

```python
    messages = list(chat_history) + [{"role": "user", "content": prompt}] if chat_history else [{"role": "user", "content": prompt}]
```

这是一个条件表达式（`A if cond else B`），拆开看：

- **有 `chat_history`**：`list(chat_history) + [新消息]`。`list(...)` 是浅拷贝，避免把新消息 `append` 进调用方持有的列表（调用方还要继续维护这份历史）。
- **无 `chat_history`**：只有一条 user 消息。

典型的多轮使用方式见 `toc_transformer`（`pageindex/page_index.py:395-421`）：上一轮输出被截断时，把 `[user prompt, assistant 半成品]` 作为 `chat_history` 传进来，新 `prompt` 是一句 *"Please continue ... from where you left off"*，让模型接着写。

另外注意：**messages 里没有 system 消息**。项目的"系统级"约束（`_SYSTEM_HARDENING`）是拼在 user prompt 开头的（见 `pageindex/page_index.py:38-45`），这样对所有 provider 行为一致。

### 第 6-7 行：重试循环 + try

```python
    for i in range(max_retries):
        try:
```

把“整次 API 调用 + 取结果”包在 try 里。任何异常（网络错误、限流、认证失败、provider 报错）都走同一条重试路径，不区分异常类型。粗糙但有效；代价是对“不可重试”的错误（如 API key 无效）也会白重试 10 次。

### 第 8-12 行：真正的 API 调用

```python
            response = litellm.completion(
                model=model,
                messages=messages,
                temperature=0,
            )
```

- `litellm.completion` 是多 provider 统一接口（OpenAI/Anthropic/DeepSeek/Ollama...），接口形状仿 OpenAI SDK。
- **`temperature=0` 是整个项目最重要的参数之一**：这里 LLM 是“解析器”不是“创作者”，同一份 PDF 每次应该解析出尽量一致的目录；而且下游有严格的 JSON 解析和正则校验，随机性只会增加失败率。
- 项目在模块顶部设了 `litellm.drop_params = True`（`pageindex/utils.py` 开头）：某些模型不支持 `temperature` 时自动丢弃该参数而不是报错，保障多模型兼容性。

### 第 13 行：取文本内容

```python
            content = response.choices[0].message.content
```

OpenAI 风格的响应结构：`choices` 是候选列表（这里永远只要第一个），`.message.content` 是模型输出的纯文本。后续由调用方用 `extract_json()`（`pageindex/utils.py:100-131`）去解析成结构化数据——**本函数只负责拿到字符串，不负责理解它**，职责单一。

### 第 14-17 行：finish_reason 翻译与返回

```python
            if return_finish_reason:
                finish_reason = "max_output_reached" if response.choices[0].finish_reason == "length" else "finished"
                return content, finish_reason
            return content
```

- Provider 原生的 `finish_reason` 取值很多（`stop`、`length`、`tool_calls`...），项目只关心一个问题：**输出是不是被 token 上限截断了？** 所以把它压缩成二值语义：`"length" → "max_output_reached"`，其他一律 `"finished"`。这是防腐层（anti-corruption layer）：业务代码依赖项目自己的词汇，不依赖 provider 细节。
- 调用方怎么用这个信号？两种典型反应：
  - `generate_toc_init`（`pageindex/page_index.py:669-674`）：`finish_reason != 'finished'` 直接 `raise`，交给上层降级；
  - `toc_transformer`（`:387-421`）：进入续写循环，拼接多轮输出。
- 注意 `return` 在 try 块内部——**成功就立刻返回，循环只为失败而存在**。

### 第 18-20 行：异常处理入口

```python
        except Exception as e:
            print('************* Retrying *************')
            logging.error(f"Error: {e}")
```

双通道输出：`print` 给盯着终端跑 `run_pageindex.py` 的用户一个醒目提示，`logging.error` 进日志系统留存证据。项目里大量用 `print` 做进度提示（如 `'start toc_transformer'`），是偏研究风格的代码。

### 第 21-22 行：退避后重试

```python
            if i < max_retries - 1:
                time.sleep(1)
```

- 还有剩余次数才 sleep；最后一次失败不白等 1 秒。
- 固定 1 秒间隔，而非指数退避（exponential backoff）。对于严格限流的 API，这里是一个可以自己动手改进的点（比如 `time.sleep(2 ** i)`）。
- 对比异步版 `llm_acompletion`：那边用 `await asyncio.sleep(1)`，不阻塞事件循环里其他并发请求。

### 第 23-27 行：重试耗尽的兜底

```python
            else:
                logging.error('Max retries reached for prompt: ' + prompt)
                if return_finish_reason:
                    return "", "error"
                return ""
```

**最值得玩味的设计决策：失败不抛异常，返回空字符串。**

- 这意味着调用方永远不需要写 `try/except`，但必须能处理“空结果”。事实上整条链路都按这个约定设计：`extract_json("")` 返回 `{}`（`pageindex/utils.py:100-131`），`{}.get('toc_detected', 'no')` 取到保守默认值 `'no'`，最终表现为“没检测到/不可信”，触发 `meta_processor` 的降级路线（`pageindex/page_index.py:1124-1162`）。
- 换句话说：**“坏输出”被统一降级为“空输出”，再由上层的校验/自愈机制兜底**，单点 LLM 故障不会直接崩掉整个几千次调用的 pipeline。
- `return_finish_reason=True` 时返回的第二个值是 `"error"`（第三种取值，区别于 `"finished"` / `"max_output_reached"`），让需要完整性信号的调用方能区分“正常结束”和“彻底失败”。
- 小瑕疵：`logging.error('... ' + prompt)` 会把整个 prompt（可能几万 token 的文档文本）写进日志，日志体积会很大。

---

## 执行流图

```
调用 llm_completion(model, prompt, ...)
        │
        ▼
 剥离 "litellm/" 前缀
        │
        ▼
 拼 messages（可选拼接 chat_history，浅拷贝不污染调用方）
        │
        ▼
 ┌─── 循环 i = 0..9 ─────────────────────────────┐
 │  litellm.completion(temperature=0)               │
 │      │成功                        │异常           │
 │      ▼                            ▼              │
 │  取 content                print + 记日志          │
 │      │                            │              │
 │  需要 finish_reason?        还有剩余次数?         │
 │   是: "length"→"max_output_   是: sleep(1)、重试 ─┘
 │       reached"，否则          否: 返回 "" 或 ("","error")
 │       "finished"
 │   否: 直接返回 content
 └── return（成功即退出循环）
```

## 设计要点速记（5 条）

1. **单一收口**：全项目同步 LLM 调用都走这里，换 provider/加监控/改重试策略只改一处。
2. **temperature=0**：解析器定位，追求确定性而非创造性。
3. **失败降级为空值而非异常**：和 `extract_json` 返 `{}`、`.get(k, '保守默认值')` 构成一整条“静默降级”链，由 `meta_processor` 的验证/降级状态机最终兜底。
4. **finish_reason 防腐层**：把 provider 细节翻译成项目自己的三值语义（finished / max_output_reached / error），支撑截断续写机制。
5. **chat_history 只为续写而生**：这不是聊天机器人，多轮对话唯一的用途是把被截断的长 JSON 接着写完。
