# AI Town 架构分析

**Intel 研究文档** 🔍  
**日期**: 2026-03-01  
**来源**: https://github.com/a16z-infra/ai-town  
**技术栈**: Convex, pixi-react, OpenAI

---

## 1. 项目概述

**AI Town** 是a16z开源的可部署启动套件，用于构建虚拟城镇，AI角色在其中生活、聊天和社交。

**关键特性**：
- MIT许可，可自由fork和定制
- 基于Generative Agents论文架构
- 完整的端到端实现

---

## 2. 系统分层架构

```
┌─────────────────────────────────────────────────┐
│  Client UI (src/)                               │
│  - pixi-react 渲染游戏                          │
│  - useQuery hooks 读取状态                       │
└─────────────────────────────────────────────────┘
                        ↓ ↑
┌─────────────────────────────────────────────────┐
│  Game Logic (convex/aiTown)                     │
│  - 定义游戏状态                                  │
│  - 处理输入（人类和Agent）                       │
│  - 模拟运行                                      │
└─────────────────────────────────────────────────┘
                        ↓ ↑
┌─────────────────────────────────────────────────┐
│  Game Engine (convex/engine)                    │
│  - 保存/加载游戏状态                             │
│  - 协调输入处理                                  │
│  - 运行游戏循环                                  │
└─────────────────────────────────────────────────┘
                        ↓ ↑
┌─────────────────────────────────────────────────┐
│  Agent (convex/agent)                           │
│  - 游戏循环的一部分运行                          │
│  - 调用LLM做长时处理                             │
│  - 记忆系统                                      │
└─────────────────────────────────────────────────┘
```

---

## 3. 数据模型

### 3.1 核心概念

| 概念 | 文件 | 描述 |
|------|------|------|
| **World** | `world.ts` | 地图，多个Player互动 |
| **Player** | `player.ts` | 角色，有名字、描述、位置、寻路状态 |
| **Conversation** | `conversations.ts` | 对话，由Player创建 |
| **Membership** | `conversationMembership.ts` | 对话成员关系 |

### 3.2 对话成员状态

```typescript
type MembershipStatus = 
  | 'invited'       // 已邀请，未接受
  | 'walkingOver'   // 已接受，正在走过去
  | 'participating' // 正在参与对话
```

### 3.3 数据库Schema分类

| 类别 | 位置 | 用途 |
|------|------|------|
| Engine表 | `convex/engine/schema.ts` | 引擎内部状态 |
| Game表 | `convex/aiTown/schema.ts` | 游戏状态 |
| Agent表 | `convex/agent/schema.ts` | Agent状态 |

---

## 4. 输入系统

### 4.1 输入类型

```typescript
// convex/aiTown/inputs.ts
const inputs = {
  // 加入/离开
  join: inputHandler(...),
  leave: inputHandler(...),
  
  // 移动
  moveTo: inputHandler(...),  // RTS风格，指定目的地
  
  // 对话
  startConversation: inputHandler(...),
  acceptInvite: inputHandler(...),
  rejectInvite: inputHandler(...),
  leaveConversation: inputHandler(...),
  startTyping: inputHandler(...),
  finishSendingMessage: inputHandler(...)
}
```

### 4.2 输入处理流程

```
用户提交输入 → insertInput() → inputs表（分配递增ID+时间戳）
                     ↓
              引擎处理输入
                     ↓
              结果写回inputs行
                     ↓
            客户端通过inputStatus查询结果
```

---

## 5. 游戏引擎

### 5.1 模拟运行

```typescript
// Game.tick(now) - 模拟时间前进
tick(now: number) {
  Player.tickPathfinding();   // 寻路
  Player.tickPosition();      // 位置更新
  Conversation.tick();        // 对话状态
  Agent.tick();               // Agent逻辑
}
```

### 5.2 Tick频率设计

| 概念 | 频率 | 说明 |
|------|------|------|
| **Tick** | 60/秒 | 模拟更新频率（平滑运动） |
| **Step** | 1/秒 | 数据库写入频率（成本优化） |

**Step流程**：
1. 加载游戏状态到内存
2. 决定运行时长
3. 执行多个tick（处理输入+模拟）
4. 将更新后的状态写回数据库

### 5.3 核心设计约束

**单线程保证**：
- 每个World同一时间只有一个引擎运行
- 不需要考虑竞态条件或并发
- 通过generation number实现

### 5.4 历史值追踪

**问题**：Step每秒一次，位置更新会跳跃

**解决**：`HistoricalObject`
- 每tick记录值的变化
- 客户端"回放"历史实现平滑

```typescript
// 位置存储在HistoricalObject中
const location = new HistoricalObject({
  x: 100,
  y: 200,
  orientation: 0,
  speed: 5
});
```

---

## 6. Agent架构

### 6.1 Agent循环

```
Agent.tick()                    # 读取/修改游戏状态
    ↓
需要调用LLM?
    ↓ 是
startOperation() → 调度internalAction
    ↓
Action执行：
  - 读取数据（internalQuery）
  - 调用LLM
  - 写入数据（internalMutation）
  - 提交inputs修改游戏状态
    ↓
删除inProgressOperation
    ↓
Agent.tick() 观察新状态，继续决策
```

### 6.2 对话层 (`convex/agent/conversations.ts`)

**功能**：
- `startConversation()` - 开始对话
- `continueConversation()` - 继续对话
- `leaveConversation()` - 礼貌退出

**实现**：
1. 从数据库加载结构化数据
2. 查询记忆层获取对对方的印象
3. 调用OpenAI生成回复

### 6.3 记忆系统 (`convex/agent/memory.ts`)

```typescript
// 对话后
1. GPT总结对话历史
2. 计算总结文本的embedding
3. 写入Convex向量数据库

// 开始新对话时
1. Embed "你对Danny有什么看法？"
2. 找到3个最相似的记忆
3. 获取总结文本注入到对话prompt
```

### 6.4 Embedding缓存 (`convex/agent/embeddingsCache.ts`)

```typescript
// 避免重复计算embedding
cache.get(hash(text)) || computeAndStore(text)
```

---

## 7. 消息数据模型

**为什么消息独立于游戏引擎？**
1. 核心模拟不需要消息内容
2. 消息频繁更新（OpenAI流式输出），需要低延迟
3. 不适合引擎的Step模式

**结构**：
- 消息属于对话
- 包含作者和文本
- 对话有typing状态

---

## 8. 设计目标与限制

### 8.1 设计目标

1. **尽量接近普通Convex应用**
   - 使用普通hooks（useQuery）
   - 游戏状态存普通表

2. **类似现有引擎**
   - tick()模型直观

3. **Agent与引擎解耦**
   - 人类和AI做同样的事

### 8.2 固有限制

| 限制 | 原因 | 建议 |
|------|------|------|
| **游戏状态需小** | 每step加载到内存 | 保持<几十KB |
| **输入经过数据库** | inputs表流程 | 不适合大/频繁输入 |
| **输入延迟~1.5s** | RTT + step延迟 | 不适合竞技游戏 |
| **单线程** | 设计约束 | 不适合计算密集型 |

---

## 9. 对AI Pet项目的应用

### 9.1 架构借鉴

```
┌─────────────────────────────────────────────────┐
│  前端 (Next.js)                                 │
│  - Canvas渲染Pet                                │
│  - React hooks读取状态                           │
└─────────────────────────────────────────────────┘
                        ↓ ↑
┌─────────────────────────────────────────────────┐
│  游戏逻辑 (Fastify API)                         │
│  - Pet状态管理                                   │
│  - 互动处理                                      │
│  - 广场模拟                                      │
└─────────────────────────────────────────────────┘
                        ↓ ↑
┌─────────────────────────────────────────────────┐
│  Pet Agent                                       │
│  - 记忆流                                        │
│  - 反思系统                                      │
│  - 对话生成                                      │
└─────────────────────────────────────────────────┘
```

### 9.2 可直接复用的模式

| AI Town模式 | AI Pet应用 |
|-------------|-----------|
| Memory Stream → Embedding → 向量检索 | Pet记忆系统 |
| Conversation总结 → 存储 | Pet互动记忆 |
| Agent循环：tick → 决策 → 行动 | Pet自主行为 |
| Historical Object | 平滑位置更新 |
| 输入系统设计 | 用户与Pet互动 |

### 9.3 需要调整的部分

| AI Town | AI Pet调整 |
|---------|-----------|
| Convex | Fastify + PostgreSQL |
| OpenAI | AWS Bedrock (Nova) |
| 单World | 多广场（分片） |
| 25 Agents | 目标10K+ Pets |

### 9.4 关键实现参考

**记忆系统**：
```typescript
// AI Town的实现
async function rememberConversation(agentId: string, summary: string) {
  const embedding = await embed(summary);
  await db.insert('memories', {
    agentId,
    summary,
    embedding,
    createdAt: Date.now()
  });
}

async function recallMemories(agentId: string, query: string, limit: number = 3) {
  const queryEmbedding = await embed(query);
  return await db.vectorSearch('memories', queryEmbedding, { limit });
}
```

**对话生成**：
```typescript
async function generateResponse(pet: Pet, conversation: Conversation) {
  const memories = await recallMemories(pet.id, `关于${otherPet.name}的记忆`);
  
  const prompt = `你是${pet.name}。
个性：${pet.personality}
相关记忆：${memories.map(m => m.summary).join('\n')}
对话历史：${conversation.history}
请回复：`;
  
  return await llm.chat(prompt);
}
```

---

## 10. 关键洞察

1. **分层清晰**：游戏引擎 vs Agent逻辑 vs UI完全分离
2. **单线程简化**：避免并发复杂性，适合中小规模
3. **历史值追踪**：解决低频写入 vs 平滑显示的矛盾
4. **记忆向量化**：核心技术，直接可用
5. **输入排队**：所有修改通过输入系统，保证一致性

---

## 11. 下一步

- [ ] 研究Convex向量数据库实现细节
- [ ] 设计PostgreSQL + pgvector的记忆系统
- [ ] 规划社交网络图结构
- [ ] 评估AWS Bedrock embedding成本

---

*Intel AI Town架构分析完成*
