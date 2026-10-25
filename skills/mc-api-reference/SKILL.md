---
name: "mc-api-reference"
description: "我的世界基岩版官方 @minecraft 模块参考文档。涵盖所有模块用途、版本查询与安装规范。Invoke when installing API modules, checking module compatibility, or needing module capability reference."
---

# @minecraft 官方模块参考

> 我的世界基岩版脚本开发仅能使用官方白名单模块。**禁止安装任何非 @minecraft 前缀的第三方 npm 包（如 lodash、axios、zod 等），在游戏沙箱中不会生效。**

---

## 模块速查表

| 模块 | 用途 | 环境限制 |
|------|------|----------|
| `@minecraft/server` | 服务端核心：实体/方块/维度/世界/事件 | 所有平台 |
| `@minecraft/server-ui` | 服务端 UI：表单对话框 | 所有平台 |
| `@minecraft/server-admin` | 服务端管理：服务器配置访问 | 仅 BDS |
| `@minecraft/server-net` | 网络模块：HTTP 请求 | 仅 BDS |
| `@minecraft/server-gametest` | 游戏测试：自动化测试框架 | 所有平台 |
| `@minecraft/common` | 通用资源：错误类定义等 | 所有平台 |
| `@minecraft/diagnostics` | 诊断功能：对接 Sentry 服务 | 所有平台 |
| `@minecraft/server-editor` | 编辑器扩展：编辑器工具/面板/操作 | 仅编辑器 |

---

## 各模块详解

### @minecraft/server

最基础的服务端核心模块，提供对实体、方块、维度、世界等游戏对象的访问和操控能力。

**核心 API**：
- `world` — 世界对象，管理玩家、消息、事件
- `system` — 系统对象，管理 tick 循环、定时任务
- `Player` — 玩家类
- `Block` — 方块类
- `ItemStack` — 物品堆叠
- `Dimension` — 维度（主世界/下界/末地）
- `Vector3` — 三维坐标

**常用导入**：
```typescript
import { world, system, Player, Block, ItemStack, Dimension, Vector3 } from '@minecraft/server';
```

---

### @minecraft/server-ui

服务端 UI 模块，用于向玩家显示表单对话框。

**表单类型**：

| 表单 | 类 | 用途 |
|------|-----|------|
| 操作表单 | `ActionFormData` | 多按钮选择菜单 |
| 模态表单 | `ModalFormData` | 含多种控件的输入表单 |
| 消息表单 | `MessageFormData` | 二选一确认对话框 |

**常用导入**：
```typescript
import { ActionFormData, ModalFormData, MessageFormData } from '@minecraft/server-ui';
```

---

### @minecraft/server-admin

服务端管理模块，提供服务器配置访问能力。**仅限 BDS（Bedrock Dedicated Server）专用服务器环境**，在本地世界或 Realm 中不可用。

**核心 API**：
- `ServerVariables` — 访问 `server.properties` 中的配置
- `ServerSecrets` — 访问安全存储中的密钥

**常用导入**：
```typescript
import { ServerVariables, ServerSecrets } from '@minecraft/server-admin';
```

---

### @minecraft/server-net

网络模块，提供 HTTP 请求能力。**仅限 BDS 专用服务器环境**。

**核心 API**：
- `http` — HTTP 客户端，支持 GET/POST 等请求

**常用导入**：
```typescript
import { http } from '@minecraft/server-net';

// 示例：发送 HTTP GET 请求
async function fetchRemoteData(): Promise<void> {
  const response = await http.get('https://api.example.com/data');
  console.log(`状态: ${response.status}, 内容: ${response.body}`);
}
```

---

### @minecraft/server-gametest

游戏测试模块，提供自动化测试框架。用于编写可在游戏中运行的测试用例。

**核心 API**：
- `register` — 注册测试
- `Test` — 测试实例
- `GameTestSequence` — 测试步骤序列

**常用导入**：
```typescript
import { register, GameTestSequence } from '@minecraft/server-gametest';
```

---

### @minecraft/common

提供通用的资源和接口，例如错误类定义。通常作为其他模块的基础依赖。

**常用导入**：
```typescript
import { ArgumentError, EngineError } from '@minecraft/common';
```

---

### @minecraft/diagnostics

提供诊断功能，目前主要用于对接 Sentry 服务进行错误追踪和性能监控。

**常用导入**：
```typescript
import { Sentry } from '@minecraft/diagnostics';
```

---

### @minecraft/server-editor

编辑器扩展模块，提供面向编辑器工具、面板和操作的接口。**仅限编辑器环境**。

**常用导入**：
```typescript
import { registerEditorExtension } from '@minecraft/server-editor';
```

---

## npm 安装规范

### 核心规则

1. **安装前必须先查询所有可用版本**
2. **用户未指定版本时，默认使用最新稳定版（正式版）模块**
3. **用户说"使用 beta"时，manifest.json 中只声明 `"version": "beta"` 即可**，游戏会自动匹配最新预览版
4. 每个模块独立安装，版本号在 manifest.json 的 `dependencies` 中声明

### 版本声明策略

| 用户意图 | manifest 中 version 值 | 说明 |
|----------|----------------------|------|
| 未指定（默认） | 具体版本号，如 `"1.16.0"` | 最新稳定版，不随游戏更新自动变化 |
| 说"用 beta" | `"beta"` | 字符串 `"beta"`，会自动跟随游戏版本升级 |
| 说"用 alpha" | `"3.0.0-alpha"` + 需搭配 `metadata` | 见下方 alpha 特别说明 |

#### alpha 特别说明

当使用 `version: "3.0.0-alpha"` 时，**必须同时声明**以下 metadata：

```json
"metadata": {
  "product_type": "addon"
}
```

> `3.0.0-alpha` + `product_type: "addon"` 的组合可以确保 **不会禁用游戏的成就系统**。

#### beta 的优势

```json
"dependencies": [
  {
    "module_name": "@minecraft/server",
    "version": "beta"
  }
]
```

只声明 `"beta"`（字符串，不是具体版本号），游戏运行时会自动使用当前引擎版本对应的最新 API 预览版。**无需手动更新版本号**，适合需要紧跟最新游戏特性的项目。

### 安装流程

#### 第 1 步：查询可用版本

```bash
npm view @minecraft/server versions --json
```

该命令输出该模块所有已发布的版本号，例如：

```json
[
  "1.0.0",
  "1.1.0",
  "1.2.0-beta.1.20.50.22-preview",
  "1.2.0",
  "1.3.0-beta.1.20.60.24-preview",
  "1.3.0",
  ...
]
```

#### 第 2 步：选择版本

- **用户指定了版本** → 使用用户指定的版本
- **用户说"使用 beta"** → manifest 中声明 `"version": "beta"`，npm 安装最新 beta 用于本地开发
- **用户未指定** → 使用最新稳定版（正式版），npm 安装最新稳定版，manifest 中写具体版本号

#### 第 3 步：执行安装

```bash
# 默认：最新稳定版
npm install @minecraft/server@latest
npm install @minecraft/server-ui@latest

# 用户说用 beta：安装最新 beta 用于代码提示，但 manifest 中写 "beta"
npm install @minecraft/server@latest-beta
```

#### 第 4 步：更新 manifest.json

在 `dependencies` 中声明模块及版本：

```json
// 默认：具体版本号
{
  "dependencies": [
    {
      "module_name": "@minecraft/server",
      "version": "1.16.0"
    },
    {
      "module_name": "@minecraft/server-ui",
      "version": "1.6.0"
    }
  ]
}

// 用户说"用 beta"：只写 "beta"
{
  "dependencies": [
    {
      "module_name": "@minecraft/server",
      "version": "beta"
    },
    {
      "module_name": "@minecraft/server-ui",
      "version": "beta"
    }
  ]
}

// 用户说"用 alpha"：搭配 product_type
{
  "dependencies": [
    {
      "module_name": "@minecraft/server",
      "version": "3.0.0-alpha"
    },
    {
      "module_name": "@minecraft/server-ui",
      "version": "3.0.0-alpha"
    }
  ],
  "metadata": {
    "product_type": "addon"
  }
}
```

### 批量安装脚本示例

```bash
# 查询所有模块版本
npm view @minecraft/server versions --json
npm view @minecraft/server-ui versions --json

# 默认：安装最新稳定版
npm install @minecraft/server@latest
npm install @minecraft/server-ui@latest

# 或：用户说用 beta → npm 装 beta，manifest 中写 "beta"
npm install @minecraft/server@latest-beta
npm install @minecraft/server-ui@latest-beta
```

---

## 版本对应关系

| 模块 | 默认版本策略 | manifest 版本值 |
|------|------------|---------------|
| `@minecraft/server` | 最新稳定版 | `"1.16.0"` 或 `"beta"` 或 `"3.0.0-alpha"` |
| `@minecraft/server-ui` | 最新稳定版 | `"1.6.0"` 或 `"beta"` 或 `"3.0.0-alpha"` |
| `@minecraft/server-gametest` | 最新稳定版 | 具体版本号或 `"beta"` |
| `@minecraft/server-admin` | 最新稳定版 | 具体版本号或 `"beta"`（仅 BDS） |
| `@minecraft/server-net` | 最新稳定版 | 具体版本号或 `"beta"`（仅 BDS） |
| `@minecraft/common` | 最新稳定版 | 具体版本号 |
| `@minecraft/diagnostics` | 最新稳定版 | 具体版本号 |
| `@minecraft/server-editor` | 最新稳定版 | 具体版本号或 `"beta"`（仅编辑器） |

> - **默认**：用户不指定时，写最新稳定版的具体版本号
> - **beta**：用户说用 beta 时，manifest 中写字符串 `"beta"`（自动跟随游戏版本升级）
> - **alpha**：用户说用 alpha 时，写 `"3.0.0-alpha"` 并搭配 `"metadata": { "product_type": "addon" }`，不会禁用成就

> **注意**：manifest.json 中的 `min_engine_version` 应与所用模块对应的游戏版本匹配。

---

## 常见问题

### Q: 为什么不能安装 lodash、axios 等 npm 包？

A: MC 基岩版的脚本运行在游戏内置的沙箱 JavaScript 引擎中，不是 Node.js 环境。只有 `@minecraft/` 开头的官方模块在白名单内，其他 npm 包无法加载。

### Q: 如何判断某个模块是否支持当前平台？

A: 参考上方"环境限制"列：
- 标记"所有平台" → 本地世界、Realm、BDS 均可使用
- 标记"仅 BDS" → 只在 BDS 专用服务器可用
- 标记"仅编辑器" → 只在编辑器模式可用

### Q: beta 版本和正式版本有什么区别？

A: beta 版本对应游戏的预览版/测试版，包含最新的 API。正式版对应游戏正式版。开发时建议使用最新 beta 以获取最新功能，发布时应切换到匹配的版本。

---

**最后更新**: 2026-10-14