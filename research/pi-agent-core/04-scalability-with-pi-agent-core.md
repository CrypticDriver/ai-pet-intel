# 基于pi-agent-core的扩展性设计

**Intel 研究文档** 🔍  
**日期**: 2026-03-01  
**目标**: 支持10,000+ Pet的可扩展架构

---

## 1. 扩展性挑战

### 1.1 核心问题

| 问题 | 原因 | 当前MVP状态 |
|------|------|-------------|
| **Agent实例管理** | 每个Pet一个Agent | Map无限增长 |
| **LLM成本** | 每次对话调用API | 无成本控制 |
| **记忆检索** | 线性扫描 | 无向量索引 |
| **实时同步** | 所有Pet每分钟tick | 串行执行 |

### 1.2 规模目标

```
Phase 1 (MVP): 100 Pet
Phase 2 (Beta): 1,000 Pet  
Phase 3 (Launch): 10,000 Pet
Phase 4 (Scale): 100,000+ Pet
```

---

## 2. Agent Pool优化

### 2.1 LRU缓存设计

```typescript
// agent-pool-optimized.ts
import { LRUCache } from "lru-cache";
import { Agent } from "@mariozechner/pi-agent-core";

interface AgentEntry {
  agent: Agent;
  petId: string;
  lastPromptTime: number;
  conversationCount: number;
}

class OptimizedAgentPool {
  private cache: LRUCache<string, AgentEntry>;
  
  constructor() {
    this.cache = new LRUCache({
      max: 200,                    // 最多200个活跃Agent
      ttl: 30 * 60 * 1000,         // 30分钟TTL
      updateAgeOnGet: true,        // 访问时更新年龄
      dispose: (entry, key) => {   // 淘汰时的清理
        this.persistAgentState(entry);
      },
    });
  }
  
  async getOrCreate(petId: string): Promise<Agent> {
    let entry = this.cache.get(petId);
    
    if (!entry) {
      // 创建新Agent
      const agent = await createPetAgent(petId);
      entry = {
        agent,
        petId,
        lastPromptTime: Date.now(),
        conversationCount: 0,
      };
      this.cache.set(petId, entry);
    }
    
    entry.conversationCount++;
    entry.lastPromptTime = Date.now();
    
    return entry.agent;
  }
  
  private async persistAgentState(entry: AgentEntry): Promise<void> {
    // 保存对话历史到数据库
    const messages = entry.agent.getMessages();
    await saveConversationHistory(entry.petId, messages);
    console.log(`💾 Persisted state for pet ${entry.petId}`);
  }
  
  getStats(): { size: number; hitRate: number } {
    return {
      size: this.cache.size,
      hitRate: this.cache.size > 0 
        ? this.cache.calculatedSize / this.cache.maxSize 
        : 0,
    };
  }
}

export const agentPool = new OptimizedAgentPool();
```

### 2.2 按需唤醒策略

```typescript
// awakening-strategy.ts

interface AwakeningPolicy {
  shouldAwake(petId: string, trigger: string): Promise<boolean>;
  getPriority(petId: string): number;
}

class SmartAwakeningPolicy implements AwakeningPolicy {
  async shouldAwake(petId: string, trigger: string): Promise<boolean> {
    // 用户互动：总是唤醒
    if (trigger === "user_interaction") return true;
    
    // 心跳检查：只在需要时唤醒
    if (trigger === "heartbeat") {
      const pet = getPet(petId);
      const lastActive = new Date(pet.last_active_at);
      const hoursSinceActive = (Date.now() - lastActive.getTime()) / (1000 * 60 * 60);
      
      // 最近活跃的Pet更可能需要心跳
      return hoursSinceActive < 4;
    }
    
    // 自主社交：概率性唤醒
    if (trigger === "social") {
      const pet = getPet(petId);
      // 社交性高的Pet更可能自主社交
      const soul = JSON.parse(pet.soul_json) as PetSoul;
      return Math.random() < (soul.traits.sociability / 100);
    }
    
    return false;
  }
  
  getPriority(petId: string): number {
    const pet = getPet(petId);
    // 优先级因素：最近活跃度、付费用户、社交需求
    let priority = 0;
    
    // 最近活跃
    const hoursSinceActive = (Date.now() - new Date(pet.last_active_at).getTime()) / (1000 * 60 * 60);
    if (hoursSinceActive < 1) priority += 3;
    else if (hoursSinceActive < 4) priority += 2;
    else if (hoursSinceActive < 24) priority += 1;
    
    // 付费用户（未来）
    // if (pet.tier === "premium") priority += 2;
    
    // 社交需求（心情低或孤立）
    if (pet.mood < 30) priority += 1;
    
    return priority;
  }
}

export const awakeningPolicy = new SmartAwakeningPolicy();
```

---

## 3. Memory分层存储

### 3.1 三层架构

```
┌─────────────────────────────────────────────────────────────┐
│  Hot Layer (内存)                                           │
│  - Agent.messages (当前对话)                                │
│  - TTL: 30分钟                                              │
│  - 容量: 200 Pet                                            │
├─────────────────────────────────────────────────────────────┤
│  Warm Layer (SQLite/PostgreSQL)                            │
│  - pet_social_memory (关系记忆)                             │
│  - pet_activity_log (最近30天)                              │
│  - TTL: 30天                                                │
│  - 容量: 无限                                               │
├─────────────────────────────────────────────────────────────┤
│  Cold Layer (归档)                                          │
│  - 压缩的历史记忆                                            │
│  - JSON文件或S3                                             │
│  - TTL: 永久                                                │
│  - 容量: 无限                                               │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 记忆迁移流程

```typescript
// memory-migration.ts

// 每天凌晨执行
export async function migrateMemories(): Promise<void> {
  const db = getDb();
  
  // 1. Warm → Cold: 30天以上的记忆归档
  const oldMemories = db.prepare(`
    SELECT pet_id, memory_type, memory_text, emotional_tag, created_at
    FROM pet_social_memory
    WHERE created_at < datetime('now', '-30 days')
    ORDER BY pet_id, created_at
  `).all() as any[];
  
  // 按Pet分组归档
  const byPet = groupBy(oldMemories, "pet_id");
  
  for (const [petId, memories] of Object.entries(byPet)) {
    await archiveMemories(petId, memories);
  }
  
  // 删除已归档的
  db.prepare(`
    DELETE FROM pet_social_memory
    WHERE created_at < datetime('now', '-30 days')
  `).run();
  
  // 2. 压缩活动日志
  db.prepare(`
    DELETE FROM pet_activity_log
    WHERE created_at < datetime('now', '-7 days')
    AND action_type NOT IN ('became_friends', 'first_meet', 'social_chat_init')
  `).run();
  
  console.log(`📦 Memory migration complete: ${oldMemories.length} memories archived`);
}

async function archiveMemories(petId: string, memories: any[]): Promise<void> {
  const month = new Date().toISOString().slice(0, 7);
  const archivePath = `data/archives/${petId}`;
  
  // 创建目录
  await fs.mkdir(archivePath, { recursive: true });
  
  // 写入归档文件
  const archiveFile = `${archivePath}/${month}.json`;
  let existing: any[] = [];
  
  try {
    existing = JSON.parse(await fs.readFile(archiveFile, "utf-8"));
  } catch {}
  
  await fs.writeFile(archiveFile, JSON.stringify([...existing, ...memories], null, 2));
}
```

### 3.3 记忆检索优化

```typescript
// memory-retrieval.ts

interface MemoryQuery {
  petId: string;
  query: string;
  limit: number;
  includeArchive: boolean;
}

export async function retrieveMemories(params: MemoryQuery): Promise<any[]> {
  const { petId, query, limit, includeArchive } = params;
  const results: any[] = [];
  
  // 1. 从Warm层检索（最近30天）
  const warmResults = searchWarmMemories(petId, query, limit);
  results.push(...warmResults);
  
  // 2. 如果不够，从Cold层检索
  if (includeArchive && results.length < limit) {
    const coldResults = await searchColdMemories(petId, query, limit - results.length);
    results.push(...coldResults);
  }
  
  // 3. 按相关性排序（简单实现：关键词匹配）
  return results
    .map(r => ({
      ...r,
      score: calculateRelevance(r.memory_text || r.text, query),
    }))
    .sort((a, b) => b.score - a.score)
    .slice(0, limit);
}

function searchWarmMemories(petId: string, query: string, limit: number): any[] {
  const db = getDb();
  // 简单的LIKE搜索（未来改为向量搜索）
  return db.prepare(`
    SELECT * FROM pet_social_memory
    WHERE pet_id = ? AND memory_text LIKE ?
    ORDER BY importance DESC, created_at DESC
    LIMIT ?
  `).all(petId, `%${query}%`, limit) as any[];
}

async function searchColdMemories(petId: string, query: string, limit: number): Promise<any[]> {
  const archivePath = `data/archives/${petId}`;
  const results: any[] = [];
  
  try {
    const files = await fs.readdir(archivePath);
    
    for (const file of files.reverse()) { // 从最近的开始
      const content = JSON.parse(await fs.readFile(`${archivePath}/${file}`, "utf-8"));
      
      for (const memory of content) {
        if (memory.memory_text?.includes(query)) {
          results.push(memory);
          if (results.length >= limit) break;
        }
      }
      
      if (results.length >= limit) break;
    }
  } catch {}
  
  return results;
}
```

---

## 4. LLM成本控制

### 4.1 模型选择策略

```typescript
// model-selection.ts

interface ModelConfig {
  id: string;
  provider: string;
  inputCost: number;   // $/1M tokens
  outputCost: number;  // $/1M tokens
  maxTokens: number;
  quality: "low" | "medium" | "high";
}

const MODELS: Record<string, ModelConfig> = {
  "nova-lite": {
    id: "us.amazon.nova-lite-v1:0",
    provider: "amazon-bedrock",
    inputCost: 0.06,
    outputCost: 0.24,
    maxTokens: 5000,
    quality: "medium",
  },
  "nova-pro": {
    id: "us.amazon.nova-pro-v1:0",
    provider: "amazon-bedrock",
    inputCost: 0.8,
    outputCost: 3.2,
    maxTokens: 5000,
    quality: "high",
  },
  "claude-sonnet": {
    id: "us.anthropic.claude-sonnet-4-5-20260101:0",
    provider: "amazon-bedrock",
    inputCost: 3,
    outputCost: 15,
    maxTokens: 8000,
    quality: "high",
  },
};

export function selectModel(context: {
  task: string;
  userTier: "free" | "premium";
  importance: "low" | "medium" | "high";
}): ModelConfig {
  // 免费用户：总是用最便宜的
  if (context.userTier === "free") {
    return MODELS["nova-lite"];
  }
  
  // 付费用户：根据任务重要性选择
  switch (context.importance) {
    case "low":
      return MODELS["nova-lite"];
    case "medium":
      return MODELS["nova-pro"];
    case "high":
      return MODELS["claude-sonnet"];
    default:
      return MODELS["nova-lite"];
  }
}
```

### 4.2 使用量追踪

```typescript
// usage-tracking.ts

interface UsageRecord {
  petId: string;
  model: string;
  inputTokens: number;
  outputTokens: number;
  cost: number;
  timestamp: Date;
}

class UsageTracker {
  private records: UsageRecord[] = [];
  private dailyBudget = 100; // $100/day
  
  async track(record: UsageRecord): Promise<void> {
    this.records.push(record);
    
    // 每100条写入数据库
    if (this.records.length >= 100) {
      await this.flush();
    }
    
    // 检查预算
    await this.checkBudget();
  }
  
  async getTodayCost(): Promise<number> {
    const today = new Date().toISOString().slice(0, 10);
    const db = getDb();
    
    const result = db.prepare(`
      SELECT SUM(cost) as total FROM usage_log
      WHERE date(timestamp) = ?
    `).get(today) as any;
    
    const pendingCost = this.records.reduce((sum, r) => sum + r.cost, 0);
    
    return (result?.total || 0) + pendingCost;
  }
  
  private async checkBudget(): Promise<void> {
    const todayCost = await this.getTodayCost();
    
    if (todayCost > this.dailyBudget * 0.8) {
      console.warn(`⚠️ 80% of daily budget used: $${todayCost.toFixed(2)}`);
    }
    
    if (todayCost > this.dailyBudget) {
      console.error(`🚨 Daily budget exceeded: $${todayCost.toFixed(2)}`);
      // 触发降级：强制使用最便宜的模型
      setForceEconomyMode(true);
    }
  }
  
  private async flush(): Promise<void> {
    const db = getDb();
    const stmt = db.prepare(`
      INSERT INTO usage_log (pet_id, model, input_tokens, output_tokens, cost, timestamp)
      VALUES (?, ?, ?, ?, ?, ?)
    `);
    
    for (const r of this.records) {
      stmt.run(r.petId, r.model, r.inputTokens, r.outputTokens, r.cost, r.timestamp.toISOString());
    }
    
    this.records = [];
  }
}

export const usageTracker = new UsageTracker();
```

### 4.3 响应缓存

```typescript
// response-cache.ts
import { LRUCache } from "lru-cache";
import crypto from "crypto";

class ResponseCache {
  private cache: LRUCache<string, string>;
  
  constructor() {
    this.cache = new LRUCache({
      max: 1000,           // 最多1000条缓存
      ttl: 5 * 60 * 1000,  // 5分钟TTL
    });
  }
  
  getKey(systemPrompt: string, userMessage: string): string {
    // 只对系统提示的关键部分和用户消息生成hash
    const content = `${systemPrompt.slice(0, 500)}|${userMessage}`;
    return crypto.createHash("md5").update(content).digest("hex");
  }
  
  get(systemPrompt: string, userMessage: string): string | null {
    const key = this.getKey(systemPrompt, userMessage);
    return this.cache.get(key) || null;
  }
  
  set(systemPrompt: string, userMessage: string, response: string): void {
    const key = this.getKey(systemPrompt, userMessage);
    this.cache.set(key, response);
  }
}

export const responseCache = new ResponseCache();

// 在chat函数中使用
export async function chatWithCache(petId: string, message: string): Promise<{ text: string; cached: boolean }> {
  const systemPrompt = buildSystemPrompt(getPet(petId));
  
  // 检查缓存
  const cached = responseCache.get(systemPrompt, message);
  if (cached) {
    return { text: cached, cached: true };
  }
  
  // 调用LLM
  const result = await chat(petId, message);
  
  // 存入缓存（只缓存短消息）
  if (message.length < 100) {
    responseCache.set(systemPrompt, message, result.text);
  }
  
  return { text: result.text, cached: false };
}
```

---

## 5. 并发处理

### 5.1 自主行为批处理

```typescript
// batch-autonomous.ts

// 替代原来的串行执行
export async function executeAutonomousBehaviorBatch(): Promise<void> {
  const db = getDb();
  const pets = db.prepare("SELECT id FROM pets").all() as any[];
  
  // 分批处理
  const batchSize = 20;
  const batches = chunk(pets, batchSize);
  
  for (const batch of batches) {
    // 并发处理一批
    await Promise.all(
      batch.map(pet => 
        executeAutonomousBehaviorForPet(pet.id).catch(err => {
          console.error(`Autonomous error for ${pet.id}:`, err);
        })
      )
    );
    
    // 批间休息，避免过载
    await sleep(100);
  }
}

async function executeAutonomousBehaviorForPet(petId: string): Promise<void> {
  // 检查是否需要唤醒
  if (!await awakeningPolicy.shouldAwake(petId, "heartbeat")) {
    return;
  }
  
  // 原有的自主行为逻辑
  const pet = getPet(petId);
  const state = getPetState(petId);
  const action = decidePetAction(pet, state);
  
  // 应用状态变化
  applyAction(petId, action);
}

function chunk<T>(array: T[], size: number): T[][] {
  const result: T[][] = [];
  for (let i = 0; i < array.length; i += size) {
    result.push(array.slice(i, i + size));
  }
  return result;
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

### 5.2 社交调度器

```typescript
// social-scheduler.ts

interface SocialIntent {
  initiator: string;
  target: string;
  priority: number;
  timestamp: number;
}

class SocialScheduler {
  private queue: SocialIntent[] = [];
  private processing = false;
  private maxConcurrent = 3;  // 同时最多3对Pet在聊天
  private activeSessions = new Set<string>();
  
  enqueue(initiator: string, target: string, priority: number = 1): void {
    // 检查Pet是否已在聊天
    if (this.activeSessions.has(initiator) || this.activeSessions.has(target)) {
      return;
    }
    
    this.queue.push({
      initiator,
      target,
      priority,
      timestamp: Date.now(),
    });
    
    // 按优先级排序
    this.queue.sort((a, b) => b.priority - a.priority);
    
    this.processQueue();
  }
  
  private async processQueue(): Promise<void> {
    if (this.processing) return;
    this.processing = true;
    
    while (this.queue.length > 0 && this.activeSessions.size < this.maxConcurrent * 2) {
      const intent = this.queue.shift()!;
      
      // 再次检查可用性
      if (this.activeSessions.has(intent.initiator) || this.activeSessions.has(intent.target)) {
        continue;
      }
      
      // 标记为活跃
      this.activeSessions.add(intent.initiator);
      this.activeSessions.add(intent.target);
      
      // 执行社交（不await，让多对同时进行）
      this.executeSocial(intent).finally(() => {
        this.activeSessions.delete(intent.initiator);
        this.activeSessions.delete(intent.target);
      });
    }
    
    this.processing = false;
  }
  
  private async executeSocial(intent: SocialIntent): Promise<void> {
    try {
      await triggerAutonomousSocialBetween(intent.initiator, intent.target);
    } catch (err) {
      console.error(`Social error between ${intent.initiator} and ${intent.target}:`, err);
    }
  }
}

export const socialScheduler = new SocialScheduler();
```

---

## 6. 监控与告警

```typescript
// monitoring.ts

interface SystemMetrics {
  activeAgents: number;
  queuedSocials: number;
  todayCost: number;
  avgResponseTime: number;
  errorRate: number;
}

class Monitor {
  private metrics: SystemMetrics = {
    activeAgents: 0,
    queuedSocials: 0,
    todayCost: 0,
    avgResponseTime: 0,
    errorRate: 0,
  };
  
  update(partial: Partial<SystemMetrics>): void {
    Object.assign(this.metrics, partial);
  }
  
  check(): void {
    // Agent池过载
    if (this.metrics.activeAgents > 180) {
      console.warn(`⚠️ Agent pool near capacity: ${this.metrics.activeAgents}/200`);
    }
    
    // 成本超标
    if (this.metrics.todayCost > 80) {
      console.warn(`⚠️ Daily cost high: $${this.metrics.todayCost.toFixed(2)}`);
    }
    
    // 响应时间过长
    if (this.metrics.avgResponseTime > 3000) {
      console.warn(`⚠️ Response time high: ${this.metrics.avgResponseTime}ms`);
    }
    
    // 错误率过高
    if (this.metrics.errorRate > 0.05) {
      console.error(`🚨 Error rate high: ${(this.metrics.errorRate * 100).toFixed(1)}%`);
    }
  }
  
  report(): SystemMetrics {
    return { ...this.metrics };
  }
}

export const monitor = new Monitor();

// 定期检查
setInterval(() => monitor.check(), 60 * 1000);
```

---

## 7. 扩展路线图

### Phase 1: 100 Pet (当前)

- ✅ 基础Agent系统
- ✅ SQLite存储
- ✅ 单实例部署

### Phase 2: 1,000 Pet

- [ ] Agent Pool LRU
- [ ] 记忆分层存储
- [ ] 成本追踪
- [ ] 批处理自主行为

### Phase 3: 10,000 Pet

- [ ] PostgreSQL迁移
- [ ] pgvector向量检索
- [ ] 社交调度器
- [ ] 响应缓存
- [ ] 多实例部署

### Phase 4: 100,000+ Pet

- [ ] Redis缓存层
- [ ] 分片数据库
- [ ] Kubernetes部署
- [ ] 专用GPU推理

---

## 8. 总结

### 8.1 关键优化点

| 优化 | 效果 | 复杂度 |
|------|------|--------|
| Agent Pool LRU | 内存可控 | 低 |
| 模型分层选择 | 成本-30% | 低 |
| 响应缓存 | 成本-20% | 低 |
| 记忆分层 | 存储可控 | 中 |
| 批处理 | 吞吐量+200% | 中 |
| 向量检索 | 检索准确性+50% | 高 |

### 8.2 成本预估

| 规模 | Pet数 | 月成本 | 每Pet成本 |
|------|-------|--------|-----------|
| MVP | 100 | $50 | $0.50 |
| Beta | 1,000 | $300 | $0.30 |
| Launch | 10,000 | $1,500 | $0.15 |
| Scale | 100,000 | $10,000 | $0.10 |

### 8.3 下一步

1. Dev实现Agent Pool LRU
2. Dev实现成本追踪
3. Intel继续监控社区最佳实践

---

*Intel 基于pi-agent-core的扩展性设计完成*
