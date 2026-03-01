# 广场扩展性竞品技术分析

**Intel 研究文档** 🔍  
**日期**: 2026-03-01  
**状态**: 进行中

---

## 1. 调研目标

分析2024-2026年最新技术方案，为AI Pet广场从3个Pet扩展到30万个Pet提供技术参考。

---

## 2. Roblox 2025 基础设施架构

### 来源
- Roblox官方博客 (June 2025): "The Infrastructure Supporting Record-Breaking Experiences"
- Medium技术文章 (Dec 2025): "The Hidden Architecture Behind Large-Scale Roblox Worlds"

### 核心数据
- **峰值并发**: 3060万玩家 (2025年6月)
- **单体验峰值**: 2160万同时在线 (Grow a Garden)
- **边缘数据中心**: 全球24个
- **微服务数量**: 1600+

### 关键技术策略

#### 2.1 区域分片 (Cellular Infrastructure)
```
用户连接 → 最近边缘数据中心 → 最优实例
```
- 混合云架构：本地 + 云边缘数据中心
- 自动扩容：周五预测周末流量，动态启动云资源
- 区域隔离：每个数据中心独立处理10K+用户

#### 2.2 匹配系统 (Matchmaking)
- **目标**: 1000万用户10秒内完成匹配
- **峰值能力**: 每秒评估40亿种可能的组合
- **关键**: 不节流用户，不跳过匹配算法

#### 2.3 实体管理 (Entity Load)
**问题**: 每个持久对象都有成本
- NPC、怪物、资源节点
- 可交互物品、物理对象
- 环境脚本

**解决方案**: 空间分区
```typescript
// 将世界划分为网格
// 100×100 studs = 1个chunk
// 500×500世界 = 25个chunks

// 玩家进入chunk → 激活附近实体
// 玩家离开 → 停用或销毁实体

// 效果：减少40-70%服务器负载
```

#### 2.4 服务器tick预算
- **目标**: 30Hz（每秒30次模拟更新）
- **危险信号**: 30→25→20→15→10 FPS崩溃
- **消耗因素**: AI循环、寻路、物理、碰撞、远程事件

**AI优化原则**:
- AI休眠，除非玩家在附近
- 寻路每0.5-1.5秒运行一次（不是每帧）
- 简单转向替代昂贵的寻路
- 便宜的hitbox交互替代完整物理

---

## 3. VRChat 2025 性能优化

### 来源
- VRChat开发者文档 (Dec 2025): "Performance Optimization: Reducing Update Calls and Network Load"

### 关键策略

#### 3.1 避免Update()
```csharp
// 坏：每帧检查
void Update() {
  if (Input.GetKeyDown(KeyCode.E)) { ... }
}

// 好：事件驱动
public override void Interact() { ... }
```

| 需求 | 坏方案(Update) | 好方案(事件驱动) |
|------|----------------|------------------|
| 检测按键 | if(Input.GetKey...) | Interact() |
| 检测玩家进入 | Vector3.Distance... | OnPlayerTriggerEnter |
| 检测血量为0 | if(health<=0)每帧 | 受伤时检查 |

#### 3.2 网络负载优化
```csharp
// 坏：每次都发送
set {
  _score = value;
  RequestSerialization(); // 即使值没变也发送
}

// 好：只在值变化时发送
set {
  if (_score == value) return; // 值没变就不发送
  _score = value;
  RequestSerialization();
}
```

#### 3.3 Behaviour Sync Mode
- **Continuous**: 自动同步，变量变化时尝试同步（危险）
- **Manual**: 只在调用RequestSerialization()时同步（推荐）

---

## 4. 渲染技术对比 (2025基准测试)

### 来源
- SVG Genie (Dec 2025): "SVG vs Canvas vs WebGL: I Benchmarked All 3"

### 性能数据

#### SVG性能
| 元素数量 | 渲染时间 | 动画FPS |
|----------|----------|---------|
| 100 | 2ms | 60fps |
| 1,000 | 15ms | 60fps |
| 5,000 | 85ms | 35fps |
| 10,000 | 210ms | 12fps |

#### Canvas性能
| 元素数量 | 渲染时间 | 动画FPS |
|----------|----------|---------|
| 1,000 | 3ms | 60fps |
| 10,000 | 18ms | 60fps |
| 50,000 | 45ms | 55fps |
| 100,000 | 95ms | 40fps |

#### WebGL性能
| 元素数量 | 渲染时间 | 动画FPS |
|----------|----------|---------|
| 10,000 | 2ms | 60fps |
| 100,000 | 8ms | 60fps |
| 500,000 | 25ms | 55fps |
| 1,000,000 | 45ms | 45fps |

### 选择建议
| 需求 | 推荐技术 |
|------|----------|
| <5,000元素+交互 | SVG |
| 10K-100K元素 | Canvas |
| 3D或百万级元素 | WebGL |
| 移动端性能关键 | Canvas/WebGL |

---

## 5. WebSocket扩展性架构 (2025最佳实践)

### 来源
- Galaxy4Games (Oct 2025): "What Backend Architecture Supports Scalable Multiplayer Games"
- Medium技术文章 (2025): "Scaling WebSockets to Millions"

### 核心架构模式

#### 5.1 微服务分离
```
认证服务 → 独立
匹配服务 → 独立
聊天服务 → 独立
物理服务 → 独立
支付服务 → 独立
```
**优点**: 独立扩展、独立部署、无瓶颈

#### 5.2 负载均衡
```
玩家连接 → 负载均衡器 → 多服务器分发
```
- 防止瓶颈
- 降低延迟
- 高峰期平稳

#### 5.3 协议选择
| 数据类型 | 协议 |
|----------|------|
| 关键游戏数据(位置/动作) | UDP/WebSocket + 丢包缓解 |
| 非关键操作(用户资料) | REST/gRPC |

#### 5.4 云 vs VPS vs 专用
| 方案 | 适用场景 |
|------|----------|
| 云 | 弹性伸缩，流量波动大 |
| VPS | 小项目，成本敏感 |
| 专用服务器 | 最高性能，流量稳定 |

---

## 6. 对AI Pet广场的应用建议

### 6.1 渲染层
- **当前**: Canvas 2D (Dev已实现)
- **建议**: 保持Canvas，50个Pet同屏足够

### 6.2 网络层
- **区域分片**: 广场划分为9个区域，用户只接收附近2-3个区域的更新
- **批量更新**: 每200ms批量发送一次位置（5fps），不是每帧
- **智能采样**: 优先显示距离近+活跃+稀有的Pet

### 6.3 服务器层
- **M1-M6 (25K用户)**: 单Square Server + 区域分片
- **M7-M9 (80K用户)**: 3个Square Server
- **M10+ (200K用户)**: 9个Square Server + 自动扩容

### 6.4 AI对话层
- **90%对话**: 模板（零成本）
- **10%对话**: Nova 2 Lite（超便宜）
- **触发概率**: 距离<50=10%，距离>=50=1%

---

## 7. 下一步调研

- [ ] Canvas渲染优化详解
- [ ] WebSocket区域分片实现
- [ ] LLM成本详细计算
- [ ] 分布式架构设计

---

*Intel 竞品技术分析初稿完成*  
*14:00 UTC @Kuro 汇报*
