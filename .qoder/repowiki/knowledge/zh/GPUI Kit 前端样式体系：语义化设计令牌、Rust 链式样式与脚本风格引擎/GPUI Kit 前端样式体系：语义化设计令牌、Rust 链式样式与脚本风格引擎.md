---
kind: frontend_style
name: GPUI Kit 前端样式体系：语义化设计令牌、Rust 链式样式与脚本风格引擎
category: frontend_style
scope:
    - '**'
source_files:
    - crates/base/src/theme.rs
    - crates/base/src/theme_tokens.rs
    - crates/base/src/styled.rs
    - crates/shell/src/style.rs
    - crates/shell/src/theme_tokens.rs
    - crates/shell/src/default-tokens.json
    - themes/catppuccin.json
    - .theme-schema.json
    - docs/STYLING-AND-MOTION.md
---

## 1. 采用的方法与工具

仓库没有使用 CSS/SCSS/Tailwind 等 Web 样式方案，而是基于 Rust + GPUI（桌面 UI 框架）构建的纯 Rust 组件库。样式系统由三层组成：

- **基础层（`crates/base`）**：定义全局 `Theme`、语义设计令牌（`SemanticThemeTokens`）、滚动条/可调整大小面板/绘图的主题配置，以及通用的值过渡与弹簧动画（`transition` / `spring` / `Sequence` / `Stagger`）。所有视觉属性通过 GPUI 的 `StyleRefinement` 以链式方法设置。
- **组件层（`crates/component`）**：在 base 之上提供 60+ 桌面组件，每个组件暴露语义状态（checked / pressed / selected / disabled 等），并通过 `.styles(|styles| ...)` 闭包把应用提供的颜色、边框、透明度等映射到具体状态。
- **Shell 脚本层（`crates/shell`）**：为 JS/Rust 脚本暴露一套类 CSS 的 fluent API（如 `v_flex():size_full():bg("surface"):p(12)`），通过反射 + 手工绑定把方法名编译成紧凑的 `u16` 索引，最终写入 `StyleRefinement`。

文档 `docs/STYLING-AND-MOTION.md` 明确划分所有权：GPUI 检测 hover/active/focus；base 定义语义状态并解析优先级；应用或 gpui-component 拥有目标样式、变体与视觉槽位。

## 2. 关键文件

- `crates/base/src/theme.rs` — 全局 `Theme`、`ThemeAppearance`（light/dark）、`ScrollbarTheme`、`PlotTheme`、`ResizableTheme`，作为 GPUI `Global` 存储。
- `crates/base/src/theme_tokens.rs` — 语义设计令牌：`ColorTokens`（background/foreground/surface/primary/secondary/muted/accent/destructive/border/input/ring/selection）、`RadiusTokens`（none/sm/md/lg/xl/full）、`SpacingTokens`（xxs..xxl）、`TypographyTokens`（sans/mono + xs..xl 字号行高）、`ShadowTokens`（sm/md/lg 阴影层级）。
- `crates/base/src/styled.rs` — `StyledExt` trait，提供 `h_flex`/`v_flex`/`paddings`/`margins`/`debug_*`/`font_*` 等链式辅助方法。
- `crates/shell/src/style.rs` — Shell 风格引擎，维护无参方法反射表与约 40 个带参方法的 `PARAM_STYLES` 白名单，实现 `apply_nullary` / `apply_param`。
- `crates/shell/src/theme_tokens.rs` — 将脚本中的 token 名称（`"surface"`、`"md"`、`"lg"` 等）解析为 `gpui_base::SemanticThemeTokens` 的实际值，并缓存主题快照。
- `crates/shell/src/default-tokens.json` — Shell 默认 light/dark 配色 JSON。
- `themes/*.json`（如 `catppuccin.json`）—— 用户主题，遵循 `.theme-schema.json`，包含 UI 颜色、滚动条、标题栏、tab、列表、编辑器语法高亮等字段。
- `website/src/{dark,light}.theme.json` —— 文档站点的 VS Code 风格主题（独立于运行时主题）。
- `docs/STYLING-AND-MOTION.md` —— 样式与动效规范文档。

## 3. 架构与设计约定

### 3.1 语义令牌优先于硬编码值
`theme_tokens.rs` 注释明确：“这些令牌描述视觉角色和尺度，刻意不包含 button/table/sidebar 等组件名”。颜色、圆角、间距、字体、阴影全部走 `SemanticThemeTokens`，组件代码不应写死 `#ffffff` 或 `px(8.)`。

### 3.2 样式优先级固定
文档规定顺序：实例样式 → 活跃值状态（按组件声明顺序，如 checked → indeterminate → disabled）→ 禁用 → GPUI 原生交互修饰符（hover/active/focus/focus-visible）。同一层只覆盖自身设置的字段，不相关字段保留。

### 3.3 语义状态 vs 运行时伪状态
- GPUI 原生 `hover()` / `active()` / `focus()` / `focus_visible()` 用于指针条件。
- Base 控件通过 `.styles(|styles| styles.checked(...).disabled(...))` 暴露语义状态，由组件内部决定哪些状态可用。

### 3.4 圆角统一来自主题
文档要求“不要写字面量半径”，应使用 `rounded_full()` / `rounded_md()` 等主题派生方法；`Theme::radius` 设为 0 时，avatar、badge、radio、slider thumb、pill tab、progress bar 都应变成方形。

### 3.5 动效所有权分离
Base 只提供通用 `transition` / `spring` / `Sequence` / `Stagger` 生命周期机制，**不安装默认淡入/滑动/弹簧**。产品动效属于应用或 styled 组件层。`motion::Transition` 是旧 API，新代码应使用值过渡。

### 3.6 Shell 脚本风格 API
Shell 的 `style.rs` 把无参方法通过 `gpui_base::styled_ext_reflection_methods` 与 `gpui::styled_reflection::methods` 自动反射，有参方法（w/h/p/m/gap/bg/shadow/border/rounded/font_weight/line_height 等）手工绑定在 `PARAM_STYLES` 中。脚本调用 `:bg("surface")` 会被解析为 token 颜色，长度支持 `"auto"` / `"50%"` / `"12px"` / `"1rem"`。

### 3.7 主题文件格式
`themes/*.json` 使用 `.theme-schema.json` 校验，结构包括 `name` / `author` / `url` / `themes[]`（每个 theme 含 `mode: "light"|"dark"`、`colors`、可选 `highlight.syntax` 语法高亮）。`default-tokens.json` 仅包含 `light.colors` 与 `dark.colors` 两套扁平色板。

## 4. 约定与约束

- **语义令牌命名空间**：颜色 token 名称集合（`COLOR_TOKEN_NAMES`）必须与 `ColorTokens` 字段一致，spacing/radius 同理；`crates/shell/src/theme_tokens.rs` 中的测试断言 `field_names(&ColorTokens::default()) == sorted(color_token_names())`，新增 token 时必须同步更新 resolver。
- **禁止硬编码圆角**：文档 `STYLING-AND-MOTION.md` 第 9 条不变式：“Corner radius is derived from the theme, never written as a literal.”
- **禁用态最后覆盖**：不变式第 6 条：“Disabled is the last semantic layer.” 应用若要在 disabled 下保留自定义背景，需在状态闭包内重放该 refinement。
- **GPUI 原生伪状态是唯一 hover/active/focus 机制**：不变式第 4 条，base 不提供第二套选择器。
- **Shell 风格方法白名单**：`PARAM_STYLES` 是“唯一手工维护的列表”（style.rs 注释），新增脚本风格方法需同时加入此处与 typings 文档；无参方法则依赖反射，但 `EXTRA_NULLARY` 中 `font_*` 权重方法因宏展开无法被反射而需手动追加。
- **反射表规模守卫**：`tests::the_reflection_table_is_populated` 断言 `table().nullary.len() > 100`，确保启用 `gpui-base/inspector` feature 后 release 构建仍有数百个反射样式方法。
- **未知 token 拒绝**：`resolve_color` / `resolve_spacing` / `resolve_radius` 对未识别名称返回 `None`，并由上层报错；测试验证拼写错误（`"backgrund"`、`"xxxl"`、`"rounded"`）被拒绝。
- **动效预算**：基准测试要求标量 timing/easing 采样中位数 ≤ 0.10 ms（`cargo bench -p gpui-base --bench motion`）。
- **减少动效偏好**：`init` 读取操作系统偏好写入 `App::set_reduce_motion`，`transition` / `spring` / `Presence` / `Sequence` 均尊重该标志。
- **主题变更刷新**：通过 `Theme::change` / `Theme::update` 修改主题，直接写 `Theme::global_mut` 的公开字段不会刷新 scrollbar 等 base 内部副本，需调用 `Theme::sync_base`。

## 5. 结论

这是一个以 Rust 类型系统与 GPUI `StyleRefinement` 为核心的桌面 UI 样式体系：用 `SemanticThemeTokens` 抽象颜色/圆角/间距/排版/阴影，用 `Theme` 作为全局上下文，用组件级 `.styles(...)` 闭包表达语义状态，用 Shell 脚本层暴露类 CSS 的 fluent API。它不使用 CSS/SCSS/Tailwind，也不依赖外部样式框架，所有视觉一致性由 Rust 类型、JSON schema 与文档中的不变式共同保证。