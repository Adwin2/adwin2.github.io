---
layout: post
title: 用 Go 构建 RAG 系统：从文档到回答的工程实践
categories: [Go, AI]
description: 学习 RAG 的过程中用 Go 做了个完整的 demo，记录文档切分、向量检索、Prompt 工程的学习笔记。
keywords: RAG, Golang, LLM, 向量检索, Embedding, Milvus
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

最近在学 RAG（Retrieval-Augmented Generation），动手用 Go 写了一个知识库问答的 demo 项目。从文档解析到向量检索到 LLM 生成，完整跑通了一遍，踩了不少坑，这里做个记录。

## 整体架构

RAG 的核心思路不复杂：用户提问 → 检索相关文档 → 把文档和问题一起丢给 LLM 生成回答。但工程化后每一步都有细节。

我的 demo 分成两条链路：

**离线链路（索引构建）：**
1. 文档采集（内部 Wiki、Confluence、飞书文档）
2. 文档解析与清洗
3. 文本切分（Chunking）
4. Embedding 向量化
5. 写入向量数据库

**在线链路（问答服务）：**
1. 用户 Query 向量化
2. 向量检索 Top-K
3. 重排序（Rerank）
4. Prompt 组装
5. LLM 生成回答

## 文档切分：最影响效果的一步

一开始很天真地按固定 token 数切，512 tokens 一段，效果很差——经常把一段完整的操作流程从中间切断，检索回来的文档半头半尾的，LLM 理解不了。

### 递归切分 + 结构感知

最终用的方案是递归切分，优先按文档结构分：

```go
type ChunkConfig struct {
    MaxTokens    int      // 单个 chunk 最大 token 数
    OverlapTokens int     // 重叠 token 数
    Separators   []string // 分隔符优先级
}

var defaultSeparators = []string{
    "\n## ",    // 二级标题
    "\n### ",   // 三级标题
    "\n\n",     // 段落
    "\n",       // 换行
    "。",       // 句号
    ". ",       // 英文句号
}

func RecursiveSplit(text string, cfg ChunkConfig) []Chunk {
    if TokenCount(text) <= cfg.MaxTokens {
        return []Chunk{{Content: text}}
    }

    // 按优先级尝试分隔符
    for _, sep := range cfg.Separators {
        parts := strings.Split(text, sep)
        if len(parts) == 1 {
            continue
        }

        var chunks []Chunk
        var current strings.Builder

        for _, part := range parts {
            candidate := current.String() + sep + part
            if TokenCount(candidate) > cfg.MaxTokens && current.Len() > 0 {
                chunks = append(chunks, Chunk{Content: current.String()})
                // 保留重叠部分，提高检索连贯性
                current.Reset()
                current.WriteString(getOverlapSuffix(current.String(), cfg.OverlapTokens))
            }
            if current.Len() > 0 {
                current.WriteString(sep)
            }
            current.WriteString(part)
        }

        if current.Len() > 0 {
            chunks = append(chunks, Chunk{Content: current.String()})
        }
        return chunks
    }

    // 兜底：硬切
    return forceSplit(text, cfg.MaxTokens, cfg.OverlapTokens)
}
```

### chunk 附加元数据

光有文本不够，每个 chunk 还要带上来源信息：

```go
type Chunk struct {
    Content    string            // 文本内容
    DocID      string            // 原始文档 ID
    DocTitle   string            // 文档标题
    Section    string            // 所属章节标题
    ChunkIndex int               // 在文档中的位置
    Metadata   map[string]string // 额外元数据（作者、更新时间等）
}
```

检索时把 `DocTitle` 和 `Section` 也加进 Prompt，LLM 回答时就能引用来源，大幅提升可信度。

### chunk 大小的选择

试了几组参数，最终结论：

| Chunk 大小 | 效果 | 适用场景 |
|-----------|------|---------|
| 256 tokens | 检索精确但上下文不足 | FAQ、定义类问题 |
| 512 tokens | 综合最优 | 大多数场景 |
| 1024 tokens | 上下文充分但检索噪声大 | 流程文档、长篇分析 |

最终用 512 tokens + 64 tokens overlap。

## Embedding 选择

对比了几个方案：

| 模型 | 维度 | 中文效果 | 延迟 |
|-----|------|---------|------|
| OpenAI text-embedding-3-small | 1536 | 好 | 高（海外API） |
| BGE-M3 (本地部署) | 1024 | 好 | 低 |
| BCE-embedding-base_v1 | 768 | 较好 | 低 |

最终选了 BGE-M3 本地部署，走 HTTP 接口调用。Go 这边封装了一个 Embedding client：

```go
type EmbeddingClient struct {
    httpClient *http.Client
    endpoint   string
}

func (c *EmbeddingClient) Embed(ctx context.Context, texts []string) ([][]float32, error) {
    reqBody := embeddingRequest{
        Input: texts,
    }

    body, _ := json.Marshal(reqBody)
    req, _ := http.NewRequestWithContext(ctx, "POST", c.endpoint+"/embeddings", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("embedding request failed: %w", err)
    }
    defer resp.Body.Close()

    var result embeddingResponse
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, fmt.Errorf("decode embedding response: %w", err)
    }

    vectors := make([][]float32, len(result.Data))
    for i, d := range result.Data {
        vectors[i] = d.Embedding
    }
    return vectors, nil
}
```

**批量 Embedding 的坑：** 一次发太多文本会 OOM（embedding 模型那边），所以控制在每批 32 条，用 errgroup 并发 4 个批次。

## 向量数据库：Milvus

选 Milvus 主要因为：
1. 开源，自己能掌控
2. Go SDK 成熟
3. 支持标量过滤（可以按文档来源、时间等字段过滤）

核心的检索代码：

```go
func (s *VectorStore) Search(ctx context.Context, query []float32, topK int, filter string) ([]SearchResult, error) {
    sp, _ := entity.NewIndexIvfFlatSearchParam(64) // nprobe

    results, err := s.client.Search(
        ctx,
        s.collection,
        nil,               // partitions
        filter,            // 标量过滤，如 "source == 'wiki'"
        []string{"content", "doc_title", "section", "doc_id"},
        []entity.Vector{entity.FloatVector(query)},
        "embedding",
        entity.COSINE,
        topK,
        sp,
    )
    if err != nil {
        return nil, fmt.Errorf("milvus search: %w", err)
    }

    var searchResults []SearchResult
    for _, r := range results {
        for i := 0; i < r.ResultCount; i++ {
            searchResults = append(searchResults, SearchResult{
                Content:  r.Fields.GetColumn("content").GetAsString(i),
                DocTitle: r.Fields.GetColumn("doc_title").GetAsString(i),
                Section:  r.Fields.GetColumn("section").GetAsString(i),
                Score:    r.Scores[i],
            })
        }
    }
    return searchResults, nil
}
```

### 混合检索

纯向量检索对关键词类 query 效果差（比如搜错误码 "ERR_10042"，语义匹配完全不行）。于是加了 BM25 关键词检索做混合：

```go
func (s *HybridSearcher) Search(ctx context.Context, query string, topK int) ([]SearchResult, error) {
    // 并发执行向量检索和关键词检索
    g, ctx := errgroup.WithContext(ctx)

    var vectorResults, bm25Results []SearchResult

    g.Go(func() error {
        queryVec, err := s.embedder.Embed(ctx, []string{query})
        if err != nil {
            return err
        }
        vectorResults, err = s.vectorStore.Search(ctx, queryVec[0], topK*2, "")
        return err
    })

    g.Go(func() error {
        var err error
        bm25Results, err = s.bm25Store.Search(ctx, query, topK*2)
        return err
    })

    if err := g.Wait(); err != nil {
        return nil, err
    }

    // RRF (Reciprocal Rank Fusion) 合并结果
    return reciprocalRankFusion(vectorResults, bm25Results, topK), nil
}
```

**RRF 融合** 简单好用，不需要归一化分数，直接按排名融合：

```go
func reciprocalRankFusion(results ...[]SearchResult, topK int) []SearchResult {
    const k = 60 // RRF 常数
    scores := make(map[string]float64) // key: chunk ID
    docMap := make(map[string]SearchResult)

    for _, resultList := range results {
        for rank, r := range resultList {
            id := r.DocID + "_" + strconv.Itoa(r.ChunkIndex)
            scores[id] += 1.0 / float64(rank+k)
            docMap[id] = r
        }
    }

    // 按融合分数排序，取 topK
    // ...
}
```

混合检索加上后，在测试集上 Recall@10 从 0.72 提升到 0.85。

## Prompt 工程

Prompt 模板是影响回答质量的最后一环。初版很简单：

```
根据以下文档回答问题：
{documents}
问题：{query}
```

效果一般。优化后的模板：

```go
const promptTemplate = `你是一个内部知识库助手。请根据提供的参考文档回答用户问题。

要求：
1. 只基于参考文档中的信息回答，不要编造
2. 如果参考文档中没有相关信息，明确告知用户
3. 在回答末尾标注信息来源（文档标题和章节）

参考文档：
{{range .Documents}}
---
来源：{{.DocTitle}} > {{.Section}}
内容：{{.Content}}
---
{{end}}

用户问题：{{.Query}}`
```

几个关键的 Prompt 技巧：
- **明确要求引用来源**：减少幻觉，用户也能验证
- **明确说"不知道就说不知道"**：避免 LLM 瞎编
- **把来源信息放在文档开头**：LLM 对开头和结尾的信息记忆更强

## 效果优化的几个教训

### 1. Query 改写

用户的提问经常很口语化。"上次那个部署报错咋整" → 向量检索效果很差。可以加一步 query 改写，用 LLM 把口语化 query 转成检索友好的形式：

```
原始 query: 上次那个部署报错咋整
改写后: 部署过程中出现错误的排查和解决方法
```

这一步用一个轻量模型就行，延迟增加约 200ms，但检索效果提升明显。

### 2. Rerank 重排序

向量检索的 Top-20 用 rerank 模型精排到 Top-5。试了 BCE-reranker，效果比直接取 Top-5 好很多，尤其是当 query 比较长或者比较模糊的时候。

### 3. 不要把太多文档塞进 Prompt

一开始贪多，塞了 10 个 chunk 进去，结果 LLM 经常从不太相关的 chunk 里拼凑出错误答案。减到 3-5 个反而效果更好。context window 大不等于应该塞满。

## 性能数据

最终在线链路的延迟分布：

| 阶段 | P50 延迟 | P99 延迟 |
|-----|---------|---------|
| Query Embedding | 15ms | 30ms |
| 向量检索 (Milvus) | 8ms | 25ms |
| BM25 检索 | 5ms | 15ms |
| Rerank | 80ms | 150ms |
| LLM 生成 (首 token) | 300ms | 800ms |
| **总计** | **~400ms** | **~1s** |

瓶颈在 LLM 生成。用流式输出（SSE）的话，用户体感延迟主要就是首 token 时间，可以接受。

## 总结

RAG 看起来简单，但工程化后有大量细节。最重要的三点：
1. **切分策略决定上限** — 切得不好，后面怎么优化都救不回来
2. **混合检索 > 纯向量检索** — 关键词和语义互补
3. **少即是多** — 宁可给 LLM 3 个高质量 chunk，也不要 10 个有噪声的
