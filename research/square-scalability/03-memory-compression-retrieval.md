# Memory压缩与检索技术调研

**Intel 研究文档** 🔍  
**日期**: 2026-03-01  
**主题**: 向量数据库对比 + 记忆压缩技术 + 成本优化

---

## 1. 向量数据库对比（2026最新）

### 1.1 主流选项快速对比

| 数据库 | 类型 | 适用规模 | 核心优势 | 核心限制 | 价格 |
|--------|------|----------|----------|----------|------|
| **Pinecone** | 全托管 | 10M-100M+ | 零运维，最简单 | 按用量计费 | $0.33/GB + 操作费 |
| **Milvus** | 开源 | 10亿级 | 最流行OSS | 运维复杂 | 免费（基础设施费） |
| **Weaviate** | 开源+托管 | <50M | 最佳混合搜索 | 14天试用限制 | $25/月起 |
| **Qdrant** | 开源+托管 | <50M | 最佳免费层(1GB) | 超10M吞吐量下降 | 1GB永久免费 |
| **pgvector** | PostgreSQL扩展 | <100M | 统一数据模型 | 不适合超1亿 | 仅PostgreSQL成本 |
| **Elasticsearch** | 搜索引擎 | 50M+ | 成熟稳定 | 延迟较高 | 云版本按需 |

### 1.2 ⭐ 推荐：pgvector + pgvectorscale

**为什么推荐给AI Pet项目**：

1. **我们已使用PostgreSQL**（pi-agent-core）
2. **性能现在足够**：
   - 50M向量 → 471 QPS @ 99% recall
   - 比Pinecone p95延迟低28倍
   - 比Qdrant高11.4倍吞吐量
3. **成本优势**：
   - 比Pinecone便宜75%+
   - 无额外数据库运维
4. **统一数据模型**：
   - Pet关系数据 + 向量记忆同一个数据库
   - 单事务保证一致性

**技术要点**：
```sql
-- 安装扩展
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pgvectorscale;

-- 创建记忆表
CREATE TABLE pet_memories (
  id SERIAL PRIMARY KEY,
  pet_id INT REFERENCES pets(id),
  content TEXT NOT NULL,
  embedding vector(1024),  -- Titan Embed v2维度
  importance FLOAT DEFAULT 0.5,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_accessed_at TIMESTAMPTZ DEFAULT NOW()
);

-- DiskANN索引（pgvectorscale）
CREATE INDEX ON pet_memories 
USING diskann (embedding)
WITH (num_neighbors=50);
```

---

## 2. Memory压缩技术

### 2.1 MemGPT架构（操作系统式记忆管理）

**核心思想**：把LLM当成操作系统，有分层内存

```
┌─────────────────────────────────────────────────┐
│  Main Context（主内存 = RAM）                    │
│  - 当前对话窗口                                  │
│  - 受LLM token限制                               │
└─────────────────────────────────────────────────┘
                    ↕ 分页 (Paging)
┌─────────────────────────────────────────────────┐
│  External Context（外部存储 = 磁盘）             │
│  - Core Memory: 压缩的核心事实                   │
│  - Recall Memory: 可搜索的记忆数据库             │
│  - Archival Memory: 长期存储                     │
└─────────────────────────────────────────────────┘
```

**关键机制**：

| 机制 | 描述 | 对AI Pet的应用 |
|------|------|----------------|
| **自我编辑** | LLM自己决定存什么、删什么 | Pet主动管理记忆 |
| **战略遗忘** | 不是失败，是特性 | 避免"上下文污染" |
| **语义搜索** | 概念相关，不只是关键词 | 回忆相关记忆 |
| **分页** | 主→外存交换 | 无限上下文假象 |

**战略遗忘的实现**：
```typescript
interface MemoryTriage {
  priority: 'high' | 'medium' | 'low';
  action: 'retain' | 'summarize' | 'delete';
}

// LLM评估每条记忆的未来价值
async function triageMemory(memory: PetMemory): Promise<MemoryTriage> {
  const evaluation = await llm.chat({
    messages: [{
      role: 'system',
      content: `评估这条记忆对Pet未来互动的重要性：
"${memory.content}"

考虑：
- 用户偏好？（高优先）
- 核心事实？（高优先）
- 重复信息？（低优先）
- 瞬时对话？（低优先）

返回JSON: {priority: "high|medium|low", action: "retain|summarize|delete"}`
    }]
  });
  return JSON.parse(evaluation);
}
```

### 2.2 HiMem架构（层级长期记忆 - 2026最新）

**论文来源**：arXiv:2601.06377（2026年1月）

**核心创新**：双层语义链接

```
┌─────────────────────────────────────────────────┐
│  Note Memory（笔记记忆）                         │
│  - 稳定知识：事实、偏好、档案                     │
│  - 紧凑表示                                      │
│  - 快速检索                                      │
└─────────────────────────────────────────────────┘
                    ↕ 语义链接
┌─────────────────────────────────────────────────┐
│  Episode Memory（情节记忆）                      │
│  - 细粒度互动片段                                │
│  - 时间和话题边界                                │
│  - 保持原始上下文                                │
└─────────────────────────────────────────────────┘
```

**话题感知事件-惊讶双通道分割**：
```
分割边界触发条件（OR规则）：
1. 话题转变（讨论目标或子话题变化）
2. 显著不连续（意图或情绪突变）
```

**三类知识提取**：
```typescript
interface NoteMemory {
  facts: string[];      // 客观事实
  preferences: string[]; // 用户偏好
  profile: string[];     // 稳定特征
}

// 多阶段提取
// Stage 1: 独立可解释的事实单元
// Stage 2: 高置信度隐式信息
// Stage 3: 非破坏性归一化（去重、指代消解、时间规范化）
```

**冲突感知记忆重整**：
```
触发条件（AND规则）：
1. Note Memory检索不足
2. Episode Memory提供了充足证据

更新操作：
- 独立 → ADD
- 可扩展 → UPDATE
- 矛盾 → DELETE

Episode Memory是不可变的（只追加）
```

**性能提升**（LoCoMo基准）：
- Multi-Hop推理：显著提升
- 时间推理：显著提升
- 开放域问题：改善

---

## 3. 记忆检索策略

### 3.1 混合检索 vs 最优努力检索

| 策略 | 描述 | 优势 | 劣势 |
|------|------|------|------|
| **混合检索** | 同时查Note和Episode | 最大召回 | 成本高 |
| **最优努力** | 先Note，不足再Episode | 效率高 | 可能遗漏 |

**推荐**：AI Pet使用最优努力检索

```typescript
async function retrieveMemory(pet: Pet, query: string): Promise<MemoryResult> {
  // Step 1: 查询Note Memory（快）
  const notes = await queryNoteMemory(pet.id, query);
  
  // Step 2: 评估是否足够
  const sufficient = await evaluateSufficiency(query, notes);
  
  if (sufficient) {
    return { source: 'note', memories: notes };
  }
  
  // Step 3: 不足时查询Episode Memory
  const episodes = await queryEpisodeMemory(pet.id, query);
  
  // Step 4: 可能触发记忆重整
  await maybeReconsolidate(pet, query, notes, episodes);
  
  return { source: 'episode', memories: episodes };
}
```

### 3.2 Generative Agents检索公式（复习）

```
Score = α × Recency + β × Importance + γ × Relevance

Recency: 指数衰减，基于最后访问时间
Importance: LLM评分1-10
Relevance: 查询与记忆的embedding余弦相似度

权重通常相等: α = β = γ = 0.33
```

---

## 4. AWS Bedrock成本分析

### 4.1 Embedding模型

**Amazon Titan Text Embeddings V2**：
- 价格：**$0.00011 / 1000 tokens**
- 维度：1024
- 最大输入：8192 tokens

**成本估算**（AI Pet项目）：

| 场景 | tokens/次 | 次数/天 | 日成本 |
|------|-----------|---------|--------|
| Pet互动记忆 | 200 | 100,000 | $2.20 |
| 记忆检索 | 50 | 500,000 | $2.75 |
| 反思生成 | 500 | 10,000 | $0.55 |
| **总计** | - | - | **$5.50/天** |

**月成本**：~$165（10K活跃Pet）

### 4.2 LLM成本（Nova Lite）

**Amazon Nova Lite**：
- 输入：$0.00006 / 1000 tokens
- 输出：$0.00024 / 1000 tokens

| 操作 | 输入tokens | 输出tokens | 次数/天 | 日成本 |
|------|------------|------------|---------|--------|
| 对话回复 | 500 | 100 | 100,000 | $5.40 |
| 反思生成 | 2000 | 300 | 10,000 | $1.92 |
| 规划 | 1000 | 200 | 10,000 | $1.08 |
| **总计** | - | - | - | **$8.40/天** |

**月成本**：~$252（10K活跃Pet）

### 4.3 总月成本

| 组件 | 月成本 |
|------|--------|
| Embedding | $165 |
| LLM | $252 |
| PostgreSQL (RDS) | ~$100 |
| **总计** | **~$517/月** |

**每Pet成本**：$0.05/月（10K Pet）

---

## 5. AI Pet记忆系统设计建议

### 5.1 推荐架构

```
┌─────────────────────────────────────────────────┐
│  Pet Memory System                               │
├─────────────────────────────────────────────────┤
│  Layer 1: Core Memory（核心记忆）                │
│  - 个性定义                                      │
│  - 与主人的关系                                  │
│  - 当前情绪状态                                  │
│  ← 始终在prompt中                                │
├─────────────────────────────────────────────────┤
│  Layer 2: Note Memory（笔记记忆）                │
│  - 稳定偏好                                      │
│  - 重要事实                                      │
│  - 朋友关系                                      │
│  ← pgvector存储，最优努力检索                    │
├─────────────────────────────────────────────────┤
│  Layer 3: Episode Memory（情节记忆）             │
│  - 互动片段                                      │
│  - 对话历史                                      │
│  - 广场事件                                      │
│  ← pgvector存储，fallback检索                    │
└─────────────────────────────────────────────────┘
```

### 5.2 实现代码框架

```typescript
interface PetMemorySystem {
  core: CoreMemory;
  notes: NoteMemory[];
  episodes: EpisodeMemory[];
}

interface CoreMemory {
  personality: string;
  ownerRelation: string;
  currentMood: string;
  // 始终在context window中
}

interface NoteMemory {
  id: string;
  content: string;
  category: 'fact' | 'preference' | 'profile';
  embedding: number[];
  createdAt: Date;
}

interface EpisodeMemory {
  id: string;
  topic: string;
  summary: string;
  dialogue: string;
  embedding: number[];
  timestamp: Date;
}

class PetAgent {
  async respond(userInput: string): Promise<string> {
    // 1. 构建context
    const coreContext = this.formatCoreMemory();
    
    // 2. 检索相关记忆（最优努力）
    const relevantMemories = await this.retrieveMemories(userInput);
    
    // 3. 生成回复
    const response = await this.llm.chat({
      messages: [
        { role: 'system', content: coreContext + relevantMemories },
        { role: 'user', content: userInput }
      ]
    });
    
    // 4. 存储新记忆
    await this.storeInteraction(userInput, response);
    
    // 5. 可能触发反思
    await this.maybeReflect();
    
    return response;
  }
}
```

### 5.3 成本优化策略

1. **批量Embedding**：合并多条记忆一次请求
2. **缓存热门记忆**：Redis缓存频繁访问的embedding
3. **压缩老记忆**：超过30天的Episode自动总结
4. **分层存储**：活跃Pet用RDS，休眠Pet用S3

---

## 6. 关键结论

### 6.1 向量数据库选择

**推荐**：pgvector + pgvectorscale
- 理由：已有PostgreSQL，性能足够，成本最低

### 6.2 记忆架构选择

**推荐**：HiMem双层架构
- Note Memory（笔记）+ Episode Memory（情节）
- 最优努力检索策略
- 冲突感知记忆重整

### 6.3 成本可控性

**10K活跃Pet**：~$500/月
**每Pet成本**：$0.05/月

可行！ ✅

---

## 7. 下一步

- [ ] 设计社交网络图结构
- [ ] 研究霸凌/孤立检测算法
- [ ] 规划MVP记忆系统实现范围

---

*Intel Memory压缩与检索调研完成*
