---
name: "frontend-dev"
description: "基岩版UI/交互规范。覆盖表单界面、玩家消息、计分板显示、NPC对话等。Invoke when building Minecraft Bedrock UI forms, player interactions, or in-game displays."
---

# 基岩版 UI 与交互规范

> 适用于我的世界基岩版脚本开发中的玩家交互界面（Form 表单、消息系统、计分板、对话等）。

---

## 一、核心原则

### 1.1 消息文案规范

**文案必须简洁清晰**，避免冗长和歧义：

```typescript
// ❌ 错误：啰嗦、含 emoji、格式混乱
player.sendMessage('✅ 恭喜你！你已经成功完成了这个超级棒的任务哦~请继续加油吧！');

// ✅ 正确：简洁、信息明确
player.sendMessage('任务完成：已获得 100 经验值');
```

**文案模板化**：所有面向玩家的文字应使用常量或函数统一管理。

```typescript
// ✅ 推荐：集中管理消息模板
const Messages = {
  ITEM_RECEIVED: (itemName: string, count: number) =>
    `获得了 ${itemName} x${count}`,
  QUEST_COMPLETED: (questName: string) =>
    `任务完成：${questName}`,
  NOT_ENOUGH_ITEMS: (itemName: string, required: number, current: number) =>
    `材料不足：需要 ${itemName} x${required}，当前拥有 x${current}`,
  COOLDOWN_ACTIVE: (seconds: number) =>
    `冷却中，请等待 ${seconds} 秒`,
} as const;

// 使用
player.sendMessage(Messages.ITEM_RECEIVED('钻石', 5));
```

### 1.2 表单交互规范

> 基岩版 UI 表单通过 `@minecraft/server-ui` 模块提供，支持三种类型。

**表单类型选择**：

| 表单类型 | 适用场景 | 最多选项 |
|----------|----------|----------|
| `ActionFormData` | 选择型菜单、确认对话框 | 不限（建议 ≤ 8） |
| `ModalFormData` | 输入型表单、设置页面 | 不限控件 |
| `MessageFormData` | 二选一确认 | 2 个按钮 |

```typescript
import { ActionFormData, ModalFormData, MessageFormData } from '@minecraft/server-ui';

// ✅ 正确：ActionForm - 菜单选择
function showMainMenu(player: Player): void {
  const form = new ActionFormData();
  form.title('主菜单');
  form.body('请选择操作：');
  form.button('查看背包', 'textures/ui/icon_chest');
  form.button('传送回家', 'textures/ui/icon_home');
  form.button('打开商店', 'textures/ui/icon_shop');
  form.button('§c退出', 'textures/ui/icon_close');

  form.show(player).then((response) => {
    if (response.canceled) return;
    switch (response.selection) {
      case 0: openBackpack(player); break;
      case 1: teleportHome(player); break;
      case 2: openShop(player); break;
      case 3: player.sendMessage('已退出'); break;
    }
  });
}

// ✅ 正确：ModalForm - 输入表单
function showSettingsForm(player: Player): void {
  const form = new ModalFormData();
  form.title('个人设置');
  form.toggle('显示伤害数字', true);
  form.slider('音量', 0, 100, 1, 80);
  form.dropdown('难度偏好', ['简单', '普通', '困难'], 1);
  form.textField('自定义称号', '输入你的称号...', '');

  form.show(player).then((response) => {
    if (response.canceled) return;
    const [showDamage, volume, difficulty, title] = response.formValues;
    // 保存设置...
  });
}

// ✅ 正确：MessageForm - 二选一确认
function showConfirmDialog(player: Player, message: string): Promise<boolean> {
  const form = new MessageFormData();
  form.title('确认操作');
  form.body(message);
  form.button1('确认');
  form.button2('取消');

  return form.show(player).then((response) => {
    return response.selection === 0;
  });
}
```

---

## 二、消息系统使用规范

### 2.1 消息类型选择

| 消息类型 | 方法 | 适用场景 |
|----------|------|----------|
| 聊天消息 | `player.sendMessage()` | 一般信息、系统提示 |
| 标题 | `player.onScreenDisplay.setTitle()` | 重要事件、Boss 提示 |
| 动作栏 | `player.onScreenDisplay.setActionBar()` | 实时状态、持续信息 |
| 计分板 | `world.scoreboard` | 排行榜、数据统计 |

```typescript
// ✅ 正确：根据场景选择合适消息类型
function notifyQuestProgress(player: Player, questName: string, progress: number, total: number): void {
  // 聊天消息：详细信息
  player.sendMessage(`任务进度 [${questName}]：${progress}/${total}`);

  // 动作栏：持续可见的进度
  player.onScreenDisplay.setActionBar(`${questName}: ${progress}/${total}`);
}

function notifyBossSpawn(player: Player, bossName: string): void {
  // 标题：重要事件
  player.onScreenDisplay.setTitle(bossName, {
    subtitle: 'Boss 已出现！',
    fadeInDuration: 10,
    stayDuration: 40,
    fadeOutDuration: 10
  });
}
```

### 2.2 颜色代码规范

> 基岩版使用 `§` 格式化代码。

| 用途 | 代码 | 示例 |
|------|------|------|
| 成功/正面 | `§a` (绿色) | `§a操作成功` |
| 警告/提示 | `§e` (黄色) | `§e背包已满` |
| 错误/危险 | `§c` (红色) | `§c权限不足` |
| 重要信息 | `§b` (青色) | `§b新消息` |
| 系统消息 | `§7` (灰色) | `§7系统提示` |
| 高亮数值 | `§6` (金色) | `§6+100 金币` |

```typescript
// ✅ 推荐：集中管理格式化函数
function success(msg: string): string { return `§a${msg}`; }
function warning(msg: string): string { return `§e${msg}`; }
function error(msg: string): string { return `§c${msg}`; }
function highlight(msg: string): string { return `§6${msg}`; }
function dim(msg: string): string { return `§7${msg}`; }

// 使用
player.sendMessage(success('操作成功！获得 ') + highlight('100 金币'));
player.sendMessage(error('权限不足，无法执行此操作'));
```

---

## 三、用户输入处理

### 3.1 输入校验

```typescript
// ✅ 正确：所有用户输入必须校验
function handleCustomCommand(player: Player, args: string[]): void {
  if (args.length < 2) {
    player.sendMessage(error('用法: /mycmd <数量> <物品ID>'));
    return;
  }

  const count = parseInt(args[0], 10);
  if (isNaN(count) || count < 1 || count > 64) {
    player.sendMessage(error('数量必须在 1-64 之间'));
    return;
  }

  const itemId = args[1];
  if (!isValidItemId(itemId)) {
    player.sendMessage(error(`无效的物品ID: ${itemId}`));
    return;
  }

  // 执行逻辑...
}
```

### 3.2 冷却与防刷

```typescript
// ✅ 正确：关键操作必须加冷却
const cooldowns = new Map<string, number>();

function handleWithCooldown(player: Player, action: string, cooldownTicks: number, callback: () => void): void {
  const key = `${player.id}_${action}`;
  const lastUse = cooldowns.get(key) ?? 0;
  const currentTick = system.currentTick;

  if (currentTick - lastUse < cooldownTicks) {
    const remainingTicks = cooldownTicks - (currentTick - lastUse);
    const remainingSeconds = Math.ceil(remainingTicks / 20);
    player.sendMessage(warning(`冷却中，请等待 ${remainingSeconds} 秒`));
    return;
  }

  cooldowns.set(key, currentTick);
  callback();
}
```

---

## 四、表单设计规范

### 4.1 表单标题与描述

- 标题：简短（≤ 20 字符），概括表单用途
- body/描述：可选，提供补充说明，超出 2 行建议省略

### 4.2 按钮规范

- 按钮文字简洁（≤ 10 字符）
- 主要操作放前面
- 危险操作（删除、退出）用 `§c` 标记
- 每个按钮使用时附带纹理图标路径

### 4.3 控件选择

| 控件 | 适用数据 | 示例 |
|------|----------|------|
| `toggle` | 布尔值 | 是否启用PVP |
| `slider` | 整数范围 | 音量 0-100 |
| `dropdown` | 固定选项列表 | 难度选择 |
| `textField` | 自由文本 | 输入称号 |

---

## 五、计分板规范

```typescript
// ✅ 正确：计分板用于数据展示
function setupLeaderboard(world: World, objectiveId: string, displayName: string): void {
  const objective = world.scoreboard.getObjective(objectiveId)
    ?? world.scoreboard.addObjective(objectiveId, displayName);

  world.scoreboard.setObjectiveAtDisplaySlot('sidebar', {
    objective: objective,
  });
}

function updatePlayerScore(player: Player, objectiveId: string, score: number): void {
  const objective = world.scoreboard.getObjective(objectiveId);
  if (!objective) return;

  const participants = objective.getParticipants();
  const participant = participants.find(p => p.displayName === player.name)
    ?? objective.addScore(player.name, 0);

  objective.setScore(player.name, score);
}
```

---

## 六、交付标准

### 用户交互
- [ ] 表单标题简洁清晰
- [ ] 错误提示有明确的解决方案
- [ ] 关键操作有冷却机制
- [ ] 重要操作有二次确认

### 消息系统
- [ ] 颜色代码语义化（绿=成功，红=错误，黄=警告）
- [ ] 消息模板统一管理
- [ ] 动作栏用于实时状态，标题用于重要事件

### 代码质量
- [ ] 所有玩家可见文字在常量/函数中管理
- [ ] 表单回调处理 `canceled` 状态
- [ ] 用户输入有完整校验

---

**最后更新**: 2026-10-14