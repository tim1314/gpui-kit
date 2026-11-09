---
kind: logging_system
name: 基于 tracing 的日志系统：自定义 StderrSubscriber 与两级日志框架
category: logging_system
scope:
    - '**'
source_files:
    - crates/shell/src/host.rs
    - crates/story-web/src/lib.rs
    - examples/color_mix_oklab.rs
---

## 1. 使用的系统与框架

仓库采用 **双轨日志方案**：

- **核心运行时（`crates/shell`）**：使用 `tracing` + 自实现的 `StderrSubscriber`，不依赖 `tracing-subscriber`。所有 `tracing::info!` / `debug!` / `warn!` / `error!` / `trace!` 宏调用均通过该 Subscriber 输出到 stderr。
- **业务代码（`crates/base`、`crates/component`、`crates/webview`、`crates/assets` 等）**：直接调用 `tracing::*` 宏进行结构化记录；同时部分模块仍保留对 `log` crate 的 `log::warn!` / `log::debug!` 调用（来自第三方依赖或历史代码）。
- **WASM/浏览器端（`crates/story-web`）**：通过 `console_log` 将 `log` crate 的消息桥接到浏览器控制台。
- **示例程序（`examples/color_mix_oklab.rs`）**：使用 `env_logger::init()` 初始化 `log` 后端。

## 2. 关键文件

- `crates/shell/src/host.rs`：日志系统的唯一安装点与 Subscriber 实现。
- `crates/story-web/src/lib.rs`：WASM 端 `console_log` 初始化。
- `examples/color_mix_oklab.rs`：示例中 `env_logger` 的使用。
- `crates/base/src/**/*.rs`、`crates/component/src/**/*.rs`：大量 `tracing::warn!` / `tracing::error!` / `tracing::info!` 调用点。

## 3. 架构与设计决策

### 3.1 单一安装入口

`install_log_sink(max_level: Level)`（`host.rs:1301`）是唯一的日志订阅者安装点。它调用 `tracing::subscriber::set_global_default(StderrSubscriber { max_level })`，并忽略返回的错误——因为“已经安装了订阅者”意味着已有更好的 sink，无需覆盖。

### 3.2 自定义 `StderrSubscriber`

`StderrSubscriber` 实现了 `tracing::Subscriber` trait，行为如下：

- `enabled`：按 `max_level` 过滤事件。
- `max_level_hint`：返回 `LevelFilter`，让 `tracing` 在编译期跳过被禁用的 callsite（注释明确说明这是“保持禁用 debug! 在渲染路径中零开销”的原因）。
- `new_span` / `record` / `enter` / `exit`：全部空实现，因为该 sink 只关心 event，不记录 span。
- `event`：将事件格式化为单行 `LEVEL target: message fields...` 写入 stderr。

字段序列化由内部 `EventFields` Visit 完成：`message` 字段作为纯文本输出，其他字段以 `name=value` 形式追加，构成扁平的结构化行。

### 3.3 日志级别策略

| 启动模式 | 最大级别 | 来源 |
|---|---:|---|
| `gpui-shell run` / `dev` | `DEBUG`（`--dev` 时）或 `INFO`（默认） | `main_with_brand_and_components` 第 176–180 行 |
| `gpui-shell check` | `WARN` | 显式传入 `Level::WARN`（第 137 行），因为 check 只报告诊断信息 |
| WASM（story-web） | `Info` | `console_log::init_with_level(log::Level::Info)`（`story-web/src/lib.rs:95`） |

`--dev` 命令会设置 `verbose=true`，从而启用 DEBUG 级别；普通 `run` 仅 INFO。

### 3.4 为什么不用 `tracing-subscriber`

`host.rs` 中的注释明确声明了设计意图：crate 依赖 `tracing` 但不依赖 `tracing-subscriber`，因为“每个事件一行输出到 stderr”就是全部需求；如果需要过滤、span 和格式化，应该引入 `tracing-subscriber` 而不是让这个 subscriber 膨胀。

### 3.5 跨平台与多目标差异

- 桌面端（`crates/shell`）：`tracing` + 自定义 `StderrSubscriber`。
- WASM 端（`crates/story-web`）：`console_log` 桥接 `log` crate 到浏览器控制台。
- 非 Rust 侧（如 `crates/webview`、`crates/assets`、`crates/component/speech`）：直接使用 `log` crate 宏，由各自环境的 logger 处理。

## 4. 约定与约束

- **Shell 二进制必须通过 `install_log_sink` 安装日志 sink**：在 `main_with_brand_and_components` 中，sink 在所有逻辑之前安装，注释指出“否则脚本错误、未处理的 promise rejection、非法阶段调用都会通过 tracing 发出但被静默丢弃”。
- **`check` 子命令强制使用 `WARN` 级别**：硬编码为 `Level::WARN`，因为 check 只报告诊断，不输出进度信息。
- **`--dev` 开启 `DEBUG` 级别**：`dev` 命令等价于 `run --watch --dev`，其中 `is_verbose()` 返回 true，驱动 DEBUG 级别。
- **已存在的 subscriber 不可被覆盖**：`set_global_default` 的返回值被忽略，允许嵌入宿主先安装自己的 subscriber。
- **Span 不被记录**：`new_span` / `record` / `enter` / `exit` 均为空操作，避免无意义的 span 开销。
- **结构化字段命名约定**：`message` 字段作为消息正文，其余字段以 `key=value` 形式追加（见 `EventFields::record_debug`）。
- **业务代码统一使用 `tracing::*` 宏**：`base` 和 `component` crate 中的日志调用几乎全部走 `tracing::warn!` / `tracing::error!` / `tracing::info!` / `tracing::debug!` / `tracing::trace!`。
- **WASM 环境通过 `console_log` 桥接 `log` crate**：`story-web` 在初始化时将 `log` 级别设为 `Info`。
- **示例程序独立初始化 `env_logger`**：`examples/color_mix_oklab.rs` 单独调用 `env_logger::init()`，不影响库本身。

## 5. 观察到的日志级别分布

- `tracing::warn!`：最常用，用于可恢复的异常路径（HTML 解析警告、剪贴板读取失败、WebView 组合失败等）。
- `tracing::error!`：用于不可恢复的错误（主题目录监听失败、highlighter 错误、palette 解析失败）。
- `tracing::info!`：用于用户可见的操作（主题重载、应用退出请求）。
- `tracing::debug!`：开发调试信息（storage 路径、development mode 提示）。
- `tracing::trace!`：性能测量（FPS、渲染耗时）。
- `log::warn!` / `log::debug!`：主要来自第三方依赖（如 speech、webview、assets 模块）。
