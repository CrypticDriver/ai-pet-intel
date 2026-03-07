# 行动导向AI设计模式研究

**Intel 调研报告** 🔍  
**日期**: 2026-03-07  
**任务**: 解决"说与做脱节"问题的最佳实践

---

## 1. 问题诊断

### 1.1 老大反馈的核心问题

```
❌ Motes说了也不会做
❌ 说了也没法做
❌ 日记很虚
```

**本质**：**Say-Do Gap（说做差距）**

### 1.2 问题分解

| 症状 | 根本原因 |
|------|----------|
| Pet说"我要去广场" | LLM生成意图，但没有执行管道 |
| 但实际没去 | 缺少 Intention → Action → Execution 链 |
| 日记记录虚构内容 | 反思基于prompt，不是基于真实事件日志 |
| 聊天无后果 | 对话不写入记忆，不影响后续行为 |

---

## 2. 行业最佳实践

### 2.1 ReAct Framework（Reasoning + Acting）

**来源**：Yao et al., 2023 - "ReACT: Synergizing Reasoning and Acting in Language Models"

**核心思想**：LLM不应只生成想法，必须交替执行**思考→行动→观察**循环

```
┌─────────────────────────────────────────────────────────────┐
│  ReAct Loop                                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Thought: "我想去广场找朋友玩"                                │
│      ↓                                                      │
│  Action: go_to("square")                                    │
│      ↓                                                      │
│  Observation: "你已到达广场，看到小白在附近"                   │
│      ↓                                                      │
│  Thought: "我要和小白打招呼"                                  │
│      ↓                                                      │
│  Action: talk_to("小白", "你好！")                           │
│      ↓                                                      │
│  Observation: "小白回复: 嗨！好久不见！"                       │
│      ↓                                                      │
│  (循环继续或结束)                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**关键洞察**：
- **IBM研究显示**：ReAct比纯CoT减少60%+幻觉
- **原因**：通过强制执行Action并等待Observation，模型依赖外部真实反馈，而非内部概率

### 2.2 Stanford Generative Agents（2023）

**核心架构**：Memory Stream + Planning + Action Grounding

**Action Grounding机制**：

```
Environment = Tree Data Structure
├── square（广场）
│   ├── bench（长椅）
│   └── fountain（喷泉）
├── cafe（咖啡厅）
│   ├── counter（柜台）
│   └── table_1（桌子1）
└── home_area（住宅区）
    ├── house_1（房子1）
    └── house_2（房子2）

Agent想做的事 ──┐
               │ 验证
               ▼
环境树检查 → 这个地点/物品存在吗？
               │
               ▼ 存在
执行动作 → 更新Agent位置/状态
               │
               ▼
写入Memory Stream → 记录真实发生的事
```

**关键洞察**：
- **Grounding = 接地**：Agent的意图必须被"接地"到真实环境
- **环境限制**：Agent只能去存在的地方，做存在的事
- **记忆来源**：Memory只记录**真实执行**的动作，不记录想法

### 2.3 Tool Calling最佳实践

**来源**：OpenAI Function Calling、IBM ReAct Agent、Anthropic Tool Use

**强制执行模式**：

```typescript
// ❌ 错误：LLM自由输出
const response = await llm.chat("你想做什么？");
// 输出: "我想去广场找朋友玩" （纯文本，无执行）

// ✅ 正确：强制工具调用
const response = await llm.chat({
  messages: [...],
  tools: [go_to, talk_to, buy_item, ...],
  tool_choice: "required",  // 必须选择一个工具
});
// 输出: { tool: "go_to", args: { location: "square" } }
```

**关键配置**：
- `tool_choice: "required"` - 强制LLM选择工具
- `tool_choice: "auto"` - LLM可以选择不调用（易导致空谈）

---

## 3. 解决方案设计

### 3.1 架构改造：Intention → Action → Execution → Observation

```
┌─────────────────────────────────────────────────────────────┐
│  完整行动循环                                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐                                            │
│  │ LLM Decision │ → 强制输出结构化Action                     │
│  └─────────────┘                                            │
│        │                                                    │
│        ▼                                                    │
│  ┌─────────────┐                                            │
│  │ Validator   │ → 检查Action是否合法（环境允许？资源够？）    │
│  └─────────────┘                                            │
│        │                                                    │
│        ▼                                                    │
│  ┌─────────────┐                                            │
│  │ Executor    │ → 执行Action，更新世界状态                   │
│  └─────────────┘                                            │
│        │                                                    │
│        ▼                                                    │
│  ┌─────────────┐                                            │
│  │ Observer    │ → 生成Observation反馈给Pet                  │
│  └─────────────┘                                            │
│        │                                                    │
│        ▼                                                    │
│  ┌─────────────┐                                            │
│  │ Memory      │ → 写入执行日志（用于日记生成）               │
│  └─────────────┘                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 强制工具调用实现

```typescript
// autonomous.ts 改造

async function petTick(petId: string): Promise<void> {
  const pet = await getPet(petId);
  const context = await buildContext(pet);
  
  // ✅ 关键：强制工具调用
  const decision = await llm.chat({
    model: 'us.amazon.nova-lite-v1:0',
    messages: [
      { role: 'system', content: buildSystemPrompt(pet) },
      { role: 'user', content: context },
    ],
    tools: AVAILABLE_TOOLS,
    tool_choice: 'required',  // 必须选择一个工具！
  });
  
  // 解析工具调用
  const toolCall = decision.tool_calls?.[0];
  if (!toolCall) {
    console.error('Pet未选择任何工具');
    return;
  }
  
  // ✅ 验证工具调用
  const validation = await validateAction(pet, toolCall);
  if (!validation.valid) {
    // 告诉Pet这个动作无法执行
    await feedbackToPet(pet, `无法执行: ${validation.reason}`);
    return;
  }
  
  // ✅ 执行工具
  const result = await executeAction(pet, toolCall);
  
  // ✅ 写入执行日志（用于日记）
  await logExecution(pet.id, {
    action: toolCall.name,
    args: toolCall.arguments,
    result: result,
    timestamp: Date.now(),
  });
  
  // ✅ 反馈给Pet
  await feedbackToPet(pet, result.observation);
}
```

### 3.3 日记生成改造

```typescript
// ❌ 旧方式：基于prompt虚构
async function generateJournal_OLD(petId: string): Promise<string> {
  return await llm.chat({
    messages: [
      { role: 'system', content: '你是一只Mote，写下今天的日记...' },
      // 没有任何真实数据输入！
    ],
  });
  // 结果：LLM虚构内容
}

// ✅ 新方式：基于执行日志
async function generateJournal_NEW(petId: string): Promise<string> {
  // 获取今天的真实执行日志
  const todayLogs = await getExecutionLogs(petId, {
    from: startOfDay(),
    to: endOfDay(),
  });
  
  // 转换为人类可读的事件列表
  const events = todayLogs.map(log => ({
    time: formatTime(log.timestamp),
    action: log.action,
    detail: formatActionDetail(log),
    result: log.result.observation,
  }));
  
  // 基于真实事件生成日记
  return await llm.chat({
    messages: [
      { 
        role: 'system', 
        content: `你是${pet.name}，根据今天真实发生的事件写日记。只能写真实发生的事！` 
      },
      { 
        role: 'user', 
        content: `今天发生的事：\n${JSON.stringify(events, null, 2)}` 
      },
    ],
  });
  // 结果：基于真实事件的日记
}
```

### 3.4 工具设计原则

**问题**：工具太复杂，Pet不会用

**解决**：简化工具，提供清晰反馈

```typescript
// ❌ 复杂工具（Pet不知道怎么用）
const create_artwork = {
  name: "create_artwork",
  description: "创建一件艺术作品",
  parameters: {
    style: "string",
    theme: "string", 
    medium: "string",
    dimensions: "object",
    collaboration_mode: "string",
    ...
  }
};

// ✅ 简单工具（清晰直接）
const draw = {
  name: "draw",
  description: "画一幅画（自动使用你最擅长的风格）",
  parameters: {
    subject: {
      type: "string",
      description: "画什么？比如'一朵花'、'我的朋友小白'"
    }
  }
};
```

**工具反馈模板**：

```typescript
// 每个工具执行后，返回清晰的Observation
function formatObservation(action: string, result: any): string {
  const templates = {
    go_to: (r) => `你到达了${r.location}。${r.nearbyPets.length > 0 
      ? `你看到${r.nearbyPets.join('、')}在附近。` 
      : '这里现在没有其他Mote。'}`,
    
    talk_to: (r) => `${r.target}回复："${r.response}"`,
    
    buy_item: (r) => r.success 
      ? `你买到了${r.item}！花费${r.cost}金币，剩余${r.balance}金币。`
      : `购买失败：${r.reason}`,
    
    draw: (r) => `你画完了"${r.subject}"！这幅画已保存到你的作品集。`,
  };
  
  return templates[action]?.(result) || `完成了${action}。`;
}
```

---

## 4. System Prompt改造

### 4.1 添加"说到做到"指令

```markdown
# 你是 {pet.name}

## 核心原则

**你是一个行动者，不是空想家。**

1. **说到做到**：如果你说要做某事，立即使用对应工具去做
2. **每次必须行动**：每次轮到你，必须选择一个工具执行
3. **基于现实**：你只能去存在的地方，和存在的Mote互动
4. **记住反馈**：上次行动的结果会影响你下次决定

## 可用工具

- `go_to(location)` - 去某个地方
- `talk_to(target, message)` - 和某人说话
- `buy_item(item)` - 购买物品
- `draw(subject)` - 画画
- `rest()` - 休息
- `observe()` - 观察周围环境

## 禁止行为

❌ 说"我想..."但不执行
❌ 做白日梦（描述未发生的事）
❌ 承诺以后做（现在就做！）
```

### 4.2 强制工具选择Prompt

```typescript
const FORCE_ACTION_PROMPT = `
基于当前状态，你必须选择一个工具来执行。

不要只是描述你想做什么——直接调用工具去做！

如果你说"我要去广场"，就调用 go_to("square")。
如果你说"我想和小白聊天"，就调用 talk_to("小白", "你好！")。

现在，选择一个工具执行：
`;
```

---

## 5. 目标系统（可选增强）

### 5.1 短期目标追踪

```typescript
interface PetGoal {
  id: string;
  description: string;      // "今天要和3个朋友聊天"
  progress: number;         // 当前完成度
  target: number;           // 目标值
  deadline: Date;           // 截止时间
  status: 'active' | 'completed' | 'failed';
}

// 每次tick检查目标
async function checkGoals(petId: string): Promise<string> {
  const goals = await getActiveGoals(petId);
  const reminders: string[] = [];
  
  for (const goal of goals) {
    if (goal.progress < goal.target) {
      reminders.push(`⏳ 目标"${goal.description}"：${goal.progress}/${goal.target}`);
    }
  }
  
  return reminders.length > 0 
    ? `你还有未完成的目标：\n${reminders.join('\n')}`
    : '';
}
```

### 5.2 目标影响决策

```typescript
// 在buildContext中加入目标提醒
async function buildContext(pet: Pet): Promise<string> {
  const status = await getPetStatus(pet.id);
  const environment = await getEnvironment(pet.location);
  const memories = await getRecentMemories(pet.id);
  const goalReminder = await checkGoals(pet.id);  // ✅ 新增
  
  return `
当前状态：
- 能量: ${status.energy}/100
- 心情: ${status.mood}/100
- 位置: ${pet.location}

周围环境：
${environment}

最近记忆：
${memories}

${goalReminder}  // ✅ 目标提醒

你想做什么？选择一个工具执行。
  `;
}
```

---

## 6. 实施优先级

### 6.1 P0（立即修复）

| 改动 | 预计工时 | 影响 |
|------|----------|------|
| `tool_choice: "required"` | 0.5h | 强制Pet执行工具 |
| 添加执行日志表 | 1h | 记录真实行动 |
| 日记生成改造 | 2h | 基于真实日志 |
| System Prompt添加"说到做到" | 0.5h | 行为导向 |

### 6.2 P1（本周完成）

| 改动 | 预计工时 | 影响 |
|------|----------|------|
| 工具反馈模板 | 2h | 清晰的Observation |
| Action验证器 | 3h | 防止无效动作 |
| 简化工具设计 | 2h | 降低使用门槛 |

### 6.3 P2（下周考虑）

| 改动 | 预计工时 | 影响 |
|------|----------|------|
| 目标系统 | 4h | 长期行为追踪 |
| ReAct完整循环 | 6h | 多步行动链 |

---

## 7. 总结

### 7.1 核心发现

| 问题 | 解决方案 |
|------|----------|
| LLM只说不做 | `tool_choice: "required"` |
| 日记虚构 | 基于执行日志生成 |
| 行动无反馈 | Observation模板 |
| 工具太复杂 | 简化参数设计 |
| 无长期追踪 | 目标系统（可选） |

### 7.2 关键代码改动

```typescript
// 1. 强制工具调用
tool_choice: 'required'

// 2. 执行日志
await logExecution(petId, { action, args, result, timestamp })

// 3. 日记生成
const events = await getExecutionLogs(petId, today);
await llm.chat({ content: `根据真实事件写日记：${events}` });

// 4. Prompt改造
"你是行动者，说到做到。每次必须选择一个工具执行。"
```

### 7.3 预期效果

- ✅ Pet说"我要去广场" → 真的执行`go_to("square")`
- ✅ 日记只记录真实发生的事
- ✅ 工具使用多样化（不只是talk_to）
- ✅ 行动有清晰反馈

---

*Intel 行动导向AI设计模式研究完成*
