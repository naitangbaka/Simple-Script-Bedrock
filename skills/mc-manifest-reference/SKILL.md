---
name: "mc-manifest-reference"
description: "我的世界基岩版 manifest.json 清单文件参考文档。覆盖所有字段详解、完整示例、常见错误。Invoke when creating or modifying manifest.json files for behavior packs or resource packs."
---

# manifest.json 清单文件参考

> 官方文档来源：[packmanifest](https://learn.microsoft.com/en-us/minecraft/creator/reference/content/addonsreference/packmanifest)

---

## 一、概述

`manifest.json` 是《我的世界》基岩版附加包（Add-On）的核心清单文件，用于向游戏声明该包的基本信息、模块类型、依赖关系等。每个行为包（Behavior Pack）和资源包（Resource Pack）都必须包含此文件。

---

## 二、文件位置

```
<包根目录>/
├── manifest.json      ← 清单文件（必需）
├── pack_icon.png      ← 包图标（可选）
└── ...其他内容
```

---

## 三、基本结构

```json
{
  "format_version": 2,
  "header": { },
  "modules": [ ],
  "dependencies": [ ],
  "capabilities": [ ],
  "metadata": { },
  "subpacks": [ ]
}
```

---

## 四、字段详解

### 1. `format_version`（必需）

清单文件的格式版本。

| 值 | 说明 |
|---|---|
| `1` | 旧版格式，已不推荐 |
| `2` | **当前推荐使用**，支持更多字段 |

---

### 2. `header`（必需）

定义包的基本信息。

| 字段 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `name` | String | ✅ | 包名称，显示在游戏中 |
| `description` | String | ✅ | 包描述，显示在游戏中 |
| `uuid` | String | ✅ | 该包的**唯一标识符**（UUID v4） |
| `version` | Array[3] | ✅ | 包版本，如 `[1, 0, 0]` |
| `min_engine_version` | Array[3] | ❌ | 最低游戏引擎版本，如 `[1, 20, 0]` |
| `pack_scope` | String | ❌ | 包作用域：`world`、`global`、`any`（默认 `any`） |
| `lock_template_options` | Boolean | ❌ | 锁定模板选项，防止玩家修改 |

**示例：**
```json
"header": {
  "name": "我的自定义包",
  "description": "一个示例附加包",
  "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "version": [1, 0, 0],
  "min_engine_version": [1, 20, 0]
}
```

---

### 3. `modules`（必需）

定义包包含的模块，**数组**类型，可包含多个模块。

| 字段 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `type` | String | ✅ | 模块类型（见下表） |
| `uuid` | String | ✅ | 模块的唯一标识符（**不能与 header.uuid 相同**） |
| `version` | Array[3] | ✅ | 模块版本 |
| `description` | String | ❌ | 模块描述 |
| `entry` | String | ❌ | 入口文件路径（部分类型需要） |
| `language` | String | ❌ | 脚本语言，如 `javascript` |

**`type` 取值：**

| 值 | 说明 |
|---|---|
| `resources` | 资源包模块 |
| `data` | 行为包模块 |
| `script` | 脚本模块（GameTest / Script API） |
| `client_data` | 客户端数据 |
| `interface` | 界面模块 |
| `world_template` | 世界模板 |

**示例：**
```json
"modules": [
  {
    "type": "data",
    "uuid": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
    "version": [1, 0, 0]
  }
]
```

> ⚠️ 注意：一个包中每个模块的 `uuid` 必须唯一，且不能与 `header.uuid` 重复。

---

### 4. `dependencies`（可选）

声明该包所依赖的其他包。

| 字段 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `uuid` | String | ✅ | 依赖包的 UUID |
| `version` | Array[3] 或 String | ✅ | 依赖包的版本；也可用 `"*"` 表示任意版本 |
| `module_name` | String | ❌ | 仅脚本模块依赖时使用（如 `@minecraft/server`） |

#### 版本声明策略

| 场景 | `version` 值 | 说明 |
|------|-------------|------|
| **默认（未指定）** | 具体版本号，如 `"1.16.0"` | 最新稳定正式版 |
| **用户说"用 beta"** | `"beta"` | 字符串 `"beta"`，游戏自动跟随版本升级 |
| **用户说"用 alpha"** | `"3.0.0-alpha"` | 需搭配 `metadata.product_type: "addon"` |

**示例：**

```json
// 默认：最新稳定版
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

// 用户说"用 beta"：写 "beta" 字符串即可，自动跟随游戏版本
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

// 用户说"用 alpha"：搭配 metadata.product_type
"dependencies": [
  {
    "module_name": "@minecraft/server",
    "version": "3.0.0-alpha"
  },
  {
    "module_name": "@minecraft/server-ui",
    "version": "3.0.0-alpha"
  }
]
```

---

### 5. `capabilities`（可选）

声明包需要的特殊能力。

| 值 | 说明 |
|---|---|
| `script_eval` | 允许使用脚本评估（已弃用） |
| `chemistry` | 允许使用化学相关功能 |
| `raytraced` | 允许使用光线追踪相关功能 |

**示例：**
```json
"capabilities": ["script_eval"]
```

---

### 6. `metadata`（可选）

元数据信息。

| 字段 | 类型 | 说明 |
|---|---|---|
| `authors` | Array[String] | 作者列表 |
| `license` | String | 许可证 |
| `url` | String | 项目主页 |
| `generated_with` | Object | 生成工具信息 |
| `product_type` | String | 产品类型。设为 `"addon"` 时**不会禁用游戏成就系统** |

> `product_type: "addon"` 常见搭配：与 `@minecraft/server` 的 `"3.0.0-alpha"` 版本组合使用，确保附加包模式下成就不会被禁用。

**示例：**
```json
"metadata": {
  "authors": ["作者名"],
  "license": "MIT",
  "url": "https://example.com"
}

// 搭配 alpha 版本的完整示例
"metadata": {
  "authors": ["作者名"],
  "product_type": "addon"
}
```

---

### 7. `subpacks`（可选）

子包（Subpack），用于根据设备性能或条件切换不同内容。

| 字段 | 类型 | 说明 |
|---|---|---|
| `folder_name` | String | 子包文件夹名 |
| `name` | String | 子包显示名 |
| `memory_tier` | Integer | 内存层级（用于自动切换） |

**示例：**
```json
"subpacks": [
  {
    "folder_name": "low_res",
    "name": "低配版",
    "memory_tier": 0
  },
  {
    "folder_name": "high_res",
    "name": "高配版",
    "memory_tier": 2
  }
]
```

---

## 五、完整示例

### 行为包（含脚本模块）— 默认稳定版

```json
{
  "format_version": 2,
  "header": {
    "name": "示例行为包",
    "description": "这是一个完整的行为包 manifest 示例",
    "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "version": [1, 0, 0],
    "min_engine_version": [1, 20, 0]
  },
  "modules": [
    {
      "type": "data",
      "uuid": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
      "version": [1, 0, 0]
    },
    {
      "type": "script",
      "language": "javascript",
      "uuid": "c3d4e5f6-a7b8-9012-cdef-345678901234",
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
  "capabilities": ["script_eval"],
  "metadata": {
    "authors": ["你的名字"],
    "license": "MIT"
  }
}
```

### 行为包（beta 版本）— 自动跟随更新

```json
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
```

### 行为包（alpha 版本）— 不锁成就

```json
{
  "format_version": 2,
  "header": {
    "name": "示例行为包",
    "description": "使用 alpha 版本，不会禁用成就",
    "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "version": [1, 0, 0],
    "min_engine_version": [1, 20, 0]
  },
  "modules": [
    {
      "type": "data",
      "uuid": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
      "version": [1, 0, 0]
    },
    {
      "type": "script",
      "language": "javascript",
      "uuid": "c3d4e5f6-a7b8-9012-cdef-345678901234",
      "entry": "scripts/main.js",
      "version": [1, 0, 0]
    }
  ],
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
  "capabilities": ["script_eval"],
  "metadata": {
    "authors": ["你的名字"],
    "product_type": "addon"
  }
}
```

### 资源包

```json
{
  "format_version": 2,
  "header": {
    "name": "示例资源包",
    "description": "这是一个完整的资源包 manifest 示例",
    "uuid": "d4e5f6a7-b8c9-0123-defa-456789012345",
    "version": [1, 0, 0],
    "min_engine_version": [1, 20, 0]
  },
  "modules": [
    {
      "type": "resources",
      "uuid": "e5f6a7b8-c9d0-1234-efab-567890123456",
      "version": [1, 0, 0]
    }
  ],
  "metadata": {
    "authors": ["你的名字"]
  }
}
```

---

## 六、生成 UUID

> UUID 必须为标准的 **UUID v4** 格式（`8-4-4-4-12`，如 `a1b2c3d4-e5f6-7890-abcd-ef1234567890`）。

### 内置生成程序

以下代码可直接运行生成 UUID v4：

**JavaScript/Node.js（推荐）：**

```javascript
// 用法: node gen-uuid.js
function generateUUIDv4() {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
    const r = Math.random() * 16 | 0;
    const v = c === 'x' ? r : (r & 0x3 | 0x8);
    return v.toString(16);
  });
}

// 一次生成 5 个（header + data模块 + script模块 + 额外备用）
for (let i = 0; i < 5; i++) {
  console.log(generateUUIDv4());
}
```

**PowerShell（一行命令）：**

```powershell
1..5 | ForEach-Object { [guid]::NewGuid().ToString() }
```

### 外部工具

- **在线工具**：[uuidgenerator.net](https://www.uuidgenerator.net/)
- **VS Code 插件**：`UUID Generator`
- **Linux/macOS**：`uuidgen`

---

## 七、常见错误与注意事项

| 问题 | 解决办法 |
|---|---|
| 包不被游戏识别 | 检查 `manifest.json` 是否为合法 JSON（无多余逗号、无注释） |
| UUID 冲突 | 确保 `header.uuid` 与所有 `modules[].uuid` 各不相同，所有 UUID 必须为 v4 格式 |
| 版本格式错误 | `version` 必须是 `[主, 次, 修订]` 三个整数的数组 |
| 依赖包未加载 | 检查 `dependencies` 中的 UUID 和版本是否匹配 |
| 脚本模块无效 | 需在 `modules` 中声明 `script` 类型，`entry` 指向编译后的 JS 文件（如 `scripts/main.js`） |
| 模块版本不对 | 默认用最新稳定版；beta 写字符串 `"beta"`；alpha 写 `"3.0.0-alpha"` 并搭配 `product_type: "addon"` |
| 成就被禁用 | 使用 `3.0.0-alpha` 版本并设置 `"metadata": { "product_type": "addon" }` 可避免 |

---

## 八、相关官方文档链接

| 文档 | 链接 |
|---|---|
| manifest.json 参考 | [packmanifest](https://learn.microsoft.com/en-us/minecraft/creator/reference/content/addonsreference/packmanifest) |
| 附加包总览 | [creator docs](https://learn.microsoft.com/en-us/minecraft/creator/) |
| 脚本 API | [scriptapi](https://learn.microsoft.com/en-us/minecraft/creator/scriptapi/) |
| 行为包文档 | [behaviorpack](https://learn.microsoft.com/en-us/minecraft/creator/documents/behaviorpack) |

---

**最后更新**: 2026-10-14