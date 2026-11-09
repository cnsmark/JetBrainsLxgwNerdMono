# 04 · 关键类与函数 API

本页按源码文件给出「类 / 函数」的签名、参数、返回值与行为说明。所有签名与仓库实现保持一致。

---

## 4.0 · 完整调用链时序图（源码级）

本节的三张时序图**严格按 build.py → merge.py → utils.py 的真实函数调用关系**绘制，与仓库实现一致。建议按顺序阅读：

1. [图 A · main() 全局编排](#图-a--main-全局编排)：展示 `main()` 从 CLI 到 `build_single_font`、并行调度、manifest 生成的完整流程
2. [图 B · build_single_font 完整链路](#图-b--build_single_font-完整链路)：展示单个字重构建的 6 步流水线，明确每个函数的入参与返回值
3. [图 C · merge_fonts 内部深度调用链](#图-c--merge_fonts-内部深度调用链)：展开最复杂的 `merge_fonts`，包含它对 `get_cjk_glyphs`、`get_cjk_cmap_entries`、`is_cjk_codepoint`、`merge_os2_ranges` 的调用

### 图 A · `main()` 全局编排

```mermaid
sequenceDiagram
    participant CLI as CLI 参数<br/>argparse
    participant M as build.py main()
    participant LC as build.py<br/>load_config()
    participant GV as build.py<br/>get_config_value()×N
    participant FC as src/config.py<br/>FontConfig
    participant BP as build.py<br/>build_single_font()
    participant MPG as ProcessPoolExecutor
    participant J as json

    CLI->>M: --config --styles --fonts-dir --output-dir --parallel
    M->>LC: load_config(config_path)
    LC-->>M: yaml_config dict
    loop 多次读取嵌套键
        M->>GV: get_config_value(yaml_config, *keys, default)
        GV-->>M: 叶子值 或 default
    end

    rect rgb(30, 58, 95)
        Note over M,FC: 构造 FontConfig 数据类
        M->>FC: FontConfig(family_name, version,<br/>visual_scale, en_width=600,<br/>cn_width=1200)
        FC-->>M: config 对象
    end

    M->>M: 校验样式键 / 源字体文件存在
    M->>M: font_paths[style] = {en_font_path,<br/>cn_font_path, display_name}
    M->>M: output_dir.mkdir(parents=True, exist_ok=True)

    alt parallel <= 1 顺序构建
        loop 每个 style
            M->>BP: build_single_font(style,<br/>en_font_path, cn_font_path,<br/>display_name, output_dir,<br/>config, metadata)
            BP-->>M: str(output_path)
        end
    else parallel > 1 多进程并行
        M->>MPG: ProcessPoolExecutor(max_workers=parallel)
        loop 每个 style
            M->>MPG: executor.submit(build_single_font, ...)
        end
        loop as_completed
            MPG-->>M: future.result()
        end
    end

    rect rgb(34, 197, 94)
        Note over M,J: 生成 fonts-manifest.json
        M->>M: manifest = {family_name, version,<br/>fonts:[{style, display_name,<br/>filename}]}
        M->>J: json.dump(manifest, indent=2)
        J-->>M: 写入 output/fonts/fonts-manifest.json
    end

    style CLI fill:#f59e0b,color:#111
    style M fill:#3b82f6,color:#fff
    style BP fill:#22c55e,color:#111
    style MPG fill:#8b5cf6,color:#fff
    style J fill:#22c55e,color:#111
```

### 图 B · `build_single_font` 完整链路（单字重构建）

```mermaid
sequenceDiagram
    participant B as build.py<br/>build_single_font
    participant MF as merge.py<br/>merge_fonts
    participant SN as merge.py<br/>scale_nerd_icons
    participant CC as merge.py<br/>center_cjk_glyphs
    participant UN as utils.py<br/>update_font_names
    participant VG as utils.py<br/>verify_glyph_width
    participant FT as fontTools.TTFont

    B->>MF: merge_fonts(base_font_path=str(en),<br/>cn_font_path=str(cn), config)
    Note right of MF: (详见图 C)
    MF-->>B: merged_font: TTFont

    B->>SN: scale_nerd_icons(merged_font, config)
    Note right of SN: glyf/coordinates.scale(1.4)<br/>hmtx[glyph] = (1200, ideal_lsb)<br/>Powerline 只水平居中

    B->>CC: center_cjk_glyphs(merged_font, config)
    Note right of CC: (内部再调 get_cjk_glyphs)<br/>glyph.coordinates.translate((delta,0))<br/>左标点右对齐 / 右标点左对齐

    rect rgb(124, 58, 237)
        Note over B,UN: update_font_names 批量写 18+ 个 NameID
        B->>UN: update_font_names(font=merged_font,<br/>family_name, style_name,<br/>full_name, postscript_name,<br/>version_str="Version X",<br/>author, copyright_str,<br/>description, url,<br/>license_desc, license_url)
        loop NameID 0..14 + 16 + 17
            UN->>UN: set_font_name(font, value, name_id)
            UN->>FT: font["name"].removeNames(nameID=name_id)<br/>font["name"].setName(..., platformID=3, platEncID=1, langID=0x409)
            FT-->>UN: OK
            opt mac=True and lang_id==0x409
                UN->>FT: font["name"].setName(..., platformID=1, platEncID=0, langID=0)
            end
        end
        loop zh-CN(0x804): NameID 1/2/4/16/17
            UN->>UN: set_font_name(font, value, name_id,<br/>mac=False, lang_id=0x804)
        end
    end

    opt 异常捕获
        B->>VG: verify_glyph_width(font=merged_font,<br/>expected_widths=[0,600,1200],<br/>file_name=postscript_name)
        loop 遍历 font.getGlyphOrder()
            VG->>VG: hmtx[name] = (width, lsb)<br/>if width not in [0,600,1200]:
        end
        alt 全部合规
            VG-->>B: OK (print Verified)
        else 存在异常字宽
            VG-->>B: raise ValueError (捕获后仅 Warning)
        end
    end

    B->>FT: merged_font.save(str(output_path))
    FT-->>B: 写入 .ttf
    B->>FT: merged_font.close()
    B-->>B: return str(output_path)

    style B fill:#f59e0b,color:#111
    style MF fill:#22c55e,color:#111,stroke-width:2px
    style SN fill:#22c55e,color:#111
    style CC fill:#22c55e,color:#111
    style UN fill:#8b5cf6,color:#fff
    style VG fill:#8b5cf6,color:#fff
```

### 图 C · `merge_fonts` 内部深度调用链

```mermaid
sequenceDiagram
    participant MF as merge.py<br/>merge_fonts
    participant TTF as fontTools.TTFont
    participant GCG as merge.py<br/>get_cjk_glyphs
    participant GCE as merge.py<br/>get_cjk_cmap_entries
    participant IU as utils.py<br/>is_cjk_codepoint
    participant MOR as utils.py<br/>merge_os2_ranges
    participant CN as copy.deepcopy

    MF->>TTF: TTFont(base_font_path)
    TTF-->>MF: base_font
    MF->>TTF: TTFont(cn_font_path)
    TTF-->>MF: cn_font

    MF->>MF: base_glyph_names = set(base_font.getGlyphOrder())

    MF->>GCG: get_cjk_glyphs(cn_font, config)
    GCG->>TTF: cn_font["cmap"].getBestCmap()
    TTF-->>GCG: {codepoint: glyph_name}
    loop 遍历 cmap 每项
        GCG->>IU: is_cjk_codepoint(cp, config.cjk_ranges)
        IU->>IU: for start,end in ranges:<br/>if start<=cp<=end: return True
        IU-->>GCG: True/False
        opt True
            GCG->>GCG: cjk_glyphs.add(glyph_name)
        end
    end
    GCG-->>MF: cjk_glyphs: Set[str]

    MF->>GCE: get_cjk_cmap_entries(cn_font, config)
    GCE->>TTF: cn_font["cmap"].getBestCmap()
    TTF-->>GCE: {codepoint: glyph_name}
    loop 遍历 cmap 每项
        GCE->>IU: is_cjk_codepoint(cp, config.cjk_ranges)
        IU-->>GCE: True/False
        opt True
            GCE->>GCE: entries[cp] = glyph_name
        end
    end
    GCE-->>MF: cjk_cmap: dict

    MF->>TTF: base_upm = base_font["head"].unitsPerEm
    MF->>TTF: cn_upm = cn_font["head"].unitsPerEm
    MF->>MF: combined_scale = (base_upm/cn_upm) × visual_scale

    rect rgb(30, 58, 95)
        Note over MF,CN: 逐字形：拷贝 + 缩放 + 设宽
        loop 对每个 cjk_glyphs
            alt glyph_name ∉ base_glyph_names<br/>且 cn_glyf.glyphs 存在
                MF->>CN: copy.deepcopy(cn_glyf[glyph_name])
                CN-->>MF: glyph
                opt 有 contours 且 xMin 未计算
                    MF->>TTF: glyph.recalcBounds(base_glyf)
                end
                MF->>TTF: glyph.coordinates.scale((combined_scale, combined_scale))
                MF->>TTF: glyph.recalcBounds(base_glyf)
                MF->>MF: orig_width, orig_lsb = cn_hmtx[glyph_name]
                MF->>MF: scaled_lsb = int(orig_lsb * combined_scale)
                MF->>TTF: base_hmtx.metrics[glyph_name] = (1200, scaled_lsb)
                MF->>MF: glyphs_added.append(glyph_name)
            end
        end
    end

    opt glyphs_added 非空
        MF->>TTF: base_font.setGlyphOrder(<原顺序> + glyphs_added)
        MF->>TTF: base_font["maxp"].numGlyphs = len(new_glyph_order)

        rect rgb(34, 197, 94)
            Note over MF,TTF: 更新所有 cmap 子表（Office/Excel 需要 format=4）
            loop base_font["cmap"].tables
                alt platformID==3 encID==1/10<br/>或 platformID==0
                    loop 每个新增字形的 cjk_cmap[cp]
                        opt table.format==4 and codepoint<=0xFFFF
                            MF->>TTF: table.cmap[codepoint] = glyph_name
                        end
                    end
                end
            end
        end

        MF->>TTF: base_font["hhea"].advanceWidthMax = max(原最大, 1200)
        MF->>TTF: base_font["hhea"].numberOfHMetrics = len(base_hmtx.metrics)
    end

    MF->>MOR: merge_os2_ranges(base_font, cn_font)
    opt 两者都有 OS/2 表
        MOR->>MOR: base.ulUnicodeRange1..4 |= source.ulUnicodeRange1..4
        MOR->>MOR: base.ulCodePageRange1..2 |= source.ulCodePageRange1..2
    end

    MF->>TTF: cn_font.close()
    MF-->>B: return base_font（已被修改）

    style MF fill:#22c55e,color:#111,stroke-width:2px
    style GCG fill:#16a34a,color:#fff
    style GCE fill:#16a34a,color:#fff
    style IU fill:#8b5cf6,color:#fff
    style MOR fill:#8b5cf6,color:#fff
    style TTF fill:#3b82f6,color:#fff
    style CN fill:#f59e0b,color:#111
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
