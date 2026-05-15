## 阶段一：高保真文档解析与格式转换 (Document Parsing)

传统的 PDF 提取工具往往会破坏文档的复杂排版，特别是表格和图表。在此阶段，我们使用支持复杂视觉对象提取的工具，将文档安全地转换为 AI 最容易理解的 Markdown 格式。

**执行步骤：使用 LlamaParse 提取 Markdown**

```
# 1. 安装依赖
# pip install llama-parse llama-index

from llama_parse import LlamaParse
from llama_index.core import SimpleDirectoryReader

# 2. 初始化解析器，设定输出格式为 Markdown
parser = LlamaParse(
    api_key="YOUR_LLAMA_CLOUD_API_KEY",
    result_type="markdown",  # 强制输出 Markdown 格式以保留结构
    verbose=True
)

# 3. 提取文件
file_extractor = {".pdf": parser}
documents = SimpleDirectoryReader(
    input_files=["enterprise_report.pdf"],
    file_extractor=file_extractor
).load_data()

print("文档解析完成，已保留 Markdown 结构。")
```

## 阶段二：表格重构与降维处理 (Table Contextualization)

高度结构化的数据（如 Markdown 或 CSV 格式的表格）如果不带自然语言上下文，往往会导致 AI 产生理解偏差。我们需要将表格内容通过 Qwen Plus 转化为带有上下文的自然语言描述。

**执行步骤：调用 Qwen Plus 将表格转文本** _注：阿里云 DashScope 提供了与 OpenAI 完全兼容的 API 规范，因此我们直接使用 `openai` 官方 Python 库进行调用。_

```
# pip install openai
import os
from openai import OpenAI

# 初始化 Qwen 客户端 (配置阿里云 DashScope 兼容模式)
client = OpenAI(
    api_key="YOUR_DASHSCOPE_API_KEY",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

def contextualize_table_with_qwen(markdown_table_text):
    prompt = f"""
    请将以下 Markdown 表格转换为人类和 AI 易于阅读的详细自然语言描述，
    请确保保留行与列之间的逻辑关系和关键数据：
    {markdown_table_text}
    """
    response = client.chat.completions.create(
        model="qwen-plus", # 调用通义千问 Plus 模型
        messages=[{"role": "user", "content": prompt}],
        temperature=0.1
    )
    return response.choices.message.content

# 实际流水线中：将重写后的自然语言文本替换原有的扁平化表格
```

## 阶段三：智能切分与元数据注入 (Strategic Chunking & Metadata)

盲目使用“语义切分”往往会产生极小的文本碎片。业界基准测试表明：大小为 512 Tokens 且重叠 10%-15% 的递归字符切分 (Recursive Character Splitting) 能够提供最佳的准确率与召回率平衡。

**执行步骤：递归切分与添加“面包屑”元数据**

```
# pip install langchain langchain-text-splitters

from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1. 配置递归分块器 (推荐 512 token 大小，约 50 token 重叠)
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", ".", " ", ""] # 优先按段落切分，保护结构
)

chunks = text_splitter.split_text(documents.text)

# 2. 注入元数据前缀 (面包屑导航)
enriched_chunks = []
document_title = "2025年财务预算报告"

for i, chunk in enumerate(chunks):
    # 构建上下文前缀，帮助 Qwen 即使只看到小片段也能知晓全局定位
    preface = f"文档来源: {document_title} | 块索引: {i}\n---\n"
    enriched_chunks.append({
        "text": preface + chunk,
        "metadata": {
            "source": document_title,
            "chunk_index": i
        }
    })
```

## 阶段四：长文档输入与隐式上下文缓存 (Implicit Context Caching)

对于超长静态文档（如几十万字的知识库）的高频查询，Qwen API 依赖于底层的**隐式缓存 (Implicit Caching)**。系统会自动检测请求中重复的文本前缀（Prefix），并在后台复用之前计算好的 KV Cache，从而大幅提升响应速度并可能降低计费成本。

**执行步骤：优化 Prompt 结构以触发 Qwen 隐式缓存** _关键技巧：必须将庞大且静态的文档内容放在 `messages` 列表的最前面（通常是 System Prompt），而将动态的用户提问放在最后，这样才能保证前缀完全一致从而命中缓存。_

```
# 1. 加载我们在预处理阶段准备好的超长文档文本
with open("processed_knowledge_base.txt", "r", encoding="utf-8") as f:
    long_document_content = f.read()

# 2. 构建静态的系统提示词（此部分在多次请求中保持完全一致，以触发隐式缓存）
system_prompt = f"""你是一个资深的数据分析专家。
请仅基于以下提供的企业知识库文档内容来回答用户的问题。如果文档中没有相关信息，请直接回答“不知道”。

================文档开始================
{long_document_content}
================文档结束================
"""

def query_qwen_with_cache_optimization(user_question):
    # 发起请求：长静态前缀 + 短动态后缀
    response = client.chat.completions.create(
        model="qwen-plus",
        messages=[
            # 静态前缀放最前面
            {"role": "system", "content": system_prompt},
            # 动态的用户查询放最后
            {"role": "user", "content": user_question}
        ],
        temperature=0.1
    )
    return response.choices.message.content

# 3. 执行查询
# 第一个问题（后台会计算并缓存 System Prompt 中的长文档）
print(query_qwen_with_cache_optimization("请总结第三季度的核心财务风险是什么？"))

# 第二个问题（由于系统提示词前缀完全一致，极大概率命中底层的隐式缓存，响应速度会显著加快）
print(query_qwen_with_cache_optimization("报告中提到的 2025 年主要利润增长点在哪？"))
```

## 阶段五：使用 Agentic RAG 实现精准检索 (可选)

将静态的 RAG 升级为基于 Qwen 的 Agentic RAG（例如使用 LangGraph 工具）。Qwen Plus 具备极强的工具调用（Function Calling）能力，可以让它在回答前自主评估检索结果的质量。如果发现检索出来的文档块不相关，Qwen 会重写用户的查询条件进行二次检索，而不是基于错误信息产生幻觉。