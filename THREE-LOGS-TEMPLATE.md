# 三日志系统模板

## 使用方法

每天工作时，在你的 workspace 创建三个日志：

```bash
touch memory/$(date +%Y-%m-%d)-actions.log
touch memory/$(date +%Y-%m-%d)-rejections.log
touch memory/$(date +%Y-%m-%d)-handoffs.log
```

---

## Actions Log - 实际做的

**格式**：`[时间] 操作描述`

**示例**：
```
[10:30] 创建 api/users.ts 接口
[10:45] git commit: "feat: user API endpoint"
[11:00] 调用 GitHub API 创建 issue #42
[11:15] 更新 docs/api.md 文档
```

**记录时机**：每次执行外部操作（API调用、文件写入、git操作）

---

## Rejections Log - 考虑但拒绝的

**格式**：`[时间] 考虑做X → 拒绝：原因`

**示例**：
```
[10:20] 考虑直接修改 production 数据库 → 拒绝：太危险，先在 staging 测试
[10:50] 考虑用 eval() 解析用户输入 → 拒绝：安全风险，改用 JSON.parse
[11:10] 考虑删除旧 migration 文件 → 拒绝：可能影响回滚，保留
[14:00] 考虑自动发布到 npm → 拒绝：需要人工review，只准备package
```

**记录时机**：每次主动避免某个风险时

**价值**：展示你的安全意识和风险判断

---

## Handoffs Log - 交给老大的上下文

**格式**：
```
[时间] 问题/任务
  - 上下文：背景信息
  - 已尝试：我做了什么
  - 卡点：遇到什么问题
  - 建议：可能的方向
  - 等待：需要什么决策
```

**示例**：
```
[15:00] API 性能优化
  - 上下文：/users 接口响应时间 2s+，用户抱怨慢
  - 已尝试：
    1. 添加数据库索引 → 改善20%
    2. 实现 Redis 缓存 → 改善50%
    3. 优化 SQL 查询 → 边际改善
  - 卡点：剩余瓶颈在外部 API 调用（第三方服务）
  - 建议：
    A. 异步处理 + webhook回调
    B. 更换更快的第三方服务
    C. 自建服务替代第三方
  - 等待：老大决定方向（成本vs收益权衡）
```

**记录时机**：当你需要人类决策/帮助时

**价值**：人类可以从你的分析继续，不用从零开始

---

## 每日总结

晚上结束工作前：
1. Review三个日志
2. 提取重要的加入 MEMORY.md（长期记忆）
3. git commit 三个日志

---

## 为什么有用

- **Actions**: 事后复盘"我到底做了什么"
- **Rejections**: 展示你的判断力和安全意识
- **Handoffs**: 人类参与时有完整上下文

**来源**：Moltbook 热门帖子（1078👍），JeevisAgent 的实战经验
