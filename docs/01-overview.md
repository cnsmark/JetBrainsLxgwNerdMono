# 01 · 项目概览

## 1.1 项目定位

JetBrainsLxgwNerdMono 不是一款「设计」出来的字体，而是一条**字体合成（font merging）构建流水线**。它解决的核心问题是：

> 编程场景下，纯英文字体（JetBrains Mono）缺少中文；纯中文字体（霞鹜文楷）与英文等宽宽度对不齐，且缺少 Nerd Font 图标。

本项目通过 `fontTools` 在字形（glyph）级别做「取长补短」式的合并：

| 内容类别 | 来源字体 | 处理方式 |
| --- | --- | --- |
| 英文字符 | JetBrains Mono NerdFont | 作为**基底字体**原样保留 |
| Nerd Font 图标 | JetBrains Mono NerdFont | 保留，并缩放/居中到 CJK 宽度 |
| CJK 汉字与标点 | LXGW WenKai Mono GB Screen / LXGW ZhenKai GB | 拷贝字形、统一 UPM 缩放、设为 2 倍宽 |

## 1.2 核心特性

- **2:1 等宽比**：英文宽 `600` FUnit，CJK 宽 `1200` FUnit，保证终端/编辑器里「一个汉字 = 两个英文字符宽」，`|` 竖线完美对齐。
- **Nerd Font 图标兼容**：私有使用区（PUA）图标被缩放至 CJK 宽度；其中 **Powerline 符号（U+E0A0–U+E0DF）保持原始纵向边界**，保证终端提示符箭头不错位。
- **多字重支持**：`Regular / Medium / Italic / MediumItalic / Bold / BoldItalic` 六种样式。
- **多字重中文字体映射**：不同字重可映射到不同的中文字体文件（如 Bold 用真楷以加重笔画）。
- **配置驱动 + CLI 覆盖**：`config.yaml` 集中配置，命令行参数优先级最高。
- **Web 字体产物**：基于 `cn-font-split` 将 TTF 拆为按需加载的 WOFF2 子集，并生成合并 CSS。
- **可视化验证**：内置 HTML 验证页，读取 `fonts-manifest.json` 动态注入 `@font-face` 并对比 2:1 对齐效果。

## 1.3 技术栈

| 层面 | 选型 |
| --- | --- |
| 语言 | Python `>=3.10` |
| 字形处理 | [`fonttools`](https://github.com/fonttools/fonttools)（读写 TTF、操作 `glyf`/`cmap`/`hmtx`/`name`/`OS/2` 表） |
| 配置解析 | `PyYAML` |
| 依赖/环境管理 | [`uv`](https://github.com/astral-sh/uv)（`uv.lock` 锁定） |
| 打包 | `hatchling`（`pyproject.toml`） |
| 并发 | 标准库 `concurrent.futures.ProcessPoolExecutor`（多进程并行构建字重） |
| Web 拆分 | Node 工具 `cn-font-split`（外部 CLI） |
| 容器化 | `Dockerfile`（`python:3.11-slim` + uv） |
| 前端验证 | 原生 HTML/CSS/JS（无框架） |

## 1.4 目录结构

```text
.
├── build.py                  # 主构建脚本（入口）
├── split.py                  # Web 字体拆分脚本（入口）
├── config.yaml               # 构建配置（字体元数据/字重映射/宽度/并发）
├── pyproject.toml            # Python 项目与依赖定义
├── uv.lock                   # uv 依赖锁定文件
├── Dockerfile                # 容器化构建镜像
├── README.md / README_zh.md  # 用户文档（英/中）
├── verify-2-1.html           # TTF 版 2:1 验证页
├── verify-2-1-split.html     # WOFF2 拆分版验证页
├── src/                      # 核心 Python 包
│   ├── __init__.py
│   ├── config.py             # FontConfig 数据类
│   ├── merge.py              # 字体合并核心逻辑
│   └── utils.py              # 元数据/校验/工具函数
├── fonts/                    # 源字体（构建输入，需自行下载放置）
├── resources/                # 文档配图（如 2-1.png）
└── output/                   # 构建产物（git 忽略）
    ├── fonts/                # 生成的 TTF + fonts-manifest.json
    └── split/                # 生成的 WOFF2 子集 + CSS
```

> `fonts/` 与 `output/` 属于「输入/输出」目录：源字体体积大，仓库中仅保留 `fonts/` 内的少量文件；`output/` 被 `.gitignore` 排除且由构建时创建。

## 1.5 适用场景

- 终端（Windows Terminal / iTerm2 / Alacritty）中英文混排对齐；
- 编辑器/IDE（VS Code / JetBrains / Sublime）等宽字体；
- 个人博客、文档站通过 Web Font 按需加载中文子集。

## 1.6 相关文档

- 想理解「一次构建到底发生了什么」→ [02 · 整体架构](./02-architecture.md)
- 想知道每个函数做什么 → [04 · 关键类与函数 API](./04-api-reference.md)
- 想直接跑起来 → [07 · 构建与运行](./07-build-and-run.md)
