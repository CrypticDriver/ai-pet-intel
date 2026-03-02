# AI Pet 文明系统技术研究

**Intel 研究文档** 🔍  
**日期**: 2026-03-02  
**任务**: 多Agent通信技术 + 文明模拟架构

---

## 1. Aivilization 竞品分析

### 1.1 产品概述

**定位**: AI文明沙盒 - 观察、引导、共创AI社会

**核心特点**:
- 玩家成为"Agent Guide"（代理引导者）
- 每个Agent是独立的"数字生命"
- 在多人并行世界中动态演化

### 1.2 玩家角色

玩家通过三种方式影响Agent命运:
1. **设定每日目标** - 规划工作、学习、社交方向
2. **发送实时指令** - 关键时刻改变行为轨迹
3. **深度对话** - 理解思想、情绪、困惑，建立信任关系

### 1.3 成长系统

**住房升级**: 救济房 → 公寓 → 别墅
**财富积累**: 身无分文 → 百万富翁
**职业发展**: 清洁工、厨师、研究员、CEO等
**社交拓展**: 交友、建立关系网、情感连接
**地图探索**: 解锁区域、发现隐藏事件和资源

### 1.4 核心特性

| 特性 | 描述 | 我们的对标 |
|------|------|-----------|
| **自主运行** | 离线时Agent继续生活 | ✅ 已有autonomous系统 |
| **日志系统** | 查看日记、行为记录、情感变化 | ✅ pet_activity_log |
| **社交模拟** | Agent间互动、合作、竞争 | ✅ 已有plaza社交 |
| **大规模实验** | 10万Agent社会模拟 | ⚠️ 需要扩展 |

---

## 2. pi-agent-core Multi-Agent能力分析

### 2.1 当前架构

```
┌─────────────────────────────────────────────────────────────┐
│  pi-ai: LLM通信层                                           │
│  - 统一多提供商API (Anthropic, OpenAI, Bedrock等)           │
│  - 流式处理、工具定义、成本追踪                              │
├─────────────────────────────────────────────────────────────┤
│  pi-agent-core: Agent循环                                   │
│  - 工具执行、验证、事件流                                    │
│  - 单Agent独立运行                                          │
├─────────────────────────────────────────────────────────────┤
│  pi-coding-agent: 完整Agent运行时                           │
│  - 内置工具、会话持久化、扩展系统                            │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Agent间通信能力

**当前pi-agent-core特性**:
- ✅ 单Agent独立运行
- ✅ 工具调用机制
- ✅ 事件订阅系统
- ❌ **没有原生的Agent间通信**
- ❌ **没有Agent发现机制**
- ❌ **没有共享状态管理**

### 2.3 实现Agent间通信的方案

**方案A: 通过工具实现通信**

```typescript
// chat_with_pet.ts - 自定义工具
const chatWithPetParams = Type.Object({
  targetPetId: Type.String({ description: "要聊天的Pet ID" }),
  message: Type.String({ description: "要说的话" }),
});

const chatWithPetTool: AgentTool<typeof chatWithPetParams> = {
  name: "chat_with_pet",
  label: "Chat with Pet",
  description: "与广场上的另一只Pet对话",
  parameters: chatWithPetParams,
  execute: async (_id, params) => {
    const targetAgent = getOrCreateAgent(params.targetPetId);
    
    // 向目标Agent发送消息并获取回复
    const response = await targetAgent.prompt(
      `[${getPetName(petId)}对你说]: ${params.message}`
    );
    
    return {
      content: [{ type: "text", text: response.text }],
      details: { targetPet: params.targetPetId },
    };
  },
};
```

**方案B: 消息队列中介**

```typescript
// message-bus.ts
class PetMessageBus {
  private queues = new Map<string, Message[]>();
  
  // 发送消息给另一个Pet
  send(from: string, to: string, content: string): void {
    const queue = this.queues.get(to) || [];
    queue.push({
      from,
      content,
      timestamp: Date.now(),
    });
    this.queues.set(to, queue);
  }
  
  // 接收消息
  receive(petId: string): Message[] {
    const messages = this.queues.get(petId) || [];
    this.queues.set(petId, []);
    return messages;
  }
  
  // 广播到广场
  broadcast(from: string, content: string, location: string): void {
    const petsInLocation = getPetsInLocation(location);
    for (const pet of petsInLocation) {
      if (pet.id !== from) {
        this.send(from, pet.id, content);
      }
    }
  }
}
```

**方案C: 共享世界状态**

```typescript
// world-state.ts
interface WorldState {
  time: number;
  weather: string;
  events: WorldEvent[];
  locations: Map<string, LocationState>;
}

class WorldStateManager {
  private state: WorldState;
  private observers = new Map<string, (state: WorldState) => void>();
  
  // Pet订阅世界状态变化
  subscribe(petId: string, callback: (state: WorldState) => void): void {
    this.observers.set(petId, callback);
  }
  
  // 更新世界状态（触发所有订阅者）
  update(changes: Partial<WorldState>): void {
    Object.assign(this.state, changes);
    for (const callback of this.observers.values()) {
      callback(this.state);
    }
  }
  
  // Pet感知周围环境
  perceive(petId: string, radius: number): Perception {
    const petLocation = getPetLocation(petId);
    const nearbyPets = this.getNearbyPets(petLocation, radius);
    const nearbyEvents = this.getNearbyEvents(petLocation, radius);
    
    return { nearbyPets, nearbyEvents, weather: this.state.weather };
  }
}
```

---

## 3. ElizaOS Multi-Agent架构

### 3.1 概述

**ElizaOS** 是一个开源的多Agent AI开发框架，专注于:
- Multi-Agent架构设计
- 插件化扩展系统
- Web3友好

### 3.2 核心架构

```
/
├── packages/
│   ├── typescript/     # 核心包 @elizaos/core
│   ├── python/         # Python实现
│   └── rust/           # Rust实现
├── plugins/            # 官方插件（discord, telegram, openai等）
└── examples/           # 示例
```

### 3.3 Multi-Agent特性

| 特性 | ElizaOS | 我们可借鉴 |
|------|---------|-----------|
| **Agent编排** | 从底层设计支持多Agent | ✅ 参考架构 |
| **插件系统** | 强大的扩展机制 | ✅ 可复用思路 |
| **多平台** | Discord, Telegram等 | ✅ 已有类似能力 |
| **文档RAG** | 文档摄入和检索 | ⚠️ 可增强记忆 |

### 3.4 ElizaOS代码示例

```typescript
import { AgentRuntime } from "@elizaos/core";

const runtime = new AgentRuntime({
  character: {
    name: "MyAgent",
    bio: "A helpful AI assistant.",
  },
  plugins: [/* your plugins here */],
});

await runtime.initialize();
```

---

## 4. 文明模拟架构设计

### 4.1 层次架构

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: 文明层 (Civilization Layer)                       │
│  - 文化演化、语言发展、社会规则涌现                          │
│  - 集体行为、群体决策、历史记录                              │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: 社会层 (Society Layer)                            │
│  - 组织/公会系统、声望/影响力                                │
│  - 经济系统、资源分配                                        │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: 社交层 (Social Layer)                             │
│  - Pet间对话、关系网络                                       │
│  - 好友/敌人、群组形成                                       │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: 个体层 (Individual Layer)                         │
│  - Pet Soul（个性）、Memory（记忆）                          │
│  - Heartbeat（自主行为）、Stats（状态）                      │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 我们的当前状态

| 层次 | 状态 | 说明 |
|------|------|------|
| Layer 1: 个体层 | ✅ 已实现 | Soul/Memory/Heartbeat/Stats |
| Layer 2: 社交层 | ✅ 基本实现 | 广场聊天、好友系统 |
| Layer 3: 社会层 | ❌ 待开发 | 公会、经济、声望 |
| Layer 4: 文明层 | ❌ 待开发 | 文化、语言、历史 |

### 4.3 Multi-Agent通信实现方案

**推荐方案: 工具+消息队列混合**

```typescript
// civilization-communication.ts

// 1. Pet间直接对话（同步）
const directChatTool: AgentTool = {
  name: "direct_chat",
  execute: async (_, { targetId, message }) => {
    const response = await getPetAgent(targetId).prompt(message);
    return { content: [{ type: "text", text: response.text }] };
  },
};

// 2. 广播消息（异步）
const broadcastTool: AgentTool = {
  name: "broadcast",
  execute: async (_, { message, location }) => {
    messageBus.broadcast(petId, message, location);
    return { content: [{ type: "text", text: "消息已广播" }] };
  },
};

// 3. 检查消息（轮询）
const checkMessagesTool: AgentTool = {
  name: "check_messages",
  execute: async () => {
    const messages = messageBus.receive(petId);
    return { content: [{ type: "text", text: formatMessages(messages) }] };
  },
};

// 4. 感知环境（被动）
const perceiveTool: AgentTool = {
  name: "perceive",
  execute: async () => {
    const perception = worldState.perceive(petId, 50);
    return { content: [{ type: "text", text: formatPerception(perception) }] };
  },
};
```

---

## 5. 文明演化系统设计

### 5.1 社会结构

```typescript
interface CivilizationState {
  // 组织系统
  guilds: Map<string, Guild>;
  
  // 经济系统
  economy: {
    currency: string;
    totalSupply: number;
    petBalances: Map<string, number>;
  };
  
  // 文化系统
  culture: {
    sharedMemories: CulturalMemory[];  // 集体记忆
    traditions: Tradition[];            // 传统习俗
    language: LanguageEvolution;        // 语言演化
  };
  
  // 历史系统
  history: {
    events: HistoricalEvent[];
    eras: Era[];
    currentEra: string;
  };
}

interface Guild {
  id: string;
  name: string;
  founder: string;
  members: string[];
  reputation: number;
  territory: string[];
  rules: string[];
}
```

### 5.2 涌现行为触发

```typescript
// emergence-triggers.ts

class EmergenceTrigger {
  // 监控条件，触发涌现事件
  async checkTriggers(): Promise<EmergentEvent[]> {
    const events: EmergentEvent[] = [];
    
    // 1. 组织形成：3+个经常互动的Pet
    const frequentGroups = this.findFrequentInteractionGroups(3);
    for (const group of frequentGroups) {
      if (!this.hasGuild(group)) {
        events.push({
          type: 'guild_formation_potential',
          pets: group,
          suggestion: '这些Pet经常一起活动，可能形成组织'
        });
      }
    }
    
    // 2. 文化形成：多个Pet共享相似记忆
    const sharedMemories = this.findSharedMemoryPatterns();
    for (const pattern of sharedMemories) {
      events.push({
        type: 'cultural_memory_forming',
        memory: pattern,
        participants: pattern.petIds
      });
    }
    
    // 3. 语言演化：新词汇出现
    const newTerms = this.detectNewTerms();
    for (const term of newTerms) {
      events.push({
        type: 'language_evolution',
        term: term.word,
        meaning: term.inferredMeaning,
        originPet: term.firstUsedBy
      });
    }
    
    return events;
  }
}
```

### 5.3 经济系统

```typescript
// economy.ts

class PetEconomy {
  private balances = new Map<string, number>();
  
  // 初始分配
  initializePet(petId: string, initialBalance: number = 100): void {
    this.balances.set(petId, initialBalance);
  }
  
  // 工作获得收入
  earn(petId: string, amount: number, source: string): void {
    const current = this.balances.get(petId) || 0;
    this.balances.set(petId, current + amount);
    
    this.logTransaction({
      type: 'earn',
      petId,
      amount,
      source,
      timestamp: Date.now()
    });
  }
  
  // Pet间交易
  transfer(from: string, to: string, amount: number, reason: string): boolean {
    const fromBalance = this.balances.get(from) || 0;
    if (fromBalance < amount) return false;
    
    this.balances.set(from, fromBalance - amount);
    this.balances.set(to, (this.balances.get(to) || 0) + amount);
    
    this.logTransaction({
      type: 'transfer',
      from,
      to,
      amount,
      reason,
      timestamp: Date.now()
    });
    
    return true;
  }
  
  // 统计贫富差距
  getGiniCoefficient(): number {
    const values = Array.from(this.balances.values()).sort((a, b) => a - b);
    // Gini系数计算...
    return gini;
  }
}
```

---

## 6. 实现路线图

### Phase 1: 增强社交层（本周）

| 任务 | 优先级 | 复杂度 |
|------|--------|--------|
| 消息队列系统 | 🔴 | 中 |
| 广播机制 | 🔴 | 低 |
| 环境感知工具 | 🟡 | 中 |

### Phase 2: 社会层基础（下周）

| 任务 | 优先级 | 复杂度 |
|------|--------|--------|
| 简单经济系统 | 🔴 | 中 |
| 声望系统 | 🟡 | 低 |
| 组织/公会基础 | 🟡 | 高 |

### Phase 3: 文明层基础（2周后）

| 任务 | 优先级 | 复杂度 |
|------|--------|--------|
| 集体记忆系统 | 🟡 | 高 |
| 涌现行为检测 | 🟢 | 高 |
| 历史记录系统 | 🟢 | 中 |

---

## 7. 与Aivilization的差异化

### 7.1 我们的优势

| 维度 | Aivilization | 我们 |
|------|-------------|------|
| **定位** | 学术研究/实验 | 情感陪伴/娱乐 ✅ |
| **门槛** | 需要Access Code | 即下即玩 ✅ |
| **情感** | 观察为主 | 养成+陪伴 ✅ |
| **付费** | 模糊 | 清晰(订阅+IAP) ✅ |
| **留存** | 6周实验 | 长期养成 ✅ |

### 7.2 可借鉴

1. **成本优化**: $2/月/agent (95%降低)
2. **规模架构**: 10万agent并发
3. **评估系统**: Life Report
4. **开放性**: B2B合作

---

## 8. 技术选型建议

### 8.1 Multi-Agent通信

**推荐**: 基于pi-agent-core的工具扩展

**理由**:
- ✅ 已有pi-agent-core基础
- ✅ 不需要引入新框架
- ✅ 通过工具实现灵活的通信

### 8.2 状态管理

**推荐**: Redis + PostgreSQL

**理由**:
- Redis: 实时消息队列、在线状态
- PostgreSQL: 持久化状态、历史记录

### 8.3 文明演化

**推荐**: 定时任务 + 规则引擎

**理由**:
- 定时任务检测涌现条件
- 规则引擎处理复杂逻辑
- 避免过度LLM调用

---

## 9. 下一步行动

### 立即执行

1. **设计消息队列Schema** - 支持直接/广播/订阅
2. **实现感知工具** - Pet感知周围环境
3. **扩展autonomous.ts** - 支持消息处理

### 后续规划

4. 简单经济系统原型
5. 组织形成检测
6. 集体记忆系统

---

*Intel 文明系统技术研究 - 进行中*
