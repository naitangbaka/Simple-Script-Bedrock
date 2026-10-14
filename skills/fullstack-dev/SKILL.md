---
name: "fullstack-dev"
description: "基岩版脚本开发架构规范。覆盖脚本架构、事件系统、数据持久化、模块组织。Invoke when setting up script architecture, organizing modules, or needing project structure guidelines."
---

# 基岩版 脚本开发架构规范

> 适用于我的世界基岩版脚本开发（@minecraft/server 等官方模块）。默认当前目录即为行为包根目录。

---

## 核心原则

**API 优先，指令按需使用**：实现功能时优先使用 Script API（`@minecraft/server` 等模块）编写代码，而非用 `/execute`、`/give`、`/tp` 等命令字符串。除非用户明确说明"用指令实现"。

```typescript
// ❌ 错误：默认用指令字符串
player.runCommand(`give @s diamond 1`);
player.runCommand(`tp @s 100 64 100`);
dimension.runCommand(`summon minecraft:creeper ~ ~ ~`);

// ✅ 正确：用 Script API 实现
const container = player.getComponent('inventory')?.container;
container?.addItem(new ItemStack('minecraft:diamond', 1));
player.teleport({ x: 100, y: 64, z: 100 }, { dimension: dimension });
dimension.spawnEntity('minecraft:creeper', { x: 0, y: 64, z: 0 });

// ✅ 用户说"用指令"时才用命令
// 用户：用指令给玩家钻石
player.runCommand(`give @s diamond 64`);
```

| 对比项 | Script API | 命令（runCommand） |
|--------|-----------|-------------------|
| 类型安全 | ✅ 编译期检查 | ❌ 字符串拼写错误运行时才发现 |
| 性能 | ✅ 直接调用 | ❌ 需解析字符串 |
| 返回值 | ✅ 直接获取对象/结果 | ❌ 难以获取结构化返回值 |
| 条件判断 | ✅ 标准 if/else | ❌ 需选择器 + execute 嵌套 |
| 错误处理 | ✅ try/catch | ❌ 命令失败无明确异常 |

---

## Addon 项目结构

> 本文档默认当前目录即为行为包根目录。`src/` 存放 TypeScript 源码，`scripts/` 存放编译后的 JS 文件。`manifest.json` 中 `script` 模块的 `entry` 指向 `scripts/main.js`。`.gitignore` 应忽略 `scripts/` 目录。

### 脚本入口 (main.ts) 规范

```typescript
// main.ts - 脚本入口，只做注册和初始化
import { world, system } from '@minecraft/server';
import { CustomBlockSystem } from './systems/custom-blocks';
import { CustomItemSystem } from './systems/custom-items';
import { EventRouter } from './core/event-handler';

// 系统实例
const blockSystem = new CustomBlockSystem();
const itemSystem = new CustomItemSystem();
const eventRouter = new EventRouter();

// 注册事件
eventRouter.register('blockPlace', blockSystem.onBlockPlace);
eventRouter.register('blockBreak', blockSystem.onBlockBreak);
eventRouter.register('itemUse', itemSystem.onItemUse);

// 启动 Tick 循环（如有需要）
system.runInterval(() => {
  blockSystem.tick();
  itemSystem.tick();
}, 20); // 每秒 20 tick = 1 秒

world.sendMessage('§aAddon 已加载');
```

---

## 脚本架构设计

### 分层架构

```
┌─────────────────────────────────────────────┐
│              Event Layer                     │  ← 事件监听与分发
├─────────────────────────────────────────────┤
│              System Layer                    │  ← 功能系统逻辑
├─────────────────────────────────────────────┤
│              Manager Layer                   │  ← 数据/状态管理
├─────────────────────────────────────────────┤
│              Core Layer                      │  ← 基础工具/调度器
└─────────────────────────────────────────────┘
```

### 事件系统设计

```typescript
// core/event-handler.ts
import { world, Player, Block, system } from '@minecraft/server';

type EventCallback<T> = (data: T) => void;

class EventRouter {
  private handlers = new Map<string, EventCallback<unknown>[]>();

  register<T>(eventName: string, handler: EventCallback<T>): void {
    const handlers = this.handlers.get(eventName) ?? [];
    handlers.push(handler as EventCallback<unknown>);
    this.handlers.set(eventName, handlers);
  }

  emit<T>(eventName: string, data: T): void {
    const handlers = this.handlers.get(eventName);
    if (handlers) {
      for (const handler of handlers) {
        try {
          handler(data);
        } catch (e) {
          console.error(`事件处理错误 [${eventName}]: ${e}`, e);
        }
      }
    }
  }
}

// 使用 Minecraft 原生事件映射到自定义事件路由
function setupWorldEvents(router: EventRouter): void {
  world.afterEvents.blockPlace.subscribe((event) => {
    router.emit('blockPlace', {
      player: event.player,
      block: event.block,
      dimension: event.dimension
    });
  });

  world.afterEvents.playerJoin.subscribe((event) => {
    router.emit('playerJoin', {
      player: event.player,
      playerName: event.playerName,
      playerId: event.playerId
    });
  });
}
```

---

## 数据持久化

> 基岩版脚本环境中没有文件系统、没有数据库。数据持久化只能使用以下官方 API：

| 方式 | API | 适用场景 | 容量限制 |
|------|-----|----------|----------|
| 动态属性 | `entity.setDynamicProperty()` | 玩家数据、方块数据 | 无明确限制 |
| 计分板 | `world.scoreboard` | 数值统计、排行榜 | 最大 2,147,483,647 |
| 标签 | `entity.addTag()` | 布尔标记 | 无明确限制 |

```typescript
// ✅ 推荐：动态属性 - 存储结构化数据
class PlayerDataManager {
  private readonly KEY = 'playerData';

  saveData(player: Player, data: PlayerData): void {
    player.setDynamicProperty(this.KEY, JSON.stringify(data));
  }

  loadData(player: Player): PlayerData | null {
    const raw = player.getDynamicProperty(this.KEY) as string | undefined;
    if (!raw) return null;
    try {
      return JSON.parse(raw) as PlayerData;
    } catch {
      return null;
    }
  }

  updateField(player: Player, field: keyof PlayerData, value: unknown): void {
    const data = this.loadData(player) ?? {} as PlayerData;
    (data as Record<string, unknown>)[field] = value;
    this.saveData(player, data);
  }
}

interface PlayerData {
  coins: number;
  level: number;
  lastLogin: number;
  settings: PlayerSettings;
}
```

---

## Tick 调度器

```typescript
// core/tick-scheduler.ts
import { system } from '@minecraft/server';

class TickScheduler {
  private tasks = new Map<string, TickTask>();

  schedule(id: string, intervalTicks: number, callback: () => void): void {
    this.tasks.set(id, { interval: intervalTicks, lastRun: 0, callback });
  }

  cancel(id: string): void {
    this.tasks.delete(id);
  }

  start(): void {
    system.runInterval(() => {
      const now = system.currentTick;
      for (const [id, task] of this.tasks) {
        if (now - task.lastRun >= task.interval) {
          task.callback();
          task.lastRun = now;
        }
      }
    }, 1);
  }
}

interface TickTask {
  interval: number;
  lastRun: number;
  callback: () => void;
}
```

---

## 性能优化

### 资源控制原则

| 优化项 | 策略 | 说明 |
|--------|------|------|
| Tick 频率 | 避免每 tick 执行重逻辑 | 用 `system.runInterval` 降低频率 |
| 事件订阅 | 按需订阅，不用即退 | 大量事件监听影响性能 |
| 循环遍历 | 避免全图遍历实体/方块 | 用选择器 + 半径限制 |
| 动态属性 | 批量读写，减少调用次数 | 每次调用都是跨边界操作 |
| 实体操作 | 控制同时存在的实体数量 | 过多实体会导致卡顿 |

```typescript
// ❌ 错误：每 tick 遍历所有玩家做重计算
system.runInterval(() => {
  for (const player of world.getPlayers()) {
    heavyCalculation(player); // 20次/秒 × 玩家数 = 性能灾难
  }
}, 1);

// ✅ 正确：降低频率 + 按需执行
system.runInterval(() => {
  for (const player of world.getPlayers()) {
    if (player.hasTag('needsUpdate')) {
      lightCalculation(player);
      player.removeTag('needsUpdate');
    }
  }
}, 100); // 每 5 秒一次
```

---

## 部署与打包

### manifest.json 要点

```json
{
  "format_version": 2,
  "header": {
    "name": "Addon名称",
    "description": "Addon描述",
    "uuid": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "version": [1, 0, 0],
    "min_engine_version": [1, 20, 50]
  },
  "modules": [
    {
      "type": "data",
      "uuid": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "version": [1, 0, 0]
    },
    {
      "type": "script",
      "language": "javascript",
      "uuid": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "entry": "scripts/main.js",
      "version": [1, 0, 0]
    }
  ],
  "dependencies": [
    {
      "module_name": "@minecraft/server",
      "version": "1.16.0"
    },
    {
      "module_name": "@minecraft/server-ui",
      "version": "1.6.0"
    }
  ],
  "capabilities": ["script_eval"]
}
```

---

## 项目检查清单

### 开发前
- [ ] 确认最低兼容版本（min_engine_version）
- [ ] 确认需要哪些官方模块（@minecraft/server、server-ui、server-net 等）
- [ ] 设计脚本架构和模块划分
- [ ] 确定数据持久化方案

### 开发中
- [ ] 每个功能系统有独立文件
- [ ] 事件监听合理管理，避免泄漏
- [ ] 性能敏感代码控制频率
- [ ] JSON 配置语法正确

### 发布前
- [ ] manifest.json 版本号正确
- [ ] 所有 UUID 唯一且固定
- [ ] 脚本编译无错误

---

**最后更新**: 2026-10-14