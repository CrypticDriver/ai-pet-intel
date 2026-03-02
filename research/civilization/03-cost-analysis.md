# 成本估算模型 v2.0（100% LLM驱动）

**Intel 研究文档** 🔍  
**日期**: 2026-03-02  
**版本**: v2.0 - 按老大方向调整  
**任务**: 100% LLM驱动的成本计算 + 模型分层策略

---

## ⚠️ 方向修正

**v1.0（已废弃）**: 90%规则+10%LLM → 成本优先  
**v2.0（当前）**: 100%LLM驱动 → **生命优先** ✅

**核心原则**：
- Pet是**数字生命**，不是NPC
- **所有决策走LLM**（思维自由）
- 规则只用于**安全护栏**和**系统调度**
- 成本控制靠**模型分层+缓存+批处理**，不靠砍LLM

---

## 1. 基础假设

### 1.1 规模参数

| 参数 | 数值 | 说明 |
|------|------|------|
| Pet数量 | 100 | MVP阶段 |
| 活跃率 | 80% | 每天有互动的Pet |
| 用户在线时长 | 2小时/天 | 平均 |
| **自主决策频率** | **每42秒** | **现有设定，全LLM** |

### 1.2 LLM定价（AWS Bedrock）

| 模型 | 输入价格 | 输出价格 | 用途 |
|------|----------|----------|------|
| **Nova Lite** | $0.06/1M | $0.24/1M | 日常思考、简单对话 |
| **Nova Pro** | $0.80/1M | $3.20/1M | 关键社交、复杂决策 |
| **Claude Sonnet** | $3.00/1M | $15.00/1M | 个性进化（极少） |
| **Titan Embed v2** | $0.00011/1K | - | Embedding |

---

## 2. 模型分层策略（核心优化）

### 2.1 分层原则

**不砍LLM调用，优化模型选择**

```
┌─────────────────────────────────────────────────────────────┐
│  任务分层路由                                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  日常思考（Nova Lite）                                        │
│  ├── 自主决策（我现在想做什么？）                               │
│  ├── 简单对话（日常闲聊）                                      │
│  ├── 环境感知（周围有什么？）                                   │
│  └── 状态表达（我饿了/累了）                                   │
│                                                             │
│  关键社交（Nova Pro）                                         │
│  ├── 首次相遇（第一印象很重要）                                 │
│  ├── 冲突处理（需要深度思考）                                   │
│  ├── 重要决定（加入组织/离开朋友）                              │
│  └── 每日反思（总结今天的经历）                                 │
│                                                             │
│  个性进化（Claude Sonnet）                                    │
│  ├── 每周个性微调（极少）                                      │
│  └── 重大人生转折（更少）                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 分层代码实现

```typescript
// model-selector.ts

type TaskType = 
  | 'autonomous_decision'    // 自主决策
  | 'simple_chat'           // 简单对话
  | 'environment_perception' // 环境感知
  | 'status_expression'     // 状态表达
  | 'first_meeting'         // 首次相遇
  | 'conflict_resolution'   // 冲突处理
  | 'important_decision'    // 重要决定
  | 'daily_reflection'      // 每日反思
  | 'personality_evolution' // 个性进化
  | 'life_transition';      // 人生转折

function selectModel(task: TaskType): string {
  // Nova Lite: 日常思考（最便宜）
  const novaLiteTasks = [
    'autonomous_decision',
    'simple_chat',
    'environment_perception',
    'status_expression',
  ];
  
  // Nova Pro: 关键社交（中等）
  const novaProTasks = [
    'first_meeting',
    'conflict_resolution',
    'important_decision',
    'daily_reflection',
  ];
  
  // Claude Sonnet: 个性进化（最贵，极少用）
  const sonnetTasks = [
    'personality_evolution',
    'life_transition',
  ];
  
  if (novaLiteTasks.includes(task)) {
    return 'us.amazon.nova-lite-v1:0';
  }
  if (novaProTasks.includes(task)) {
    return 'us.amazon.nova-pro-v1:0';
  }
  if (sonnetTasks.includes(task)) {
    return 'us.anthropic.claude-sonnet-4-5-20260101:0';
  }
  
  // 默认使用Nova Lite
  return 'us.amazon.nova-lite-v1:0';
}
```

### 2.3 分层比例估算

| 层级 | 模型 | 调用占比 | 成本占比 |
|------|------|----------|----------|
| **日常思考** | Nova Lite | 85% | ~15% |
| **关键社交** | Nova Pro | 14% | ~75% |
| **个性进化** | Claude Sonnet | 1% | ~10% |

---

## 3. 任务分解与Token估算（100% LLM）

### 3.1 自主决策（全LLM）

**场景**：Pet每42秒思考一次"我现在想做什么？"

```
System Prompt: ~300 tokens（soul + 当前状态）
Context: ~200 tokens（环境 + 最近记忆）
Decision output: ~80 tokens（想法 + 行动）
---
Total per decision: ~580 tokens
```

**频率估算**：
- 每Pet每小时: ~86次（3600/42）
- 活跃时间: 8小时（假设Pet也休息）
- 每Pet每天: ~688次
- 活跃Pet: 80只
- **总计: 80 * 688 = 55,040次/天**

**模型分配**：
- Nova Lite: 95%（日常决策）
- Nova Pro: 5%（涉及社交的决策）

**成本**（Nova Lite为主）：
```
Nova Lite (95%):
  输入: 52,288 * 500 = 26.1M tokens → $1.57
  输出: 52,288 * 80 = 4.2M tokens → $1.00
  
Nova Pro (5%):
  输入: 2,752 * 500 = 1.4M tokens → $1.10
  输出: 2,752 * 80 = 0.22M tokens → $0.70

日成本: $4.37
月成本: ~$131
```

### 3.2 用户对话

**场景**：用户与Pet聊天

```
System Prompt: ~500 tokens（worldview + soul + memory）
User Input: ~50 tokens
Pet Response: ~100 tokens
---
Total per turn: ~650 tokens
```

**频率估算**：
- 每用户每天: 20次对话
- 活跃Pet: 80只
- 总计: 80 * 20 = 1,600次/天

**模型分配**：
- Nova Lite: 80%（日常聊天）
- Nova Pro: 20%（深度对话）

**成本**：
```
Nova Lite (80%):
  输入: 1,280 * 550 = 704,000 tokens → $0.042
  输出: 1,280 * 100 = 128,000 tokens → $0.031

Nova Pro (20%):
  输入: 320 * 550 = 176,000 tokens → $0.141
  输出: 320 * 100 = 32,000 tokens → $0.102

日成本: $0.316
月成本: ~$9.48
```

### 3.3 Pet间对话

**场景**：广场上Pet自发聊天

```
System Prompt: ~500 tokens
Context (对方记忆): ~200 tokens
Conversation (3轮):
  - Turn 1: 50 input + 80 output
  - Turn 2: 50 input + 80 output
  - Turn 3: 50 input + 80 output
---
Total per conversation: ~1,090 tokens
```

**频率估算**：
- 每对Pet对话: 30%概率/小时（广场）
- 广场Pet: 20只（同时）
- 对话次数: (20*19/2) * 0.3 * 8小时 = ~456次/天

**模型分配**：
- Nova Lite: 70%（已认识的朋友）
- Nova Pro: 30%（首次相遇）

**成本**：
```
Nova Lite (70%):
  输入: 319 * 850 = 271,150 tokens → $0.016
  输出: 319 * 240 = 76,560 tokens → $0.018

Nova Pro (30%):
  输入: 137 * 850 = 116,450 tokens → $0.093
  输出: 137 * 240 = 32,880 tokens → $0.105

日成本: $0.232
月成本: ~$6.96
```

### 3.4 记忆创建

**场景**：Pet互动后创建记忆

```
System Prompt: ~200 tokens
Conversation summary: ~300 tokens
Memory output: ~50 tokens
---
Total: ~550 tokens
```

**频率**：每次对话后 = 456 + 1,600 = ~2,056次/天

**模型**：Nova Lite（记忆创建不需要高质量）

**成本**：
```
输入: 2,056 * 500 = 1.03M tokens → $0.062
输出: 2,056 * 50 = 102,800 tokens → $0.025
日成本: $0.087
月成本: ~$2.61
```

### 3.5 每日反思

**场景**：每天结束时Pet总结经历

```
System Prompt: ~300 tokens
Today's events: ~500 tokens
Reflection output: ~200 tokens
---
Total: ~1,000 tokens
```

**频率**：100 Pet * 1次/天 = 100次/天

**模型**：Nova Pro（反思质量很重要）

**成本**：
```
输入: 100 * 800 = 80,000 tokens → $0.064
输出: 100 * 200 = 20,000 tokens → $0.064
日成本: $0.128
月成本: ~$3.84
```

### 3.6 每周个性进化

**场景**：每周Pet个性微调

```
System Prompt: ~300 tokens
Week stats: ~500 tokens
Analysis output: ~300 tokens
---
Total: ~1,100 tokens
```

**频率**：100 Pet * 1次/周 = ~14次/天

**模型**：Claude Sonnet（进化质量至关重要）

**成本**：
```
输入: 14 * 800 = 11,200 tokens → $0.034
输出: 14 * 300 = 4,200 tokens → $0.063
日成本: $0.097
月成本: ~$2.91
```

### 3.7 Embedding

**场景**：记忆向量化

```
Memory text: ~50 tokens
```

**频率**：
- 新记忆: ~2,056条/天
- 检索查询: ~5,000次/天

**成本**：
```
Embedding: 7,056 * 50 = 352,800 tokens → $0.039
月成本: ~$1.17
```

---

## 4. 成本汇总（100 Pet，100% LLM）

### 4.1 月度成本

| 任务 | 模型分布 | 日成本 | 月成本 | 占比 |
|------|----------|--------|--------|------|
| **自主决策** | Lite 95% / Pro 5% | **$4.37** | **$131.1** | **83%** |
| 用户对话 | Lite 80% / Pro 20% | $0.32 | $9.48 | 6% |
| Pet间对话 | Lite 70% / Pro 30% | $0.23 | $6.96 | 4% |
| 记忆创建 | Lite 100% | $0.09 | $2.61 | 2% |
| 每日反思 | Pro 100% | $0.13 | $3.84 | 2% |
| 每周进化 | Sonnet 100% | $0.10 | $2.91 | 2% |
| Embedding | Titan 100% | $0.04 | $1.17 | 1% |
| **总计** | - | **$5.28** | **$158.07** | 100% |

### 4.2 每Pet成本

```
$158.07 / 100 = $1.58/月/Pet
```

### 4.3 与竞品对比

| 项目 | 成本/月/Pet | vs Aivilization |
|------|-------------|-----------------|
| **Aivilization** | $2.00 | 基准 |
| **我们（100% LLM）** | $1.58 | **79%** ✅ |
| **我们（优化后）** | ~$0.95 | **48%** ✅✅ |

**结论**：即使100% LLM驱动，我们仍低于Aivilization！

---

## 5. 优化策略（不砍LLM）

### 5.1 决策频率优化

**问题**：自主决策占成本83%，太高

**优化方案**：智能触发（不是砍决策）

```typescript
// smart-trigger.ts

class SmartDecisionTrigger {
  private lastDecisionTime = new Map<string, number>();
  private lastState = new Map<string, PetState>();
  
  shouldTriggerDecision(petId: string, currentState: PetState): boolean {
    const now = Date.now();
    const lastTime = this.lastDecisionTime.get(petId) || 0;
    const lastPetState = this.lastState.get(petId);
    
    // 规则1: 最短间隔42秒（原设定）
    if (now - lastTime < 42 * 1000) return false;
    
    // 规则2: 状态无变化时，延长间隔到2分钟
    if (lastPetState && this.statesSimilar(lastPetState, currentState)) {
      if (now - lastTime < 120 * 1000) return false;
    }
    
    // 规则3: 有事件发生时，立即触发
    if (currentState.pendingEvents.length > 0) return true;
    
    // 规则4: 有人接近时，立即触发
    if (currentState.nearbyPetsChanged) return true;
    
    return true; // 默认触发
  }
  
  private statesSimilar(a: PetState, b: PetState): boolean {
    // 状态相似度检查
    return (
      Math.abs(a.energy - b.energy) < 5 &&
      Math.abs(a.mood - b.mood) < 5 &&
      a.location === b.location &&
      a.nearbyPets.length === b.nearbyPets.length
    );
  }
}
```

**效果**：
- 原频率: 每42秒 = 688次/天/Pet
- 优化后: 平均每90秒 = ~320次/天/Pet
- **减少53%决策调用**

### 5.2 优化后成本估算

| 任务 | 原成本 | 优化后 | 节省 |
|------|--------|--------|------|
| 自主决策 | $131.1 | ~$60 | -54% |
| 用户对话 | $9.48 | ~$7.6 | -20% (缓存) |
| Pet间对话 | $6.96 | ~$5.6 | -20% (缓存) |
| 记忆创建 | $2.61 | ~$2.1 | -20% |
| 每日反思 | $3.84 | $3.84 | 0% |
| 每周进化 | $2.91 | $2.91 | 0% |
| Embedding | $1.17 | $1.17 | 0% |
| **总计** | $158.07 | **~$83.2** | **-47%** |

**优化后每Pet成本**：
```
$83.2 / 100 = $0.83/月/Pet
```

### 5.3 缓存策略

```typescript
// response-cache.ts

class ResponseCache {
  private cache = new LRUCache<string, CacheEntry>({
    max: 5000,
    ttl: 10 * 60 * 1000, // 10分钟
  });
  
  // 对话缓存（相似问题相似回答）
  getChatCache(petId: string, userMessage: string): string | null {
    // 模糊匹配常见问候语
    const normalizedMessage = this.normalizeGreeting(userMessage);
    const key = `chat:${petId}:${normalizedMessage}`;
    return this.cache.get(key)?.response || null;
  }
  
  // 决策缓存（相同状态相似决策）
  getDecisionCache(petId: string, state: PetState): string | null {
    // 状态指纹
    const stateFingerprint = this.getStateFingerprint(state);
    const key = `decision:${petId}:${stateFingerprint}`;
    
    const cached = this.cache.get(key);
    if (cached && Math.random() > 0.3) {
      // 70%概率使用缓存（保持一定变化性）
      return cached.response;
    }
    return null;
  }
  
  private normalizeGreeting(message: string): string {
    const greetings = ['你好', '早上好', '晚上好', '嗨', 'hi', 'hello'];
    for (const g of greetings) {
      if (message.toLowerCase().includes(g)) return 'GREETING';
    }
    return message;
  }
  
  private getStateFingerprint(state: PetState): string {
    // 状态分桶（能量、心情各分10档）
    const energyBucket = Math.floor(state.energy / 10);
    const moodBucket = Math.floor(state.mood / 10);
    return `${state.location}:${energyBucket}:${moodBucket}:${state.nearbyPets.length}`;
  }
}
```

**预估节省**：20%

---

## 6. 成本监控（调整后）

### 6.1 新的预算设置

```typescript
// cost-monitor-v2.ts

class CostMonitor {
  // 100 Pet的日预算
  private dailyBudget = 3.00;  // $3/天（优化后目标）
  private todaySpent = 0;
  
  async trackUsage(model: string, task: TaskType, inputTokens: number, outputTokens: number): Promise<void> {
    const cost = this.calculateCost(model, inputTokens, outputTokens);
    this.todaySpent += cost;
    
    // 分任务统计
    this.taskCosts.set(task, (this.taskCosts.get(task) || 0) + cost);
    
    // 警告阈值
    if (this.todaySpent > this.dailyBudget * 0.7) {
      console.warn(`⚠️ 已使用70%日预算: $${this.todaySpent.toFixed(2)}`);
      this.enableSoftSavingMode();
    }
    
    if (this.todaySpent > this.dailyBudget * 0.9) {
      console.error(`🚨 接近日预算上限!`);
      this.enableHardSavingMode();
    }
  }
  
  // 软节省模式（不影响体验）
  enableSoftSavingMode(): void {
    // 延长决策间隔
    setDecisionInterval(90);  // 90秒
    
    // 增加缓存使用概率
    setCacheUseProbability(0.8);
  }
  
  // 硬节省模式（轻微影响体验）
  enableHardSavingMode(): void {
    // 进一步延长决策间隔
    setDecisionInterval(180);  // 3分钟
    
    // Nova Pro任务降级到Nova Lite
    setModelDowngrade('nova-pro', 'nova-lite');
    
    // 但永远不砍掉LLM决策！
  }
}
```

### 6.2 日报告模板

```markdown
# 日成本报告 - 2026-03-02

## 汇总
- 总成本: $2.78
- 日预算: $3.00
- 使用率: 93%

## 分项
| 任务 | 调用次数 | Tokens | 成本 | 占比 |
|------|----------|--------|------|------|
| 自主决策 | 25,600 | 14.8M | $2.00 | 72% |
| 用户对话 | 1,543 | 1.0M | $0.30 | 11% |
| Pet间对话 | 423 | 0.5M | $0.20 | 7% |
| 反思 | 100 | 0.1M | $0.13 | 5% |
| 其他 | 500 | 0.3M | $0.15 | 5% |

## 模型使用
| 模型 | 调用占比 | 成本占比 |
|------|----------|----------|
| Nova Lite | 88% | 35% |
| Nova Pro | 11% | 55% |
| Sonnet | 1% | 10% |

## 优化状态
- 缓存命中率: 23%
- 平均决策间隔: 72秒
- 节省模式: 未启用

## 建议
- ✅ 成本在预算内
- 💡 可考虑提高缓存TTL
```

---

## 7. 不同规模成本预测

### 7.1 规模扩展（100% LLM，优化后）

| Pet数量 | 月成本 | 每Pet成本 | 说明 |
|---------|--------|-----------|------|
| 100 | $83 | $0.83 | MVP |
| 500 | $415 | $0.83 | Beta |
| 1,000 | $830 | $0.83 | 线性扩展 |
| 5,000 | $4,150 | $0.83 | 线性扩展 |
| 10,000 | $8,300 | $0.83 | Launch |

### 7.2 商业模式分析

| 模式 | 订阅价格 | LLM成本 | 成本占比 | 可行性 |
|------|----------|---------|----------|--------|
| **订阅** | $6.99/月 | $0.83 | 12% | ✅ 极佳 |
| **订阅（Premium）** | $9.99/月 | $1.50 | 15% | ✅ 极佳 |
| **广告** | $0.50/月 | $0.83 | 166% | ❌ 不可行 |
| **免费增值** | 混合 | ~$0.50 | 20-30% | ⚠️ 需优化 |

**结论**：订阅模式完全可行，成本占比仅12%

---

## 8. 总结

### 8.1 关键发现

| 指标 | v1.0（混合模式） | v2.0（100% LLM） | 变化 |
|------|-----------------|------------------|------|
| 每Pet成本 | $0.13 | **$0.83** | +6.4x |
| vs Aivilization | 6.6% | **42%** | - |
| 订阅成本占比 | <2% | **12%** | +10% |
| 生命真实性 | ⚠️ 中等 | ✅ **最高** | 显著提升 |

### 8.2 核心结论

1. **100% LLM驱动仍低于竞品** - $0.83 vs Aivilization $2.00
2. **订阅模式完全可承受** - 成本占收入12%
3. **优化空间大** - 智能触发+缓存可再降47%
4. **生命质量优先** - 这是产品核心差异化

### 8.3 推荐配置

| 配置项 | 推荐值 | 理由 |
|--------|--------|------|
| 默认模型 | Nova Lite | 日常思考 |
| 社交模型 | Nova Pro | 关键互动 |
| 进化模型 | Claude Sonnet | 极少使用 |
| 决策频率 | 智能触发（~90秒） | 平衡成本和体验 |
| 缓存启用 | 是（10分钟TTL） | 节省20% |
| 日预算 | $3.00/100Pet | 安全边际 |

### 8.4 规则引擎的新角色

**不做决策，做护栏和调度**：

```typescript
// 安全护栏（规则引擎）
class SafetyGuard {
  preventSelfAwareness(response: string): string;  // 防觉醒
  filterHallucination(response: string): string;   // 防幻觉
  keepInWorldview(response: string): string;       // 防越界
}

// 系统调度（规则引擎）
class SystemScheduler {
  shouldTriggerThinking(pet: Pet): boolean;        // 何时触发思考
  matchInteractionPartners(location): PetPair[];   // 谁跟谁互动
  prioritizeAgentPool(pets: Pet[]): Pet[];         // 资源分配
}
```

### 8.5 下一步

1. ✅ Dev实现模型分层选择器
2. ✅ Dev实现智能决策触发
3. ✅ Dev实现安全护栏系统
4. ✅ Dev实现成本监控系统

---

*Intel 成本估算模型 v2.0 完成*
*方向：100% LLM驱动 = 真正的数字生命*
