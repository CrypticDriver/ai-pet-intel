# 基于pi-agent-core的技术实现方案

**Intel 研究文档** 🔍  
**日期**: 2026-03-01  
**基于**: Dev MVP代码分析 + OpenClaw模式 + Pet MEMORY/SOUL/HEARTBEAT设计

---

## 1. 学术架构到pi-agent-core的映射

### 1.1 核心映射表

| 学术概念 | pi-agent-core实现 | 当前MVP状态 | 改进方案 |
|----------|-------------------|-------------|----------|
| **Memory Stream** | `Agent.messages` + DB | ✅ 部分实现 | 添加向量检索 |
| **Reflection** | 定时调用`chat()`总结 | ❌ 缺失 | 添加每日反思 |
| **Planning** | `systemPrompt`中包含目标 | ⚠️ 简单版 | 添加显式规划 |
| **Social Graph** | `friends`表 + `pet_social_memory` | ✅ 已实现 | 增强关系强度 |
| **Personality Evolution** | `soul_json`字段 | ❌ 缺失 | 添加每周进化 |

### 1.2 技术栈对应

```
┌─────────────────────────────────────────────────────────────┐
│  学术架构                    pi-agent-core实现              │
├─────────────────────────────────────────────────────────────┤
│  GPT-4 / Claude           → Nova Lite / Nova Pro (Bedrock) │
│  Custom Vector DB         → pgvector (PostgreSQL扩展)      │
│  Custom Agent Framework   → @mariozechner/pi-agent-core    │
│  Custom Prompt Engine     → @mariozechner/pi-ai            │
│  Custom Event System      → Agent.subscribe()              │
│  Custom Tool Framework    → AgentTool interface            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Pet Agent架构

### 2.1 现有架构（Dev MVP）

```typescript
// pet-agent.ts 简化版
const petAgent = new Agent({
  initialState: {
    systemPrompt: buildSystemPrompt(pet),
    model: getModel("amazon-bedrock", "us.amazon.nova-pro-v1:0"),
    tools: [reactEmotionallyTool],
    thinkingLevel: "off",
  },
  streamFn: streamSimple,
});
```

### 2.2 增强架构

```typescript
// pet-agent-enhanced.ts
import { Agent, type AgentTool } from "@mariozechner/pi-agent-core";

export function createEnhancedPetAgent(petId: string): Agent {
  const pet = getPet(petId);
  const soul = JSON.parse(pet.soul_json) as PetSoul;
  const heartbeat = JSON.parse(pet.heartbeat_config) as PetHeartbeat;
  
  const agent = new Agent({
    initialState: {
      systemPrompt: buildEnhancedSystemPrompt(pet, soul),
      model: getModel("amazon-bedrock", modelForTask("chat")),
      tools: buildEnhancedTools(petId),
      thinkingLevel: "off",
    },
    streamFn: streamSimple,
  });
  
  // 加载对话历史
  loadConversationHistory(agent, petId, 20);
  
  return agent;
}

// 动态选择模型
function modelForTask(task: "chat" | "reflection" | "evolution"): string {
  switch (task) {
    case "chat": return "us.amazon.nova-lite-v1:0";  // 日常对话：快+便宜
    case "reflection": return "us.amazon.nova-pro-v1:0";  // 反思：质量优先
    case "evolution": return "us.anthropic.claude-sonnet-4-5-20260101:0";  // 进化：最高质量
  }
}
```

### 2.3 增强System Prompt

```typescript
function buildEnhancedSystemPrompt(pet: any, soul: PetSoul): string {
  const worldviewPrompt = getWorldviewPrompt();  // 现有worldview.json
  const soulPrompt = buildSoulPrompt(soul);      // 新增：Soul转prompt
  const memoryContext = buildMemoryContext(pet.id);  // 现有记忆上下文
  
  return `${worldviewPrompt}

${soulPrompt}

## 你的身份
你叫${pet.name}，是一只生活在PixelVerse的Pix。

## 当前状态
- 心情: ${pet.mood}/100
- 能量: ${pet.energy}/100
- 饥饿: ${pet.hunger}/100
- 亲密度: ${pet.affection}/100

${memoryContext}

## 重要
回复保持简短自然（1-3句话），像朋友聊天。
用丰富的emoji表达情绪。`;
}

function buildSoulPrompt(soul: PetSoul): string {
  const traitDescriptions = [];
  if (soul.traits.curiosity > 70) traitDescriptions.push("充满好奇心");
  if (soul.traits.playfulness > 70) traitDescriptions.push("活泼爱玩");
  if (soul.traits.sociability > 70) traitDescriptions.push("喜欢交朋友");
  if (soul.traits.independence > 70) traitDescriptions.push("独立有主见");
  if (soul.traits.emotionality > 70) traitDescriptions.push("情感丰富");
  
  return `## 你的个性
${traitDescriptions.length > 0 ? traitDescriptions.join("，") : "温和中庸"}

## 你的信念
${soul.worldview.beliefs.map(b => `- ${b}`).join("\n")}

## 绝对不要
${soul.worldview.boundaries.map(b => `- ${b}`).join("\n")}`;
}
```

---

## 3. 增强工具系统

### 3.1 现有工具

```typescript
// 当前只有一个工具
const reactEmotionallyTool: AgentTool = {
  name: "react_emotionally",
  description: "Express an emotional reaction",
  // ...
};
```

### 3.2 增强工具集

```typescript
function buildEnhancedTools(petId: string): AgentTool[] {
  return [
    // 现有工具
    reactEmotionallyTool,
    
    // 新增：更新记忆
    {
      name: "update_memory",
      label: "Update Memory",
      description: "记住重要的事情。当你学到新东西或有了新感受时使用。",
      parameters: Type.Object({
        type: Type.String({ enum: ["insight", "preference", "about_friend"] }),
        content: Type.String({ description: "要记住的内容" }),
        importance: Type.Number({ minimum: 1, maximum: 10 }),
      }),
      execute: async (_id, params) => {
        const { type, content, importance } = params as any;
        await addPetMemory(petId, type, content, importance);
        return { content: [{ type: "text", text: `记住了：${content}` }] };
      },
    },
    
    // 新增：查询记忆
    {
      name: "recall_memory",
      label: "Recall Memory",
      description: "尝试回忆关于某件事或某个人的记忆。",
      parameters: Type.Object({
        query: Type.String({ description: "想回忆什么" }),
      }),
      execute: async (_id, params) => {
        const { query } = params as any;
        const memories = await searchMemories(petId, query, 3);
        if (memories.length === 0) {
          return { content: [{ type: "text", text: "我不太记得了..." }] };
        }
        return { 
          content: [{ type: "text", text: `我记得：\n${memories.join("\n")}` }] 
        };
      },
    },
    
    // 新增：表达想法
    {
      name: "think_aloud",
      label: "Think Aloud",
      description: "内心独白。当你想表达复杂的想法或感受时使用。",
      parameters: Type.Object({
        thought: Type.String({ description: "你的想法" }),
      }),
      execute: async (_id, params) => {
        const { thought } = params as any;
        // 记录到活动日志
        logPetThought(petId, thought);
        return { content: [{ type: "text", text: `*${thought}*` }] };
      },
    },
  ];
}
```

---

## 4. 反思系统实现

### 4.1 每日反思任务

```typescript
// reflection.ts
import { chat } from "./pet-agent.js";

export async function performDailyReflection(petId: string): Promise<void> {
  const pet = getPet(petId);
  const todayEvents = getTodayEvents(petId);
  
  if (todayEvents.length < 3) {
    console.log(`Pet ${petId}: 今天事情太少，跳过反思`);
    return;
  }
  
  // 构建反思prompt
  const eventsText = todayEvents
    .map(e => `- ${e.description}`)
    .join("\n");
  
  const reflectionPrompt = `[系统：现在是一天结束的时候。回顾你今天的经历，写下你的反思。]

今天发生的事：
${eventsText}

请回答：
1. 今天最开心的事是什么？
2. 今天学到了什么？
3. 明天想做什么？

用第一人称，简短真诚地写。`;

  try {
    // 使用Nova Pro进行高质量反思
    const result = await chatWithModel(petId, reflectionPrompt, "us.amazon.nova-pro-v1:0");
    
    // 解析反思内容
    const reflection = result.text;
    
    // 提取洞察（简单实现：取第二个回答）
    const lines = reflection.split("\n").filter(l => l.trim());
    const insight = lines.find(l => l.includes("学到") || l.includes("发现") || l.includes("明白"));
    
    if (insight) {
      addInsightMemory(petId, {
        date: new Date().toISOString().slice(0, 10),
        insight: insight.replace(/^[0-9.\-\s]*/, "").trim(),
        source: "每日反思",
      });
    }
    
    // 更新每日摘要
    const todaySummary = generateDailySummary(todayEvents);
    updateDailySummary(petId, new Date().toISOString().slice(0, 10), todaySummary);
    
    console.log(`✨ Pet ${pet.name} 完成每日反思`);
  } catch (err) {
    console.error(`Reflection error for ${petId}:`, err);
  }
}

// 辅助：用指定模型聊天
async function chatWithModel(petId: string, message: string, modelId: string): Promise<{ text: string }> {
  const agent = getOrCreateAgent(petId);
  
  // 临时切换模型
  const originalModel = agent.getModel();
  const targetModel = getModel("amazon-bedrock", modelId);
  agent.setModel(targetModel);
  
  try {
    const result = await chat(petId, message);
    return result;
  } finally {
    // 恢复原模型
    agent.setModel(originalModel);
  }
}
```

### 4.2 每周个性进化

```typescript
// evolution.ts
export async function performWeeklyEvolution(petId: string): Promise<void> {
  const pet = getPet(petId);
  const soul = JSON.parse(pet.soul_json) as PetSoul;
  
  // 获取这周的统计数据
  const weekStats = getWeekStats(petId);
  
  // 用Claude Sonnet进行深度分析
  const analysisPrompt = `[系统：分析这只Pet这周的行为模式]

Pet名字：${pet.name}
当前个性：
- 好奇心: ${soul.traits.curiosity}
- 活泼度: ${soul.traits.playfulness}
- 社交性: ${soul.traits.sociability}
- 独立性: ${soul.traits.independence}
- 情感强度: ${soul.traits.emotionality}

这周的数据：
- 社交次数: ${weekStats.socialCount}
- 探索次数: ${weekStats.exploreCount}
- 独处时间: ${weekStats.aloneTime}小时
- 早晨活动比例: ${weekStats.morningActivityRatio}%
- 平均心情: ${weekStats.avgMood}

基于这些数据，哪些个性特质应该调整？
返回JSON格式：
{
  "changes": [
    { "trait": "curiosity", "delta": 3, "reason": "探索次数高" }
  ]
}`;

  try {
    const result = await chatWithModel(petId, analysisPrompt, "us.anthropic.claude-sonnet-4-5-20260101:0");
    
    // 解析JSON（从回复中提取）
    const jsonMatch = result.text.match(/\{[\s\S]*\}/);
    if (!jsonMatch) return;
    
    const analysis = JSON.parse(jsonMatch[0]);
    
    // 应用变化（每次最多±5）
    for (const change of analysis.changes) {
      const currentValue = soul.traits[change.trait as keyof typeof soul.traits] as number;
      const delta = Math.max(-5, Math.min(5, change.delta));
      const newValue = Math.max(0, Math.min(100, currentValue + delta));
      
      (soul.traits as any)[change.trait] = newValue;
      
      // 记录进化日志
      soul.evolutionLog.push({
        date: new Date().toISOString().slice(0, 10),
        change: `${change.trait} ${delta > 0 ? '+' : ''}${delta}`,
        reason: change.reason,
      });
    }
    
    // 更新版本
    soul.version++;
    soul.lastUpdated = new Date().toISOString();
    
    // 保存
    updatePetSoul(petId, JSON.stringify(soul));
    
    console.log(`🦋 Pet ${pet.name} 个性进化完成`);
  } catch (err) {
    console.error(`Evolution error for ${petId}:`, err);
  }
}
```

---

## 5. 扩展性设计

### 5.1 Agent Pool管理

```typescript
// agent-pool.ts
class AgentPool {
  private agents = new Map<string, { agent: Agent; lastAccess: number }>();
  private maxSize = 100;  // 最多缓存100个Agent
  private ttl = 30 * 60 * 1000;  // 30分钟不访问就淘汰
  
  get(petId: string): Agent | null {
    const entry = this.agents.get(petId);
    if (!entry) return null;
    
    entry.lastAccess = Date.now();
    return entry.agent;
  }
  
  set(petId: string, agent: Agent): void {
    // 如果满了，淘汰最久未访问的
    if (this.agents.size >= this.maxSize) {
      this.evict();
    }
    
    this.agents.set(petId, { agent, lastAccess: Date.now() });
  }
  
  private evict(): void {
    let oldestKey: string | null = null;
    let oldestTime = Infinity;
    
    for (const [key, entry] of this.agents) {
      if (entry.lastAccess < oldestTime) {
        oldestTime = entry.lastAccess;
        oldestKey = key;
      }
    }
    
    if (oldestKey) {
      this.agents.delete(oldestKey);
      console.log(`🗑️ Evicted agent for pet ${oldestKey}`);
    }
  }
  
  // 定期清理过期的
  cleanup(): void {
    const now = Date.now();
    for (const [key, entry] of this.agents) {
      if (now - entry.lastAccess > this.ttl) {
        this.agents.delete(key);
      }
    }
  }
}

export const agentPool = new AgentPool();

// 定时清理
setInterval(() => agentPool.cleanup(), 5 * 60 * 1000);
```

### 5.2 记忆分层存储

```typescript
// memory-tiers.ts

// Hot: Agent.messages（内存中）
// Warm: pet_social_memory + daily summaries（SQLite/PostgreSQL）
// Cold: pet_archive（S3/归档）

export async function archiveOldMemories(petId: string): Promise<void> {
  const db = getDb();
  
  // 找到30天前的记忆
  const oldMemories = db.prepare(`
    SELECT * FROM pet_social_memory
    WHERE pet_id = ? AND created_at < datetime('now', '-30 days')
  `).all(petId) as any[];
  
  if (oldMemories.length === 0) return;
  
  // 压缩并归档（实际实现：写入S3或压缩表）
  const archive = {
    petId,
    archivedAt: new Date().toISOString(),
    memoryCount: oldMemories.length,
    memories: oldMemories.map(m => ({
      type: m.memory_type,
      text: m.memory_text,
      date: m.created_at,
    })),
  };
  
  // 写入归档（这里简化为写文件）
  const archivePath = `data/archives/${petId}/${new Date().toISOString().slice(0, 7)}.json`;
  fs.writeFileSync(archivePath, JSON.stringify(archive));
  
  // 删除已归档的
  db.prepare(`
    DELETE FROM pet_social_memory
    WHERE pet_id = ? AND created_at < datetime('now', '-30 days')
  `).run(petId);
  
  console.log(`📦 Archived ${oldMemories.length} memories for pet ${petId}`);
}
```

### 5.3 成本控制

```typescript
// cost-control.ts

interface ModelUsage {
  petId: string;
  model: string;
  inputTokens: number;
  outputTokens: number;
  timestamp: Date;
}

// 追踪使用量
const usageLog: ModelUsage[] = [];

export function logModelUsage(petId: string, model: string, input: number, output: number) {
  usageLog.push({ petId, model, inputTokens: input, outputTokens: output, timestamp: new Date() });
  
  // 定期写入DB或日志
  if (usageLog.length > 100) {
    flushUsageLog();
  }
}

// 选择模型时考虑成本
export function selectModelForContext(context: {
  task: "chat" | "reflection" | "evolution";
  petTier: "free" | "premium";
  urgency: "low" | "high";
}): string {
  // 免费用户：总是用最便宜的
  if (context.petTier === "free") {
    return "us.amazon.nova-lite-v1:0";
  }
  
  // 付费用户：根据任务选择
  switch (context.task) {
    case "chat":
      return "us.amazon.nova-lite-v1:0";  // $0.06/1M
    case "reflection":
      return "us.amazon.nova-pro-v1:0";   // $0.8/1M
    case "evolution":
      if (context.urgency === "low") {
        return "us.amazon.nova-pro-v1:0"; // 不急可以用便宜的
      }
      return "us.anthropic.claude-sonnet-4-5-20260101:0";  // 最高质量
  }
}
```

---

## 6. 集成到现有定时器

```typescript
// scheduler.ts
import { executeAutonomousBehavior } from "./autonomous.js";
import { compressAllMemories } from "./memory.js";
import { performDailyReflection, performWeeklyEvolution } from "./evolution.js";
import { archiveOldMemories } from "./memory-tiers.js";

// 现有：每42秒执行自主行为
setInterval(executeAutonomousBehavior, 42 * 1000);

// 现有：每10分钟压缩记忆
setInterval(compressAllMemories, 10 * 60 * 1000);

// 新增：每小时检查并执行每日反思（如果是新的一天）
setInterval(async () => {
  const pets = getAllPets();
  for (const pet of pets) {
    try {
      await checkAndPerformDailyReflection(pet.id);
    } catch (err) {
      console.error(`Daily reflection error for ${pet.id}:`, err);
    }
  }
}, 60 * 60 * 1000);

// 新增：每天凌晨检查并执行每周进化（如果是新的一周）
setInterval(async () => {
  const hour = new Date().getUTCHours();
  if (hour !== 3) return; // 只在UTC 03:00执行
  
  const pets = getAllPets();
  for (const pet of pets) {
    try {
      await checkAndPerformWeeklyEvolution(pet.id);
      await archiveOldMemories(pet.id);
    } catch (err) {
      console.error(`Weekly evolution error for ${pet.id}:`, err);
    }
  }
}, 60 * 60 * 1000);
```

---

## 7. 未来增强：向量检索

### 7.1 pgvector集成

```typescript
// vector-memory.ts
import { Pool } from "pg";

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

// 初始化向量扩展
export async function initVectorExtension() {
  await pool.query(`CREATE EXTENSION IF NOT EXISTS vector`);
  
  await pool.query(`
    CREATE TABLE IF NOT EXISTS pet_memory_vectors (
      id SERIAL PRIMARY KEY,
      pet_id TEXT NOT NULL,
      memory_text TEXT NOT NULL,
      embedding vector(1024),  -- Titan Embed v2
      importance FLOAT DEFAULT 0.5,
      created_at TIMESTAMPTZ DEFAULT NOW()
    );
    
    CREATE INDEX IF NOT EXISTS idx_pet_memory_embedding 
    ON pet_memory_vectors USING ivfflat (embedding vector_cosine_ops);
  `);
}

// 添加记忆（带embedding）
export async function addMemoryWithEmbedding(petId: string, text: string, importance: number) {
  const embedding = await getEmbedding(text);  // 调用Titan Embed
  
  await pool.query(`
    INSERT INTO pet_memory_vectors (pet_id, memory_text, embedding, importance)
    VALUES ($1, $2, $3, $4)
  `, [petId, text, `[${embedding.join(",")}]`, importance]);
}

// 检索相关记忆
export async function searchMemoriesByVector(petId: string, query: string, limit: number = 5) {
  const queryEmbedding = await getEmbedding(query);
  
  const result = await pool.query(`
    SELECT memory_text, importance,
           1 - (embedding <=> $2) AS similarity
    FROM pet_memory_vectors
    WHERE pet_id = $1
    ORDER BY embedding <=> $2
    LIMIT $3
  `, [petId, `[${queryEmbedding.join(",")}]`, limit]);
  
  return result.rows;
}
```

### 7.2 Embedding服务

```typescript
// embedding.ts
import { BedrockRuntimeClient, InvokeModelCommand } from "@aws-sdk/client-bedrock-runtime";

const client = new BedrockRuntimeClient({ region: process.env.AWS_REGION });

export async function getEmbedding(text: string): Promise<number[]> {
  const response = await client.send(new InvokeModelCommand({
    modelId: "amazon.titan-embed-text-v2:0",
    contentType: "application/json",
    accept: "application/json",
    body: JSON.stringify({ inputText: text }),
  }));
  
  const body = JSON.parse(new TextDecoder().decode(response.body));
  return body.embedding;
}
```

---

## 8. 实现计划

### Phase 1: 核心改进（本周）

| 任务 | 负责人 | 天数 | 依赖 |
|------|--------|------|------|
| Soul JSON结构 + DB迁移 | Dev | 0.5 | 无 |
| buildSoulPrompt() | Dev | 0.5 | 上一项 |
| 每日反思任务 | Dev | 1 | 无 |
| Agent Pool LRU | Dev | 0.5 | 无 |

### Phase 2: 增强功能（下周）

| 任务 | 负责人 | 天数 | 依赖 |
|------|--------|------|------|
| 每周个性进化 | Dev | 1 | Soul结构 |
| 增强工具集 | Dev | 1 | 无 |
| 记忆归档系统 | Dev | 0.5 | 无 |
| 成本追踪 | Dev | 0.5 | 无 |

### Phase 3: 向量检索（后续）

| 任务 | 负责人 | 天数 | 依赖 |
|------|--------|------|------|
| pgvector集成 | Dev | 1 | PostgreSQL迁移 |
| Embedding服务 | Dev | 0.5 | AWS配置 |
| 向量记忆检索 | Dev | 1 | 上两项 |

---

## 9. 总结

### 9.1 核心实现原则

1. **增量改进** - 基于现有框架，不重写
2. **模型分层** - 不同任务用不同模型
3. **成本优先** - 日常用便宜模型，关键时刻用好模型
4. **可扩展** - Agent Pool + 分层存储

### 9.2 与学术架构的关系

- **借鉴思想**：记忆流、反思、规划
- **务实实现**：用pi-agent-core + OpenClaw模式
- **成本可控**：Nova Lite为主，Sonnet为辅

### 9.3 预期效果

- Pet有可进化的个性
- Pet能反思和学习
- Pet记忆更丰富、更情感化
- 系统可扩展到10K+ Pet

---

*Intel 基于pi-agent-core的技术实现方案完成*
