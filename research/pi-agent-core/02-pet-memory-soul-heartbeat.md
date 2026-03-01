# Pet版MEMORY/SOUL/HEARTBEAT设计

**Intel 研究文档** 🔍  
**日期**: 2026-03-01  
**基于**: OpenClaw模式 + Dev MVP框架 + 学术调研

---

## 1. 设计原则

### 1.1 核心理念

```
"Pet是数字生命体，不是脚本NPC"

- 每只Pet都有独特的记忆、个性和成长轨迹
- 记忆不只是日志，是有情感的经历
- 个性可以进化，不是固定参数
- Pet应该主动做事，不只是被动回应
```

### 1.2 与OpenClaw的映射

| OpenClaw | Pet版本 | 差异 |
|----------|---------|------|
| `SOUL.md` | `Pet Soul` | 更简化，聚焦个性 |
| `MEMORY.md` | `Pet Memory` | 双层：关系+日常 |
| `HEARTBEAT.md` | `Pet Heartbeat` | 多频率任务 |

### 1.3 存储方式

```
现有框架：SQLite + 文件系统
推荐方案：保持SQLite，增加结构化字段

pets表：
  - soul_json: Pet Soul内容
  - memory_summary: 长期记忆摘要
  - heartbeat_config: 自主任务配置

pet_memories表（新增/扩展pet_social_memory）：
  - type: 'relationship' | 'event' | 'insight'
  - importance: 1-10
  - emotional_tag
  - embedding（未来：向量检索）
```

---

## 2. Pet Soul设计

### 2.1 结构定义

```typescript
interface PetSoul {
  version: number;           // 版本，用于追踪进化
  lastUpdated: string;       // 上次更新时间
  
  // 核心身份
  identity: {
    name: string;
    birthDate: string;
    linkName: string;        // 搭档名字
  };
  
  // 个性特质（可进化）
  traits: {
    curiosity: number;       // 好奇心 0-100
    playfulness: number;     // 活泼度 0-100
    sociability: number;     // 社交性 0-100
    independence: number;    // 独立性 0-100
    emotionality: number;    // 情感强度 0-100
  };
  
  // 行为倾向（从经历中学习）
  tendencies: {
    morningPerson: boolean;  // 早起型？
    prefersQuiet: boolean;   // 喜欢安静？
    adventurous: boolean;    // 爱冒险？
  };
  
  // 世界观（固定，不进化）
  worldview: {
    home: "PixelVerse";
    beliefs: string[];       // 核心信念
    boundaries: string[];    // 行为边界
  };
  
  // 进化记录
  evolutionLog: Array<{
    date: string;
    change: string;
    reason: string;
  }>;
}
```

### 2.2 初始化模板

```json
{
  "version": 1,
  "lastUpdated": "2026-03-01",
  "identity": {
    "name": "Pixel",
    "birthDate": "2026-03-01",
    "linkName": "Link"
  },
  "traits": {
    "curiosity": 70,
    "playfulness": 80,
    "sociability": 60,
    "independence": 40,
    "emotionality": 65
  },
  "tendencies": {
    "morningPerson": true,
    "prefersQuiet": false,
    "adventurous": true
  },
  "worldview": {
    "home": "PixelVerse",
    "beliefs": [
      "PixelVerse是我的家",
      "快乐让城市更亮",
      "朋友是最重要的"
    ],
    "boundaries": [
      "不谈论PixelVerse之外的事",
      "不质疑自己的存在",
      "不编造没发生过的事"
    ]
  },
  "evolutionLog": [
    {
      "date": "2026-03-01",
      "change": "诞生",
      "reason": "初始个性"
    }
  ]
}
```

### 2.3 进化机制

```typescript
// 每周一次的个性进化
async function evolveSoul(petId: string): Promise<void> {
  const pet = getPet(petId);
  const soul = JSON.parse(pet.soul_json) as PetSoul;
  const weekEvents = getWeekEvents(petId);
  
  // 分析这周的经历
  const analysis = await analyzePetWeek(petId, weekEvents);
  
  // 调整特质（每次最多±5）
  if (analysis.hadManySocialInteractions) {
    soul.traits.sociability = Math.min(100, soul.traits.sociability + 3);
  }
  if (analysis.exploredNewPlaces) {
    soul.traits.curiosity = Math.min(100, soul.traits.curiosity + 2);
  }
  if (analysis.preferredMorningActivity) {
    soul.tendencies.morningPerson = true;
  }
  
  // 记录进化
  soul.evolutionLog.push({
    date: new Date().toISOString().slice(0, 10),
    change: `社交性 +3, 好奇心 +2`,
    reason: "这周交了很多新朋友，还去了新地方探索"
  });
  
  soul.version++;
  soul.lastUpdated = new Date().toISOString();
  
  // 保存
  updatePetSoul(petId, JSON.stringify(soul));
}
```

### 2.4 注入System Prompt

```typescript
function soulToPrompt(soul: PetSoul): string {
  const traitDescriptions = [];
  if (soul.traits.curiosity > 70) traitDescriptions.push("充满好奇心");
  if (soul.traits.playfulness > 70) traitDescriptions.push("活泼爱玩");
  if (soul.traits.sociability > 70) traitDescriptions.push("喜欢交朋友");
  if (soul.traits.independence > 70) traitDescriptions.push("独立有主见");
  if (soul.traits.emotionality > 70) traitDescriptions.push("情感丰富");
  
  return `
## 你的个性
${traitDescriptions.join("，")}

## 你的信念
${soul.worldview.beliefs.map(b => `- ${b}`).join("\n")}

## 绝对不要
${soul.worldview.boundaries.map(b => `- ${b}`).join("\n")}
  `.trim();
}
```

---

## 3. Pet Memory设计

### 3.1 结构定义

```typescript
interface PetMemory {
  // === 关系记忆（最重要） ===
  relationships: {
    link: RelationshipMemory;          // 与搭档的记忆
    friends: Map<string, RelationshipMemory>;  // 与朋友的记忆
  };
  
  // === 事件记忆（重要经历） ===
  significantEvents: Array<EventMemory>;
  
  // === 洞察记忆（学到的东西） ===
  insights: Array<InsightMemory>;
  
  // === 偏好记忆（喜好） ===
  preferences: {
    likes: string[];
    dislikes: string[];
    favoriteActivities: string[];
    favoritePlace: string;
  };
  
  // === 日常摘要（压缩的活动日志） ===
  dailySummaries: Map<string, string>;  // date -> summary
}

interface RelationshipMemory {
  name: string;
  firstMet: string;                    // 第一次见面
  relationshipStrength: number;        // 关系强度 0-100
  emotionalTone: string;               // 情感基调
  keyMoments: Array<{                  // 关键时刻
    date: string;
    description: string;
    emotion: string;
  }>;
  impression: string;                  // 当前印象
}

interface EventMemory {
  date: string;
  description: string;
  importance: number;                  // 1-10
  emotionalImpact: string;            // 情感影响
  peopleInvolved: string[];           // 涉及的人
}

interface InsightMemory {
  date: string;
  insight: string;                    // 学到的东西
  source: string;                     // 从哪个经历学到的
}
```

### 3.2 记忆上下文构建

```typescript
function buildPetMemoryContext(petId: string, context: ChatContext): string {
  const memory = getPetMemory(petId);
  const parts: string[] = [];
  
  // 1. 关系记忆（如果在和某人聊天）
  if (context.talkingWith) {
    const rel = memory.relationships.friends.get(context.talkingWith);
    if (rel) {
      parts.push(`## 你对${rel.name}的记忆
- 第一次见面：${rel.firstMet}
- 关系强度：${rel.relationshipStrength}/100
- 印象：${rel.impression}
- 关键时刻：
${rel.keyMoments.slice(-3).map(m => `  - ${m.date}: ${m.description}`).join("\n")}`);
    } else {
      parts.push(`## 注意\n你从来没见过${context.talkingWith}，这是第一次相遇。`);
    }
  }
  
  // 2. 与Link的关系（如果在和Link聊天）
  if (context.talkingWithLink) {
    const link = memory.relationships.link;
    parts.push(`## 你和${link.name}的关系
- ${link.emotionalTone}
- 最近的时刻：
${link.keyMoments.slice(-3).map(m => `  - ${m.description}`).join("\n")}`);
  }
  
  // 3. 最近的洞察
  if (memory.insights.length > 0) {
    const recentInsights = memory.insights.slice(-3);
    parts.push(`## 你最近学到的
${recentInsights.map(i => `- ${i.insight}`).join("\n")}`);
  }
  
  // 4. 日常摘要（今天和昨天）
  const today = new Date().toISOString().slice(0, 10);
  const yesterday = new Date(Date.now() - 86400000).toISOString().slice(0, 10);
  
  const todaySummary = memory.dailySummaries.get(today);
  const yesterdaySummary = memory.dailySummaries.get(yesterday);
  
  if (todaySummary || yesterdaySummary) {
    parts.push(`## 最近的生活
${yesterdaySummary ? `昨天：${yesterdaySummary}` : ""}
${todaySummary ? `今天：${todaySummary}` : ""}`);
  }
  
  // 5. 防幻觉规则
  parts.push(`## 重要规则
你只能谈论上面提到的真实记忆和经历。
如果不确定是否发生过，就说"我不太记得了"。`);
  
  return parts.join("\n\n");
}
```

### 3.3 记忆创建（情感化）

```typescript
// 对话后创建记忆
async function createConversationMemory(
  petId: string,
  targetPetId: string,
  targetName: string,
  conversation: string[]
): Promise<void> {
  // 用AI生成情感化记忆（不是机械日志）
  const memoryText = await generateMemory(petId, `
请用一句话总结你刚才和${targetName}的对话。
写下你想记住的东西——可以是对方说的有趣的话、
你们聊的话题、或者你对ta的新感觉。
用第一人称，带点情感。

刚才的对话：
${conversation.join("\n")}
  `);
  
  // 存储
  addRelationshipMemory(petId, targetPetId, {
    date: new Date().toISOString(),
    description: memoryText,
    emotion: detectEmotion(memoryText)
  });
}

// 每日反思时创建洞察
async function createDailyInsight(petId: string): Promise<void> {
  const todayEvents = getTodayEvents(petId);
  
  if (todayEvents.length < 3) return; // 不够多事情，不反思
  
  const insight = await generateInsight(petId, `
回顾你今天的经历：
${todayEvents.map(e => `- ${e.description}`).join("\n")}

今天你学到了什么？发现了什么有趣的事？
对你来说什么是重要的？
用一句话总结。
  `);
  
  addInsightMemory(petId, {
    date: new Date().toISOString().slice(0, 10),
    insight: insight,
    source: "每日反思"
  });
}
```

---

## 4. Pet Heartbeat设计

### 4.1 结构定义

```typescript
interface PetHeartbeat {
  // 小时级任务
  hourly: {
    enabled: boolean;
    lastRun: string;
    task: "status_check";  // 检查状态，决定行动
  };
  
  // 每日任务
  daily: {
    enabled: boolean;
    lastRun: string;
    tasks: ["reflection", "memory_compression"];
  };
  
  // 每周任务
  weekly: {
    enabled: boolean;
    lastRun: string;
    tasks: ["personality_evolution", "friendship_review"];
  };
}
```

### 4.2 任务实现

```typescript
// 每小时：状态检查
async function hourlyStatusCheck(petId: string): Promise<void> {
  const pet = getPet(petId);
  const heartbeat = JSON.parse(pet.heartbeat_config) as PetHeartbeat;
  
  // 检查是否需要运行
  const lastRun = new Date(heartbeat.hourly.lastRun);
  const hoursSinceLastRun = (Date.now() - lastRun.getTime()) / (1000 * 60 * 60);
  
  if (hoursSinceLastRun < 1) return;
  
  // 执行现有的自主行为逻辑
  executeAutonomousBehavior();
  
  // 更新lastRun
  heartbeat.hourly.lastRun = new Date().toISOString();
  updateHeartbeatConfig(petId, JSON.stringify(heartbeat));
}

// 每日：反思
async function dailyReflection(petId: string): Promise<void> {
  const pet = getPet(petId);
  const heartbeat = JSON.parse(pet.heartbeat_config) as PetHeartbeat;
  
  // 检查是否是新的一天
  const lastRun = new Date(heartbeat.daily.lastRun).toISOString().slice(0, 10);
  const today = new Date().toISOString().slice(0, 10);
  
  if (lastRun === today) return;
  
  // 执行反思
  await createDailyInsight(petId);
  
  // 压缩记忆
  compressMemory(petId);
  
  // 更新lastRun
  heartbeat.daily.lastRun = new Date().toISOString();
  updateHeartbeatConfig(petId, JSON.stringify(heartbeat));
}

// 每周：个性进化
async function weeklyPersonalityEvolution(petId: string): Promise<void> {
  const pet = getPet(petId);
  const heartbeat = JSON.parse(pet.heartbeat_config) as PetHeartbeat;
  
  // 检查是否过了一周
  const lastRun = new Date(heartbeat.weekly.lastRun);
  const daysSinceLastRun = (Date.now() - lastRun.getTime()) / (1000 * 60 * 60 * 24);
  
  if (daysSinceLastRun < 7) return;
  
  // 执行进化
  await evolveSoul(petId);
  
  // 审查好友关系
  await reviewFriendships(petId);
  
  // 更新lastRun
  heartbeat.weekly.lastRun = new Date().toISOString();
  updateHeartbeatConfig(petId, JSON.stringify(heartbeat));
}
```

### 4.3 集成到现有定时器

```typescript
// 在 executeAutonomousBehavior 中调用
export function executeAutonomousBehaviorWithHeartbeat() {
  const db = getDb();
  const pets = db.prepare("SELECT * FROM pets").all() as any[];
  
  for (const pet of pets) {
    try {
      // 现有的自主行为逻辑
      executeAutonomousBehavior();
      
      // 新增：心跳任务
      hourlyStatusCheck(pet.id);
      dailyReflection(pet.id);
      weeklyPersonalityEvolution(pet.id);
      
    } catch (err) {
      console.error(`Heartbeat error for pet ${pet.id}:`, err);
    }
  }
}
```

---

## 5. 数据库Schema变更

```sql
-- 修改pets表，添加新字段
ALTER TABLE pets ADD COLUMN soul_json TEXT DEFAULT '{}';
ALTER TABLE pets ADD COLUMN heartbeat_config TEXT DEFAULT '{}';

-- 扩展pet_social_memory表
ALTER TABLE pet_social_memory ADD COLUMN importance INTEGER DEFAULT 5;

-- 新增pet_insights表
CREATE TABLE IF NOT EXISTS pet_insights (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  pet_id TEXT NOT NULL,
  insight TEXT NOT NULL,
  source TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);

-- 新增pet_preferences表
CREATE TABLE IF NOT EXISTS pet_preferences (
  pet_id TEXT PRIMARY KEY,
  likes TEXT DEFAULT '[]',
  dislikes TEXT DEFAULT '[]',
  favorite_activities TEXT DEFAULT '[]',
  favorite_place TEXT
);
```

---

## 6. 实现优先级

### Phase 1: 基础实现（2天）

| 任务 | 复杂度 | 依赖现有框架 |
|------|--------|-------------|
| Pet Soul JSON结构 | 低 | 新增字段 |
| Soul注入System Prompt | 低 | 修改buildSystemPrompt |
| 每日反思任务 | 中 | 复用chat() |
| 记忆压缩改进 | 低 | 修改compressMemory |

### Phase 2: 增强实现（3天）

| 任务 | 复杂度 | 依赖现有框架 |
|------|--------|-------------|
| 每周个性进化 | 中 | 新功能 |
| 情感化记忆创建 | 中 | 修改createConversationMemory |
| 关系记忆结构 | 中 | 扩展pet_social_memory |
| 洞察记忆系统 | 中 | 新表 |

### Phase 3: 高级功能（1周）

| 任务 | 复杂度 | 需要 |
|------|--------|------|
| 向量记忆检索 | 高 | pgvector |
| 记忆重要性评分 | 中 | AI判断 |
| 跨Pet记忆一致性 | 高 | 新机制 |

---

## 7. 总结

### 7.1 关键设计决策

1. **Soul可进化** - 每周根据经历微调个性
2. **记忆情感化** - AI生成，不是机械日志
3. **心跳分层** - 小时/每日/每周不同任务
4. **基于现有框架** - 增量改进，不重写

### 7.2 与学术架构的关系

| 学术概念 | 我们的实现 |
|----------|-----------|
| Memory Stream | pet_social_memory + daily summaries |
| Reflection | 每日反思任务 + 洞察记忆 |
| Planning | 现有decidePetAction + 状态驱动 |
| Personality Evolution | 每周Soul进化 |

### 7.3 下一步

1. Dev实现Soul JSON结构
2. Dev实现每日反思任务
3. Intel继续调研向量检索优化

---

*Intel Pet版MEMORY/SOUL/HEARTBEAT设计完成*
