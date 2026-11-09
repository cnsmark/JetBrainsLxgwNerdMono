# 04 · 关键类与函数 API

本页按源码文件给出「类 / 函数」的签名、参数、返回值与行为说明。所有签名与仓库实现保持一致。

### 一次完整构建的函数调用链（build_single_font）

```mermaid
sequenceDiagram
    participant Main as build.py main()
    participant B as build.py<br/>build_single_font()
    participant M as merge.py
    participant U as utils.py
    participant FS as 文件系统

    Main->>B: 对每个 style 调用
    B->>M: merge_fonts(en_font_path, cn_font_path, config)
    Note over M: 内部调用<br/>get_cjk_glyphs()<br/>get_cjk_cmap_entries()<br/>(循环字形) copy.deepcopy()<br/>coordinates.scale()
    M->>U: merge_os2_ranges(base, cn)
    M-->>B: TTFont（合并后）
    B->>M: scale_nerd_icons(font, config)
    Note over M: PUA 图标 1.4× 缩放<br/>Powerline 特判
    B->>M: center_cjk_glyphs(font, config)
    Note over M: 宽字形居中<br/>成对标点分侧
    B->>U: update_font_names(font, ...)
    Note over U: 内部 set_font_name() × N<br/>写 Windows + zh-CN 名称
    B->>U: verify_glyph_width(font, [0,600,1200])
    B->>FS: merged_font.save(<Name>-<Style>.ttf)
    FS-->>B: OK
    B-->>Main: str(输出路径)

    style Main fill:#1e3a5f,color:#fff
    style B fill:#f59e0b,color:#111
    style M fill:#22c55e,color:#111
    style U fill:#8b5cf6,color:#fff
```

---

## 4.1 `src/config.py` · 配置类

### `FontConfig`（`@dataclass`）

构建过程的强类型配置载体，带默认值，可在构造时按需覆盖。

```python
@dataclass
class FontConfig:
    family_name: str = "JetBrainsLxgwNerdMono"
    family_name_compact: str = "JetBrainsLxgwNerdMono"
    version: str = "1.0"
    visual_scale: float = 1.08
    en_width: int = 600
    cn_width: int = 1200
    cjk_ranges: Tuple[Tuple[int, int], ...] = (...)
```

| 字段 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `family_name` | `str` | `"JetBrainsLxgwNerdMono"` | 字体族名，用于 name 表与拆分 CSS 的 `fontFamily` |
| `family_name_compact` | `str` | 同 `family_name` | 「紧凑族名」，用于拼接 PostScript 名与输出文件名 |
| `version` | `str` | `"1.0"` | 版本字符串，写入 NameID 5 |
| `visual_scale` | `float` | `1.08` | CJK 视觉放大系数，叠加在 UPM 归一化之上 |
| `en_width` | `int` | `600` | 英文/图标的 advance width（FUnit） |
| `cn_width` | `int` | `1200` | CJK 的 advance width（FUnit），须为 `en_width` 的 2 倍以实现 2:1 |
| `cjk_ranges` | `Tuple[Tuple[int,int],...]` | 见 [06](./06-configuration.md#67-cjk-码位范围cjk_ranges) | 参与合并的 CJK Unicode 区间 |

> 在 `build.py` 中，`family_name_compact` 被赋值为 `family_name`，因此输出文件名为 `JetBrainsLxgwNerdMono-<Style>.ttf`。

---

## 4.2 `build.py` · 构建编排

### `load_config(config_path: Path) -> Dict[str, Any]`

读取 YAML 配置。

- **参数**：`config_path` — `config.yaml` 路径。
- **返回**：配置字典；文件不存在时返回 `{}`（下游走默认值）。
- **行为**：`yaml.safe_load`，空文件返回 `{}`。

### `get_config_value(yaml_config: Dict[str, Any], *keys: str, default: Any = None) -> Any`

安全读取嵌套配置。

- **参数**：`yaml_config` — 配置字典；`*keys` — 逐层键名；`default` — 缺失时的回退值。
- **返回**：叶子值或 `default`。
- **行为**：逐层 `.get()`；任一层非 dict 或值为 `None` 时立即返回 `default`。

```python
get_config_value(cfg, "build", "parallel", default=1)
```

### `build_single_font(style, en_font_path, cn_font_path, display_name, output_dir, config, metadata) -> str`

**单字重构建单元**，是本项目的核心流水线函数。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `style` | `str` | 样式键（`Regular`/`Medium`/...），用于日志与 PostScript 名 |
| `en_font_path` | `Path` | 英文字体路径（基底） |
| `cn_font_path` | `Path` | 中文字体路径（CJK 来源） |
| `display_name` | `str` | name 表中的展示名（如 `"Medium Italic"`） |
| `output_dir` | `Path` | 输出目录 |
| `config` | `FontConfig` | 运行期配置 |
| `metadata` | `dict` | `author/copyright/description/url/license/license_url` |

**执行步骤**

1. `merge_fonts(str(en), str(cn), config)` → 合并 TTFont；
2. `scale_nerd_icons(font, config)` → 缩放 Nerd 图标；
3. `center_cjk_glyphs(font, config)` → CJK 居中；
4. `update_font_names(...)` → 写 name 表（`postscript_name = f"{family_name_compact}-{style}"`）；
5. `verify_glyph_width(font, [0, en_width, cn_width], postscript_name)` — 捕获 `ValueError` **仅打印告警**；
6. `merged_font.save(output_dir / f"{postscript_name}.ttf")`，随后 `close()`。

**返回**：输出文件路径字符串。

### `main() -> None`

CLI 主入口。

**命令行参数**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--config` | `<repo>/config.yaml` | 配置文件路径 |
| `--styles` | 配置中的 `build.styles` | 逗号分隔样式列表 |
| `--fonts-dir` | `config.fonts_dir` 或 `fonts/` | 源字体目录 |
| `--output-dir` | `config.build.output_dir` 或 `output/fonts` | 输出目录 |
| `--parallel` | `config.build.parallel` 或 `1` | 并行进程数 |

**流程**：解析参数 → 加载 YAML → 按 `CLI > YAML > 默认` 求值 → 构造 `FontConfig` → 校验样式/文件 → 创建输出目录 → 顺序或并行构建 → 生成 `fonts-manifest.json`。

**退出码**：配置缺失、样式非法、源字体缺失时 `sys.exit(1)`。

---

## 4.3 `src/merge.py` · 核心合并

### `get_cjk_glyphs(font: TTFont, config: FontConfig) -> Set[str]`

返回字体 `cmap` 中所有落在 `config.cjk_ranges` 内的**字形名集合**。基于 `font["cmap"].getBestCmap()`。

### `get_cjk_cmap_entries(font: TTFont, config: FontConfig) -> dict`

返回 `{码位: 字形名}`，仅包含 CJK 码位。用于后续向多个 cmap 子表回填。

### `merge_fonts(base_font_path: str, cn_font_path: str, config: FontConfig) -> TTFont`

**核心合并函数**：把中文源字体的 CJK 字形注入英文字体，返回合并后的 `TTFont`（即 `base_font`）。

| 参数 | 说明 |
| --- | --- |
| `base_font_path` | 英文字体（JetBrains Mono NerdFont）路径 |
| `cn_font_path` | 中文字体（LXGW）路径 |
| `config` | 配置对象 |

**算法要点**

1. 读取两种字体，记录 `base_font.getGlyphOrder()` 作为「已存在字形」集合。
2. 计算缩放：
   ```text
   upm_scale      = base_upm / cn_upm          # 例如 1000 / 2048
   combined_scale = upm_scale * visual_scale    # 叠加视觉放大
   ```
3. 遍历 CJK 字形：
   - 若基底已存在同名 glyph → 跳过（**不覆盖英文/图标**）；
   - 深拷贝 `cn_glyf[glyph_name]`（注意不是 `.glyphs[]`）；
   - 有轮廓则 `coordinates.scale(combined_scale)` 并 `recalcBounds`；
   - `hmtx[glyph] = (config.cn_width, int(orig_lsb * combined_scale))` —— **宽度固定为 1200**，保留 LSB 比例。
4. 若有新增字形：
   - 追加 glyph order，更新 `maxp.numGlyphs`；
   - 遍历所有 Unicode cmap 子表回填映射（platform 3 enc 1/10 与 platform 0；`format=4` 跳过非 BMP 码位）；
   - 更新 `hhea.advanceWidthMax` 与 `numberOfHMetrics`；
   - `merge_os2_ranges(base_font, cn_font)`。
5. 关闭 `cn_font`，返回 `base_font`。无新增字形时直接返回。

**副作用**：就地修改 `base_font` 的 `glyf`/`hmtx`/`cmap`/`hhea`/`OS/2`/`maxp`；关闭源中文字体。

### `scale_nerd_icons(font: TTFont, config: FontConfig) -> None`

将 PUA 图标（`U+E000–F8FF`、`U+F0000–FFFFD`）缩放/居中到 CJK 宽度。

| 子类 | 处理 |
| --- | --- |
| 普通图标 | 固定 `scale_factor = 1.4` 放大 → 水平居中（`ideal_lsb=(1200-w)/2`）→ 垂直中心对齐到 `y=360` |
| Powerline（`U+E0A0–E0DF`） | **不缩放、不垂直位移**，仅水平居中并设宽度 1200，保持纵向边界以对齐终端文本 |

- 仅处理当前 `advance width == config.en_width`（600）的字形。
- 更新 `hmtx[glyph] = (cn_width, ideal_lsb)`。
- 打印统计：Powerline 数量与普通缩放数量。

### `center_cjk_glyphs(font: TTFont, config: FontConfig) -> None`

在 1200 宽度内对 CJK 字形做水平定位修正。

| 情形 | 处理 |
| --- | --- |
| 左标点（`【《〈「『〔〖〘〚（［｛' "` 等） | 右对齐：`ideal_lsb = cn_width - glyph_width` |
| 右标点（`】》〉」』〕〗〙〛）］｝' "` 等） | 左对齐：`ideal_lsb = 0` |
| 宽字形（`glyph_width > cn_width/2`） | 居中：`ideal_lsb = (cn_width - glyph_width)//2` |
| 窄字形（`≤ cn_width/2`，如部分标点） | **跳过**，保留原位置 |

- 仅处理 `advance width == cn_width` 的字形；位移量 `>1` 才应用。
- 打印统计：`Centered / Paired punctuation / Skipped(narrow)`。

---

## 4.4 `src/utils.py` · 基础能力

### `set_font_name(font, name, name_id, mac=True, lang_id=0x409) -> None`

写入单条 `name` 记录。

- 当 `lang_id == 0x409`（英文）时，先 `removeNames(nameID=name_id)`，避免残留；
- 写入 Windows 平台 `(platformID=3, platEncID=1)`；
- 当 `mac and lang_id == 0x409` 时，额外写入 Mac 平台 `(platformID=1, platEncID=0, langID=0)`。

### `update_font_names(font, family_name, style_name, full_name, postscript_name, version_str, author="", copyright_str="", description="", url="", license_desc="", license_url="") -> None`

批量写回字体元数据。

| NameID | 内容 | 条件 |
| --- | --- | --- |
| 0 | Copyright | 非空时 |
| 1 / 2 | Family / Subfamily | 总是 |
| 3 | Unique ID：`"{version};JBLXGW;{postscript}"` | 总是 |
| 4 / 5 / 6 | Full / Version / PostScript | 总是 |
| 7 | Trademark | 置空 |
| 8 / 9 | Manufacturer / Designer | `author` 非空 |
| 10 | Description | 非空 |
| 11 / 12 | Vendor URL / Designer URL | `url` 非空 |
| 13 / 14 | License / License URL | 非空 |
| 16 / 17 | Preferred Family / Subfamily | 总是（Windows 兼容） |

另外，对 NameID 1/2/4/16/17 追加 **zh-CN（`lang_id=0x804`）** 名称，且不写 Mac 条目，提升 Windows/Excel 中文识别。

### `merge_os2_ranges(target_font: TTFont, source_font: TTFont) -> None`

按位或合并 OS/2 表的 Unicode Range 与 Code Page Range：

```text
ulUnicodeRange1..4  |= source 对应位
ulCodePageRange1..2 |= source 对应位
```

任一方缺少 `OS/2` 表时直接返回。这是 Windows 应用判定「支持中文」的关键。

### `is_cjk_codepoint(codepoint: int, cjk_ranges: Tuple[Tuple[int,int], ...]) -> bool`

线性扫描区间，判断码位是否属于 CJK。

### `verify_glyph_width(font, expected_widths: List[int], file_name: str | None = None) -> None`

遍历 `font.getGlyphOrder()` 的 `hmtx` advance width，若存在既不在 `expected_widths` 又非 `0` 的字形，则抛 `ValueError`（消息含前 10 个异常字形示例）；全部通过则打印 `Verified glyph widths in <name>`。

---

## 4.5 `split.py` · Web 字体拆分

### 常量

| 常量 | 值 | 用途 |
| --- | --- | --- |
| `STYLE_DISPLAY_NAMES` | `{Regular: Regular, Medium: Medium, Italic: Italic, MediumItalic: "Medium Italic", Bold: Bold, BoldItalic: "Bold Italic"}` | 拼接 full name 做 `local()` 匹配 |
| `STYLE_KEYS` | `["MediumItalic","BoldItalic","Medium","Italic","Bold","Regular"]` | **长名优先**的样式识别顺序 |

### `check_cn_font_split_installed() -> bool`

`shutil.which("cn-font-split") is not None`。

### `enhance_local_font_matching(css_path: Path, family_name: str, style_key: str) -> None`

把生成 CSS 中的 `local("Family"),url(` 替换为：

```text
local("<Family>-<Style>"),local("<Family> <Display>"),local("<Family>"),url(
```

即依次尝试 PostScript 名、Full 名、Family 名，提升浏览器/系统本地已安装字体的命中率。文件不存在时告警返回；无变化则不写回。

### `split_font(font_path: Path, output_base_dir: Path, config: FontConfig, test_html: bool = True) -> Path`

单字体拆分。

- 未安装 `cn-font-split` → `RuntimeError`；
- 输出目录：`output_base_dir / <font_stem>`；
- 由文件名匹配 `STYLE_KEYS` 推导 `font_weight`（Bold→700、Medium→500，否则 400）与 `font_style`（含 Italic→italic）；
- 调用命令：
  ```text
  cn-font-split run -i <ttf> -o <dir>
    --css.fontFamily <family_name>
    --css.fontWeight <w> --css.fontStyle <s>
    --css.fontDisplay swap
    --renameOutputFont [index].[ext]
    --testHtml true|false
  ```
- 成功后调用 `enhance_local_font_matching(<dir>/result.css, ...)`；
- 返回该字重的输出目录。

### `generate_merged_css(split_dirs: List[Path], output_file: Path) -> None`

为每个拆分目录生成 `@import "./<font_name>/result.css";`，写入 `all.css`。

### `split_all_fonts(input_dir: Path, output_dir: Path, config: FontConfig) -> None`

`glob("*.ttf")` → 逐个 `split_font` → 收集成功目录 → 若有则生成 `all.css`。**单个失败不影响其它字体**。

### `main() -> None`

CLI：`--input-dir`（默认 `output/fonts`）、`--output-dir`（默认 `output/split`）；配置 `logging` 后执行拆分。

---

## 4.6 前端验证页 API

`verify-2-1.html` 内部函数（纯浏览器 JS）：

| 函数 | 作用 |
| --- | --- |
| `injectFontFaces(fonts, familyName)` | 生成并注入 `@font-face`，family 名为 `<Family>-<Style>` |
| `populateSelect(fonts)` | 填充字重下拉，默认选中第一项并应用 |
| `applyFont(style)` | 设置展示文本的 `font-family` |
| `init()` | `fetch` manifest → 注入字体 → 填下拉 → 改标题；失败进 `fallbackMode()` |
| `fallbackMode()` | 用内置 4 个字重（Regular/Medium/Italic/MediumItalic）兜底 |

**依赖的 manifest 契约**：

```json
{
  "family_name": "JetBrainsLxgwNerdMono",
  "version": "1.3",
  "fonts": [
    { "style": "Regular", "display_name": "Regular", "filename": "JetBrainsLxgwNerdMono-Regular.ttf" }
  ]
}
```

---

## 4.7 相关文档

- 各函数所在的模块背景 → [03 · 模块职责](./03-modules.md)
- 配置字段与 CLI → [06 · 配置说明](./06-configuration.md) / [07 · 构建与运行](./07-build-and-run.md)
