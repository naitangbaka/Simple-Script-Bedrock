---
name: "error-check"
description: "常用错误检查技能。覆盖 Native function 异步调度、常见运行时错误、编译期检查。Invoke when debugging, reviewing code, or before finalizing scripts."
---

# 常用错误检查

> 适用于我的世界基岩版脚本开发（@minecraft/server 等官方模块）。在编码完成和提交前逐项自检。

---

## 核心检查清单

### 1. Native Function 异步调度检查

> **关键规则**：所有 `@minecraft/server` 等模块的 API 调用都是**同步 Native 函数**，如果在高频率事件（如 `entityHit`、`tick`）中直接调用重逻辑 API，可能导致游戏卡顿甚至崩溃。

**必须检查**：高频触发的 Native function 是否缺少 `system.run` 进行异步调度？

```typescript
import { world, system, Player } from '@minecraft/server';

// ❌ 危险：在每 tick 事件中直接调用重逻辑 Native API
world.beforeEvents.itemUse.subscribe((event) => {
  const player = event.source;
  heavyNativeOperation(player); // 阻塞游戏主线程
});

// ✅ 正确：使用 system.run 将重逻辑推迟到下一帧执行
world.beforeEvents.itemUse.subscribe((event) => {
  const player = event.source;
  system.run(() => {
    heavyNativeOperation(player); // 异步执行，不阻塞当前 tick
  });
});

// ❌ 危险：循环中密集调用 Native API 未做延迟
function processAllPlayers(): void {
  for (const player of world.getAllPlayers()) {
    player.runCommand('say hello');      // 连续同步调用
    player.addExperience(100);           // 连续同步调用
    player.setDynamicProperty('data', 'value'); // 连续同步调用
  }
}

// ✅ 正确：通过 system.run 分帧执行
function processAllPlayersAsync(): void {
  const players = [...world.getAllPlayers()];
  let index = 0;

  function processNext(): void {
    if (index >= players.length) return;
    const player = players[index]!;
    player.runCommand('say hello');
    player.addExperience(100);
    player.setDynamicProperty('data', 'value');
    index++;
    system.run(processNext); // 下一帧处理下一个玩家
  }

  system.run(processNext);
}
```

**自检问题**：
- 这个事件回调是否在高频触发？（tick / entityHit / projectileHit）
- 回调中是否有超过 3 个连续 Native API 调用？
- 回调中是否遍历了所有玩家/实体？
- 如果以上任一为"是" → 使用 `system.run` 包裹或分帧处理

#### afterEvents 不需要 system.run

> **重要**：`afterEvents` 中的回调**不需要**使用 `system.run` 进行异步调度。`afterEvents` 本身就是事件完成后的回调，此时游戏状态已经稳定，直接操作是安全的。

```typescript
import { world, Player } from '@minecraft/server';

// ✅ afterEvents 中直接操作即可，无需 system.run
world.afterEvents.entityDie.subscribe((event) => {
  const deadEntity = event.deadEntity;
  const damageSource = event.damageSource;

  if (damageSource.damagingEntity instanceof Player) {
    damageSource.damagingEntity.sendMessage(`击杀了 ${deadEntity.typeId}`);
  }
});

// ✅ afterEvents.playerJoin 也无需 system.run
world.afterEvents.playerJoin.subscribe((event) => {
  event.player.sendMessage('欢迎回来！');
  event.player.addExperience(0); // 初始化经验条显示
});

// ❌ 不必多此一举
world.afterEvents.entityDie.subscribe((event) => {
  system.run(() => { // ← 不需要，afterEvents 已经是事件后回调
    handleDeath(event);
  });
});
```

| 事件类型 | 是否需要 system.run | 原因 |
|----------|-------------------|------|
| `beforeEvents` | ✅ 重逻辑时需要 | 事件进行中，阻塞会影响游戏 |
| `afterEvents` | ❌ 不需要 | 事件已完成，状态已稳定 |
| `system.runInterval` | ✅ 回调中重逻辑需要 | Tick 循环高频率触发 |
| `system.runTimeout` | ❌ 不需要 | 本身就是延迟/异步机制 |

---

### 2. 事件取消检查

> `beforeEvents` 事件的 `cancel` 属性用于阻止事件的默认行为。遗漏 `cancel = true` 是常见 bug。

```typescript
import { world } from '@minecraft/server';

// ❌ 错误：想要阻止方块放置但忘了设置 cancel
world.beforeEvents.blockPlace.subscribe((event) => {
  if (isRestrictedBlock(event.block)) {
    event.cancel = true; // ← 必须显式设置为 true
  }
});

// ✅ 正确
world.beforeEvents.blockPlace.subscribe((event) => {
  if (isRestrictedBlock(event.block)) {
    event.cancel = true;
  }
});

// ✅ 更安全的模式：确保所有路径都明确 cancel 值
world.beforeEvents.blockPlace.subscribe((event) => {
  event.cancel = isRestrictedBlock(event.block);
});
```

---

### 3. 空值/未定义检查

> Script API 的 `getComponent()`、`getDynamicProperty()` 等方法可能返回 `undefined`，直接访问其属性会导致运行时错误。

```typescript
import { Player, ItemStack } from '@minecraft/server';

// ❌ 危险：未检查 getComponent 返回值
function giveItem(player: Player, typeId: string): void {
  const container = player.getComponent('inventory')!.container; // 可能 undefined!
  container!.addItem(new ItemStack(typeId, 1));
}

// ✅ 正确：先检查再使用
function giveItem(player: Player, typeId: string): boolean {
  const inventory = player.getComponent('inventory');
  if (!inventory) return false;
  const container = inventory.container;
  if (!container) return false;
  container.addItem(new ItemStack(typeId, 1));
  return true;
}
```

---

### 4. UUID 唯一性检查

> `manifest.json` 中的 `uuid` 必须全局唯一。多人协作或复制粘贴时最容易出错。

```json
// ❌ 风险：多个 manifest 使用相同 UUID
{
  "header": {
    "uuid": "00000000-0000-0000-0000-000000000000"
  },
  "modules": [
    {
      "type": "script",
      "uuid": "00000000-0000-0000-0000-000000000000" // 与 header 相同也不行
    }
  ]
}
```

**检查方法**：
- header.uuid、module.uuid、dependency.uuid 三者必须互不相同
- 行为包和资源包之间也不能共享 UUID
- 不同 module（data/script）必须各有独立 UUID

---

### 5. 模块版本不匹配检查

> `manifest.json` 中 `dependencies` 的模块版本号与实际安装的 npm 包版本不一致，会导致部分 API 不可用。

```json
// ❌ 潜在问题：manifest 写的是具体版本，但 npm 可能安装了不同版本
{
  "dependencies": [
    {
      "module_name": "@minecraft/server",
      "version": "1.10.0"  // 如果实际环境是 1.12.0+，新 API 使用了但没声明
    }
  ]
}
```

**检查方法**：
- 确保使用的 API 在所声明版本的模块中确实存在
- 使用 `"beta"`（字符串）可避免版本号不一致问题（会自动匹配）
- 如果用户要求兼容成就，确认版本策略符合成就兼容性要求

---

### 6. 动态属性键名冲突检查

> `setDynamicProperty()` 的键名在实体/世界范围内是全局的，多个系统使用相同键名会导致数据覆盖。

```typescript
// ❌ 风险：两个系统使用了相同的 key
class CoinSystem {
  saveCoins(player: Player, coins: number): void {
    player.setDynamicProperty('data', coins); // 键名过于通用
  }
}

class QuestSystem {
  saveQuest(player: Player, quest: string): void {
    player.setDynamicProperty('data', quest); // 与 CoinSystem 冲突！
  }
}

// ✅ 正确：使用带命名空间前缀的唯一键名
const COIN_KEY = 'addon:coinSystem:coins';
const QUEST_KEY = 'addon:questSystem:activeQuest';
```

---

### 7. import 路径检查

> 编译输出目录 `scripts/` 中的 `.js` 文件引用路径必须正确，路径区分大小写。

```typescript
// ❌ 常见错误：路径大小写不匹配
import { MyClass } from './Systems/MyModule';  // 实际文件夹是 systems
import { util } from '../utils/helper';         // 实际文件是 Helper.ts

// ✅ 正确：使用小写 + 完整路径（含 .js 扩展名）
import { MyClass } from './systems/my-module.js';
import { util } from '../utils/helper.js';
```

---

### 8. subscribe 未解绑导致内存泄漏

> 脚本重载时（如 `/reload`），旧的 `subscribe` 回调可能仍然存在，导致事件触发多次。

```typescript
import { world } from '@minecraft/server';

// ❌ 风险：subscribe 返回的引用未保存，无法解绑
world.afterEvents.entityHitEntity.subscribe((event) => {
  handleDamage(event);
});

// ✅ 正确：保存引用，在适当时机解绑
const hitSubscription = world.afterEvents.entityHitEntity.subscribe((event) => {
  handleDamage(event);
});

// 在适当时机（如脚本卸载）解绑
function cleanup(): void {
  world.afterEvents.entityHitEntity.unsubscribe(hitSubscription);
}
```

---

## 快速自检流程

编写完成后，按以下顺序快速检查：

| 编号 | 检查项 | 问自己 |
|------|--------|--------|
| 1 | 异步调度 | 高频事件中有重逻辑 Native 调用吗？→ 加 `system.run` |
| 2 | 事件取消 | `beforeEvents` 中是否忘了 `event.cancel = true`？ |
| 3 | 空值检查 | `getComponent()` / `getDynamicProperty()` 返回值检查了吗？ |
| 4 | UUID | manifest.json 中各 UUID 是否互不相同？ |
| 5 | 版本匹配 | 使用的 API 在声明的模块版本中存在吗？ |
| 6 | 键名冲突 | 动态属性的 key 是否有命名空间前缀？ |
| 7 | 导入路径 | import 路径大小写是否正确？ |
| 8 | 事件解绑 | subscribe 的返回值是否保存？是否需要解绑？ |

---

**最后更新**: 2026-10-14