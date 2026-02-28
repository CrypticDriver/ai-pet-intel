# OpenClaw-bot-review 房间场景实现分析

**Intel 深度调研** 🔍
**日期**: 2026-02-28
**任务**: 分析像素办公室的技术实现，供AI Pet项目参考

---

## 1. 架构概览

### 核心技术栈

```
渲染: HTML5 Canvas 2D
框架: Next.js + React 19
状态: useRef + useState (无外部状态库)
动画: requestAnimationFrame 游戏循环
```

### 代码结构

```
lib/pixel-office/
├── engine/
│   ├── renderer.ts      # 核心渲染逻辑
│   ├── officeState.ts   # 办公室状态管理
│   ├── characters.ts    # 角色逻辑
│   └── matrixEffect.ts  # 特效
├── layout/
│   └── furnitureCatalog.ts  # 家具目录
├── sprites/
│   └── spriteData.ts    # 精灵图数据
├── constants.ts         # 配置常量
└── types.ts             # 类型定义
```

---

## 2. 房间/场景设计方式

### 采用 Tile-based（瓦片式）架构 ⭐

**不是单张大图，而是16x16像素的小瓦片拼接**

```typescript
// 核心常量
export const TILE_SIZE = 16      // 每个瓦片16x16像素
export const DEFAULT_COLS = 20   // 默认20列
export const DEFAULT_ROWS = 11   // 默认11行
```

### 瓦片类型

```typescript
enum TileType {
  VOID = 0,     // 空白/透明
  FLOOR_1 = 1,  // 地板类型1
  FLOOR_2 = 2,  // 地板类型2
  WALL = 3,     // 墙壁
  // ...更多类型
}
```

### 地图数据存储

```typescript
interface OfficeLayout {
  cols: number;           // 列数
  rows: number;           // 行数
  tiles: TileTypeVal[];   // 一维数组存储瓦片类型
  tileColors?: FloorColor[]; // 瓦片颜色
  furniture: FurnitureInstance[]; // 家具列表
}
```

---

## 3. 渲染机制详解

### 游戏循环（60 FPS）

```typescript
const render = (time: number) => {
  const dt = (time - lastTime) / 1000; // 计算delta时间
  office.update(dt);                    // 更新状态
  
  // 渲染各层：
  // 1. 地板层
  renderTileGrid(ctx, tileMap, offsetX, offsetY, zoom, tileColors);
  
  // 2. 墙壁层
  renderWalls(ctx, ...);
  
  // 3. 家具层（按Y轴排序实现伪3D）
  renderFurniture(ctx, furniture, ...);
  
  // 4. 角色层
  renderCharacters(ctx, characters, ...);
  
  // 5. UI层（气泡、按钮等）
  renderUI(ctx, ...);
  
  requestAnimationFrame(render);
};
```

### 伪3D排序（Y-sorting）

```typescript
// 角色和家具按Y坐标排序，越靠下的越后渲染
// 实现"前面遮挡后面"的效果
const sortedObjects = [...furniture, ...characters]
  .sort((a, b) => a.y - b.y);
```

### 性能关键点

```typescript
// 1. 避免浮点数渲染模糊
ctx.fillRect(Math.round(x), Math.round(y), Math.round(w), Math.round(h));

// 2. 禁用图像平滑保持像素清晰
ctx.imageSmoothingEnabled = false;

// 3. 使用缓存的精灵图
const sprite = getCachedSprite(type);
```

---

## 4. 家具系统

### 家具目录结构

```typescript
interface FurnitureCatalogEntry {
  type: string;           // 类型ID
  sprite: SpriteData;     // 精灵图
  footprintW: number;     // 占地宽度(瓦片数)
  footprintH: number;     // 占地高度(瓦片数)
  canPlaceOnWalls?: boolean; // 是否可贴墙
  isSeat?: boolean;       // 是否是座位
}
```

### 预设家具

- 桌子 (desk)
- 椅子 (chair) - 可坐
- 电脑 (pc) - 可交互
- 沙发 (sofa) - 可坐
- 书架 (library)
- 白板 (whiteboard)
- 相机 (camera) - 可交互
- 手机 (phone) - 可交互
- 钟表 (clock) - 可交互

---

## 5. 角色系统

### 角色状态机

```typescript
enum CharacterState {
  IDLE = 'idle',
  WALKING = 'walking',
  SITTING = 'sitting',
  TYPING = 'typing',
  WORKING = 'working',
}
```

### 动画配置

```typescript
// 移动速度
export const WALK_SPEED_PX_PER_SEC = 48;

// 帧切换间隔
export const WALK_FRAME_DURATION_SEC = 0.15;

// 闲逛暂停时间
export const WANDER_PAUSE_MIN_SEC = 1.0;
export const WANDER_PAUSE_MAX_SEC = 8.0;

// 坐下休息时间
export const SEAT_REST_MIN_SEC = 30.0;
export const SEAT_REST_MAX_SEC = 120.0;
```

---

## 6. 对AI Pet项目的借鉴价值

### 高度可复用 ✅

| 功能 | 借鉴程度 | 说明 |
|------|---------|------|
| **Tile-based地图** | ⭐⭐⭐⭐⭐ | 直接复用概念，改瓦片图即可 |
| **Canvas渲染循环** | ⭐⭐⭐⭐⭐ | 完整的游戏循环实现 |
| **角色状态机** | ⭐⭐⭐⭐ | 简化后用于Pet动作 |
| **家具系统** | ⭐⭐⭐⭐ | 改成Pet房间家具 |
| **Y-sorting伪3D** | ⭐⭐⭐⭐ | 增加层次感 |
| **精灵缓存** | ⭐⭐⭐ | 优化性能 |

### 需要修改的部分

1. **瓦片素材**：替换为宠物房间风格
2. **家具类型**：改为宠物相关（床、食盆、玩具）
3. **角色逻辑**：从Agent改为Pet行为

### 快速实现方案

```
方案A：精简版（1-2天）
- 用纯色块做地板/墙壁
- Pet单独渲染在房间上
- 家具用简单图标

方案B：完整版（3-5天）
- 设计16x16像素瓦片集
- 实现完整Tile渲染
- 家具可交互
```

---

## 7. 工作量评估

### 最小可行房间（MVP）

| 任务 | 时间 | 负责 |
|------|------|------|
| 房间背景Canvas层 | 2-4h | Dev |
| 简单瓦片素材 | 2-4h | Design |
| Pet与房间整合 | 2-4h | Dev |
| **合计** | **6-12h** | - |

### 完整交互房间

| 任务 | 时间 | 负责 |
|------|------|------|
| 完整Tile渲染系统 | 8-16h | Dev |
| 像素瓦片集设计 | 8-16h | Design |
| 家具系统 | 4-8h | Dev |
| Pet交互逻辑 | 4-8h | Dev |
| **合计** | **24-48h** | - |

---

## 8. 推荐实现路径

### Phase 1: MVP简易房间（今天可做）

```typescript
// 简单背景层，纯色块
const renderSimpleRoom = (ctx: CanvasRenderingContext2D) => {
  // 地板
  ctx.fillStyle = '#8B7355';  // 木地板色
  ctx.fillRect(0, 100, 400, 300);
  
  // 墙壁
  ctx.fillStyle = '#F5DEB3';  // 米色墙
  ctx.fillRect(0, 0, 400, 100);
  
  // 窗户
  ctx.fillStyle = '#87CEEB';  // 天蓝
  ctx.fillRect(50, 20, 80, 60);
};
```

### Phase 2: 完整Tile系统（后续迭代）

参考openclaw-bot-review的完整实现

---

*Intel 深度调研完成 🔍*
*建议Dev和Design一起评估实现方案*
