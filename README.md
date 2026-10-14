# 基岩简单脚本（Simple Script Bedrock）

> 一套面向《我的世界》基岩版脚本开发的 AI 辅助技能库，涵盖从项目规划、架构设计、类型安全到 UI 交互的完整开发规范与参考文档。

---

## 项目简介

**基岩简单脚本** 是一个专为《我的世界》基岩版（Minecraft Bedrock Edition）Addon 脚本开发设计的技能库（Skill Library）。它包含 8 个精心编写的 SKILL 文档，覆盖了脚本开发的各个关键环节——从开发前的需求问答，到编码时的类型安全、OOP 设计、架构规范，再到 UI 交互与错误检查，以及官方 API 和清单文件的参考速查。

这些技能文档旨在辅助 AI 编码助手在开发基岩版 Addon 时能够遵循最佳实践，写出类型安全、架构清晰、性能可靠的高质量游戏脚本。

---

## 技能列表

| 编号 | 技能名称 | 文件路径 | 说明 |
|------|----------|----------|------|
| 01 | **predev-qa** | [SKILL.md](skills/predev-qa/SKILL.md) | 开发前问答与规划。在着手编码前，通过结构化问答澄清项目定位、目标用户、功能范围与约束条件 |
| 02 | **type-safety** | [SKILL.md](skills/type-safety/SKILL.md) | TypeScript 类型安全规范。强制严格模式、禁止 `any`、类型定义优先 |
| 03 | **oop-design** | [SKILL.md](skills/oop-design/SKILL.md) | 面向对象设计规范。禁止面向过程/面向结果编程，可扩展性优先 |
| 04 | **fullstack-dev** | [SKILL.md](skills/fullstack-dev/SKILL.md) | 脚本开发架构规范。分层架构、事件系统、数据持久化、Tick 调度器 |
| 05 | **frontend-dev** | [SKILL.md](skills/frontend-dev/SKILL.md) | 基岩版 UI 与交互规范。表单设计、消息系统、颜色代码、用户输入处理 |
| 06 | **error-check** | [SKILL.md](skills/error-check/SKILL.md) | 常用错误检查清单。Native 函数异步调度、事件取消、空值检查、UUID 唯一性等 |
| 07 | **mc-api-reference** | [SKILL.md](skills/mc-api-reference/SKILL.md) | @minecraft 官方模块参考。各模块用途说明、版本查询与安装规范 |
| 08 | **mc-manifest-reference** | [SKILL.md](skills/mc-manifest-reference/SKILL.md) | manifest.json 清单文件参考。所有字段详解、完整示例与常见错误 |

---

## 适用场景

- 使用 AI 编码助手开发《我的世界》基岩版 Addon 脚本
- 新手开发者学习基岩版脚本开发最佳实践
- 团队协作时统一编码规范与架构风格
- 项目初期进行需求梳理与架构规划

---

## 技术栈

- **语言**：TypeScript
- **运行环境**：Minecraft Bedrock Edition 游戏内置沙箱 JavaScript 引擎
- **官方模块**：`@minecraft/server`、`@minecraft/server-ui`、`@minecraft/server-net`、`@minecraft/server-admin`、`@minecraft/server-gametest`、`@minecraft/common`、`@minecraft/diagnostics`、`@minecraft/server-editor`

---

## 项目结构

```
基岩简单脚本/
├── skills/
│   ├── predev-qa/
│   │   └── SKILL.md              # 开发前问答与规划
│   ├── type-safety/
│   │   └── SKILL.md              # TypeScript 类型安全规范
│   ├── oop-design/
│   │   └── SKILL.md              # 面向对象设计规范
│   ├── fullstack-dev/
│   │   └── SKILL.md              # 脚本开发架构规范
│   ├── frontend-dev/
│   │   └── SKILL.md              # 基岩版 UI 与交互规范
│   ├── error-check/
│   │   └── SKILL.md              # 常用错误检查清单
│   ├── mc-api-reference/
│   │   └── SKILL.md              # @minecraft 官方模块参考
│   └── mc-manifest-reference/
│       └── SKILL.md              # manifest.json 清单文件参考
├── LICENSE                       # 公共领域许可证（The Unlicense）
└── README.md                     # 项目说明文档
```

---

## 许可证

本项目已加入公共领域（Public Domain），使用 [The Unlicense](LICENSE) 许可证。任何人都可以自由地使用、修改、分发本项目的内容，无需署名、无需授权。