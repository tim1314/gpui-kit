---
kind: error_handling
name: GPUI Kit 错误处理体系：领域错误枚举 + ShellError 脚本边界
category: error_handling
scope:
    - '**'
source_files:
    - crates/shell/src/error.rs
    - crates/shell/src/host_modules.rs
    - crates/shell/src/spec.rs
    - crates/shell/src/capability.rs
    - crates/shell/src/plugin.rs
    - crates/shell/src/scope.rs
    - crates/shell/src/value.rs
    - crates/base/src/motion.rs
    - crates/base/src/motion/easing.rs
    - crates/base/src/questionnaire/types.rs
    - crates/component/src/speech/recognizer.rs
    - crates/component/src/diff/parser.rs
---

## 1. 总体方案

GPUI Kit 在 Rust 侧采用「标准库 `std::error::Error` + 领域专用 `enum`/`struct` 错误类型」，没有引入第三方错误框架（如 `anyhow`、`thiserror`）。跨脚本引擎的边界通过一个统一的 `ShellError` 包装，将样式解析、值转换、spec arena 等内部错误转换为脚本作者可见的异常。

核心分层：
- **领域层**：每个子系统定义自己的错误枚举（`SpecError`、`CapabilityError`、`ManifestProblem`、`SpeechError`、`EasingError`、`SpringError`、`QuestionnaireValidationError` 等），实现 `Display` + `std::error::Error`。
- **宿主函数层**：`HostError` 表示宿主函数返回的错误，仅携带消息，模块名和函数名由引擎在边界处附加。
- **脚本边界层**：`ShellError` 是 engine-neutral 的类型，注释明确说明「必须不命名特定 VM，各引擎在边界处将其转为自身错误」。

## 2. 关键文件与类型

| 文件 | 类型 | 用途 |
|---|---|---|
| `crates/shell/src/error.rs` | `ShellError(String)` + `type Result<T>` | 引擎无关的脚本级错误，提供 `runtime(message)` 构造器 |
| `crates/shell/src/host_modules.rs` | `HostError` + `HostResult = Result<HostValue, HostError>` | 宿主函数返回值，参数校验失败时返回 |
| `crates/shell/src/spec.rs` | `SpecError { Claimed, Expired, AlreadyParented, SelfParent, DuplicateChildView }` | UI spec 树构建期错误 |
| `crates/shell/src/capability.rs` | `CapabilityError { NotGranted, OutsideRoots, ExecuteDenied, StorageDenied }` | 权限/能力检查失败 |
| `crates/shell/src/plugin.rs` | `ManifestError` + `ManifestProblem` | 插件清单解析错误，附带来源路径 |
| `crates/shell/src/scope.rs` | `StaleContext` | 捕获了过期 `cx` 后使用 |
| `crates/base/src/motion.rs` | `SpringError` | 弹簧动画参数错误 |
| `crates/base/src/motion/easing.rs` | `EasingError` | 缓动函数参数错误 |
| `crates/base/src/questionnaire/types.rs` | `QuestionnaireValidationError` / `QuestionnaireSchemaError` | 问卷校验错误 |
| `crates/component/src/speech/recognizer.rs` | `SpeechError` | 语音识别错误 |
| `crates/component/src/diff/parser.rs` | `DiffParseError` | diff 解析错误 |

## 3. 架构约定

### 3.1 错误类型设计模式
- 所有领域错误均为 `#[derive(Debug, PartialEq)] enum`，字段语义清晰，例如 `AlreadyParented { component: &'static str }`、`OutsideRoots { path: PathBuf, access: Access }`。
- 每个错误都实现 `std::fmt::Display`，消息面向最终用户（脚本作者或插件作者），而非开发者日志。`ManifestProblem` 的注释明确要求措辞遵循 CapabilityError 风格：先说哪里错了，再说应该写什么。
- 所有错误都实现 `std::error::Error`（空 impl），以适配通用错误链。

### 3.2 脚本边界隔离
`error.rs` 顶部注释是约束性文档：样式解析、值强制转换、spec arena 产生的错误必须到达脚本作者作为脚本级异常，不能命名特定 VM；每个引擎在边界处把 `ShellError` 转成自己的错误。

### 3.3 宿主函数错误传递
`host_modules.rs` 中 `HostError` 只携带 message，注释说明「引擎会把模块名和函数名加进去，宿主函数编写者无需重复自己的名字」。同步/异步宿主函数统一返回 `HostResult`，异步分支在后台执行器上运行，调用方无法访问 `App`/`Ctx`，从类型层面杜绝越界回调。

### 3.4 值转换错误集中化
`crates/shell/src/value.rs` 中所有 JS→Rust 值转换失败都走 `ShellError::runtime(format!(...))`，这是脚本边界的统一入口。

## 4. 约定与约束

- **约定**：领域错误用 `enum` 表达离散状态，带可读 `Display`；宿主函数错误用简单 `struct HostError(String)`；脚本边界错误统一为 `ShellError`。
- **约定**：错误消息面向终端用户（脚本作者/插件作者），包含可操作建议（如 `declare capabilities.fs.read in the manifest`）。
- **约定**：`From<String>` / `From<&str>` 为 `HostError` 实现，便于直接 `Err("msg".into())`。
- **约定**：`ManifestError` 把来源路径作为独立字段而非变体，使调用方可在不解包的情况下匹配问题类型。
- **约束（文档声明）**：`ShellError` 不得命名特定 VM，否则 `engine/` 之上不再 engine-independent——这是跨引擎抽象的硬性要求。
- **约束（类型系统）**：异步宿主函数的 future 必须是 `Send + 'static`，且注释明确指出「这不是外加限制，而是模块头对同步函数要求的物理化：宿主函数不得重新进入引擎」。
- **约束（测试）**：`SpecError` 的单元测试覆盖 Claimed / Expired / AlreadyParented / SelfParent / DuplicateChildView 全部变体，确保错误传播路径可验证。

## 5. 未观察到的内容

仓库中没有发现 `panic!` 用于正常控制流（搜索命中集中在测试代码中的 `expect`/`unwrap`），也没有 `catch_unwind` 或 `recover` 机制；也没有中间件式的错误拦截层。错误主要通过 `Result` 向上冒泡，在脚本引擎边界统一转换为脚本异常。