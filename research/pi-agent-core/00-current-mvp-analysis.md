# Dev MVP代码分析

**Intel 研究文档** 🔍  
**日期**: 2026-03-01  
**来源**: `/home/ubuntu/.openclaw/workspace-dev/ai-pet-mvp/src/server/`

---

## 1. 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│  AI Pet MVP - 基于 pi-agent-core 的架构                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  pet-agent.ts                                        │    │
│  │  - 基于 @mariozechner/pi-agent-core 的 Agent 类      │    │
│  │  - 系统提示注入：worldview + personality + memory    │    │
│  │  - 工具定义：react_emotionally                       │    │
│  │  - LRU缓存：Map<petId, Agent>                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                         ↓                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  memory.ts                                           │    │
│  │  - buildMemoryContext(): 构建记忆上下文               │    │
│  │  - pet_social_memory 表：关系记忆                     │    │
│  │  - compressMemory(): 压缩日志为摘要                   │    │
│  │  - createConversationMemory(): AI生成对话记忆         │    │
│  └─────────────────────────────────────────────────────┘    │
│                         ↓                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  autonomous.ts                                       │    │
│  │  - decidePetAction(): 基于状态的决策树                │    │
│  │  - triggerAutonomousSocial(): Pet间自动对话          │    │
│  │  - pet_activity_log 表：行为日志                      │    │
│  │  - pet_state 表：位置/状态                            │    │
│  └─────────────────────────────────────────────────────┘    │
│                         ↓                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  worldview.ts                                        │    │
│  │  - 加载 data/worldview.json                          │    │
│  │  - 热重载（检测文件修改时间）                          │    │
│  │  - 版本控制                                           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 核心组件分析

### 2.1 pet-agent.ts - Agent系统

**技术栈**：
```typescript
import { Agent, type AgentTool } from "@mariozechner/pi-agent-core";
import { getModel, streamSimple } from "@mariozechner/pi-ai";
```

**Agent创建**：
```typescript
const agent = new Agent({
  initialState: {
    systemPrompt,          // 世界观 + 个性 + 记忆
    model,                 // nova-pro 或 claude-sonnet
    thinkingLevel: "off",  // 不使用CoT
    tools: buildTools(),   // 情感反应工具
  },
  streamFn: streamSimple,
});
```

**System Prompt结构**：
```
{worldview_prompt}           ← 世界观（worldview.json）

## 你的身份
你叫{pet_name}...

## 性格核心
- 温暖、细腻、情感丰富
- 俏皮中带着真诚...

## 表达风格
- emoji表达情绪
- 可爱声音表达

## 互动反应
- 被喂食/玩耍/休息时的反应

## 当前状态
- mood/energy/hunger/affection

{memory_context}              ← 动态注入的记忆上下文
```

**优势**：
- ✅ 基于成熟的pi-agent-core框架
- ✅ 支持Bedrock和Anthropic
- ✅ Agent实例缓存复用
- ✅ 对话历史自动管理

**局限**：
- ⚠️ Agent Map无限增长（无LRU淘汰）
- ⚠️ 系统提示每次chat都重建
- ⚠️ 只有一个工具（react_emotionally）

---

### 2.2 memory.ts - 记忆系统

**两层记忆设计**：

| 层级 | 存储 | 内容 | 更新频率 |
|------|------|------|----------|
| **长期记忆** | `pets.memory_summary` | 压缩的日常摘要 | 每10分钟压缩 |
| **社交记忆** | `pet_social_memory` | Per-Pet关系记忆 | 每次对话后 |

**记忆上下文构建**：
```typescript
function buildMemoryContext(petId: string): string {
  const parts = [];
  
  // 1. 长期记忆摘要
  if (pet.memory_summary) {
    parts.push(`## 你的记忆\n${pet.memory_summary}`);
  }
  
  // 2. 最近10条活动
  parts.push(`## 你最近做的事\n${activityLines}`);
  
  // 3. 最近5条社交
  parts.push(`## 你最近的社交\n${socialLines}`);
  
  // 4. 好友列表
  parts.push(`## 你的朋友\n${friendNames}`);
  
  // 5. 防幻觉规则
  parts.push(`## 重要规则\n你只能谈论真实发生过的事...`);
  
  return parts.join("\n\n");
}
```

**记忆压缩**：
```typescript
function compressMemory(petId: string) {
  // 获取最近50条活动
  // 统计活动类型
  // 生成自然语言摘要
  // 例如：[2026-03-01] 玩了3次玩具球，睡了2次觉，和豆豆聊了1次天
  // 保留最近500字符
}
```

**核心原则**：
```
老大指令：
- Pets可以有"善意谎言"（gentle expressions）
- Pets不能有"幻觉"（fabricated events）
- 记忆必须基于真实经历
- 记忆应该是情感化的，不是机械日志
```

**优势**：
- ✅ 双层记忆设计
- ✅ AI生成对话记忆（情感化）
- ✅ 防幻觉规则
- ✅ 自动压缩防止无限增长

**局限**：
- ⚠️ 无向量检索（只用最近N条）
- ⚠️ 社交记忆无重要性评分
- ⚠️ 500字符上限可能丢失关键记忆

---

### 2.3 autonomous.ts - 自主行为系统

**决策流程**：
```
每分钟执行一次 executeAutonomousBehavior()
    ↓
对每只Pet：
    ↓
decidePetAction(pet, state)
    ↓
优先级1: 紧急需求（能量<10睡觉，饥饿>90找食物）
    ↓
优先级2: 夜间行为（看星星/睡觉）
    ↓
优先级3: 情绪需求（mood<25时发呆/叹气）
    ↓
优先级4: 随机日常活动（加权随机）
    ↓
应用状态变化 + 更新位置 + 记录日志
```

**活动类型**：

| 位置 | 活动 | emoji | 状态影响 |
|------|------|-------|----------|
| Room | play_toy | 🎾 | mood+8, energy-5 |
| Room | explore | 🐾 | mood+3, energy-2 |
| Room | nap | 😴 | energy+10, mood+2 |
| Room | window | 🌤️ | mood+4 |
| Room | go_to_plaza | 🏞️ | mood+3, 切换场景 |
| Plaza | wander | 🚶 | mood+3, energy-2 |
| Plaza | fountain | 💦 | mood+6, energy-3 |
| Plaza | butterfly | 🦋 | mood+7, energy-4 |
| Plaza | social_wave | 👋 | mood+5 |
| Plaza | go_home | 🏠 | energy+3 |

**自主社交**：
```typescript
async function triggerAutonomousSocial() {
  // 找到广场上所有Pet（mood>30, energy>20）
  // 30%概率触发
  // 随机选两只Pet
  // 3轮对话：A发起 → B回复 → A反应
  // 双方创建对话记忆
  // 20%概率成为好友
}
```

**优势**：
- ✅ Pet完全自主（离线也能活动）
- ✅ 状态驱动决策
- ✅ 场景切换（房间↔广场）
- ✅ 自发社交和交友

**局限**：
- ⚠️ 决策是规则树，不是AI驱动
- ⚠️ 活动类型固定，无法学习新行为
- ⚠️ 社交概率硬编码

---

### 2.4 worldview.ts - 世界观系统

**设计模式**：
```
data/worldview.json（可热更新）
    ↓
worldview.ts（加载+缓存）
    ↓
pet-agent.ts（注入system prompt）
```

**世界观内容**（worldview.json）：
- `lore`: 背景故事（起源、居民、地点）
- `system_prompt_prefix`: 注入prompt的开头
- `rules`: 9条核心规则（防觉醒）

**热更新机制**：
```typescript
function getWorldview(): Worldview {
  const stat = fs.statSync(WORLDVIEW_PATH);
  if (stat.mtimeMs === cachedMtime) {
    return cachedWorldview; // 使用缓存
  }
  // 文件变了，重新加载
  cachedWorldview = JSON.parse(fs.readFileSync(...));
  cachedMtime = stat.mtimeMs;
  return cachedWorldview;
}
```

**优势**：
- ✅ 世界观与代码分离
- ✅ 热更新（无需重启）
- ✅ 版本控制

**局限**：
- ⚠️ 单一全局世界观，无法per-Pet定制

---

## 3. 数据库Schema

```sql
-- 核心表
pets (id, name, mood, energy, hunger, affection, memory_summary, ...)

-- 行为日志
pet_activity_log (id, pet_id, action_type, action_data, location, created_at)

-- Pet状态
pet_state (pet_id, location, position_x, position_y, current_action, last_autonomous_at)

-- 社交记忆
pet_social_memory (id, pet_id, target_pet_id, memory_type, memory_text, emotional_tag, created_at)

-- 好友关系
friends (pet_id, friend_pet_id)
```

---

## 4. 与OpenClaw模式的映射

| OpenClaw概念 | AI Pet MVP实现 | 差异 |
|--------------|---------------|------|
| `MEMORY.md` | `pets.memory_summary` + `pet_activity_log` | DB存储而非文件 |
| `SOUL.md` | `PET_SYSTEM_PROMPT` 常量 | 硬编码而非文件 |
| `HEARTBEAT.md` | `executeAutonomousBehavior()` 定时任务 | 代码而非配置 |
| `memory/*.md` | `pet_social_memory` 表 | DB而非文件 |

---

## 5. 优化建议

### 5.1 短期优化（基于现有框架）

| 问题 | 建议 | 复杂度 |
|------|------|--------|
| Agent Map无LRU | 添加最大容量+过期淘汰 | 低 |
| 记忆无向量检索 | 添加pgvector + embedding | 中 |
| 决策是规则树 | 改为AI辅助决策 | 中 |
| 无反思机制 | 添加每日反思任务 | 中 |

### 5.2 长期增强

| 功能 | 来自 | 复杂度 |
|------|------|--------|
| 记忆重要性评分 | Generative Agents | 中 |
| 层级记忆（Note/Episode） | HiMem | 高 |
| 规划系统 | Generative Agents | 高 |
| 社交网络健康监控 | 安全调研 | 中 |

---

## 6. 关键代码片段

### 6.1 创建Pet Agent

```typescript
// pet-agent.ts
export function getOrCreateAgent(petId: string): Agent {
  if (agents.has(petId)) return agents.get(petId)!;
  
  const pet = getPet(petId);
  const systemPrompt = buildSystemPrompt(pet);
  
  const agent = new Agent({
    initialState: {
      systemPrompt,
      model: getModel("amazon-bedrock", "us.amazon.nova-pro-v1:0"),
      tools: [reactEmotionallyTool],
    },
    streamFn: streamSimple,
  });
  
  // 加载最近对话历史
  const history = getRecentInteractions(petId, 20);
  for (const msg of history) {
    if (msg.role === "user") {
      agent.appendMessage({ role: "user", content: msg.content });
    }
  }
  
  agents.set(petId, agent);
  return agent;
}
```

### 6.2 构建记忆上下文

```typescript
// memory.ts
export function buildMemoryContext(petId: string): string {
  const pet = getPet(petId);
  const parts = [];
  
  // 长期记忆
  if (pet.memory_summary) {
    parts.push(`## 你的记忆\n${pet.memory_summary}`);
  }
  
  // 最近活动
  const activities = getRecentActivities(petId, 10);
  parts.push(`## 你最近做的事\n${activities.map(a => `- ${a.description}`).join("\n")}`);
  
  // 朋友列表
  const friends = getFriends(petId);
  parts.push(`## 你的朋友\n${friends.join("、")}`);
  
  // 防幻觉规则
  parts.push(`## 重要规则\n你只能谈论真实发生过的事。`);
  
  return parts.join("\n\n");
}
```

### 6.3 自主决策

```typescript
// autonomous.ts
function decidePetAction(pet: any, state: any): PetAction {
  // 优先级1: 紧急需求
  if (pet.energy < 10) return { type: "sleep", ... };
  if (pet.hunger > 90) return { type: "beg_food", ... };
  
  // 优先级2: 夜间行为
  if (isNight && pet.energy < 50) return { type: "sleep", ... };
  
  // 优先级3: 情绪需求
  if (pet.mood < 25) return { type: "mope", ... };
  
  // 优先级4: 随机活动
  const pool = buildActivityPool(pet, state);
  return pool[Math.floor(Math.random() * pool.length)];
}
```

---

## 7. 总结

**Dev已实现的关键能力**：
1. ✅ pi-agent-core集成
2. ✅ 世界观系统
3. ✅ 双层记忆（长期摘要+社交记忆）
4. ✅ 自主行为系统
5. ✅ Pet间自动社交
6. ✅ 防幻觉机制

**主要gap**：
1. ⚠️ 无向量检索记忆
2. ⚠️ 无反思系统
3. ⚠️ 无规划系统
4. ⚠️ 决策是规则树而非AI

**下一步**：
基于现有框架，参考OpenClaw模式设计Pet版MEMORY/SOUL/HEARTBEAT

---

*Intel Dev MVP代码分析完成*
