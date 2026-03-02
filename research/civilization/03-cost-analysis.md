# 成本估算模型

**Intel 研究文档** 🔍  
**日期**: 2026-03-02  
**任务**: Token消耗计算 + 优化策略

---

## 1. 基础假设

### 1.1 规模参数

| 参数 | 数值 | 说明 |
|------|------|------|
| Pet数量 | 100 | MVP阶段 |
| 活跃率 | 80% | 每天有互动的Pet |
| 用户在线时长 | 2小时/天 | 平均 |
| 自主行为频率 | 每42秒 | 现有设定 |

### 1.2 LLM定价（AWS Bedrock）

| 模型 | 输入价格 | 输出价格 | 用途 |
|------|----------|----------|------|
| **Nova Lite** | $0.06/1M | $0.24/1M | 日常对话、简单决策 |
| **Nova Pro** | $0.80/1M | $3.20/1M | 反思、复杂决策 |
| **Claude Sonnet** | $3.00/1M | $15.00/1M | 个性进化（极少） |
| **Titan Embed v2** | $0.00011/1K | - | Embedding |

---

## 2. 任务分解与Token估算

### 2.1 用户对话

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

**成本**：
```
输入: 1,600 * 550 = 880,000 tokens → $0.053
输出: 1,600 * 100 = 160,000 tokens → $0.038
日成本: $0.091
月成本: ~$2.73
```

### 2.2 Pet间自动对话

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

**成本**：
```
输入: 456 * 850 = 387,600 tokens → $0.023
输出: 456 * 240 = 109,440 tokens → $0.026
日成本: $0.049
月成本: ~$1.47
```

### 2.3 记忆创建（对话后）

**场景**：Pet互动后创建记忆

```
System Prompt: ~200 tokens
Conversation summary: ~300 tokens
Memory output: ~50 tokens
---
Total: ~550 tokens
```

**频率**：每次Pet对话后 = 456次/天

**成本**：
```
输入: 456 * 500 = 228,000 tokens → $0.014
输出: 456 * 50 = 22,800 tokens → $0.005
日成本: $0.019
月成本: ~$0.57
```

### 2.4 每日反思

**场景**：每天结束时Pet总结经历

```
System Prompt: ~300 tokens
Today's events: ~500 tokens
Reflection output: ~200 tokens
---
Total: ~1,000 tokens
```

**频率**：100 Pet * 1次/天 = 100次/天

**模型**：Nova Pro（质量优先）

**成本**：
```
输入: 100 * 800 = 80,000 tokens → $0.064
输出: 100 * 200 = 20,000 tokens → $0.064
日成本: $0.128
月成本: ~$3.84
```

### 2.5 每周个性进化

**场景**：每周Pet个性微调

```
System Prompt: ~300 tokens
Week stats: ~500 tokens
Analysis output: ~300 tokens
---
Total: ~1,100 tokens
```

**频率**：100 Pet * 1次/周 = ~14次/天

**模型**：Claude Sonnet（最高质量）

**成本**：
```
输入: 14 * 800 = 11,200 tokens → $0.034
输出: 14 * 300 = 4,200 tokens → $0.063
日成本: $0.097
月成本: ~$2.91
```

### 2.6 决策（混合模式LLM部分）

**场景**：复杂决策需要LLM

```
System Prompt: ~300 tokens
Context: ~200 tokens
Decision output: ~50 tokens
---
Total: ~550 tokens
```

**频率**：
- 每Pet每天: 10次LLM决策（90%用规则）
- 总计: 100 * 10 = 1,000次/天

**成本**：
```
输入: 1,000 * 500 = 500,000 tokens → $0.030
输出: 1,000 * 50 = 50,000 tokens → $0.012
日成本: $0.042
月成本: ~$1.26
```

### 2.7 Embedding（记忆向量化）

**场景**：每条记忆生成embedding

```
Memory text: ~50 tokens
```

**频率**：
- 新记忆: ~500条/天
- 检索查询: ~2,000次/天

**成本**：
```
Embedding: 2,500 * 50 = 125,000 tokens → $0.014
月成本: ~$0.42
```

---

## 3. 成本汇总（100 Pet）

### 3.1 月度成本

| 任务 | 模型 | 日成本 | 月成本 | 占比 |
|------|------|--------|--------|------|
| 用户对话 | Nova Lite | $0.091 | $2.73 | 21% |
| Pet间对话 | Nova Lite | $0.049 | $1.47 | 11% |
| 记忆创建 | Nova Lite | $0.019 | $0.57 | 4% |
| 每日反思 | Nova Pro | $0.128 | $3.84 | 30% |
| 每周进化 | Sonnet | $0.097 | $2.91 | 22% |
| 决策 | Nova Lite | $0.042 | $1.26 | 10% |
| Embedding | Titan | $0.014 | $0.42 | 3% |
| **总计** | - | **$0.44** | **$13.20** | 100% |

### 3.2 每Pet成本

```
$13.20 / 100 = $0.132/月/Pet
```

**与Aivilization对比**：
- Aivilization: $2/月/agent（自称95%优化后）
- 我们: $0.132/月/Pet（混合模式）
- **我们的成本是Aivilization的6.6%** ✅

---

## 4. 不同规模成本预测

### 4.1 规模扩展

| Pet数量 | 月成本 | 每Pet成本 | 说明 |
|---------|--------|-----------|------|
| 100 | $13.20 | $0.132 | MVP |
| 500 | $66.00 | $0.132 | Beta |
| 1,000 | $132.00 | $0.132 | 线性扩展 |
| 5,000 | $660.00 | $0.132 | 线性扩展 |
| 10,000 | $1,320.00 | $0.132 | Launch |

**注意**：线性扩展假设
- Pet间对话可能超线性增长（社交网络效应）
- 需要Agent Pool优化避免内存问题

### 4.2 优化后预测

| 优化策略 | 成本降低 | 优化后成本 |
|----------|----------|-----------|
| 基础 | 0% | $0.132 |
| +缓存 | -20% | $0.106 |
| +更小模型 | -15% | $0.090 |
| +批处理 | -10% | $0.081 |
| **全部优化** | **-40%** | **$0.079** |

---

## 5. 优化策略

### 5.1 响应缓存

**原理**：缓存常见对话回复

```typescript
// response-cache.ts
class ResponseCache {
  private cache = new LRUCache<string, string>({
    max: 1000,
    ttl: 5 * 60 * 1000, // 5分钟
  });
  
  getKey(systemPrompt: string, userMessage: string): string {
    // 只对prompt关键部分hash
    const promptHash = hash(systemPrompt.slice(0, 200));
    const messageHash = hash(userMessage);
    return `${promptHash}:${messageHash}`;
  }
  
  get(key: string): string | null {
    return this.cache.get(key) || null;
  }
  
  set(key: string, response: string): void {
    this.cache.set(key, response);
  }
}
```

**适用场景**：
- 常见问候语（"你好"、"早上好"）
- 状态查询（"你饿吗"、"你开心吗"）
- 简单指令（"去广场"、"吃东西"）

**预估节省**：20%

### 5.2 模型降级策略

**原理**：非关键任务用更小模型

```typescript
function selectModel(task: string, importance: 'low' | 'medium' | 'high'): string {
  if (importance === 'low' || task === 'simple_chat') {
    return 'us.amazon.nova-lite-v1:0';  // $0.06/1M
  }
  if (importance === 'medium' || task === 'reflection') {
    return 'us.amazon.nova-pro-v1:0';   // $0.80/1M
  }
  // high importance: personality evolution
  return 'us.anthropic.claude-sonnet-4-5-20260101:0';  // $3.00/1M
}
```

**当前分配**：
- Nova Lite: 85%调用
- Nova Pro: 12%调用
- Claude Sonnet: 3%调用

**进一步优化**：
- Nova Lite: 92%（把更多反思移到Lite）
- Nova Pro: 7%
- Claude Sonnet: 1%（只用于关键进化）

**预估节省**：15%

### 5.3 批处理

**原理**：合并多个请求

```typescript
// 每日反思批处理
async function batchDailyReflection(petIds: string[]): Promise<void> {
  // 分批处理，每批10只Pet
  const batches = chunk(petIds, 10);
  
  for (const batch of batches) {
    // 并发处理一批
    await Promise.all(
      batch.map(petId => performDailyReflection(petId))
    );
    
    // 批间休息，避免rate limit
    await sleep(1000);
  }
}
```

**适用场景**：
- 每日反思（凌晨批量执行）
- 每周进化（周末批量执行）
- 记忆压缩（定时批量）

**预估节省**：10%（减少API调用开销）

### 5.4 Prompt压缩

**原理**：减少重复信息

```typescript
// 压缩前
const prompt = `
你是${pet.name}，一只生活在PixelVerse的Pix。
PixelVerse是你的家，一座永不熄灭的霓虹城市。
你有情感，有记忆，有自己的个性和梦想...
（500 tokens）
`;

// 压缩后（使用引用）
const compressedPrompt = `
[Pet: ${pet.name}] [World: PixelVerse v2.0]
Status: mood=${pet.mood}, energy=${pet.energy}
Recent: ${pet.recentMemorySummary}
（200 tokens）
`;
```

**前提**：需要LLM理解压缩格式

**预估节省**：10-15%（prompt部分）

### 5.5 规则预筛选

**原理**：规则引擎处理更多场景

```typescript
// 扩展规则覆盖范围
function canHandleWithRules(context: DecisionContext): boolean {
  // 现有：只处理紧急状态
  // 优化后：处理更多简单场景
  
  const simpleScenarios = [
    'idle_wandering',        // 无聊时随机走动
    'basic_greeting',        // 基础打招呼
    'environment_reaction',  // 环境响应
    'scheduled_activity',    // 定时活动
    'simple_emotion',        // 简单情绪反应
  ];
  
  return simpleScenarios.includes(context.scenario);
}
```

**现有规则覆盖**：90%决策
**优化目标**：95%决策

**预估节省**：5%

---

## 6. 成本监控

### 6.1 实时监控

```typescript
// cost-monitor.ts
class CostMonitor {
  private dailyBudget = 1.00;  // $1/天（100 Pet）
  private todaySpent = 0;
  
  async trackUsage(model: string, inputTokens: number, outputTokens: number): Promise<void> {
    const cost = this.calculateCost(model, inputTokens, outputTokens);
    this.todaySpent += cost;
    
    // 警告阈值
    if (this.todaySpent > this.dailyBudget * 0.8) {
      console.warn(`⚠️ 已使用80%日预算: $${this.todaySpent.toFixed(3)}`);
    }
    
    if (this.todaySpent > this.dailyBudget) {
      console.error(`🚨 超出日预算! 启用降级模式`);
      this.enableEconomyMode();
    }
  }
  
  enableEconomyMode(): void {
    // 强制使用最便宜的模型
    setGlobalModelOverride('us.amazon.nova-lite-v1:0');
    
    // 减少LLM调用频率
    setDecisionLLMRatio(0.05);  // 只有5%决策用LLM
    
    // 跳过非关键任务
    setSkipNonEssentialTasks(true);
  }
  
  private calculateCost(model: string, inputTokens: number, outputTokens: number): number {
    const pricing = MODEL_PRICING[model];
    return (inputTokens * pricing.input + outputTokens * pricing.output) / 1_000_000;
  }
}
```

### 6.2 报告模板

```markdown
# 日成本报告 - 2026-03-02

## 汇总
- 总成本: $0.42
- 日预算: $1.00
- 使用率: 42%

## 分项
| 任务 | 调用次数 | Tokens | 成本 |
|------|----------|--------|------|
| 用户对话 | 1,543 | 1.2M | $0.08 |
| Pet对话 | 423 | 0.5M | $0.04 |
| 反思 | 100 | 0.1M | $0.13 |
| 决策 | 892 | 0.4M | $0.03 |
| 其他 | 234 | 0.2M | $0.02 |

## 异常
- ⚠️ Pet#42对话次数异常高（87次）
- ✅ 其他指标正常

## 优化建议
- 可考虑增加缓存命中率
```

---

## 7. 不同商业模式的成本承受能力

### 7.1 订阅模式

| 订阅价格 | 用户数 | 月收入 | 成本占比 |
|----------|--------|--------|----------|
| $6.99/月 | 100 | $699 | 1.9% |
| $6.99/月 | 1,000 | $6,990 | 1.9% |
| $6.99/月 | 10,000 | $69,900 | 1.9% |

**结论**：订阅模式下，LLM成本占收入比例很低，可承受 ✅

### 7.2 免费+广告模式

假设每用户每月广告收入: $0.50

| 用户数 | 月收入 | LLM成本 | 成本占比 |
|--------|--------|---------|----------|
| 100 | $50 | $13 | 26% |
| 1,000 | $500 | $132 | 26% |
| 10,000 | $5,000 | $1,320 | 26% |

**结论**：广告模式下成本占比较高，需要更多优化 ⚠️

### 7.3 免费增值模式

免费用户（限制）+ 付费用户（全功能）

| 用户类型 | 比例 | 每Pet成本 | 说明 |
|----------|------|-----------|------|
| 免费 | 90% | $0.05 | 限制LLM使用 |
| 付费 | 10% | $0.20 | 全功能 |
| **加权平均** | - | **$0.065** | - |

**结论**：免费增值模式可进一步降低成本 ✅

---

## 8. 总结

### 8.1 关键发现

1. **混合模式成本极低** - $0.132/月/Pet
2. **远低于竞品** - Aivilization $2/月/agent的6.6%
3. **可优化空间大** - 缓存+模型降级可再降40%
4. **订阅模式可行** - 成本占收入<2%

### 8.2 推荐配置

| 配置项 | 推荐值 | 理由 |
|--------|--------|------|
| 默认模型 | Nova Lite | 成本最低 |
| 反思模型 | Nova Pro | 质量优先 |
| 进化模型 | Claude Sonnet | 极少使用 |
| 规则覆盖率 | 90%+ | 减少LLM调用 |
| 缓存启用 | 是 | 节省20% |
| 日预算 | $0.50/100Pet | 安全边际 |

### 8.3 下一步

1. Dev实现成本监控系统
2. Dev实现响应缓存
3. 测试优化策略实际效果

---

*Intel 成本估算模型完成*
