# PageIndex 中的 LLM：代码级拆解系列

本系列文档拆解 **PageIndex** 项目中 LLM 的全部用法。所有代码片段均摘自本仓库真实源码，并标注了 `文件路径:行号`，可直接对照阅读。

> 前置知识：Python、基本 LLM/RAG 概念。建议先读过 `TUTORIAL_PART1_OVERVIEW.md`。

## 文档目录

| 文件 | 内容 |
|---|---|
| `01_LLM调用基础设施.md` | 所有 LLM 调用的收口层：`llm_completion` / `llm_acompletion` / `extract_json`，temperature=0、重试、截断续写机制 |
| `02_树索引构建全流程.md` | LLM 如何一步步把 PDF 变成树索引：`<physical_index_X>` 标签、三条处理路线、建树、递归细分、summary 生成 |
| `03_LLM角色拆解.md` | LLM 在项目中扮演的 6 大角色：分类器、抽取器、定位器、审计员、摘要生成器、检索推理引擎 |
| `04_防御与自愈机制.md` | 项目"不信任 LLM"的另一半代码：prompt injection 加固、输出机器校验、verify→fix→降级状态机 |

## 一张图总览

```
 PDF ──► get_page_tokens()                 [(page_text, token_len), ...]
              │
              ▼  每页包 <physical_index_X> 标签
        check_toc() ──► 三条路线分流
              │   ├── A. 有TOC带页码  process_toc_with_page_numbers
              │   ├── B. 有TOC无页码  process_toc_no_page_numbers
              │   └── C. 无TOC       process_no_toc
              ▼
        LLM 生成扁平 TOC ──► 机器校验 marker / 页码范围
              │
              ▼
        verify_toc() 抽查（LLM 审计 LLM）
              │   accuracy == 1.0  → 通过
              │   accuracy > 0.6   → fix_incorrect_toc_with_retries() 夹逼修复
              │   accuracy ≤ 0.6   → 降级换路线重跑（A→B→C→报错）
              ▼
        post_processing() + list_to_tree()      ← 纯代码建树，不用 LLM
              │
              ▼
        process_large_node_recursively()        ← 大节点递归重跑，长出深层
              │
              ▼
        write_node_id() + generate_summaries_for_structure()
              │
              ▼
        results/*_structure.json（树索引）
```

## 核心设计哲学

1. **LLM 只做代码做不了的事**（语义判断：识别标题、判层级、定页码），分组、对齐、建树、校验全部是确定性代码。
2. **每次 LLM 输出都过机器校验**，重要结论用第二次独立的 LLM 调用审计，不可信就修复或整条路线降级。
3. **索引期 LLM 是 parser**（temperature=0，输出 JSON），**检索期 LLM 是 reasoner**（带 tool 的 agent 做开放推理），两者可配不同模型（`pageindex/config.yaml` 的 `model` 与 `retrieve_model`）。
