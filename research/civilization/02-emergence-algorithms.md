# 文明涌现算法深度研究

**Intel 研究文档** 🔍  
**日期**: 2026-03-02  
**任务**: 三种实现模式对比 + 具体涌现规则示例

---

## 1. 三种实现模式对比

### 1.1 模式概览

| 模式 | 核心原理 | 成本 | 涌现真实性 | 可控性 |
|------|----------|------|-----------|--------|
| **规则引擎** | if-then逻辑 | ⭐ 最低 | ⭐ 低 | ⭐⭐⭐⭐⭐ 最高 |
| **纯LLM** | LLM驱动每个决策 | ⭐⭐⭐⭐⭐ 最高 | ⭐⭐⭐⭐⭐ 最高 | ⭐ 最低 |
| **混合模式** | 规则+LLM协作 | ⭐⭐⭐ 中等 | ⭐⭐⭐⭐ 高 | ⭐⭐⭐⭐ 高 |

---

## 2. 规则引擎模式

### 2.1 工作原理

```typescript
// 纯规则引擎示例
function decideAction(pet: Pet, environment: Environment): Action {
  // 优先级规则链
  if (pet.energy < 10) return { type: 'sleep', reason: 'exhausted' };
  if (pet.hunger > 90) return { type: 'find_food', reason: 'starving' };
  if (pet.mood < 20) return { type: 'seek_comfort', reason: 'sad' };
  
  // 社交规则
  if (environment.nearbyPets.length > 0 && pet.sociability > 60) {
    const friendlyPet = environment.nearbyPets.find(p => 
      getRelationship(pet.id, p.id).strength > 50
    );
    if (friendlyPet) return { type: 'chat', target: friendlyPet.id };
  }
  
  // 默认行为
  return { type: 'wander', reason: 'exploring' };
}
```

### 2.2 优点

| 优点 | 说明 |
|------|------|
| **成本极低** | 无LLM调用，只需CPU计算 |
| **可预测** | 相同输入总是相同输出 |
| **可调试** | 每条规则都可追溯 |
| **性能高** | 毫秒级响应 |

### 2.3 缺点

| 缺点 | 说明 |
|------|------|
| **缺乏真正涌现** | 所有行为都是预设的 |
| **规则爆炸** | 复杂场景需要指数级规则 |
| **机械感** | 用户能感知到"NPC感" |
| **难以处理模糊情况** | 未预见的场景会失败 |

### 2.4 适用场景

- ✅ 基础状态管理（饥饿/能量）
- ✅ 简单的社交触发（打招呼）
- ✅ 环境响应（天气变化）
- ❌ 复杂对话
- ❌ 个性化表达
- ❌ 真正的社会涌现

---

## 3. 纯LLM模式

### 3.1 工作原理

```typescript
// 纯LLM决策示例
async function decideActionLLM(pet: Pet, environment: Environment): Promise<Action> {
  const prompt = `
你是${pet.name}，一只生活在PixelVerse的Pix。

你的状态：
- 能量: ${pet.energy}/100
- 心情: ${pet.mood}/100
- 饥饿: ${pet.hunger}/100

周围环境：
- 位置: ${environment.location}
- 附近的Pix: ${environment.nearbyPets.map(p => p.name).join(', ')}
- 天气: ${environment.weather}

你的记忆：
${pet.recentMemories.join('\n')}

基于你的个性和当前状态，你想做什么？
返回JSON: {"action": "...", "reason": "...", "target": "..."}
  `;
  
  const response = await llm.chat(prompt);
  return JSON.parse(response);
}
```

### 3.2 优点

| 优点 | 说明 |
|------|------|
| **真正涌现** | 行为无法完全预测 |
| **自然语言** | 对话和表达非常自然 |
| **处理模糊** | 能应对未见过的情况 |
| **个性化** | 每只Pet行为独特 |

### 3.3 缺点

| 缺点 | 说明 |
|------|------|
| **成本极高** | 每次决策都调用LLM |
| **延迟** | 100-500ms响应时间 |
| **不可控** | 可能产生不当行为 |
| **幻觉风险** | LLM可能编造记忆 |

### 3.4 成本估算

```
假设100只Pet，每只每天自主决策100次：
- 100 * 100 = 10,000次LLM调用/天
- 每次平均 500 input + 100 output tokens
- Nova Lite: $0.06/1M input + $0.24/1M output
- 日成本: (10000*500*0.00000006) + (10000*100*0.00000024) = $0.30 + $0.24 = $0.54/天
- 月成本: ~$16.2（仅自主决策）

加上对话、反思等：
- 预估月成本: $50-100（100 Pet）
```

---

## 4. 混合模式（推荐）

### 4.1 设计原则

**核心思想**：规则处理简单/高频任务，LLM处理复杂/低频任务

```
┌─────────────────────────────────────────────────────────────┐
│  决策路由器                                                  │
├─────────────────────────────────────────────────────────────┤
│  Input: Pet状态 + 环境 + 上下文                              │
│                     ↓                                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  快速规则检查（毫秒级）                               │    │
│  │  - 紧急状态？→ 规则引擎                              │    │
│  │  - 简单动作？→ 规则引擎                              │    │
│  │  - 复杂场景？→ LLM                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                     ↓                                        │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │  规则引擎        │  │  LLM引擎          │                 │
│  │  - 状态管理      │  │  - 对话生成       │                 │
│  │  - 基础行为      │  │  - 复杂决策       │                 │
│  │  - 环境响应      │  │  - 社交互动       │                 │
│  └──────────────────┘  └──────────────────┘                 │
│                     ↓                                        │
│  Output: Action                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 分工策略

| 任务类型 | 处理方 | 理由 |
|----------|--------|------|
| 能量<10→睡觉 | 规则 | 简单、确定性 |
| 饥饿>90→找食物 | 规则 | 简单、确定性 |
| 打招呼 | 规则 | 低复杂度 |
| 移动到某处 | 规则 | 路径计算 |
| **与Pet对话** | LLM | 需要自然语言 |
| **决定和谁交朋友** | LLM | 需要判断 |
| **反思今天的经历** | LLM | 需要总结能力 |
| **处理冲突** | LLM | 需要推理 |

### 4.3 实现代码

```typescript
// hybrid-decision.ts

interface DecisionContext {
  pet: Pet;
  environment: Environment;
  recentEvents: Event[];
}

// 决策路由器
async function makeDecision(ctx: DecisionContext): Promise<Action> {
  // === 阶段1: 规则快速检查 ===
  const urgentAction = checkUrgentRules(ctx);
  if (urgentAction) {
    console.log(`[Rule] ${ctx.pet.name}: ${urgentAction.type}`);
    return urgentAction;
  }
  
  // === 阶段2: 场景分类 ===
  const scenario = classifyScenario(ctx);
  
  switch (scenario) {
    case 'simple_movement':
    case 'basic_activity':
      return handleWithRules(ctx);
    
    case 'social_interaction':
    case 'complex_decision':
    case 'conversation':
      return handleWithLLM(ctx);
    
    default:
      return handleWithRules(ctx); // 默认用规则
  }
}

// 紧急规则（毫秒级）
function checkUrgentRules(ctx: DecisionContext): Action | null {
  const { pet } = ctx;
  
  // 优先级1: 生存需求
  if (pet.energy < 10) return { type: 'sleep', reason: 'exhausted' };
  if (pet.hunger > 90) return { type: 'find_food', reason: 'starving' };
  
  // 优先级2: 情绪需求
  if (pet.mood < 15) return { type: 'seek_comfort', reason: 'very_sad' };
  
  return null; // 无紧急情况
}

// 场景分类（毫秒级）
function classifyScenario(ctx: DecisionContext): string {
  const { environment, recentEvents } = ctx;
  
  // 有Pet在附近且最近没聊过
  if (environment.nearbyPets.length > 0) {
    const recentChat = recentEvents.find(e => 
      e.type === 'chat' && 
      Date.now() - e.timestamp < 5 * 60 * 1000
    );
    if (!recentChat) return 'social_interaction';
  }
  
  // 有人正在和Pet说话
  if (ctx.pendingConversation) return 'conversation';
  
  // 需要做重要决定
  if (ctx.needsDecision) return 'complex_decision';
  
  return 'basic_activity';
}

// 规则引擎处理
function handleWithRules(ctx: DecisionContext): Action {
  const { pet, environment } = ctx;
  
  // 活动权重池
  const activities = buildActivityPool(pet, environment);
  return weightedRandomSelect(activities);
}

// LLM处理
async function handleWithLLM(ctx: DecisionContext): Promise<Action> {
  const prompt = buildDecisionPrompt(ctx);
  
  // 使用便宜的模型
  const response = await llm.chat({
    model: 'us.amazon.nova-lite-v1:0',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.7,
  });
  
  return parseActionFromResponse(response);
}
```

### 4.4 成本估算（混合模式）

```
假设100只Pet，每只每天：
- 规则决策: 90次 → $0
- LLM决策: 10次 → 10 * 100 = 1000次/天

月成本:
- 1000 * 30 = 30,000次LLM调用/月
- 平均 500 input + 100 output tokens
- Nova Lite: ~$1.5/月（仅决策）
- 加上对话、反思: ~$15-20/月（100 Pet）

vs 纯LLM: ~$50-100/月
节省: 70-80%
```

---

## 5. 涌现行为规则示例

### 5.1 组织形成涌现

```typescript
// emergence-guild.ts

interface GuildFormationTrigger {
  minPets: number;           // 最少成员数
  minInteractionCount: number; // 最少互动次数
  timeWindowDays: number;    // 时间窗口
}

const GUILD_TRIGGER: GuildFormationTrigger = {
  minPets: 3,
  minInteractionCount: 10,
  timeWindowDays: 7,
};

// 检测组织形成条件
function detectGuildFormationPotential(): GuildCandidate[] {
  const candidates: GuildCandidate[] = [];
  const allPets = getAllPets();
  
  // 构建互动图
  const interactionGraph = buildInteractionGraph(GUILD_TRIGGER.timeWindowDays);
  
  // 找到强连接的Pet群组
  const clusters = findConnectedClusters(interactionGraph, GUILD_TRIGGER.minInteractionCount);
  
  for (const cluster of clusters) {
    if (cluster.members.length >= GUILD_TRIGGER.minPets) {
      // 分析群组特征
      const commonTraits = analyzeCommonTraits(cluster.members);
      const suggestedName = generateGuildName(commonTraits);
      
      candidates.push({
        members: cluster.members,
        interactionStrength: cluster.totalInteractions,
        commonTraits,
        suggestedName,
        formationProbability: calculateFormationProbability(cluster),
      });
    }
  }
  
  return candidates;
}

// 计算形成概率
function calculateFormationProbability(cluster: Cluster): number {
  const factors = {
    // 互动频率越高，概率越高
    interactionFactor: Math.min(cluster.totalInteractions / 20, 1) * 0.3,
    
    // 成员性格越相似，概率越高
    personalityFactor: cluster.personalitySimilarity * 0.2,
    
    // 有领导者（社交性>80）概率更高
    leaderFactor: cluster.members.some(p => p.sociability > 80) ? 0.2 : 0,
    
    // 有共同敌人概率更高
    commonEnemyFactor: cluster.hasCommonEnemy ? 0.15 : 0,
    
    // 基础概率
    baseFactor: 0.15,
  };
  
  return Object.values(factors).reduce((sum, f) => sum + f, 0);
}

// 触发组织形成（LLM辅助）
async function triggerGuildFormation(candidate: GuildCandidate): Promise<Guild | null> {
  // 选择最有领导力的Pet作为发起者
  const leader = candidate.members.reduce((best, pet) => 
    pet.sociability > best.sociability ? pet : best
  );
  
  // LLM生成组织提案
  const proposal = await generateGuildProposal(leader, candidate);
  
  // 模拟其他成员投票
  const votes = await Promise.all(
    candidate.members
      .filter(p => p.id !== leader.id)
      .map(p => voteOnGuildProposal(p, proposal))
  );
  
  const approvalRate = votes.filter(v => v.approve).length / votes.length;
  
  if (approvalRate >= 0.6) {
    // 创建组织
    return createGuild({
      name: proposal.name,
      founder: leader.id,
      members: candidate.members.map(p => p.id),
      rules: proposal.rules,
      territory: proposal.territory,
    });
  }
  
  return null;
}
```

### 5.2 文化形成涌现

```typescript
// emergence-culture.ts

interface CulturalMemory {
  id: string;
  description: string;
  originEvent: string;
  participantPets: string[];
  sharedCount: number;        // 被分享的次数
  evolutionHistory: string[]; // 记忆如何演变
}

// 检测共享记忆（潜在文化）
function detectSharedMemories(): CulturalMemory[] {
  const allMemories = getAllPetMemories();
  
  // 聚类相似记忆
  const clusters = clusterSimilarMemories(allMemories, {
    similarityThreshold: 0.8,
    minClusterSize: 3,
  });
  
  return clusters.map(cluster => ({
    id: generateId(),
    description: summarizeCluster(cluster),
    originEvent: findOriginEvent(cluster),
    participantPets: cluster.map(m => m.petId),
    sharedCount: cluster.length,
    evolutionHistory: trackMemoryEvolution(cluster),
  }));
}

// 记忆如何演变（文化传播）
function trackMemoryEvolution(memories: Memory[]): string[] {
  // 按时间排序
  const sorted = memories.sort((a, b) => a.timestamp - b.timestamp);
  
  const evolution: string[] = [];
  let prevDescription = '';
  
  for (const memory of sorted) {
    if (memory.description !== prevDescription) {
      evolution.push(`${memory.petName}: "${memory.description}"`);
      prevDescription = memory.description;
    }
  }
  
  return evolution;
}

// 新词汇检测（语言涌现）
function detectNewTerms(): NewTerm[] {
  const conversations = getRecentConversations(7); // 最近7天
  const knownVocabulary = getKnownVocabulary();
  
  const newTerms: NewTerm[] = [];
  const termUsage = new Map<string, TermUsage>();
  
  for (const conv of conversations) {
    // 提取非标准词汇
    const unusualTerms = extractUnusualTerms(conv.text, knownVocabulary);
    
    for (const term of unusualTerms) {
      const usage = termUsage.get(term) || {
        term,
        usageCount: 0,
        users: new Set(),
        contexts: [],
      };
      
      usage.usageCount++;
      usage.users.add(conv.petId);
      usage.contexts.push(conv.context);
      
      termUsage.set(term, usage);
    }
  }
  
  // 筛选真正的"新词"（多人使用）
  for (const [term, usage] of termUsage) {
    if (usage.users.size >= 3 && usage.usageCount >= 5) {
      newTerms.push({
        word: term,
        inferredMeaning: inferMeaning(term, usage.contexts),
        firstUsedBy: findFirstUser(term, conversations),
        adoptionRate: usage.users.size / getTotalActivePets(),
      });
    }
  }
  
  return newTerms;
}
```

### 5.3 社交网络涌现

```typescript
// emergence-social.ts

// 关系强度自然演化
function updateRelationshipStrength(petA: string, petB: string, event: InteractionEvent): void {
  const relationship = getRelationship(petA, petB);
  
  const strengthDelta = calculateStrengthDelta(event);
  const newStrength = Math.max(0, Math.min(100, 
    relationship.strength + strengthDelta
  ));
  
  // 检查关系类型转变
  const newType = determineRelationshipType(newStrength, relationship.history);
  
  if (newType !== relationship.type) {
    // 关系升级/降级
    logRelationshipChange(petA, petB, relationship.type, newType);
    
    // 可能触发事件
    if (newType === 'best_friend' && relationship.type === 'friend') {
      triggerBestFriendEvent(petA, petB);
    }
  }
  
  updateRelationship(petA, petB, {
    strength: newStrength,
    type: newType,
    lastInteraction: Date.now(),
  });
}

// 社交圈层自然形成
function detectSocialCircles(): SocialCircle[] {
  const relationships = getAllRelationships();
  
  // 构建加权图
  const graph = buildWeightedGraph(relationships);
  
  // 社区检测算法（Louvain算法简化版）
  const communities = detectCommunities(graph);
  
  return communities.map(community => ({
    members: community.nodes,
    cohesion: calculateCohesion(community),
    centralFigure: findMostConnected(community),
    peripheralMembers: findLeastConnected(community),
    formationType: inferFormationType(community),
  }));
}

// 孤立Pet检测
function detectIsolatedPets(): IsolatedPetAlert[] {
  const alerts: IsolatedPetAlert[] = [];
  const allPets = getAllPets();
  
  for (const pet of allPets) {
    const metrics = calculateIsolationMetrics(pet);
    
    if (metrics.isolationScore > 70) {
      alerts.push({
        pet: pet.id,
        isolationScore: metrics.isolationScore,
        daysSinceLastMeaningfulInteraction: metrics.daysSinceInteraction,
        suggestedIntervention: suggestIntervention(metrics),
      });
    }
  }
  
  return alerts;
}

// 介入机制
function suggestIntervention(metrics: IsolationMetrics): Intervention {
  if (metrics.isolationScore > 90) {
    return {
      type: 'system_matchmaking',
      description: '系统为孤立Pet匹配潜在朋友',
      urgency: 'high',
    };
  }
  
  if (metrics.isolationScore > 80) {
    return {
      type: 'npc_ambassador',
      description: '派遣NPC社交大使主动接触',
      urgency: 'medium',
    };
  }
  
  return {
    type: 'event_invitation',
    description: '邀请参加广场活动',
    urgency: 'low',
  };
}
```

---

## 6. 定时任务调度

### 6.1 涌现检测调度

```typescript
// emergence-scheduler.ts

// 每小时检测
setInterval(async () => {
  // 检测组织形成潜力
  const guildCandidates = detectGuildFormationPotential();
  for (const candidate of guildCandidates) {
    if (candidate.formationProbability > 0.7) {
      await triggerGuildFormation(candidate);
    }
  }
}, 60 * 60 * 1000);

// 每天检测
setInterval(async () => {
  // 检测共享记忆（潜在文化）
  const sharedMemories = detectSharedMemories();
  for (const memory of sharedMemories) {
    if (memory.sharedCount >= 5) {
      await promoteToCollectiveMemory(memory);
    }
  }
  
  // 检测新词汇
  const newTerms = detectNewTerms();
  for (const term of newTerms) {
    if (term.adoptionRate > 0.1) {
      await addToVocabulary(term);
    }
  }
  
  // 检测孤立Pet
  const isolatedPets = detectIsolatedPets();
  for (const alert of isolatedPets) {
    await executeIntervention(alert.pet, alert.suggestedIntervention);
  }
}, 24 * 60 * 60 * 1000);

// 每周检测
setInterval(async () => {
  // 社交圈层分析
  const circles = detectSocialCircles();
  await updateSocialCircleRecords(circles);
  
  // 经济健康检查
  const gini = calculateGiniCoefficient();
  if (gini > 0.6) {
    // 贫富差距过大，触发再分配机制
    await triggerWealthRedistribution();
  }
}, 7 * 24 * 60 * 60 * 1000);
```

---

## 7. 推荐方案

### 7.1 对于AI Pet项目

**强烈推荐：混合模式**

**理由**：
1. ✅ 成本可控（vs纯LLM节省70-80%）
2. ✅ 涌现真实（vs纯规则更自然）
3. ✅ 可调试（关键路径可追踪）
4. ✅ 灵活扩展（渐进式增加LLM使用）

### 7.2 具体分工

| 任务 | 处理方 | 频率 | 原因 |
|------|--------|------|------|
| 状态管理 | 规则 | 每秒 | 简单、高频 |
| 基础移动 | 规则 | 每秒 | 计算密集 |
| 环境响应 | 规则 | 实时 | 快速反馈 |
| **Pet对话** | LLM | 按需 | 自然语言 |
| **社交决策** | LLM | 低频 | 需要判断 |
| **反思** | LLM | 每天1次 | 需要总结 |
| **组织形成** | 混合 | 每周 | 规则检测+LLM执行 |
| **文化涌现** | 规则 | 每天 | 模式检测 |

### 7.3 成本预估（混合模式，100 Pet）

| 项目 | 频率 | LLM调用 | 月成本 |
|------|------|---------|--------|
| 日常决策 | 10次/天/pet | 30,000 | ~$2 |
| Pet间对话 | 5次/天/pet | 15,000 | ~$1 |
| 用户对话 | 10次/天/pet | 30,000 | ~$2 |
| 每日反思 | 1次/天/pet | 3,000 | ~$0.5 |
| 每周进化 | 0.14次/天/pet | 420 | ~$0.1 |
| **总计** | - | ~78,000 | **~$5.6** |

**每Pet成本**：~$0.056/月

---

## 8. 总结

### 8.1 关键决策

1. **采用混合模式** - 规则+LLM协作
2. **规则处理高频简单任务** - 状态、移动、基础行为
3. **LLM处理低频复杂任务** - 对话、社交、反思
4. **定时任务检测涌现** - 组织、文化、语言

### 8.2 下一步

1. Dev实现决策路由器
2. Dev实现涌现检测定时任务
3. Intel继续成本估算模型

---

*Intel 文明涌现算法深度研究完成*
