# JetBrainsLxgwNerdMono · Code Wiki

> 面向开发者的项目代码文档。本文档系统性描述该仓库的**整体架构、模块职责、关键类与函数、依赖关系以及运行方式**。

## 项目一句话简介

**JetBrainsLxgwNerdMono** 是一个字体构建流水线：以 *JetBrains Mono NerdFont* 为英文/图标基底，把 *LXGW 霞鹜文楷（Mono GB Screen / 真楷 GB）* 的 CJK 字形合并进来，产出 **英文:中文 = 1:2 宽度比** 的等宽编程字体，并可进一步拆分为 Web 用的 WOFF2 子集。

## 文档目录

| 文档 | 内容 |
| --- | --- |
| [01 · 项目概览](./01-overview.md) | 定位、核心特性、技术栈、目录结构 |
| [02 · 整体架构](./02-architecture.md) | 分层架构、构建流水线、数据流、时序与状态图 |
| [03 · 模块职责](./03-modules.md) | 目录/模块划分与各自职责边界 |
| [04 · 关键类与函数 API](./04-api-reference.md) | 核心数据结构与函数签名、参数、行为说明 |
| [05 · 依赖关系](./05-dependencies.md) | Python 依赖、外部工具依赖、模块间依赖图 |
| [06 · 配置说明](./06-configuration.md) | `config.yaml` 全字段解析与优先级规则 |
| [07 · 构建与运行](./07-build-and-run.md) | 本地/uv/Docker 构建、Web 拆分、验证页与产物结构 |

## 快速导航（按角色）

- **只想构建字体** → [07 · 构建与运行](./07-build-and-run.md)
- **想改合并逻辑** → [03 · 模块职责](./03-modules.md) + [04 · 关键类与函数 API](./04-api-reference.md#43-srcmergepy--核心合并)
- **想调参（宽度/字重/字体映射）** → [06 · 配置说明](./06-configuration.md)
- **想了解整体是怎么跑起来的** → [02 · 整体架构](./02-architecture.md)

## 源码地图（核心文件）

| 文件 | 作用 |
| --- | --- |
| `build.py` | 构建入口：读取配置、逐字重合并、产出 TTF 与 `fonts-manifest.json` |
| `split.py` | Web 字体入口：调用 `cn-font-split` 生成 WOFF2 子集与 CSS |
| `src/config.py` | `FontConfig` 数据类：宽度、缩放比、CJK 码位范围 |
| `src/merge.py` | 核心合并逻辑：CJK 拷贝、Nerd 图标缩放、CJK 居中 |
| `src/utils.py` | 名称表/OS2 元数据更新、码位判断、字宽校验 |
| `config.yaml` | 唯一的构建配置文件 |
| `verify-2-1.html` / `verify-2-1-split.html` | 2:1 等宽比可视化验证页 |
| `Dockerfile` | 容器化构建环境 |
