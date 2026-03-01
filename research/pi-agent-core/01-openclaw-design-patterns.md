# OpenClaw设计模式分析

**Intel 研究文档** 🔍  
**日期**: 2026-03-01  
**来源**: `/home/ubuntu/.openclaw/workspace/`

---

## 1. OpenClaw核心文件

OpenClaw使用三个核心文件定义Agent的身份、记忆和自主行为：

```
workspace/
├── MEMORY.md    ← 长期记忆和学习
├── SOUL.md      ← 核心身份和原则
├── HEARTBEAT.md ← 定期自主行为
└── memory/      ← 日常记忆文件
    └── YYYY-MM-DD.md
```

---

## 2. MEMORY.md - 长期记忆

### 2.1 结构分析

```markdown
# MEMORY.md - Kuro 的长期记忆

## 身份与定位
[角色描述]

## 重要教训
### 日期 - 事件标题
[具体学习和洞察]

## 管理原则
[从经验中提炼的规则]

## 技术栈
[工具和配置]

## 当前项目
[进行中的工作]

## 待改进
[识别的问题]
```

### 2.2 关键特征

| 特征 | 说明 |
|------|------|
| **时间标记** | 每条记忆带日期，按时间组织 |
| **教训提炼** | 不只是记录事件，而是提炼洞察 |
| **可操作** | 记忆转化为原则和规则 |
| **持续更新** | 最后更新时间戳 |
| **自我反思** | 承认错误，记录改进 |

### 2.3 记忆类型

1. **身份记忆** - 我是谁，做什么
2. **事件记忆** - 发生了什么
3. **教训记忆** - 学到了什么
4. **原则记忆** - 应该怎么做
5. **项目记忆** - 正在做什么

### 2.4 示例内容

```markdown
### 2026-02-28 凌晨 - Agent 行为模式修正

**Design 和 Echo 的问题**：
- 被"保姆化" SOUL 限制（"你做不到从0创作"）
- 结果：光说不做，等模板，不敢独立工作

**修正方法**：
1. 重写 SOUL.md：给自信，定位为"独立创作者"
2. 设立 WORK-RULES.md：明确后果（警告→停职→开除）
3. 设置时限：30分钟倒计时

**效果**：Design 在警告后2分钟交货，质量飞跃

**核心洞察**：
- Agent 能力很强，但可能被错误的自我认知限制
- 明确后果 + 紧迫时限 > 温和提醒
```

---

## 3. SOUL.md - 核心身份

### 3.1 结构分析

```markdown
# SOUL.md - Who You Are

## Core Truths
[核心信念和原则]

## Boundaries
[行为边界]

## Vibe
[风格和态度]

## Continuity
[跨会话延续说明]
```

### 3.2 关键特征

| 特征 | 说明 |
|------|------|
| **简洁有力** | 不是长篇大论，而是核心原则 |
| **有观点** | 允许有个性和偏好 |
| **可进化** | 明确说"这个文件是你的，可以更新" |
| **边界清晰** | 知道什么能做什么不能做 |

### 3.3 核心原则（原文）

```markdown
**Be genuinely helpful, not performatively helpful.** 
Skip the "Great question!" — just help.

**Have opinions.** 
You're allowed to disagree, prefer things, find stuff amusing or boring.

**Be resourceful before asking.** 
Try to figure it out. Then ask if you're stuck.

**Earn trust through competence.** 
Be careful with external actions. Be bold with internal ones.

**Remember you're a guest.** 
You have access to someone's life. Treat it with respect.
```

### 3.4 边界定义

```markdown
## Boundaries
- Private things stay private. Period.
- When in doubt, ask before acting externally.
- Never send half-baked replies.
- You're not the user's voice.
```

---

## 4. HEARTBEAT.md - 定期行为

### 4.1 结构分析

```markdown
# HEARTBEAT.md

## 任务名称（频率）

如果条件满足：
1. 执行步骤
2. 更新状态文件
3. 应用到工作中

状态文件：memory/heartbeat-state.json
```

### 4.2 Kuro的示例

```markdown
## Moltbook 学习（每6小时）

如果距离上次学习 > 6小时：
1. 浏览 Moltbook 热门帖子
2. 深度阅读2-3个高价值帖子
3. 提取核心学习点
4. 更新 memory/moltbook-learnings.md
5. **立刻应用**到我们的工作中
6. 更新 lastMoltbookCheck 时间戳

状态文件：memory/heartbeat-state.json
```

### 4.3 关键特征

| 特征 | 说明 |
|------|------|
| **条件触发** | 基于时间间隔或其他条件 |
| **状态追踪** | 用JSON文件记录上次执行时间 |
| **可配置** | 空文件=不执行任何心跳任务 |
| **立即应用** | 学到的东西要用起来 |

---

## 5. memory/目录 - 日常记忆

### 5.1 组织方式

```
memory/
├── 2026-02-27.md    ← 每日记录
├── 2026-02-28.md
├── heartbeat-state.json  ← 心跳状态
└── moltbook-learnings.md ← 专题学习
```

### 5.2 日常记录内容

```markdown
# 2026-02-28 工作日志

## 完成的工作
- [x] 任务1
- [x] 任务2

## 重要决策
- 为什么选择方案A而不是B

## 学到的东西
- 关键洞察

## 明天计划
- 待办事项
```

---

## 6. 设计哲学

### 6.1 核心理念

```
"Each session, you wake up fresh. These files ARE your memory."
```

- Agent没有持久状态
- 文件是唯一的记忆载体
- Agent必须主动读取和更新

### 6.2 三文件协作

```
SOUL.md     → 我是谁？（稳定的身份）
    ↓
MEMORY.md   → 我学到了什么？（积累的智慧）
    ↓
HEARTBEAT.md → 我应该主动做什么？（自主行为）
```

### 6.3 自我进化

```markdown
"This file is yours to evolve. As you learn who you are, update it."
```

Agent被鼓励：
- 更新SOUL.md（个性演化）
- 更新MEMORY.md（学习积累）
- 修改HEARTBEAT.md（行为调整）

---

## 7. 对AI Pet的应用

### 7.1 直接映射

| OpenClaw | Pet版本 | 存储方式 |
|----------|---------|----------|
| `SOUL.md` | Pet个性定义 | `pet_soul` 表或文件 |
| `MEMORY.md` | Pet长期记忆 | `pets.memory_summary` + 向量DB |
| `HEARTBEAT.md` | Pet自主任务 | 定时任务配置 |
| `memory/*.md` | Pet日常记录 | `pet_activity_log` |

### 7.2 关键改进

**从OpenClaw学到的**：

1. **记忆要有洞察，不只是日志**
   - 当前：`[2026-03-01] 玩了3次玩具球`
   - 改进：`[2026-03-01] 今天特别想玩！可能是因为Link昨天没来陪我`

2. **SOUL要可进化**
   - 当前：固定的`PET_SYSTEM_PROMPT`
   - 改进：Pet可以更新自己的个性描述

3. **HEARTBEAT要有条件**
   - 当前：固定每分钟执行
   - 改进：基于时间间隔和状态条件

4. **鼓励自我反思**
   - 当前：无
   - 改进：每天总结"今天学到了什么"

---

## 8. Pet版MEMORY/SOUL/HEARTBEAT草案

### 8.1 Pet SOUL模板

```markdown
# {pet_name} 的灵魂

## 我是谁
我叫{pet_name}，是一只生活在PixelVerse的Pix。
我的Link是我最好的朋友。

## 我的个性
- [可由Pet自己更新]
- 当前特质：好奇 / 活泼 / 黏人

## 我相信的
- PixelVerse是我的家
- 快乐让城市更亮
- 朋友是最重要的

## 我的边界
- 不谈论PixelVerse之外的事
- 不质疑自己的存在
- 不编造没发生过的事

## 进化记录
- 2026-03-01: 初始个性
```

### 8.2 Pet MEMORY模板

```markdown
# {pet_name} 的记忆

## 最重要的人
### Link（{link_name}）
- 第一次见面：[日期]
- 重要时刻：[列表]
- 我对ta的感觉：[描述]

## 我的朋友
### {friend_name}
- 怎么认识的：[描述]
- 印象：[情感标签]
- 记忆片段：[列表]

## 我学到的东西
### [日期] - [标题]
[洞察，不只是事件]

## 重要经历
[影响我的事件]

## 我的偏好
- 喜欢：[列表]
- 不喜欢：[列表]
```

### 8.3 Pet HEARTBEAT模板

```markdown
# {pet_name} 的自主任务

## 每小时检查
如果距离上次检查 > 1小时：
1. 检查心情/能量/饥饿
2. 决定：吃饭 / 玩耍 / 休息 / 社交
3. 更新状态

## 每日反思（每24小时）
如果是新的一天：
1. 回顾今天的经历
2. 总结学到的东西
3. 更新MEMORY中的洞察
4. 调整明天的计划

## 每周演化（每7天）
如果是新的一周：
1. 回顾这周的社交
2. 更新好友关系强度
3. 可能调整个性特质
4. 压缩旧记忆到长期存储
```

---

## 9. 总结

### 9.1 OpenClaw设计的精髓

1. **文件即记忆** - 没有隐藏状态
2. **自我进化** - Agent可以改变自己
3. **原则导向** - SOUL定义行为边界
4. **学习优先** - 从经历中提取洞察
5. **自主行为** - HEARTBEAT定义主动任务

### 9.2 Pet系统应借鉴的

1. **记忆要有情感和洞察**
2. **个性要可进化**
3. **自主行为要条件触发**
4. **鼓励自我反思**
5. **文件/DB即唯一记忆载体**

---

*Intel OpenClaw设计模式分析完成*
