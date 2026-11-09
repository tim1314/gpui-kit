---
kind: configuration_system
name: GPUI Shell 配置系统：插件清单、能力白名单与默认环境变量
category: configuration_system
scope:
    - '**'
source_files:
    - crates/shell/src/plugin.rs
    - crates/shell/src/capability.rs
    - crates/shell/src/default-env
    - crates/shell/src/host.rs
    - crates/shell/src/runtime.rs
    - crates/shell/src/theme_tokens.rs
    - crates/shell/src/typings.rs
---

## 1. 采用的系统与方式

GPUI Kit 的运行时配置集中在 `crates/shell` crate，采用「Rust 代码 + JSON 清单 + 纯文本白名单」的组合：

- **应用清单**：每个脚本化应用根目录下的 `gpui-shell.json`（常量 `MANIFEST_FILE`），由 `plugin.rs` 用 `serde` + `schemars` 解析为 `PluginManifest`。
- **能力/权限声明**：清单中的 `capabilities` 字段映射到 `capability.rs` 的 `Capabilities` / `HttpRequestGrant` / `ExecuteGrant`，是沙箱白名单而非任意配置。
- **默认环境变量白名单**：`crates/shell/src/default-env` 是一个每行一个变量名的纯文本文件，通过 `include_str!` 编译进二进制，由 `default_env_variables()` 提供。
- **主题 token 默认值**：`crates/shell/src/default-tokens.json` 通过 `include_str!` 嵌入，作为语义主题 token 的初始值。
- **CLI 参数**：`host.rs` 中手写 `parse()`，没有引入 clap/structopt，因为命令面很小且错误消息需要精确控制。

## 2. 关键文件与包

- `crates/shell/src/plugin.rs` — 清单结构体、Git 依赖解析、`manifest_schema()` 生成 JSON Schema、`discover/load` 生命周期。
- `crates/shell/src/capability.rs` — `Capabilities`、`HttpRequestGrant`、`ExecuteGrant`、`is_openable_url`、`default_env_variables()`。
- `crates/shell/src/default-env` — 允许脚本直接读取的环境变量名列表（无注释、无 secret）。
- `crates/shell/src/host.rs` — CLI 解析、应用目录解析、`read_local_manifest()`、打包/检查/类型生成入口。
- `crates/shell/src/runtime.rs` — 应用数据目录定位 (`app_data_dir`、`data_home`)、应用身份校验与路径哈希。
- `crates/shell/src/theme_tokens.rs` — 从 `default-tokens.json` 加载并暴露给 JS 侧的主题 token 访问。
- `crates/shell/src/typings.rs` — 把 Rust 类型导出为 TypeScript 声明（含 `SemanticThemeTokens`、`CONFIG_FILE_NAME = "jsconfig.json"`）。

## 3. 架构与约定

### 3.1 清单即权限边界

`plugin.rs` 的模块文档明确区分两类概念：**permission（能力）** 属于 data，必须在代码执行前向用户展示；**contribution（行为贡献）** 属于 code，在脚本里注册。因此清单只承载 `id`、`name`、`version`、`shell-version`、`entry`、`dependencies`、`capabilities` 七个字段，命令/面板/快捷键/设置/主题不重复声明。

清单解析流程：
1. `PluginManager::discover` 仅扫描目录、读 JSON，**不执行任何脚本**。
2. 校验 `shell-version` 与当前 `SHELL_VERSION`（= `CARGO_PKG_VERSION`）兼容。
3. 将 `capabilities` 反序列化为 `Capabilities`，再传给 `Policy` 和 `ShellRuntime`。
4. 只有 `load` 阶段才真正加载 entry 模块。

### 3.2 环境变量分层

`capability.rs` 把环境变量分为两层：

| 层级 | 来源 | 是否需要显式授权 | 说明 |
|---|---|---:|---|
| 默认层 | `default-env` 文件 | 否 | 描述用户机器/会话/偏好，不含密钥或代理凭据 |
| 请求层 | 清单 `capabilities.process.env` | 是 | 应用额外需要的变量名 |

`default_env_variables()` 使用 `OnceLock` 缓存解析结果，变量名匹配忽略大小写（`HTTP_PROXY` 也匹配 `http_proxy`）。`default-env` 的注释明确禁止把 tokens、keys、passwords、credentials 以及 `DBUS_SESSION_BUS_ADDRESS`、`SSH_AUTH_SOCK` 等传递他人权限的地址放入该文件。

### 3.3 应用身份与数据目录

`runtime.rs` 对应用 `id` 做严格校验：只能包含小写字母、数字、`.`、`-`、`_`，不能为空，不能包含 `..`。未安装的应用使用 `path_identity(root)`（目录名 + FNV-1a 路径摘要）作为临时 id，已安装应用必须使用清单 `id`，否则移动目录后数据会丢失。

数据目录遵循平台惯例：优先 `XDG_DATA_HOME`，否则 macOS → `~/Library/Application Support`，Windows → `%APPDATA%`，Linux → `~/.local/share`，最终路径为 `<data>/gpui-shell/apps/<id>`。

### 3.4 主题 token 配置

`theme_tokens.rs` 从内嵌的 `default-tokens.json` 构建 `SemanticThemeTokens`，并通过全局状态暴露给 JS 侧 `theme.tokens.*` API。JS 侧的 TypeScript 类型由 `typings.rs` 同步生成，保证 Rust 与 TS 的类型定义一致。

### 3.5 CLI 配置

`host.rs` 的手写解析器支持 `run` / `dev` / `check` / `types` / `agents` / `new` / `list` / `bundle` 子命令，以及 `--dir`、`--watch`、`--packaged`、`--dev`、`--print-spec`、`--platform`、`--icon`、`--out`、`--download-url` 等选项。未知 flag 直接报错，不会静默当作路径处理。

## 4. 约定与约束

- **清单文件名固定**：`gpui-shell.json`（`plugin.rs: MANIFEST_FILE`），打包应用必须存在，否则 `host.rs` 在 packaged 模式下拒绝启动。
- **清单字段最小集**：只允许 `id`、`name`、`version`、`shell-version`、`entry`、`dependencies`、`capabilities`、`window`；其他字段不被识别（见模块文档注释）。
- **能力默认拒绝**：`Capabilities` 默认值为空，脚本没有任何 fs/process/network/storage/clipboard/exit/env 权限，直到清单或宿主显式授予。
- **URL 打开限制**：`is_openable_url` 只允许 `http`/`https` 且有 host 的 URL，防止任意 scheme 被交给系统处理器。
- **环境变量白名单强制**：不在 `default-env` 且未在清单 `capabilities.process.env` 列出的变量，脚本无法读取（`capability.rs` 注释明确此规则）。
- **应用 id 字符集**：`runtime.rs` 的 `validate_identity` 只允许 `[a-z0-9._-]`，非法字符直接返回 error。
- **开发模式开关**：`--dev` 仅在 `run`/`dev` 子命令上合法（`host.rs` 解析器校验），开启后会调用 `set_development_mode(true)` 放宽 eval 与内置原型链。
- **清单 schema 自动生成**：`manifest_schema()` 基于 `ManifestFile` 的 `JsonSchema` 派生，避免手写 schema 与解析器不一致。
- **主题 token 默认值不可变**：`default-tokens.json` 通过 `include_str!` 编译进二进制，运行时只读，修改需重新编译。
- **JS 配置文件名**：`typings.rs` 定义 `CONFIG_FILE_NAME = "jsconfig.json"`，用于编辑器集成。

## 5. 未覆盖范围

仓库中没有发现 `.env` 文件加载、`.toml` 应用级配置、feature flags 开关、远程配置拉取或 secrets manager 集成。配置体系完全围绕 gpui-shell 的插件模型展开，面向的是“脚本化应用”的权限与元数据，而非通用应用程序配置框架。