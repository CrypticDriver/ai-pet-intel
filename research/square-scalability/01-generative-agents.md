# Generative Agents 论文笔记

**Intel 研究文档** 🔍  
**日期**: 2026-03-01  
**论文**: Generative Agents: Interactive Simulacra of Human Behavior  
**作者**: Joon Sung Park et al. (Stanford University)  
**来源**: arXiv:2304.03442, ACM UIST 2023

---

## 1. 论文核心贡献

**创造了"生成式Agent"**：能够模拟可信人类行为的计算软件Agent

**Agent能做什么**：
- 醒来、做早餐、去上班
- 艺术家画画、作家写作
- 形成观点、注意到彼此、发起对话
- 记住并反思过去的日子
- 为明天制定计划

---

## 2. 核心架构：三大组件

```
感知 (Perception)
    ↓
记忆流 (Memory Stream) ← 存储所有经历
    ↓
检索 (Retrieval) → 反思 (Reflection) → 规划 (Planning)
    ↓
行动 (Action)
```

---

## 3. 记忆流 (Memory Stream)

### 3.1 定义
**Agent的经历数据库**，所有记录和推理都用自然语言。

### 3.2 记忆对象结构
```typescript
interface MemoryRecord {
  content: string;           // 文本内容
  createdAt: Date;          // 创建时间
  lastAccessedAt: Date;     // 最后访问时间
}
```

### 3.3 记忆类型

**观察 (Observation)**：
- Agent直接感知到的事物
- 自己的行为或观察到的其他Agent/物体行为

**示例**（咖啡店员工Isabella Rodriguez）：
```
(1) Isabella Rodriguez正在摆放糕点
(2) Maria Lopez正在喝咖啡并为化学考试学习
(3) Isabella和Maria正在讨论在Hobbs咖啡馆举办情人节派对
(4) 冰箱是空的
```

### 3.4 记忆检索函数

**输入**：Agent当前情况  
**输出**：相关记忆子集（传给LLM）

**评分因素**：

| 因素 | 计算方式 | 说明 |
|------|----------|------|
| **新近度 (Recency)** | 指数衰减 | 最近访问的记忆得分高 |
| **重要性 (Importance)** | LLM评分 1-10 | 1=琐事（刷牙），10=大事（离婚/入学） |
| **相关性 (Relevance)** | 余弦相似度 | 查询与记忆的embedding相似度 |

**综合评分**：
```
score = α × recency + β × importance + γ × relevance
// 论文中三者权重相等
```

---

## 4. 反思 (Reflection)

### 4.1 定义
**更高层次的抽象记忆**，由Agent自己生成，也存储在记忆流中。

### 4.2 触发时机
当最近事件的**重要性得分总和**超过阈值时触发。
**实际频率**：每天约2-3次。

### 4.3 生成过程

**步骤1：提取关键问题**
```
提示词：
"根据以上信息，关于这些陈述中的主体，
我们能回答的3个最重要的高层问题是什么？"
```

**步骤2：生成洞察**
```
提示词：
"从以上陈述中，你能推断出什么5个高层洞察？
（示例格式：洞察（因为1, 5, 3））"
```

**输出示例**：
```
"Klaus Mueller专注于他的士绅化研究（因为1, 2, 8, 15）"
```

### 4.4 反思的层级
反思可以基于之前的反思生成 → 形成层级结构

---

## 5. 规划 (Planning)

### 5.1 为什么需要规划？
**问题**：没有规划，LLM可能建议：
- 12:00 吃午饭
- 12:30 再吃午饭
- 13:00 又吃午饭

**规划确保行为的一致性和可信度**

### 5.2 规划方法：自顶向下递归

```
每日大计划（5-8点）
    ↓
小时级细化
    ↓
5-15分钟级行动
```

### 5.3 规划基础
- Agent的基本描述
- 前一天经历的总结

### 5.4 计划调整
Agent持续感知世界：
- 如果观察到重要事物 → 可能改变计划
- 如果与其他Agent互动 → 对话后可能重新规划

---

## 6. 环境：Smallville

### 6.1 设置
- 25个Agent
- 模拟小镇：房屋、大学、商店、公园、咖啡馆
- 2D sprite游戏（基于Phaser.js）

### 6.2 Agent初始化

每个Agent用一段文字描述身份：
```
"John Lin是Willow Market药店的店员，喜欢帮助别人。
他和妻子Mei Lin（大学教授）及儿子Eddy Lin（音乐理论学生）住在一起。
John Lin非常爱他的家人。
John Lin认识隔壁的老夫妇Sam Moore和Jennifer Moore好几年了。
John Lin认为Sam Moore是一个善良的人。
John Lin和Tom Moreno是同事，他们是朋友，喜欢一起讨论本地政治..."
```

### 6.3 行动表示
```
Agent输出："Isabella Rodriguez正在写日记"
         → 翻译成环境中的行动
         → 翻译成emoji显示在角色头上
```

---

## 7. 涌现行为 (Emergent Behavior)

### 7.1 信息扩散
**初始**：只有发起者知道  
**两天后**：调查所有Agent

| 话题 | 初始知情率 | 最终知情率 |
|------|------------|------------|
| 市长选举提名 | 4% | 32% |
| 情人节派对 | 4% | 48% |

### 7.2 关系形成
模拟开始和结束时询问Agent是否认识彼此 → 关系密度显著增加

### 7.3 协调能力
**情人节派对测试**：
- 12个受邀Agent
- 5个准时到场
- 3个表示有冲突
- 4个表示有兴趣但没计划

---

## 8. 局限性

### 8.1 记忆合成困难
大量记忆难以正确综合，尤其是Agent有庞大世界模型时

### 8.2 行为适当性判断错误
- 认为宿舍浴室可容纳多人
- 去已关门的商店

**可能解决方案**：更明确地在环境描述中添加规范

### 8.3 过于礼貌
- 对话过于正式
- Agent很少说"不"
- 可能是instruction tuning导致

---

## 9. 成本与规模

**25个Agent，2天游戏时间**：
- 花费：数千美元
- 真实时间：多天

---

## 10. 对AI Pet项目的应用

### 10.1 记忆流实现

```typescript
interface PetMemoryRecord {
  id: string;
  content: string;           // "我和豆豆在广场玩耍"
  type: 'observation' | 'reflection';
  importance: number;        // 1-10
  embedding: number[];       // 向量嵌入
  createdAt: Date;
  lastAccessedAt: Date;
}

function retrieveMemories(pet: Pet, query: string, limit: number = 10): PetMemoryRecord[] {
  const queryEmbedding = embed(query);
  
  return pet.memories
    .map(m => ({
      memory: m,
      score: 
        recencyScore(m.lastAccessedAt) * 0.33 +
        m.importance / 10 * 0.33 +
        cosineSimilarity(m.embedding, queryEmbedding) * 0.33
    }))
    .sort((a, b) => b.score - a.score)
    .slice(0, limit)
    .map(r => r.memory);
}
```

### 10.2 反思系统

```typescript
async function generateReflection(pet: Pet): Promise<void> {
  // 触发条件：最近记忆重要性总和 > 阈值
  const recentImportance = pet.memories
    .filter(m => isRecent(m.createdAt, 24 * 60 * 60 * 1000))
    .reduce((sum, m) => sum + m.importance, 0);
  
  if (recentImportance < REFLECTION_THRESHOLD) return;
  
  const recent100 = pet.memories.slice(-100);
  
  // 生成洞察
  const reflection = await llm.chat({
    model: 'nova-lite',
    messages: [{
      role: 'system',
      content: `你是${pet.name}，一只PixelVerse的Pix。
根据以下最近经历，总结3个重要的洞察：
${recent100.map(m => m.content).join('\n')}`
    }]
  });
  
  // 存储反思
  pet.memories.push({
    content: reflection,
    type: 'reflection',
    importance: 8, // 反思通常重要
    createdAt: new Date(),
    lastAccessedAt: new Date()
  });
}
```

### 10.3 规划系统

```typescript
async function generateDailyPlan(pet: Pet): Promise<Plan> {
  const yesterdaySummary = summarizeYesterday(pet);
  
  // 生成高层计划
  const highLevelPlan = await llm.chat({
    messages: [{
      role: 'system',
      content: `你是${pet.name}。
昨天的总结：${yesterdaySummary}
你的个性：${pet.personality}
为今天生成5-8个计划点。`
    }]
  });
  
  // 细化到小时
  const hourlyPlan = await refinePlanToHourly(highLevelPlan);
  
  // 细化到5-15分钟
  const detailedPlan = await refinePlanToDetailed(hourlyPlan);
  
  return detailedPlan;
}
```

### 10.4 社交关系形成

```typescript
// 关键：Pet通过互动自然形成关系
// 不是预设的关系图，而是涌现的

async function afterInteraction(pet1: Pet, pet2: Pet, conversation: string): Promise<void> {
  // 更新记忆
  pet1.memories.push({
    content: `我和${pet2.name}聊了：${conversation}`,
    type: 'observation',
    importance: assessImportance(conversation),
    createdAt: new Date()
  });
  
  // 可能触发反思
  await maybeReflect(pet1);
  
  // 关系强度自然增长（通过记忆检索时的相关性体现）
}
```

---

## 11. 关键洞察总结

| 组件 | 作用 | 对AI Pet的意义 |
|------|------|----------------|
| **记忆流** | 存储所有经历 | Pet记住与用户/其他Pet的互动 |
| **检索** | 找出相关记忆 | Pet能"想起"相关的过去 |
| **反思** | 形成高层洞察 | Pet发展出个性和偏好 |
| **规划** | 确保行为一致 | Pet不会重复无意义的行为 |

---

## 12. 下一步

- [ ] 分析AI Town代码实现
- [ ] 研究成本优化（Nova Lite替代GPT-3.5）
- [ ] 设计社交网络图结构
- [ ] 规划MVP实现范围

---

*Intel Generative Agents论文笔记完成*
