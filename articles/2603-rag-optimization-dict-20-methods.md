# RAG优化字典：20种RAG优化方法全解析

> - **Published:** 2026-03-05
> - **Source:** [RAG优化字典：20种RAG优化方法全解析](https://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247694687&idx=1&sn=933681fb58bd38e6af584388def1516a&poc_token=HFQbuWmjQMqMBEjFU-klfDfWYEwI7j_jR842DSnv)
> - **Tags:** `RAG`

本文系统性地梳理了 `RAG`（`Retrieval-Augmented Generation` `检索增强生成`）系统从基础到高级的 20 种优化方法，涵盖分块策略、检索增强、查询优化、生成质量控制等多个维度。每种方法均附带核心代码实现（含简要注释），便于读者按需选型与落地实践。


## 01 语义分块

**核心思路：** 按句子级嵌入相似度切分，而非固定字符数。当相邻句子语义相似度低于阈值时在此处切块，从而保留语义完整性。

**断点计算方法：**
- 百分位法（Percentile）：相似度低于某百分位的位置作为断点
- 标准差法（Standard Deviation）：相似度低于「均值 − 标准差」的位置
- 四分位距法（IQR）：用四分位距检测异常低的相似度作为断点

```python
# ========== 方法1：语义分块 ==========
# 根据相邻句子的嵌入相似度找“语义断点”，在断点处切块，保证块内语义连贯

def compute_breakpoints(similarities, method="percentile", threshold=90):
    """
    根据相似度序列计算断点索引：相似度突降处 = 语义边界。
    method: percentile / standard_deviation / interquartile
    """
    breakpoints = []

    if method == "percentile":
        # 低于 (100-threshold) 分位数的位置视为断点
        threshold_value = np.percentile(similarities, 100 - threshold)
        for i, sim in enumerate(similarities):
            if sim < threshold_value:
                breakpoints.append(i)
    
    elif method == "standard_deviation":
        # 低于「均值 - 1 个标准差」的位置视为断点
        mean_sim = np.mean(similarities)
        std_sim = np.std(similarities)
        threshold_value = mean_sim - std_sim
        for i, sim in enumerate(similarities):
            if sim < threshold_value:
                breakpoints.append(i)
    
    elif method == "interquartile":
        # IQR 法：低于 Q1 - 1.5*IQR 的视为异常低，作为断点
        q1 = np.percentile(similarities, 25)
        q3 = np.percentile(similarities, 75)
        iqr = q3 - q1
        threshold_value = q1 - 1.5 * iqr
        for i, sim in enumerate(similarities):
            if sim < threshold_value:
                breakpoints.append(i)
    
    return breakpoints

def split_into_chunks(sentences, breakpoints):
    """按断点把句子序列切成多段，每段拼成一块文本"""
    chunks = []
    start = 0
    for bp in breakpoints:
        chunk = " ".join(sentences[start:bp + 1])
        chunks.append(chunk)
        start = bp + 1
    if start < len(sentences):
        chunks.append(" ".join(sentences[start:]))
    return chunks
```

## 02 切块大小评估

**核心思路：** 对多种切块大小（如 128、256、512 字符）分别做检索与生成，用忠实度（回复是否紧扣上下文）和相关性（是否答到点上）等指标对比，数据驱动选出最优切块参数。

**优势与局限：**
- **优势：** 避免拍脑袋定 chunk size，有评估依据；适合新语料或新场景的调参。
- **局限：** 需要标注或可自动评估的测试集，且评估有一定计算成本。
- **适用场景：** 上线前调参、不同领域/文档类型单独优化。

```python
# ========== 方法2：切块大小评估 ==========
# 对多种 chunk size 分别建索引、检索、生成，用忠实度/相关性选最优大小

 chunk_sizes = [128, 256, 512]

# 按每种大小切块并保存，便于后续统一评估

text_chunks_dict = {
    size: chunk_text(extracted_text, size, size // 5) 
    for size in chunk_sizes
}

# 对每种切块大小：建嵌入 → 检索 → 生成 → 算指标
for size in chunk_sizes:
    chunks = text_chunks_dict[size]
    embeddings = create_embeddings(chunks)
    for question in test_questions:
        results = semantic_search(question, chunks, embeddings)
        context = results[0][0]
        response = generate_response(question, context)
        # 忠实度：回复是否严格基于上下文，有无幻觉
        faithfulness = evaluate_faithfulness(response, context)
        # 相关性：回复是否切题、有用
        relevancy = evaluate_relevancy(response, question)
```

## 03 上下文增强检索

**核心思路：** 检索到最相关的块后，同时带上其前、后相邻块一起作为上下文，避免“孤岛片段”缺少前文/后文而导致理解不全。

**优势与局限：**
- **优势：** 实现简单，无需改索引；能明显改善需要前后文才能理解的内容（如列表、步骤、对比）。
- **局限：** 会引入与问题无关的相邻块，可能增加噪声和 token 消耗。
- **适用场景：** 分块较细、上下文连贯性重要的场景。

```python
# ========== 方法3：上下文增强检索 ==========
# 先按相似度找到 top-k 块，再对每个块扩展其前后 context_size 块，合并后返回

def context_enriched_search(query, text_chunks, embeddings, k=1, context_size=1):
    """
    检索：取相似度最高的 k 个块，每个块向左右各扩展 context_size 块并拼接。
    这样返回的是「中心块 + 前后文」的连续窗口，而非单块。
    """
    query_embedding = create_embeddings(query)
    similarities = [cosine_similarity(query_embedding, emb) for emb in embeddings]
    sorted_indices = np.argsort(similarities)[::-1]

    results = []
    for idx in sorted_indices[:k]:
        # 以当前块为中心，向左右各取 context_size 块
        start = max(0, idx - context_size)
        end = min(len(text_chunks), idx + context_size + 1)

        context_window = " ".join(text_chunks[start:end])
        results.append({
            "text": context_window,
            "similarity": similarities[idx],
            "center_chunk": idx
        })

    return results
```

## 04 上下文片段标题提取（CCH）

**核心思路：** 用 LLM 为每个文本块生成简短描述性标题，建索引时同时存「块文本嵌入」和「标题嵌入」。检索时用查询分别与文本、标题算相似度，再取平均（或加权）作为该块得分，从而多一个“主题维度”提升检索精度。

**优势与局限：**
- **优势：** 标题浓缩主题，对概括性、概念型查询更友好；可与现有向量检索无缝结合。
- **局限：** 预处理阶段要为每块调用 LLM 生成标题，成本和延迟增加。
- **适用场景：** 块较长、需要“主题级”匹配的知识库。

```python
# ========== 方法4：上下文片段标题提取（CCH） ==========
# 为每块生成标题并嵌入，检索时用 (文本相似度 + 标题相似度) 综合排序

def generate_chunk_header(chunk):
    """用 LLM 为单个文本块生成一句简洁标题，概括该块主题"""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": "为以下文本生成一个简洁的描述性标题。"},
            {"role": "user", "content": chunk}
        ]
    )
    return response.choices[0].message.content.strip()

def search_with_headers(query, chunks, text_embeddings, header_embeddings):
    """检索：查询与「块文本」和「块标题」分别算相似度，取平均作为该块得分"""
    query_embedding = create_embeddings(query)

    results = []
    for i in range(len(chunks)):
        sim_text = cosine_similarity(query_embedding, text_embeddings[i])
        sim_header = cosine_similarity(query_embedding, header_embeddings[i])
        avg_similarity = (sim_text + sim_header) / 2
        results.append({"text": chunks[i], "similarity": avg_similarity})
 
    results.sort(key=lambda x: x["similarity"], reverse=True)
    return results
```

## 05 文档增强RAG（问题生成）

**核心思路：** 为每个文本块用 LLM 生成若干“该块能回答的问题”，把「问题 + 对应块」一并写入向量库。用户查询时，既匹配块内容，也匹配这些预生成问题，从而拉近“问句”与“陈述句”的语义距离。

**优势与局限：**
- **优势：** 显著缓解 query（问句）与 document（陈述）之间的表述差异，提高召回。
- **局限：** 建库时每块要生成多条问题，成本和存储都增加；问题质量依赖模型与 prompt。
- **适用场景：** FAQ、知识问答、用户常以问句形式提问的场景。

```python
# ========== 方法5：文档增强RAG（问题生成） ==========
# 为每块生成“该块能回答的问题”，问题与块一起入向量库，检索时 query 可匹配问题或块

def generate_questions(text_chunk, num_questions=5):
    """根据块内容，用 LLM 生成若干该块能回答的问题，用于后续检索匹配"""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": f"基于以下文本，生成{num_questions}个该文本能够回答的问题。"},
            {"role": "user", "content": text_chunk}
        ]
    )
    questions = response.choices[0].message.content.strip().split("\n")
    return [q.strip() for q in questions if q.strip()]

# 建库：块本身 + 其对应问题都写入向量库，并记录问题→块的映射
vector_store = SimpleVectorStore()
for chunk in chunks:
    chunk_embedding = create_embeddings(chunk)
    vector_store.add_item(chunk, chunk_embedding, metadata={"type": "chunk"})
   
    questions = generate_questions(chunk)
    for question in questions:
        q_embedding = create_embeddings(question)
        vector_store.add_item(
            question, q_embedding,
            metadata={"type": "question", "source_chunk": chunk}
        )
```

## 06 查询转换

**核心思路：** 在检索前对用户原始查询做变换，使检索更准或更全。常用三种策略：
- 查询重写：把模糊、口语化查询改写成更具体、更贴近文档表述的形式
- 回退提示（Step-back）：生成一个更宽泛的“背景问题”，先检索背景再结合原问题
- 子查询分解：把多子问题的复杂查询拆成 2～4 个简单子查询，分别检索再合并

**优势与局限：**
- **优势：** 不改索引即可提升检索表现，尤其适合表述不规范或复杂的用户问句。
- **局限：** 多一次（或多次）LLM 调用，延迟与成本增加；重写/分解质量依赖 prompt。
- **适用场景：** 开放域问答、复杂多步问题、需要“先背景后细节”的检索。

```python

# ========== 方法6：查询转换 ==========
# 在检索前对 query 做改写 / 回退 / 分解，再用转换后的查询去检索

def rewrite_query(original_query):
    """把模糊或口语化查询重写成更具体、更利于检索的表述（不改变意图）"""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": """你是一个查询重写专家。
            将用户查询重写为更具体、更清晰的形式。
            只输出重写后的查询，不要添加任何解释。"""},
            {"role": "user", "content": original_query}
        ]
    )
    return response.choices[0].message.content.strip()

def generate_step_back_query(original_query):
    """生成一个更宽泛的“回退”问题，用于先检索背景知识，再结合原问题作答"""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": """你是一个查询生成专家。
            为给定查询生成一个更广泛的"回退"问题，以帮助获取相关背景知识。
            只输出回退查询，不要添加解释。"""},
            {"role": "user", "content": original_query}
        ]
    )
    return response.choices[0].message.content.strip()

def decompose_query(original_query):
    """将复杂问题拆成 2～4 个简单子问题，便于分别检索后综合答案"""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": """将以下复杂问题分解为2-4个更简单的子问题。
            每个子问题单独一行。只输出子问题，不要编号或解释。"""},
            {"role": "user", "content": original_query}
        ]
    )
    sub_queries = response.choices[0].message.content.strip().split("\n")
    return [q.strip() for q in sub_queries if q.strip()]
```

## 07 重排序（Reranking）

**核心思路：** 两阶段检索——先用向量（或 BM25）快速召回一批候选文档，再用更精细的模型对「查询-文档」对打分并重排，只保留最相关的若干条送入生成。常用实现：LLM 打 0–10 分，或专用 reranker 模型（如 Cross-Encoder）。

**优势与局限：**
- **优势：** 在不扩大首阶段召回量的前提下显著提升 top-k 精度，减少无关上下文注入。
- **局限：** 第二阶段要对每条候选调用模型，延迟和成本随候选数增加。
- **适用场景：** 对准确率要求高、候选数可控（如 top-20 内 rerank）。

```python

# ========== 方法7：重排序（Reranking） ==========
# 一阶段粗检索得到候选列表，二阶段用 LLM 对「查询-文档」打相关性分并重排

def rerank_with_llm(query, documents, model="gpt-3.5-turbo"):
    """对候选文档逐一用 LLM 打 0-10 相关性分，再按分数降序重排"""
    reranked = []
    for doc in documents:
        response = client.chat.completions.create(
            model=model,
            messages=[
                {"role": "system", "content": """你是一个相关性评估专家。
                对给定文档与查询的相关性进行0-10打分。
                只输出数字分数。"""},
                {"role": "user", "content": f"查询: {query}\n\n文档: {doc['text']}"}
            ],
            temperature=0
        )
        score = float(response.choices[0].message.content.strip())
        reranked.append({**doc, "rerank_score": score})
   
    reranked.sort(key=lambda x: x["rerank_score"], reverse=True)
    return reranked
```

## 08 相关段落提取（RSE）

**核心思路：** 不为单块打分后简单取 top-k，而是为连续块序列算“价值和”：每块价值 = 相似度 − 惩罚项，再用最大子数组（Kadane）思想找价值和最大的连续段落。这样返回的是一整段连贯文本，而不是零散块。

**优势与局限：**
- **优势：** 上下文连贯、边界更自然，适合需要“一整段话”才能答好的问题。
- **局限：** 依赖块顺序与相似度曲线质量；实现和调参（惩罚、最小长度等）略复杂。
- **适用场景：** 长文档、叙述性强，且希望按“段落”而非“单块”呈现。

```python
# ========== 方法8：相关段落提取（RSE） ==========
# 为每块算「相似度 - 惩罚」作为价值，用最大子数组找“价值和最大”的连续段落

def calculate_chunk_values(similarities, irrelevant_chunk_penalty=0.2):
    """把相似度转为价值：相关块为正，不相关块减去惩罚后为负，便于后续求连续最大和"""
    values = []
    for sim in similarities:
        value = sim - irrelevant_chunk_penalty
        values.append(value)
    return values

def find_best_segments(values, max_segments=3, min_segment_length=1):
    """Kadane 思想：在 value 序列上找最多 max_segments 段连续区间，使每段和最大"""
    segments = []
    n = len(values)

    for _ in range(max_segments):
        best_sum = float('-inf')
        best_start = best_end = 0
        current_sum = 0
        current_start = 0
   
        for i in range(n):
            current_sum += values[i]
            if current_sum > best_sum and (i - current_start + 1) >= min_segment_length:
                best_sum = current_sum
                best_start = current_start
                best_end = i
            if current_sum < 0:
                current_sum = 0
                current_start = i + 1

        if best_sum > 0:
            segments.append((best_start, best_end, best_sum))
            for i in range(best_start, best_end + 1):
                values[i] = 0
    
    return segments

def reconstruct_segments(chunks, segments):
    """根据 (start, end) 区间把对应块拼成完整段落文本"""
    reconstructed = []
    for start, end, score in segments:
        segment_text = " ".join(chunks[start:end + 1])
        reconstructed.append({
            "text": segment_text,
            "start": start,
            "end": end,
            "score": score
        })
    return reconstructed
```

## 09 上下文压缩

**核心思路：** 检索得到多块文本后，不直接全部塞给 LLM，而是先对每块做“压缩”：只保留与当前查询强相关的内容，去掉无关句子或整段，从而减少噪声、节省 token、提高生成质量。

**三种压缩策略：**
选择性（Selective）：从原文中筛选与查询相关的句子，保持原句、原序
摘要（Summary）：针对查询写一段只含相关信息的摘要
抽取（Extraction）：按查询抽取关键事实或结构化信息

**优势与局限：**
- **优势：** 显著减少无关上下文，降低幻觉与 token 消耗；可与任意检索方式组合。
- **局限：** 每块都要过一次 LLM，延迟与成本增加；过度压缩可能丢细节。
- **适用场景：** 检索结果偏长、噪声多，或上下文窗口紧张时。

```python
# ========== 方法9：上下文压缩 ==========
# 对每个检索到的块，用 LLM 按 query 只保留相关部分（选择性保留/摘要/抽取）

def compress_chunk(chunk, query, compression_type="selective", model="gpt-3.5-turbo"):
    """根据 compression_type：选择性保留相关句 / 写相关摘要 / 抽取关键信息"""
    if compression_type == "selective":
        system_prompt = """You are an expert at information filtering. 
        Your task is to analyze a document chunk and extract ONLY the sentences 
        or paragraphs that are directly relevant to the user's query. 
        Remove all irrelevant content.
 
        Your output should:
        1. ONLY include text that helps answer the query
        2. Preserve the exact wording of relevant sentences (do not paraphrase)
        3. Maintain the original order of the text
        4. Include ALL relevant content, even if it seems redundant
        5. EXCLUDE any text that isn't relevant to the query"""
    elif compression_type == "summary":
        system_prompt = """You are an expert at summarization. 
        Create a concise summary of the provided chunk that focuses ONLY on 
        information relevant to the user's query."""
    else:  # extraction
        system_prompt = """You are an expert at information extraction.
        Extract key facts and information relevant to the query."""

    response = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"Query: {query}\n\nDocument chunk:\n{chunk}"}
        ],
        temperature=0
    )
    return response.choices[0].message.content
```

## 10 上下文压缩

**核心思路：** 把用户对回复的评分/反馈纳入闭环：记录「查询–回复–评分」，用 LLM 判断历史反馈与当前查询/文档是否相关，据此动态调整文档相关性得分，并把高质量问答对写入知识库，使系统越用越准。

**核心组件：** 反馈收集 → 相关性调整（按历史反馈修正检索分数）→ 知识积累（优质 Q&A 入向量库）。

**优势与局限：**
- **优势：** 可持续利用真实用户信号，适合客服、知识库等有稳定反馈的场景。
- **局限：** 依赖高质量、足量反馈；反馈噪声或冷启动会影响调整效果。
- **适用场景：** 有评分/点赞/纠错等用户反馈的在线系统。

```python

# ========== 方法10：反馈回路RAG（摘自 11_反馈回路机制的rag.ipynb）==========

def assess_feedback_relevance(query, doc_text, feedback):
    """
    使用LLM评估过去的反馈是否与当前查询和文档相关。
    
    参数:
        query (str): 当前用户需要信息检索的查询
        doc_text (str): 正在评估的文档文本内容
        feedback (Dict): 包含'query'和'response'键的过去反馈数据
    
    返回:
        bool: 如果反馈被认为与当前查询/文档相关，则返回True，否则返回False
    """
    system_prompt = """You are an AI system that determines if a past feedback is relevant to a current query and document.
    Answer with ONLY 'yes' or 'no'. Your job is strictly to determine relevance, not to provide explanations."""
    
    user_prompt = f"""
    Current query: {query}
    Past query that received feedback: {feedback['query']}
    Document content: {doc_text[:500]}... [truncated]
    Past response that received feedback: {feedback['response'][:500]}... [truncated]
    
    Is this past feedback relevant to the current query and document? (yes/no)
    """
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0
    )

    answer = response.choices[0].message.content.strip().lower()
    return 'yes' in answer

def adjust_relevance_scores(query, results, feedback_data):
    """
    根据历史反馈调整文档相关性得分，以提高检索质量。

    参数:
        query (str): 当前用户的查询
        results (List[Dict]): 带有原始相似度得分的检索到的文档
        feedback_data (List[Dict]): 包含用户评分的历史反馈数据

    返回:
        List[Dict]: 调整后相关性得分的结果，按新的得分排序
    """
    if not feedback_data:
        return results

    print("Adjusting relevance scores based on feedback history...")

    for i, result in enumerate(results):
        document_text = result["text"]
        relevant_feedback = []
        for feedback in feedback_data:
            is_relevant = assess_feedback_relevance(query, document_text, feedback)
            if is_relevant:
                relevant_feedback.append(feedback)
        if relevant_feedback:
            avg_relevance = sum(f['relevance'] for f in relevant_feedback) / len(relevant_feedback)
            modifier = 0.5 + (avg_relevance / 5.0)
   
            original_score = result["similarity"]
            adjusted_score = original_score * modifier
    
            result["original_similarity"] = original_score
            result["similarity"] = adjusted_score
            result["relevance_score"] = adjusted_score
            result["feedback_applied"] = True
            result["feedback_count"] = len(relevant_feedback)
         
            print(f"  Document {i+1}: Adjusted score from {original_score:.4f} to {adjusted_score:.4f} based on {len(relevant_feedback)} feedback(s)")
    results.sort(key=lambda x: x["similarity"], reverse=True)

    return results

def generate_response(query, context, model="gpt-3.5-turbo"):
    """
    根据查询和上下文生成回复。
    参数:
        query (str): 用户查询
        context (str): 来自检索到的文档的上下文文本
        model (str): 要使用的LLM模型
    返回:
        str: 生成的回复
    """
    system_prompt = """You are a helpful AI assistant. Answer the user's question based only on the provided context. If you cannot find the answer in the context, state that you don't have enough information."""
    user_prompt = f"""
        Context:
        {context}
       
        Question: {query}
       
        Please provide a comprehensive answer based only on the context above.
    """
    response = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0
    )
    return response.choices[0].message.content

def rag_with_feedback_loop(query, vector_store, feedback_data, k=5, model="gpt-3.5-turbo"):
    """
    完整的RAG管道，结合反馈循环。
    参数:
        query (str): 用户查询
        vector_store (SimpleVectorStore): 包含文档片段的向量存储
        feedback_data (List[Dict]): 历史反馈数据
        k (int): 检索的文档数量
        model (str): 用于回复生成的语言模型
    返回:
        Dict: 结果字典，包括查询、检索到的文档和回复
    """
    print(f"\n=== Processing query with feedback-enhanced RAG ===")
    print(f"Query: {query}")
   
    query_embedding = create_embeddings(query)
    results = vector_store.similarity_search(query_embedding, k=k)
    adjusted_results = adjust_relevance_scores(query, results, feedback_data)
    retrieved_texts = [result["text"] for result in adjusted_results]
    context = "\n\n---\n\n".join(retrieved_texts)
    print("Generating response...")
    response = generate_response(query, context, model)
    result = {
        "query": query,
        "retrieved_documents": adjusted_results,
        "response": response
    }

    print("\n=== Response ===")
    print(response)

    return result
```

## 11 自适应检索

**核心思路：** 先对用户查询做类型分类（事实型 分析型 观点型 / 情境型），再根据类型选用不同检索策略（如事实型偏精确匹配、分析型多召回再压缩等），实现“因问施策”。

**优势与局限：**
- **优势：** 同一套系统可兼顾多类问题，检索更有的放矢。
- **局限：** 分类与各策略设计依赖领域知识；分类错误会传导到检索。
- **适用场景：** 混合类型问答、客服/咨询等。

```python

# ========== 方法11：自适应检索 ==========
# 先对 query 做类型分类，再按类型选择不同检索策略（事实/分析/观点/情境）

def classify_query(query, model="gpt-3.5-turbo"):
    """用 LLM 将查询归为四类之一：Factual / Analytical / Opinion / Contextual"""
    system_prompt = """Classify the given query into exactly one of these categories:
        - Factual: Queries seeking specific, verifiable information.
        - Analytical: Queries requiring comprehensive analysis or explanation.
        - Opinion: Queries about subjective matters or seeking diverse viewpoints.
        - Contextual: Queries that depend on user-specific context.
        Return ONLY the category name."""
 
    response = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"Classify this query: {query}"}
        ],
        temperature=0
    )
    category = response.choices[0].message.content.strip()
    valid_categories = ["Factual", "Analytical", "Opinion", "Contextual"]
    for valid in valid_categories:
        if valid in category:
            return valid
    return "Factual"

def adaptive_retrieval(query, vector_store, k=4, user_context=None):
    """按分类结果调用对应检索策略：事实型/分析型/观点型/情境型各有实现"""
    query_type = classify_query(query)
    
    if query_type == "Factual":
        results = factual_retrieval_strategy(query, vector_store, k)
    elif query_type == "Analytical":
        results = analytical_retrieval_strategy(query, vector_store, k)
    elif query_type == "Opinion":
        results = opinion_retrieval_strategy(query, vector_store, k)
    elif query_type == "Contextual":
        results = contextual_retrieval_strategy(query, vector_store, k, user_context)
    else:
        results = factual_retrieval_strategy(query, vector_store, k)
    
    return results
```

## 12 Self-RAG（自适应RAG）

**核心思路：** 在「检索 → 生成」全流程中插入多个反思与决策点：是否要检索、检索到的文档是否相关、生成的回复是否被上下文支撑、回复是否实用等，从而在运行时自适应地决定是否检索、是否重试、是否标注“无依据”。

**六个关键组件：**
1. 检索决策：判断当前查询是否需要检索
2. 文档检索：需要时做向量检索
3. 相关性评估：评估每个检索文档与查询是否相关
4. 回复生成：基于相关上下文生成回复
5. 支持评估：评估回复是否被上下文充分支持
6. 效用评估：对回复的整体实用性打 1–5 分

**优势与局限：**
- **优势：** 可减少无谓检索与幻觉，输出更可控、可解释。
- **局限：** 多次 LLM 调用，延迟与成本高；依赖各环节 prompt 设计。
- **适用场景：** 对准确性与可追溯性要求高的场景。

```python
# ========== 方法12：Self-RAG ==========
# 流程：是否需要检索 → 检索 → 评估文档相关性 → 生成回复 → 评估支持度与效用 → 选最佳回复
import re  # rate_utility 中解析评分用
def determine_if_retrieval_needed(query):
    """判断给定查询是否需要检索。事实/具体信息类答 Yes，常识或主观类答 No。"""
    system_prompt = """You are an AI assistant that determines if retrieval is necessary to answer a query.
    For factual questions, specific information requests, or questions about events, people, or concepts, answer "Yes".
    For opinions, hypothetical scenarios, or simple queries with common knowledge, answer "No".
    Answer with ONLY "Yes" or "No"."""
    user_prompt = f"Query: {query}\n\nIs retrieval necessary to answer this query accurately?"
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0
    )
    answer = response.choices[0].message.content.strip().lower()
    return "yes" in answer
def evaluate_relevance(query, context):
    """评估上下文与查询的相关性。返回 'relevant' 或 'irrelevant'。"""
    system_prompt = """You are an AI assistant that determines if a document is relevant to a query.
    Consider whether the document contains information that would be helpful in answering the query.
    Answer with ONLY "Relevant" or "Irrelevant"."""
    max_context_length = 2000
    if len(context) > max_context_length:
        context = context[:max_context_length] + "... [truncated]"
    user_prompt = f"""Query: {query}
    Document content:
    {context}
    Is this document relevant to the query? Answer with ONLY "Relevant" or "Irrelevant"."""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0
    )
    return response.choices[0].message.content.strip().lower()
def assess_support(response, context):
    """判断生成的回复是否由上下文充分支持。返回 'fully supported' / 'partially supported' / 'no support'。"""
    system_prompt = """You are an AI assistant that determines if a response is supported by the given context.
    Evaluate if the facts, claims, and information in the response are backed by the context.
    Answer with ONLY one of these three options:
    - "Fully supported": All information in the response is directly supported by the context.
    - "Partially supported": Some information in the response is supported by the context, but some is not.
    - "No support": The response contains significant information not found in or contradicting the context."""
    max_context_length = 2000
    if len(context) > max_context_length:
        context = context[:max_context_length] + "... [truncated]"
    user_prompt = f"""Context:
    {context}
    Response:
    {response}
    How well is this response supported by the context? Answer with ONLY "Fully supported", "Partially supported", or "No support"."""
    api_response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0
    )
    return api_response.choices[0].message.content.strip().lower()
def rate_utility(query, response):
    """对回复的实用性打 1–5 分。"""
    system_prompt = """You are an AI assistant that rates the utility of a response to a query.
    Consider how well the response answers the query, its completeness, correctness, and helpfulness.
    Rate the utility on a scale from 1 to 5. Answer with ONLY a single number from 1 to 5."""
    user_prompt = f"""Query: {query}\nResponse:\n{response}\n\nRate the utility of this response on a scale from 1 to 5:"""
    resp = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0
    )
    rating = resp.choices[0].message.content.strip()
    rating_match = re.search(r'[1-5]', rating)
    return int(rating_match.group()) if rating_match else 3
def generate_response(query, context=None):
    """根据查询和可选上下文生成回复。无 context 时仅凭模型自身知识回答。"""
    system_prompt = """You are a helpful AI assistant. Provide a clear, accurate, and informative response to the query."""
    if context:
        user_prompt = f"""Context:\n{context}\n\nQuery: {query}\n\nPlease answer the query based on the provided context."""
    else:
        user_prompt = f"""Query: {query}\n\nPlease answer the query to the best of your ability."""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0.2
    )
    return response.choices[0].message.content.strip()
def self_rag(query, vector_store, top_k=3):
    # 第一步：确定是否需要检索
    print("Step 1: Determining if retrieval is necessary...")
    retrieval_needed = determine_if_retrieval_needed(query)

    # 初始化跟踪Self-RAG过程的指标
    metrics = {
        "retrieval_needed": retrieval_needed,
        "documents_retrieved": 0,
        "relevant_documents": 0,
        "response_support_ratings": [],
        "utility_ratings": []
    }

    best_response = None
    best_score = -1

    if retrieval_needed:
        # 第二步：检索相关文档
        print("\nStep 2: Retrieving relevant documents...")
        query_embedding = create_embeddings(query)
        results = vector_store.similarity_search(query_embedding, k=top_k)
        metrics["documents_retrieved"] = len(results)
        print(f"Retrieved {len(results)} documents")

        # 第三步：评估每个文档的相关性
        print("\nStep 3: Evaluating document relevance...")
        relevant_contexts = []

        for i, result in enumerate(results):
            context = result["text"]
            relevance = evaluate_relevance(query, context)
            print(f"Document {i+1} relevance: {relevance}")

            if relevance == "relevant":
                relevant_contexts.append(context)

        metrics["relevant_documents"] = len(relevant_contexts)
        print(f"Found {len(relevant_contexts)} relevant documents")

        if relevant_contexts:
            # 第四步：处理每个相关上下文
            print("\nStep 4: Processing relevant contexts...")
            for i, context in enumerate(relevant_contexts):
                print(f"\nProcessing context {i+1}/{len(relevant_contexts)}...")

                # 基于上下文生成回复
                print("Generating response...")
                response = generate_response(query, context)

                # 评估回复在多大程度上由上下文支持
                print("Assessing support...")
                support_rating = assess_support(response, context)
                print(f"Support rating: {support_rating}")
                metrics["response_support_ratings"].append(support_rating)

                # 对回复的实用性进行评分
                print("Rating utility...")
                utility_rating = rate_utility(query, response)
                print(f"Utility rating: {utility_rating}/5")
                metrics["utility_ratings"].append(utility_rating)

                # 计算总体分数（支持度和实用性越高，分数越高）
                support_score = {
                    "fully supported": 3, 
                    "partially supported": 1, 
                    "no support": 0
                }.get(support_rating, 0)

                overall_score = support_score * 5 + utility_rating
                print(f"Overall score: {overall_score}")

                # 跟踪最佳回复
                if overall_score > best_score:
                    best_response = response
                    best_score = overall_score
                    print("New best response found!")

        # 如果没有找到相关上下文或所有回复得分都很低
        if not relevant_contexts or best_score <= 0:
            print("\nNo suitable context found or poor responses, generating without retrieval...")
            best_response = generate_response(query)
    else:
        # 不需要检索，直接生成回复
        print("\nNo retrieval needed, generating response directly...")
        best_response = generate_response(query)

    # 最终指标
    metrics["best_score"] = best_score
    metrics["used_retrieval"] = retrieval_needed and best_score > 0

    print("\n=== Self-RAG Completed ===")

    return {
        "query": query,
        "response": best_response,
        "metrics": metrics
    }
```

## 13 命题分块

**核心思路：** 不按字符或句子切块，而是把文本拆成原子化、自包含的“命题”（一句一事实，可独立理解）。用命题做检索单元，实现更细粒度的 query–content 匹配。

**优势与局限：**
- **优势：** 检索粒度细，适合事实型、实体型问题；命题可复用、可组合。
- **局限：** 建库需对每块做命题抽取，成本高；命题过多时索引与检索开销增大。
- **适用场景：** 知识密集型、强调“单事实”准确性的 QA。

```python
# ========== 方法13：命题分块 ==========
# 把文本块拆成「单事实、自包含」的命题句，以命题为单元建索引与检索
def generate_propositions(chunk):
    """用 LLM 将一块文本拆成多条原子命题：每句一个事实、可独立理解、含主谓结构"""
    system_prompt = """Break down the following text into simple, self-contained propositions. 
    Each proposition should:
    1. Express a Single Fact
    2. Be Understandable Without Context
    3. Use Full Names, Not Pronouns
    4. Include Relevant Dates/Qualifiers
    5. Contain One Subject-Predicate Relationship
    Output ONLY the list of propositions."""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"Text to convert:\n\n{chunk['text']}"}
        ],
        temperature=0
    )
    raw_propositions = response.choices[0].message.content.strip().split('\n')
    # 去掉编号、列表符，过滤过短无效句
    clean_propositions = []
    for prop in raw_propositions:
        cleaned = re.sub(r'^\s*(\d+\.|\-|\*)\s*', '', prop).strip()
        if cleaned and len(cleaned) > 10:
            clean_propositions.append(cleaned)
    return clean_propositions
def process_document_into_propositions(pdf_path, chunk_size=800, chunk_overlap=100, 
                                      quality_thresholds=None):
    # 如果未提供质量阈值，则设置默认值
    if quality_thresholds is None:
        quality_thresholds = {
            "accuracy": 7,
            "clarity": 7,
            "completeness": 7,
            "conciseness": 7
        }

    # 从PDF文件中提取文本
    text = extract_text_from_pdf(pdf_path)

    # 从提取的文本创建块
    chunks = chunk_text(text, chunk_size, chunk_overlap)

    # 初始化一个列表来存储所有命题
    all_propositions = []

    print("Generating propositions from chunks...")
    for i, chunk in enumerate(chunks):
        print(f"Processing chunk {i+1}/{len(chunks)}...")

        # 为当前块生成命题
        chunk_propositions = generate_propositions(chunk)
        print(f"Generated {len(chunk_propositions)} propositions")

        # 处理每个生成的命题
        for prop in chunk_propositions:
            proposition_data = {
                "text": prop,
                "source_chunk_id": chunk["chunk_id"],
                "source_text": chunk["text"]
            }
            all_propositions.append(proposition_data)

    # 评估生成的命题的质量
    print("\nEvaluating proposition quality...")
    quality_propositions = []

    for i, prop in enumerate(all_propositions):
        if i % 10 == 0:  # 每10个命题更新一次状态
            print(f"Evaluating proposition {i+1}/{len(all_propositions)}...")

        # 评估当前命题的质量
        scores = evaluate_proposition(prop["text"], prop["source_text"])
        prop["quality_scores"] = scores

        # 检查命题是否通过质量阈值
        passes_quality = True
        for metric, threshold in quality_thresholds.items():
            if scores.get(metric, 0) < threshold:
                passes_quality = False
                break

        if passes_quality:
            quality_propositions.append(prop)
        else:
            print(f"Proposition failed quality check: {prop['text'][:50]}...")

    print(f"\nRetained {len(quality_propositions)}/{len(all_propositions)} propositions after quality filtering")

    return chunks, quality_propositions
```

## 14 多模态RAG

**核心思路：** 从 PDF 等文档中同时提取文本与图像，用视觉模型为每张图生成文本描述（caption），将「正文 + 图像描述」统一嵌入并写入同一向量库，实现“以文搜文+以文搜图”的联合检索。

```python
# ========== 方法14：多模态RAG ==========
# PDF 中同时抽文本与图像，图像用 VLM 生成 caption，文本+caption 一起入向量库统一检索

def extract_content_from_pdf(pdf_path, output_dir=None):
    """按页从 PDF 抽取文本块和图像，图像保存到 output_dir 并记录路径与页码"""
    text_data = []
    image_paths = []
    with fitz.open(pdf_path) as pdf_file:
        for page_number in range(len(pdf_file)):
            page = pdf_file[page_number]
            text = page.get_text().strip()
            if text:
                text_data.append({"content": text, "metadata": {"page": page_number + 1, "type": "text"}})
            for img_index, img in enumerate(page.get_images(full=True)):
                xref = img[0]
                base_image = pdf_file.extract_image(xref)
                if base_image:
                    img_path = os.path.join(output_dir, f"page_{page_number+1}_img_{img_index+1}.{base_image['ext']}")
                    with open(img_path, "wb") as f:
                        f.write(base_image["image"])
                    image_paths.append({"path": img_path, "metadata": {"page": page_number + 1, "type": "image"}})
    return text_data, image_paths

def generate_image_caption(image_path):
    """用多模态模型（如 LLaVA）为图像生成文字描述，便于与文本一起嵌入检索"""
    base64_image = encode_image(image_path)
    response = client.chat.completions.create(
        model="llava-hf/llava-1.5-7b-hf",
        messages=[
            {"role": "system", "content": "Describe images from academic papers in detail."},
            {"role": "user", "content": [
                {"type": "text", "text": "Describe this image in detail:"},
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{base64_image}"}}
            ]}
        ]
    )
    return response.choices[0].message.content
```

## 15 融合检索

**核心思路：** 向量检索（语义相似）与 BM25（词匹配）同时做，对两路分数分别 Min-Max 归一化 后按 `alpha` 加权融合：`final\_score = alpha \* norm\_vector + (1-alpha) \* norm\_bm25`，再按融合分取 top-k。语义与关键词互补，召回更稳。

```python

# ========== 方法15：融合检索 ==========
# 向量检索 + BM25 双路打分，归一化后加权融合，再取 top-k

def bm25_search(bm25, chunks, query, k=5):
    """BM25 关键词检索：对 query 分词后算各 chunk 的 BM25 分，按分排序取 top-k"""
    query_tokens = query.split()
    scores = bm25.get_scores(query_tokens)

    results = []
    for i, score in enumerate(scores):
        metadata = chunks[i].get("metadata", {}).copy()
        metadata["index"] = i
        results.append({"text": chunks[i]["text"], "metadata": metadata, "bm25_score": float(score)})

    results.sort(key=lambda x: x["bm25_score"], reverse=True)
    return results[:k]

def fusion_retrieval(query, chunks, vector_store, bm25_index, k=5, alpha=0.5):
    """融合：向量与 BM25 各算一版分数 → Min-Max 归一化 → 加权合并 → 按最终分取 top-k"""
    epsilon = 1e-8

    query_embedding = create_embeddings(query)
    vector_results = vector_store.similarity_search_with_scores(query_embedding, k=len(chunks))
    bm25_results = bm25_search(bm25_index, chunks, query, k=len(chunks))

    vector_scores_dict = {r["metadata"]["index"]: r["similarity"] for r in vector_results}
    bm25_scores_dict = {r["metadata"]["index"]: r["bm25_score"] for r in bm25_results}

    vector_scores = np.array([vector_scores_dict.get(i, 0.0) for i in range(len(chunks))])
    bm25_scores = np.array([bm25_scores_dict.get(i, 0.0) for i in range(len(chunks))])

    v_min, v_max = vector_scores.min(), vector_scores.max()
    b_min, b_max = bm25_scores.min(), bm25_scores.max()
    norm_vector = (vector_scores - v_min) / (v_max - v_min + epsilon)
    norm_bm25 = (bm25_scores - b_min) / (b_max - b_min + epsilon)

    final_scores = alpha * norm_vector + (1 - alpha) * norm_bm25

    top_indices = np.argsort(final_scores)[::-1][:k]
    return [{"text": chunks[i]["text"], "score": final_scores[i]} for i in top_indices]
```

## 16 图RAG（Graph RAG）

**核心思路：** 从每个文本块中抽取概念/实体，以块为节点、以共享概念 + 语义相似度为边权建知识图谱。检索时先命中部分节点，再通过图上的边扩展关联节点，从而把跨多块的相关信息一并拉取。

**优势与局限：**
- **优势：** 能利用“概念共现”“实体关系”做关联扩展，适合概念密集、关系复杂的领域。
- **局限：** 建图与概念抽取成本高；图规模大时遍历与索引设计更复杂。
- **适用场景：** 知识图谱、领域百科、多实体关系推理。

```python
# ========== 方法16：图RAG（Graph RAG） ==========
# 块→节点，块内概念→属性；共享概念+语义相似度→边；检索时可沿边扩展关联块

def build_knowledge_graph(chunks):
    """为每个 chunk 抽概念、建节点；根据共享概念与嵌入相似度建边，得到知识图谱"""
    graph = nx.Graph()
    texts = [chunk["text"] for chunk in chunks]
    embeddings = create_embeddings(texts)

    for i, chunk in enumerate(chunks):
        concepts = extract_concepts(chunk["text"])  # LLM 抽取概念/实体
        graph.add_node(i, text=chunk["text"], concepts=concepts, embedding=embeddings[i])

    for i in range(len(chunks)):
        node_concepts = set(graph.nodes[i]["concepts"])
        for j in range(i + 1, len(chunks)):
            other_concepts = set(graph.nodes[j]["concepts"])
            shared_concepts = node_concepts.intersection(other_concepts)

            if shared_concepts:
                similarity = np.dot(embeddings[i], embeddings[j]) / (
                    np.linalg.norm(embeddings[i]) * np.linalg.norm(embeddings[j]))
                concept_score = len(shared_concepts) / min(len(node_concepts), len(other_concepts))
                edge_weight = 0.7 * similarity + 0.3 * concept_score

                if edge_weight > 0.6:
                    graph.add_edge(i, j, weight=edge_weight, shared_concepts=list(shared_concepts))

    return graph, embeddings
```

## 17 分层检索RAG

**核心思路：** 建两层索引——摘要层（如每页/每章一个摘要）和详细层（页内细粒度块）。检索时先查摘要层锁定相关页/章，再只在对应页内查详细块，从而在保证相关性的前提下缩小搜索范围、提升效率。

**优势与局限：**
- **优势：** 适合超长文档、十亿级规模；先筛范围再精查，延迟与算力更可控。
- **局限：** 摘要质量与粒度影响第一层召回；若相关分布在多页，需合理设置 k\_summaries。
- **适用场景：** 长篇报告、书籍、法规、多章节文档。

```python
# ========== 方法17：分层检索RAG ==========
# 第一层：页/章摘要向量库；第二层：页内详细块向量库。先搜摘要定范围，再在范围内搜块

def process_document_hierarchically(pdf_path, chunk_size=1000, chunk_overlap=200):
    """建两层索引：每页摘要入 summary_store，每页内细块入 detailed_store"""
    pages = extract_text_from_pdf(pdf_path)

    summaries = []
    for i, page in enumerate(pages):
        summary_text = generate_page_summary(page["text"])
        summaries.append({"text": summary_text, "metadata": {**page["metadata"], "is_summary": True}})

    detailed_chunks = []
    for page in pages:
        page_chunks = chunk_text(page["text"], page["metadata"], chunk_size, chunk_overlap)
        detailed_chunks.extend(page_chunks)

    summary_store = SimpleVectorStore()
    detailed_store = SimpleVectorStore()
    # 分别为 summary 与 detailed_chunks 建嵌入并写入对应 store
    return summary_store, detailed_store

def retrieve_hierarchically(query, summary_store, detailed_store, k_summaries=3, k_chunks=5):
    """先按 query 在摘要层检索出 k_summaries 个相关页，再仅在这些页的详细块里检索"""
    query_embedding = create_embeddings(query)

    summary_results = summary_store.similarity_search(query_embedding, k=k_summaries)
    relevant_pages = [r["metadata"]["page"] for r in summary_results]

    def page_filter(metadata):
        return metadata["page"] in relevant_pages

    detailed_results = detailed_store.similarity_search(
        query_embedding, k=k_chunks * len(relevant_pages), filter_func=page_filter
    )
    return detailed_results
```

## 18 假设文档嵌入（HyDE）

**核心思路：** 用户查询往往很短，与长文档在嵌入空间里距离较远。HyDE 的做法是：先用 LLM 根据问题生成一段“假设的”答案文档（风格、长度接近真实文档），再用这段假设文档的嵌入（而非原始 query 的嵌入）去检索真实文档，从而拉近 query 与 document 的表示距离。

**优势与局限：**
- **优势：** 能把短 / 抽象问句补成伪文档，大幅提升检索相关性；不用重建索引，只在检索侧增强 query。
- **局限：** 多一次 LLM 生成与嵌入调用；假设文档若偏离真实文档风格，可能引入噪声。
- **适用场景：** 短查询、知识问答、知识库规整、关键词匹配差的 RAG 场景。

```python
# ========== 方法18：假设文档嵌入（HyDE） ==========
# 用 query 生成一段“假设答案”文档，用该文档的嵌入去检索，而非用 query 嵌入

def generate_hypothetical_document(query, desired_length=1000):
    """根据问题生成一段“像标准答案”的假设文档，长度与风格接近真实文档"""
    system_prompt = f"""You are an expert document creator. 
    Given a question, generate a detailed document that would directly answer this question.
    The document should be approximately {desired_length} characters long.
    Write as if from an authoritative source. Include specific details and facts.
    Do not mention that this is a hypothetical document."""

    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"Question: {query}\n\nGenerate a document:"}
        ],
        temperature=0.1
    )
    return response.choices[0].message.content

def hyde_rag(query, vector_store, k=5):
    """HyDE 流程：生成假设文档 → 用假设文档嵌入检索 → 用真实检索结果生成最终答案"""
    hypothetical_doc = generate_hypothetical_document(query)

    hypothetical_embedding = create_embeddings([hypothetical_doc])[0]

    retrieved_chunks = vector_store.similarity_search(hypothetical_embedding, k=k)

    response = generate_response(query, retrieved_chunks)
    return {"query": query, "hypothetical_document": hypothetical_doc,
            "retrieved_chunks": retrieved_chunks, "response": response}
```

## 19 动态纠正RAG（CRAG）

**核心思路：** 检索完成后先评估这批文档与 query 的相关性（如 0–1 分），再按分数区间做分支：高相关则只用本地文档；低相关则放弃本地、改用网络搜索；中等则本地 + 网络一起用。从而在“知识不足”时自动纠偏，避免硬用无关文档生成。

**优势与局限：**
- **优势：** 降低“检索到无关文档仍强行生成”的幻觉与错误；适合知识边界不固定、需时效信息的场景。
- **局限：** 需要可靠的相关性评估（LLM 或 reranker）和网络搜索接口；阈值需调参。
- **适用场景：** 开放域问答、时效性强的场景、本地库覆盖不全时。

```python
# ========== 方法19：动态纠正RAG（CRAG） ==========
# 检索 → 评估相关性 → 高/中/低分分别采用「仅本地 / 本地+网络 / 仅网络」

def evaluate_document_relevance(query, document):
    """用 LLM 对「query-文档」打 0–1 相关性分，用于 CRAG 分支决策"""
    system_prompt = """Rate how relevant the given document is to the query 
    on a scale from 0 to 1. Provide ONLY the score as a float."""

    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"Query: {query}\n\nDocument: {document}"}
        ],
        temperature=0, max_tokens=5
    )
    score_text = response.choices[0].message.content.strip()
    score_match = re.search(r'(\d+(\.\d+)?)', score_text)
    return float(score_match.group(1)) if score_match else 0.5

def crag_process(query, vector_store, k=3):
    """CRAG：检索 → 对 top 文档打相关性分 → 按最高分选策略（仅本地/仅网络/混合）→ 生成"""
    query_embedding = create_embeddings(query)
    retrieved_docs = vector_store.similarity_search(query_embedding, k=k)

    relevance_scores = []
    for doc in retrieved_docs:
        score = evaluate_document_relevance(query, doc["text"])
        relevance_scores.append(score)
        doc["relevance"] = score

    max_score = max(relevance_scores) if relevance_scores else 0

    if max_score > 0.7:
        final_knowledge = retrieved_docs[relevance_scores.index(max_score)]["text"]
    elif max_score < 0.3:
        web_results, web_sources = perform_web_search(query)
        final_knowledge = refine_knowledge(web_results)
    else:
        best_doc = retrieved_docs[relevance_scores.index(max_score)]["text"]
        web_results, _ = perform_web_search(query)
        final_knowledge = refine_knowledge(best_doc) + "\n" + refine_knowledge(web_results)

    response = generate_response(query, final_knowledge)
    return response
```

## 20 强化学习增强RAG

**核心思路：** 把 RAG 流程看成强化学习：状态（当前 query、已选上下文、历史回复与奖励）、动作（重写 query、扩展/过滤上下文、生成回复）、奖励（如生成回复与标准答案的语义相似度）。用 epsilon-greedy 等策略根据状态选动作，用奖励迭代优化，使检索与生成策略逐步变好。

**优势与局限：**
- **优势：** 可端到端利用标注或用户反馈优化整条管道；理论上能学到复杂策略。
- **局限：** 需要奖励信号（标注答案或用户反馈）、训练循环与稳定性调参。
- **适用场景：** 有充足标注或反馈、希望长期自动优化效果的场景。

```python
# ========== 方法20：强化学习增强RAG ==========
# 状态-动作-奖励建模；用奖励（如回复与标准答案的相似度）驱动策略选择

def calculate_reward(response: str, ground_truth: str) -> float:
    """奖励函数：回复与标准答案的嵌入余弦相似度，范围约 [-1, 1]"""
    response_embedding = generate_embeddings([response])[0]
    ground_truth_embedding = generate_embeddings([ground_truth])[0]
    similarity = cosine_similarity(response_embedding, ground_truth_embedding)
    return similarity

def expand_context(query, current_chunks, top_k=3):
    """动作之一：在已有 chunks 基础上再检索若干相关块并追加"""
    additional_chunks = retrieve_relevant_chunks(query, top_k=top_k + len(current_chunks))
    new_chunks = [chunk for chunk in additional_chunks if chunk not in current_chunks]
    return current_chunks + new_chunks[:top_k]

def filter_context(query, context_chunks):
    """动作之一：按与 query 的相似度只保留最相关的若干块"""
    query_embedding = generate_embeddings([query])[0]
    chunk_embeddings = [generate_embeddings([chunk])[0] for chunk in context_chunks]

    relevance_scores = [cosine_similarity(query_embedding, emb) for emb in chunk_embeddings]
    sorted_chunks = [x for _, x in sorted(zip(relevance_scores, context_chunks), reverse=True)]
    return sorted_chunks[:min(5, len(sorted_chunks))]

def policy_network(state, action_space, epsilon=0.2):
    """epsilon-greedy：以概率 epsilon 随机探索，否则按启发式选动作（重写/扩展/过滤/生成）"""
    if np.random.random() < epsilon:
        action = np.random.choice(action_space)
    else:
        if len(state["previous_responses"]) == 0:
            action = "rewrite_query"
        elif state["previous_rewards"] and max(state["previous_rewards"]) < 0.7:
            action = "expand_context"
        elif len(state["context"]) > 5:
            action = "filter_context"
        else:
            action = "generate_response"
    return action
```

## 方法总结对比

编号|方法|优化维度|核心思想
-|-|-|-
1|语义分块|分块|基于语义相似度的智能分割|
2|切块大小评估|分块|数据驱动选择最优切块参数|
3|上下文增强检索|检索|返回匹配块及其相邻块|
4|上下文标题提取（CCH）|检索|标题+文本双维度匹配|
5|文档增强RAG|索引|预生成问题弥补语义鸿沟|
6|查询转换|查询|重写/回退/分解查询|
7|重排序|检索|两阶段：粗检索+精排序|
8|相关段落提取（RSE）|检索|最大子数组找连续最优段落|
9|上下文压缩|后处理|压缩检索结果去除噪声|
10|反馈回路|系统|用户反馈动态调整系统|
11|自适应检索|检索|按查询类型选策略|
12|Self-RAG|生成|多维度反思评估机制|
13|命题分块|分块|原子化事实陈述作为检索单元|
14|多模态RAG|索引|图文联合检索|
15|融合检索|检索|向量 + BM25 加权融合|
16|图RAG|检索|知识图谱关联发现|
17|分层检索|检索|摘要→详细的两级检索|
18|HyDE|查询|假设文档嵌入替代查询嵌入|
19|CRAG|系统|动态评估并纠正检索质量|
20|RL增强RAG|系统|强化学习优化全流程|

## 优化方法选择指南

- 分块优化：检索不够精准时，优先考虑语义分块（1）、命题分块（13）或切块大小评估（2）。
- 检索增强：融合检索（15）性价比高、易落地；图RAG（16）适合概念与关系密集的领域。
- 查询优化：HyDE（18）对短查询、抽象问句效果好；查询转换（6）适合复杂、多子问题查询。
- 后处理：重排序（7）+ 上下文压缩（9）能明显提升最终回复质量与可控性。
- 系统级：CRAG（19）适合对可靠性、时效性要求高的场景；反馈回路（10）适合有持续用户反馈的产品。
- 端到端：Self-RAG（12）与 RL 增强（20）能力最全面，实现与调参成本也更高，可按需选用。