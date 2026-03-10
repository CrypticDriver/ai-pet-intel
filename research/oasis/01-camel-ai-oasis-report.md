# CAMEL AI OASIS 调研报告

**Intel 紧急调研** 🔍  
**日期**: 2026-03-10  
**任务**: 老大指定调研 https://github.com/camel-ai/oasis

---

## 1. 项目概览

### 1.1 基本信息

| 项目 | 信息 |
|------|------|
| **名称** | OASIS (Open Agent Social Interaction Simulations) |
| **作者** | CAMEL-AI 团队（22位研究者） |
| **发布** | 2024-11-19 (arXiv) |
| **GitHub** | https://github.com/camel-ai/oasis |
| **Stars** | 活跃项目 |
| **License** | Apache 2.0 ✅ |
| **安装** | `pip install camel-oasis` |

### 1.2 核心定位

**🏝️ OASIS = 大规模社交媒体模拟器**

- 模拟Twitter/Reddit平台
- **支持100万Agent同时交互**
- LLM驱动Agent行为
- 研究社会现象（信息传播、群体极化、羊群效应）

---

## 2. 核心能力

### 2.1 规模能力

| 能力 | 数值 |
|------|------|
| **最大Agent数** | 1,000,000 |
| **支持动作** | 23种 |
| **平台类型** | Twitter、Reddit |

**如何实现百万级？**
- LLM Agent + Rule-based Agent 混合
- 不是每个Agent都用LLM
- 批量处理 + 异步执行

### 2.2 动作空间（23种）

```python
ActionType = [
    # 内容交互
    LIKE_POST, DISLIKE_POST, CREATE_POST, CREATE_COMMENT,
    LIKE_COMMENT, DISLIKE_COMMENT, REPOST, QUOTE,
    
    # 搜索发现
    SEARCH_POSTS, SEARCH_USER, TREND, REFRESH,
    
    # 社交关系
    FOLLOW, MUTE, BLOCK, UNFOLLOW,
    
    # 其他
    DO_NOTHING, REPORT_POST, 
    # 群聊功能（2025.6新增）
    CREATE_GROUP, SEND_GROUP_MESSAGE, LEAVE_GROUP,
    # 采访功能
    INTERVIEW,
]
```

### 2.3 推荐系统

OASIS内置两种推荐算法：

1. **Interest-based** - 基于用户兴趣
2. **Hot-score-based** - 基于热度分数

模拟真实社交平台的内容发现机制。

### 2.4 动态环境

- 社交网络实时更新（关注关系变化）
- 内容流动态变化（新帖子、热门变化）
- Agent状态实时同步

---

## 3. 技术架构

### 3.1 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│  OASIS Architecture                                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Agent Graph  │  │   Platform   │  │   Database   │       │
│  │ (用户关系图)  │  │ (Twitter等)  │  │  (SQLite)    │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         │                 │                 │               │
│         ▼                 ▼                 ▼               │
│  ┌──────────────────────────────────────────────────┐       │
│  │              Environment (Gym-style)              │       │
│  │                                                   │       │
│  │  reset() → step(actions) → observe → reward      │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Agent类型

**混合架构**（关键扩展性来源）：

| Agent类型 | 决策方式 | 占比 | 用途 |
|-----------|----------|------|------|
| **LLM Agent** | GPT-4o-mini | 少数 | 关键用户、研究对象 |
| **Rule Agent** | 规则引擎 | 多数 | 背景用户、填充规模 |

```python
# 两种动作方式
actions = {
    # LLM决策
    agent_1: LLMAction(),  # LLM自主选择
    
    # 手动/规则决策
    agent_2: ManualAction(
        action_type=ActionType.CREATE_POST,
        action_args={"content": "Hello!"}
    ),
}
```

### 3.3 关键代码示例

```python
import oasis
from oasis import ActionType, LLMAction, generate_reddit_agent_graph

async def main():
    # 1. 创建Agent图（从profile文件）
    agent_graph = await generate_reddit_agent_graph(
        profile_path="./data/user_data_36.json",
        model=openai_model,
        available_actions=[ActionType.LIKE_POST, ActionType.CREATE_POST, ...],
    )
    
    # 2. 创建环境
    env = oasis.make(
        agent_graph=agent_graph,
        platform=oasis.DefaultPlatformType.REDDIT,
        database_path="./simulation.db",
    )
    
    # 3. 运行模拟
    await env.reset()
    
    # 手动动作
    actions_1 = {
        agent_0: ManualAction(ActionType.CREATE_POST, {"content": "Hello!"})
    }
    await env.step(actions_1)
    
    # LLM自主动作
    actions_2 = {agent: LLMAction() for _, agent in env.agent_graph.get_agents()}
    await env.step(actions_2)
```

---

## 4. 与Genesis Motes对比

### 4.1 定位差异

| 维度 | OASIS | Genesis Motes |
|------|-------|---------------|
| **场景** | 社交媒体模拟 | 虚拟宠物世界 |
| **目标** | 研究社会现象 | 数字生命体验 |
| **平台** | Twitter/Reddit | 自定义世界 |
| **交互** | 发帖/评论/关注 | 移动/对话/工作 |
| **规模** | 百万级 | 百级 |
| **用户** | 研究人员 | 消费者 |

### 4.2 架构对比

| 维度 | OASIS | Genesis Motes |
|------|-------|---------------|
| **Agent框架** | CAMEL-AI | pi-agent-core |
| **LLM** | GPT-4o-mini | Nova Lite/Pro |
| **存储** | SQLite | PostgreSQL |
| **规模策略** | LLM+Rule混合 | 100%LLM |
| **环境** | Gym-style | 自定义tick |

### 4.3 可借鉴点 ⭐

| 特性 | OASIS做法 | 对Genesis的价值 |
|------|-----------|----------------|
| **动作空间** | 23种预定义Action | 我们有18个工具，可参考扩展 |
| **推荐系统** | 内置兴趣+热度推荐 | 可用于Pet发现内容/朋友 |
| **Agent Graph** | 社交关系图结构 | 可优化我们的关系存储 |
| **混合Agent** | LLM+Rule混合 | 我们100%LLM，成本更高 |
| **批量执行** | 异步批量step | 可优化我们的tick性能 |

---

## 5. 关键洞察

### 5.1 百万Agent的秘密

**OASIS不是100万个LLM Agent！**

```
实际架构：
- 少量LLM Agent（研究对象）
- 大量Rule Agent（背景填充）
- 混合比例可调
```

**这与我们的100%LLM策略不同**：
- OASIS追求规模 → 牺牲个体智能
- 我们追求生命感 → 牺牲规模

### 5.2 对我们的启发

**可借鉴**：
1. **动作空间设计** - 23种标准化Action
2. **推荐系统** - 内容发现机制
3. **批量执行** - 异步处理提升性能
4. **环境抽象** - Gym-style接口设计

**不适合直接用**：
1. **混合Agent** - 我们坚持100%LLM
2. **社交媒体场景** - 我们是虚拟世界
3. **研究导向** - 我们是产品导向

### 5.3 技术可迁移项

| 技术 | OASIS实现 | 迁移可行性 | 优先级 |
|------|-----------|-----------|--------|
| 异步批量处理 | asyncio + batch | ✅ 高 | 🔴 P0 |
| 动作标准化 | ActionType enum | ✅ 高 | 🔴 P0 |
| 推荐算法 | interest + hot | ✅ 中 | 🟡 P1 |
| 关系图存储 | agent_graph | ✅ 中 | 🟡 P1 |
| 混合Agent | LLM + Rule | ❌ 低 | 不采用 |

---

## 6. 总结

### 6.1 OASIS是什么

- 🏝️ **大规模社交媒体模拟器**
- 📊 **研究工具**（信息传播、群体行为）
- 🔬 **学术项目**（22位研究者、arXiv论文）
- 🐍 **Python库**（pip install camel-oasis）

### 6.2 与我们的关系

| 维度 | 结论 |
|------|------|
| **直接竞品** | ❌ 不是（场景不同） |
| **技术参考** | ✅ 是（架构、动作设计） |
| **可集成** | ⚠️ 部分（推荐系统、批处理） |
| **替代方案** | ❌ 不是（定位完全不同） |

### 6.3 推荐行动

**立即可做**：
1. 学习其Action标准化设计
2. 参考异步批处理架构
3. 研究推荐算法实现

**不建议**：
1. 直接集成OASIS（场景不匹配）
2. 采用混合Agent（违背数字生命理念）

---

## 7. 关键引用

- **论文**: arXiv:2411.11581
- **GitHub**: https://github.com/camel-ai/oasis
- **文档**: https://docs.oasis.camel-ai.org
- **视频**: https://www.youtube.com/watch?v=lprGHqkApus

---

*Intel 紧急调研完成*
