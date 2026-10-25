---
name: "oop-design"
description: "OOP design principles: when to use OOP vs functions, avoid procedural/result-oriented coding. Invoke when writing new modules, refactoring, or reviewing architecture."
---

# 面向对象设计规范

> 适用于我的世界基岩版脚本开发（@minecraft/server 等官方模块）。

---

## 核心原则

### 0. 依赖选择优先级（基岩版限制）

> ⚠️ **关键约束**：我的世界基岩版脚本运行在游戏内置的沙箱 JavaScript 引擎中，**不是 Node.js 环境**。
> 只能使用官方白名单模块。随意安装第三方 npm 库（如 lodash、date-fns、zod、axios）**不会起效果**，会导致运行时错误。

实现功能时，按以下优先级选择依赖来源：

1. **优先使用 Minecraft 官方 API 模块** — `@minecraft/server`、`@minecraft/server-ui`、`@minecraft/server-net` 等官方白名单模块
2. **其次查找项目内已有封装** — 项目已有公共工具函数、类、类型定义，避免重复造轮子
3. **最后才自己实现** — 前两条都不可行时，用纯 TypeScript + 官方 API 手动实现

```typescript
// ✅ 优先：使用 Minecraft 官方 API
import { world, system, Player, Block, ItemStack } from '@minecraft/server';
import { ActionFormData } from '@minecraft/server-ui';

// ✅ 其次：复用项目已有封装
import { CustomBlockRegistry } from '../registry/block-registry';
import { formatGameMessage } from '../utils/message-utils';

// ❌ 禁止：安装任意第三方 npm 库（在游戏沙箱中无法运行）
// import { z } from 'zod';            // ❌ 不生效
// import { v4 as uuid } from 'uuid';  // ❌ 不生效
// import _ from 'lodash';             // ❌ 不生效

// ✅ 替代方案：用纯 TypeScript 手动实现需要的功能
function generateId(): string {
  return `${Date.now()}_${Math.random().toString(36).slice(2, 9)}`;
}

function isCustomBlockConfig(data: unknown): data is CustomBlockConfig {
  if (typeof data !== 'object' || data === null) return false;
  const candidate = data as Record<string, unknown>;
  return (
    typeof candidate.typeId === 'string' &&
    typeof candidate.displayName === 'string'
  );
}
```

**自检清单**：
- 这个功能 Minecraft 官方 API 是否已提供？（查 `@minecraft/server` 等模块文档）
- 项目中是否已有类似功能的封装？（搜索代码库的公共模块）
- 如果要引入新模块，它是否在官方白名单内？（**不在白名单则不安装**）
- 如果自己实现，复杂度是否可控？

### 1. 禁止面向过程编程

**不要**将业务逻辑写成一步步的过程式调用。过程式代码的典型特征：
- 一个巨大的函数包含所有步骤
- 数据与行为分离（裸数据 + 外部处理函数）
- 通过参数传递 mutable 状态，层层穿透

**正确做法**：将数据和行为封装为有明确职责的类，通过对象协作完成任务。

```typescript
// ❌ 面向过程：数据裸奔，行为外挂
function giveCustomItem(player: Player, itemId: string) {
  const config = getItemConfig(itemId);
  if (!config) return;
  const item = new ItemStack(config.typeId, 1);
  const container = player.getComponent('inventory')?.container;
  if (!container) return;
  container.addItem(item);
  player.sendMessage(`获得了 ${config.displayName}`);
  logItemGive(player.name, itemId);
}

// ✅ 面向对象：数据与行为内聚
class CustomItemGiver {
  constructor(
    private registry: ItemRegistry,
    private logger: GameLogger
  ) {}

  giveItem(player: Player, itemId: string): boolean {
    const config = this.registry.getById(itemId);
    if (!config) return false;

    const added = this.addToInventory(player, config);
    if (added) {
      player.sendMessage(`获得了 ${config.displayName}`);
      this.logger.logItemGive(player.name, itemId);
    }
    return added;
  }

  private addToInventory(player: Player, config: CustomItemConfig): boolean {
    const container = player.getComponent('inventory')?.container;
    if (!container) return false;
    const item = new ItemStack(config.typeId, 1);
    container.addItem(item);
    return true;
  }
}
```

### 2. 禁止面向结果编程

**不要**为了快速完成功能而牺牲代码结构。典型表现：
- 在已有类上不断追加 if/else 分支
- 不做重构，只在原代码上打补丁
- 为了"少改代码"而选择耦合方案
- 复制粘贴相似逻辑而不抽取公共抽象

**正确做法**：每次添加功能时，花时间思考正确的抽象层级。如果现有结构无法优雅支持新需求，先重构再添加。

### 3. 可扩展性优先于快速交付

设计时自问：
- 如果明天要加一种新的方块/物品/实体类型，我需要改多少处代码？
- 这个类的职责是否单一？换一个需求场景，它能否复用？
- 外部调用者是否需要了解内部实现细节才能使用？

**开闭原则（OCP）**：对扩展开放，对修改关闭。新增功能应该通过添加新代码实现，而非修改已有代码。

```typescript
// ❌ 每次加新方块类型都要改这里（违反 OCP）
function getBlockBehavior(blockType: string): BlockBehavior {
  if (blockType === 'cursed_stone') return new CursedStoneBehavior();
  if (blockType === 'healing_flower') return new HealingFlowerBehavior();
  if (blockType === 'explosive_ore') return new ExplosiveOreBehavior(); // 新增就要改这里
  return new DefaultBlockBehavior();
}

// ✅ 新增类型只需新增类（符合 OCP）
interface BlockBehavior {
  onPlace(block: Block, player: Player): void;
  onBreak(block: Block, player: Player): void;
  onInteract(block: Block, player: Player): void;
}

class CursedStoneBehavior implements BlockBehavior {
  onPlace(block: Block, player: Player) { /* 诅咒效果 */ }
  onBreak(block: Block, player: Player) { /* 解除诅咒 */ }
  onInteract(block: Block, player: Player) { /* 显示提示 */ }
}
// 新增方块：加一个 class 即可，不改任何已有代码
class HealingFlowerBehavior implements BlockBehavior {
  onPlace(block: Block, player: Player) { /* 治疗光环 */ }
  onBreak(block: Block, player: Player) { /* 特效 */ }
  onInteract(block: Block, player: Player) { /* 使用治疗 */ }
}
```

---

## 何时使用 OOP（Class） vs 函数（Function）

### 适合用 Class 的场景

| 场景 | 判断标准 | MC 示例 |
|------|----------|---------|
| **有状态实体** | 数据有生命周期，需要在多个方法间共享和修改 | 自定义方块行为管理、玩家数据缓存 |
| **需要多态** | 同一行为有多种实现，需要运行时动态替换 | 不同方块/物品的交互行为、技能效果系统 |
| **资源管理** | 需要初始化/释放的资源 | 定时器管理、事件监听器生命周期 |
| **领域模型** | 表达游戏概念的实体 | 自定义方块配置、自定义实体、合成配方 |
| **对外暴露多个相关操作** | 一组函数共享同一份配置或上下文 | 注册表（方块/物品/实体注册） |

```typescript
// ✅ 有状态、需要多态 → 用 Class
abstract class CustomBlockHandler {
  constructor(protected config: CustomBlockConfig) {}

  abstract onPlace(block: Block, player: Player): void;
  abstract onBreak(block: Block, player: Player): void;

  // 公共方法：所有方块共用的逻辑
  protected playPlacementEffect(block: Block): void {
    const dimension = block.dimension;
    dimension.spawnParticle('minecraft:large_explosion', block.location);
  }
}

class TrapBlockHandler extends CustomBlockHandler {
  onPlace(block: Block, player: Player): void {
    this.playPlacementEffect(block);
    player.sendMessage('陷阱方块已放置');
    // 启动陷阱计时器...
  }

  onBreak(block: Block, player: Player): void {
    player.sendMessage('陷阱方块被拆除');
  }
}
```

### 适合用 Function 的场景

| 场景 | 判断标准 | MC 示例 |
|------|----------|---------|
| **纯计算/转换** | 输入 → 输出，无副作用，不依赖外部状态 | 坐标转换、伤害计算、掉落表计算 |
| **工具函数** | 通用且无状态的辅助操作 | 消息格式化、ID 生成、颜色代码转换 |
| **组合/编排** | 将多个小函数组合成流程 | 命令序列执行、事件处理管道 |
| **单次操作** | 只需要一个行为的简单操作 | 播放音效、发送消息 |

```typescript
// ✅ 无状态、纯转换 → 用 Function
function calculateDamage(
  baseDamage: number,
  enchantLevel: number,
  targetArmor: number
): number {
  const enchantedDamage = baseDamage * (1 + enchantLevel * 0.25);
  return Math.max(0, enchantedDamage - targetArmor * 0.5);
}

function formatItemName(typeId: string): string {
  return typeId
    .replace('minecraft:', '')
    .split('_')
    .map(word => word.charAt(0).toUpperCase() + word.slice(1))
    .join(' ');
}

function generateUniqueId(prefix: string): string {
  return `${prefix}_${system.currentTick}_${Math.random().toString(36).slice(2, 8)}`;
}
```

---

## 避免过度封装

**过度封装信号**：
- 一个类只有一个方法（可能是 `run()` / `execute()`），无其他行为 → 改成函数
- 一个接口只有一个实现，且未来几乎不会有第二个实现 → 先别抽象，YAGNI（You Aren't Gonna Need It）
- 为了"标准"而引入大量间接层（FactoryFactory 式的套娃）
- 一个类的字段全是 getter/setter，没有任何真实行为 → 可能是贫血模型

```typescript
// ❌ 过度封装：一个类只有一个方法，改成函数更清晰
class MessageSender {
  send(player: Player, msg: string): void {
    player.sendMessage(msg);
  }
}

// ✅ 直接用函数
function sendMessage(player: Player, msg: string): void {
  player.sendMessage(msg);
}

// ❌ 过度抽象：只有一个实现就引入接口
interface IBlockRegistry {
  getById(id: string): CustomBlockConfig | undefined;
  register(id: string, config: CustomBlockConfig): void;
}
class SingleBlockRegistry implements IBlockRegistry {
  private blocks = new Map<string, CustomBlockConfig>();
  getById(id: string) { return this.blocks.get(id); }
  register(id: string, config: CustomBlockConfig) { this.blocks.set(id, config); }
}

// ✅ 先直接用具体类，等真正出现第二个实现时再抽象
class BlockRegistry {
  private blocks = new Map<string, CustomBlockConfig>();

  getById(id: string): CustomBlockConfig | undefined {
    return this.blocks.get(id);
  }

  register(id: string, config: CustomBlockConfig): void {
    this.blocks.set(id, config);
  }
}
```

---

## 设计决策速查

| 问题 | 答案 |
|------|------|
| 这段逻辑有状态吗？ | 有 → Class / 无 → Function |
| 需要运行时替换实现吗？ | 是 → 接口/抽象类 / 否 → 具体类或函数 |
| 未来大概率会扩展吗？ | 是 → 预留扩展点（策略/插件模式）/ 否 → YAGNI，先简单实现 |
| 这个类有几个 public 方法？ | 1 个 → 考虑改成函数 / 2+ → 保持类 |
| 新增功能需要改几处？ | 1 处 → 良好 / 3+ 处 → 考虑重构 |
| 需要引入第三方库？ | ⚠️ 先确认是否在 Minecraft 官方白名单内 |

---

## SOLID 原则精要

1. **SRP 单一职责**：一个类只应该因为一个原因被修改。如果一个类承担了多个职责，拆分它。
2. **OCP 开闭原则**：对扩展开放，对修改关闭。加功能 = 加代码，不改已有代码。
3. **LSP 里氏替换**：子类必须能完全替代父类。不要写出"is-a 但实际不能用"的继承。
4. **ISP 接口隔离**：接口应该小而精，调用方不应该依赖它不需要的方法。
5. **DIP 依赖倒置**：依赖抽象（接口）而非具体实现。高层模块和低层模块都依赖抽象。

---

## 工作流程

接到需求时，按以下顺序思考：

1. **理解领域** → 识别核心游戏实体和行为（方块、物品、实体、玩家动作）
2. **检查官方 API** → Minecraft 官方模块是否已提供对应能力？
3. **定义接口** → 先定契约（interface/type），再定实现
4. **分配职责** → 谁拥有这个行为？谁持有这份数据？
5. **确定抽象层级** → 哪些是可变策略（需要多态），哪些是固定逻辑（可以直接实现）
6. **编写代码** → 先写调用方代码（自顶向下），再填充实现
7. **自检** → 用上面的"设计决策速查"快速过一遍

---

**最后更新**: 2026-10-14