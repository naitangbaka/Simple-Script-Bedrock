---
name: "type-safety"
description: "TypeScript类型安全开发规范。强制严格类型检查、禁止any、类型定义优先。Invoke when writing TypeScript code, defining interfaces, or reviewing type safety."
---

# TypeScript 类型安全开发规范

> 本项目（我的世界基岩版开发）使用 TypeScript 编写游戏脚本。严格遵循类型安全原则，禁止使用 any 类型，所有代码必须有明确的类型定义。

---

## 核心原则

### 1. 强制 TypeScript

- **所有游戏脚本代码必须是 .ts 文件**
- **禁止使用纯 JavaScript**
- **新功能必须先定义类型**

### 2. 禁止 any 类型

```typescript
// ❌ 错误: 使用 any
function processBlockData(data: any): any {
  return data.typeId;
}

// ✅ 正确: 使用具体类型
function processBlockData(data: BlockData): string {
  return data.typeId;
}

// ✅ 可接受: 使用 unknown（需要类型守卫）
function processUnknownBlock(data: unknown): string {
  if (typeof data === 'object' && data !== null && 'typeId' in data) {
    return String((data as { typeId: string }).typeId);
  }
  throw new TypeError('Expected block data');
}
```

### 3. 类型定义优先

开发流程:
1. 定义类型/接口
2. 编写函数/类
3. 实现业务逻辑

```typescript
// 步骤1: 先定义类型
interface CustomBlockConfig {
  typeId: string;
  displayName: string;
  properties: BlockProperties;
}

interface BlockProperties {
  hardness: number;
  resistance: number;
  lightEmission: number;
}

// 步骤2: 基于类型实现功能
function registerCustomBlock(config: CustomBlockConfig): void {
  // 实现...
}
```

---

## TypeScript 配置

### 严格模式配置

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "lib": ["ES2020","dom"],
    "moduleResolution": "node",
    
    // 严格类型检查
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    
    // 额外检查
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    
    // 模块解析
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": false,
    "sourceMap": false,

    // 输出
    "outDir": "./scripts",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "scripts"]
}
```

> **关键配置说明**：
> - `"declaration": false` — **不生成 `.d.ts` 声明文件**。MC 游戏引擎不需要类型声明，生成它们只会污染 scripts 输出目录。
> - `"sourceMap": false` — **不生成 `.js.map` 文件**。游戏沙箱不加载 source map，保留它们只会让目录杂乱。
> - `"outDir": "./scripts"` — 编译输出直接到行为包的 scripts 目录，由 manifest.json 的 `entry` 指向。

---

## package.json 构建规范

> 默认构建配置应保持输出目录干净，不生成冗余文件。

### 标准配置

```json
{
  "name": "addon-name",
  "version": "1.0.0",
  "description": "Addon 描述",
  "main": "scripts/main.js",
  "scripts": {
    "build": "tsc",
    "typecheck": "tsc --noEmit",
    "typecheck:watch": "tsc --noEmit --watch"
  },
  "dependencies": {
    "@minecraft/server": "^2.10.0",
    "@minecraft/server-ui": "^2.2.0"
  }
}
```

### 规范说明

| 字段 | 要求 | 说明 |
|------|------|------|
| `main` | 指向编译入口 `"scripts/main.js"` | 与 manifest.json 中 script module 的 entry 一致 |
| `build` | `"tsc"` | 调用 tsconfig.json 的编译配置，默认不生成 d.ts 和 map |
| `typecheck` | `"tsc --noEmit"` | 仅做类型检查，不产出任何文件 |
| `dependencies` | 仅含 `@minecraft/*` 官方模块 | 禁止非官方 npm 包 |

### 目录清洁原则

```bash
# ✅ 干净的 scripts 目录（只有 .js 文件）
scripts/
├── main.js
├── types/
│   ├── index.js
│   └── ...
├── core/
│   ├── event-handler.js
│   └── ...
└── systems/
    ├── custom-blocks.js
    └── ...

# ❌ 杂乱：多余的 d.ts 和 map 文件
scripts/
├── main.js
├── main.d.ts          # ← 无用，游戏引擎不加载
├── main.js.map        # ← 无用，沙箱不支持 source map
├── types/
│   ├── index.js
│   ├── index.d.ts      # ← 污染目录
│   ├── index.js.map    # ← 污染目录
│   └── ...
```

---

## 类型命名规范

### 命名约定

| 类型 | 命名方式 | 示例 |
|------|----------|------|
| **接口** | PascalCase | `CustomItemConfig`, `GameEventData` |
| **类型别名** | PascalCase | `BlockId`, `EntityType` |
| **Const 对象** | PascalCase | `GameMode`, `Dimension` |
| **泛型参数** | 单字母大写 | `T`, `K`, `V`, `R` |

### 命名示例

```typescript
// ✅ 正确: 接口命名
interface CustomItemConfig {
  typeId: string;
  displayName: string;
  maxStackSize: number;
}

interface CustomEntityConfig {
  typeId: string;
  health: number;
  speed: number;
}

// ✅ 正确: 类型别名
type BlockId = string;
type ItemId = string;
type GameTick = number;

// ✅ 正确: Const 对象（推荐方案，避免 enum 同名冲突）
const GameMode = {
  SURVIVAL: 'survival',
  CREATIVE: 'creative',
  ADVENTURE: 'adventure',
  SPECTATOR: 'spectator'
} as const;

type GameMode = typeof GameMode[keyof typeof GameMode];

// ✅ 正确: 事件数据接口
interface PlayerJoinEventData {
  playerName: string;
  playerId: string;
  timestamp: number;
}
```

---

### 类型导出规范

```typescript
// types/blocks.ts
export interface CustomBlockConfig {
  typeId: string;
  displayName: string;
  properties: BlockProperties;
  components?: BlockComponent[];
}

export interface BlockProperties {
  hardness: number;
  resistance: number;
  lightEmission: number;
  friction: number;
}

export interface BlockComponent {
  type: string;
  data: Record<string, unknown>;
}

// types/index.ts
export * from './blocks';
export * from './items';
export * from './entities';
export * from './events';
export * from './ui';
export * from './config';
```

---

## 泛型使用

### 泛型接口

```typescript
// 通用注册表接口
interface Registry<T, ID = string> {
  getById(id: ID): T | undefined;
  getAll(): T[];
  register(id: ID, entry: T): void;
  unregister(id: ID): boolean;
}

// 实现：方块注册表
class BlockRegistry implements Registry<CustomBlockConfig> {
  private blocks = new Map<string, CustomBlockConfig>();

  getById(id: string): CustomBlockConfig | undefined {
    return this.blocks.get(id);
  }

  getAll(): CustomBlockConfig[] {
    return Array.from(this.blocks.values());
  }

  register(id: string, entry: CustomBlockConfig): void {
    this.blocks.set(id, entry);
  }

  unregister(id: string): boolean {
    return this.blocks.delete(id);
  }
}
```

### 泛型工具类型

```typescript
// 使用 TypeScript 内置工具类型
type BlockCreate = Omit<CustomBlockConfig, 'typeId'>;
type BlockUpdate = Partial<BlockCreate>;
type BlockReadonly = Readonly<CustomBlockConfig>;

// 自定义工具类型
type Nullable<T> = T | null;
type AsyncResult<T> = Promise<{ success: boolean; data?: T; error?: string }>;
type EventHandler<T = void> = T extends void ? () => void : (data: T) => void;
```

---

## instanceof vs typeId 类型判断

> 在校验 Minecraft 实体/玩家等对象类型时，**优先使用 `instanceof` 而非字符串 `typeId` 比较**。

### 规则

| 方式 | 推荐度 | 说明 |
|------|--------|------|
| `instanceof Player` | ✅ 优先 | 类型安全，TypeScript 自动收窄类型，不依赖字符串 |
| `entity.typeId === 'minecraft:player'` | ⚠️ 仅在必要时 | 字符串可能拼错，无类型收窄，且需确认 typeId 准确值 |

### 示例

```typescript
import { world, Player, Entity } from '@minecraft/server';

// ✅ 推荐：使用 instanceof 判断玩家
world.afterEvents.entityDie.subscribe((event) => {
  const source = event.damageSource.damagingEntity;

  if (source instanceof Player) {
    // TypeScript 已自动收窄 source 为 Player 类型
    source.sendMessage('击杀了目标');
    source.addExperience(10);
    const inventory = source.getComponent('inventory'); // 只有 Player 有的方法可直接用
  }
});

// ❌ 不推荐：用 typeId 字符串比较
world.afterEvents.entityDie.subscribe((event) => {
  const source = event.damageSource.damagingEntity;

  if (source?.typeId === 'minecraft:player') {
    // TypeScript 不知道 source 是 Player，需要手动类型断言
    const player = source as Player;
    player.sendMessage('击杀了目标');
  }
});

// ✅ 对非玩家的实体判断（无专用类时用 typeId）
world.afterEvents.entityDie.subscribe((event) => {
  const deadEntity = event.deadEntity;

  // 对特定实体类型判断 — 只能用 typeId，因为没有 Creeper、Zombie 等专用类
  if (deadEntity.typeId === 'minecraft:creeper') {
    // 苦力怕死亡特殊处理
  }
});
```

**总结**：
- `Player` 有专用类 → 用 `instanceof Player`
- 其他实体（Creeper、Zombie、Skeleton 等）没有专用类 → 用 `typeId` 字符串比较
- 自定义实体 → 用 `typeId` 或 `hasTag()` 判断

---

## 类型守卫

### 使用 Zod 进行运行时校验

> 推荐统一使用 `zod` 定义运行时校验，兼作类型守卫的数据源。

```typescript
import { z } from 'zod';

// 1. 定义 Schema（单一来源）
const CustomItemSchema = z.object({
  typeId: z.string(),
  displayName: z.string(),
  maxStackSize: z.number().min(1).max(64)
});

// 2. 推导出类型
type CustomItem = z.infer<typeof CustomItemSchema>;

// 3. 类型守卫函数
function isCustomItem(data: unknown): data is CustomItem {
  return CustomItemSchema.safeParse(data).success;
}

// 4. 解析（校验失败抛错）
function parseCustomItem(data: unknown): CustomItem {
  return CustomItemSchema.parse(data);
}

// 使用示例
function processItemConfig(data: unknown): void {
  if (isCustomItem(data)) {
    // TypeScript 已收窄 data 为 CustomItem
    console.log(data.displayName);
  }
}
```

### 简单场景的手写守卫

对于不需要 `zod` 的简单场景，手写守卫必须逐字段做 `typeof` 校验：

```typescript
function isBlockProperties(data: unknown): data is BlockProperties {
  if (typeof data !== 'object' || data === null) return false;
  const candidate = data as Record<string, unknown>;
  return (
    typeof candidate.hardness === 'number' &&
    typeof candidate.resistance === 'number' &&
    typeof candidate.lightEmission === 'number' &&
    typeof candidate.friction === 'number'
  );
}
```

---

## const 对象替代 enum

> 统一采用 `const 对象 + as const` 方案，避免 enum 的运行时体积与 tree-shaking 问题。

```typescript
// ✅ 推荐: const 对象 + as const
const Dimension = {
  OVERWORLD: 'minecraft:overworld',
  NETHER: 'minecraft:nether',
  THE_END: 'minecraft:the_end'
} as const;

type Dimension = typeof Dimension[keyof typeof Dimension];

// ✅ 数字常量同理
const ErrorCode = {
  UNKNOWN_BLOCK: 1001,
  UNKNOWN_ITEM: 1002,
  INVALID_CONFIG: 1003,
  REGISTRATION_FAILED: 1004
} as const;

type ErrorCode = typeof ErrorCode[keyof typeof ErrorCode];
```

### 为什么不用 enum

- `enum` 的产物包含一个**反向映射对象**（尤其数字枚举），即使只用其中一个成员，整个对象也会被打包保留
- `const 对象 + as const` 在类型层无运行时开销，未使用的成员可被彻底移除（tree-shaking 更彻底）
- `const 对象 + as const` 可配合类型推导，支持字面量狭窄化（`typeof X[keyof typeof X]`）

---

## 类型检查命令

```bash
# 类型检查（不生成输出）
tsc --noEmit

# 严格模式检查
tsc --strict --noEmit

# 显示详细错误信息
tsc --noEmit --pretty
```

### package.json 脚本

```json
{
  "scripts": {
    "build": "tsc",
    "typecheck": "tsc --noEmit",
    "typecheck:watch": "tsc --noEmit --watch"
  }
}
```

---

## 常见类型错误

### 1. 隐式 any

```typescript
// ❌ 错误: 隐式 any
function processItemConfig(data) {
  return data.typeId;
}

// ✅ 正确: 明确类型
function processItemConfig(data: CustomItemConfig): string {
  return data.typeId;
}
```

### 2. 索引访问（noUncheckedIndexedAccess）

```typescript
// 本配置已开启 noUncheckedIndexedAccess，数组/对象索引进出来的是 T | undefined

// ❌ 错误: 不考虑 undefined 的情况
const blockTypes: string[] = ['stone', 'dirt', 'wood'];
const firstBlock: string = blockTypes[0]; // 类型为 string | undefined

// ✅ 正确: 处理 undefined
const firstBlock = blockTypes[0];
if (firstBlock !== undefined) {
  useBlock(firstBlock);
}

// 或使用非空断言（仅在业务保证索引必然存在时）
const firstBlock = blockTypes[0]!;
```

### 3. 数组方法

```typescript
// ❌ 错误: filter 后可能为 undefined
const found = items.filter(item => item.typeId === id)[0];

// ✅ 正确: 使用 find
const found = items.find(item => item.typeId === id);

// 或处理可能为 undefined 的情况
const found = items.filter(item => item.typeId === id)[0];
if (found) {
  useItem(found);
}
```

### 4. 对象解构

```typescript
// ❌ 错误: 可能缺少属性
const { typeId, displayName } = config;

// ✅ 正确: 使用可选链或默认值
const typeId = config?.typeId ?? 'unknown';
const displayName = config?.displayName ?? 'Unnamed';

// 或在接口中明确定义可选属性
interface ConfigData {
  typeId: string;
  displayName?: string;
}
```

### 5. 判别联合窄化失效（Discriminated Union Narrowing）

> 这是 `strict` + `noUncheckedIndexedAccess` 项目中最常见的隐性 bug 来源：把判别联合对象的
> **判别字段**与**载荷字段**拆成独立变量后，TypeScript 无法再关联它们，导致窄化完全失效。

#### 错误根源 1：拆分判别字段到独立变量

```typescript
type BlockEvent = 
  | { kind: 'placed'; block: BlockData; position: Vector3 }
  | { kind: 'broken'; block: BlockData; brokenBy: string };

// ❌ 错误: kind 和 data 拆成两个独立变量，TS 丢失关联
function handleBlockEvent(event: BlockEvent) {
  const kind = event.kind;
  if (kind === 'placed') {
    event.position; // TS 可能无法正确窄化
  }
}

// ✅ 正确: 直接使用 event.kind 判别
function handleBlockEvent(event: BlockEvent) {
  if (event.kind === 'placed') {
    const { block, position } = event; // TS 已完成窄化
    console.log(position);
  }
}
```

### 6. 未使用声明（TS6133 / TS6192）

> `noUnusedLocals` / `noUnusedParameters` 已开启：未使用的导入、局部变量、函数、参数都会报错。
> 修复方式是**删除**，而不是加 `void` 或忽略。

```typescript
// ❌ 错误: TS6133 未使用
import { ItemStack } from '@minecraft/server';
function unusedHelper() { /* ... */ }

// ✅ 正确: 删除未使用的声明
```

---

## 类型安全最佳实践

### 1. 使用 readonly

```typescript
interface GameConfig {
  readonly maxPlayers: number;
  readonly worldName: string;
}

const config: GameConfig = {
  maxPlayers: 10,
  worldName: 'My World'
};

// config.maxPlayers = 20; // 错误! 只读属性
```

### 2. 使用 as const

```typescript
const blockIds = {
  STONE: 'minecraft:stone',
  DIRT: 'minecraft:dirt',
  OAK_LOG: 'minecraft:oak_log'
} as const;

// blockIds.STONE 的类型是 'minecraft:stone' 而不是 string
```

### 3. 使用 satisfies

```typescript
const gameSettings = {
  maxPlayers: 10,
  difficulty: 'normal',
  allowCheats: false
} satisfies Record<string, string | number | boolean>;

// 保持字面量类型的同时验证类型
```

### 4. 避免不安全的类型断言

```typescript
// ❌ 错误: 不安全的类型断言
const itemConfig = rawData as CustomItemConfig;

// ✅ 正确: 使用类型守卫
if (isCustomItem(rawData)) {
  const itemConfig = rawData; // TypeScript 自动推断
  // 安全使用 itemConfig
}

// 或使用 unknown 中间类型 + 手动验证
function validateItemConfig(data: unknown): CustomItemConfig {
  if (!isCustomItem(data)) {
    throw new Error('Invalid item config');
  }
  return data;
}
```

---

## 类型检查清单

### 开发前
- [ ] 定义所有需要的类型/接口
- [ ] 检查 types/ 目录是否已有相关类型
- [ ] 确认 tsconfig.json 启用 strict 模式

### 开发中
- [ ] 所有函数参数有类型
- [ ] 所有返回值有类型
- [ ] 没有使用 any 类型
- [ ] 使用 unknown 时需要类型守卫
- [ ] 状态/常量使用 const 对象而非 enum

### 提交前
- [ ] 运行 `tsc --noEmit` 通过
- [ ] 运行 `npm run typecheck` 通过
- [ ] 没有类型相关的 TODO 注释
- [ ] 新增类型已导出到 types/index.ts（如需要）

---

**最后更新**: 2026-10-14