---
kind: dependency_management
name: 基于 Cargo workspace + lockfile 的多 crate 依赖管理与 Zed 快照锁定策略
category: dependency_management
scope:
    - '**'
source_files:
    - Cargo.toml
    - Cargo.lock
    - .github/dependabot.yml
    - script/check-gpui-pin.ts
    - crates/component/Cargo.toml
    - website/package.json
    - flake.nix
    - .cargo/config.toml
---

## 1. 使用的系统与工具

- **Rust/Cargo**：仓库是 Cargo workspace（`Cargo.toml` 中 `resolver = "2"`，`members` 列出 `crates/*` 与 `examples/*`），所有 Rust 依赖通过 `[workspace.dependencies]` 集中声明，子 crate 以 `workspace = true` 引用。
- **Cargo.lock**：根目录存在 `Cargo.lock`，作为整个 workspace 的确定性构建锁文件；Nix flake (`flake.nix`) 通过 `cargoLock.lockFile = ./Cargo.lock` 直接复用该锁文件。
- **Dependabot**：`.github/dependabot.yml` 仅对 `package-ecosystem: cargo` 启用，且 `allow.dependency-name` 白名单只放行 `gpui` 和 `wry`，其余依赖不会自动提 PR。
- **Bun / npm**：文档站点 `website/package.json` 使用 Bun 运行脚本（`bunx --bun astro ...`），依赖版本范围较宽泛（如 `astro: 7.3.1`、`vue: ^3.5.0`、`tailwindcss: ^4.1.15`）。
- **Nix flake**：`flake.nix` 声明系统级构建依赖（`pkg-config`、`gtk3`、`vulkan-loader`、`fontconfig` 等）与 `rust-bin.beta.latest`，并通过 `nativeBuildInputs` / `buildInputs` 注入到 `buildRustPackage`。

## 2. 关键文件

- `Cargo.toml` — workspace 定义、`[workspace.dependencies]` 集中声明、Clippy lint、profile 优化配置。
- `Cargo.lock` — 全仓唯一锁文件。
- `.github/dependabot.yml` — 仅允许 Dependabot 更新 `gpui` 与 `wry`。
- `script/check-gpui-pin.ts` — 校验所有 `gpui-pre-*` crate 在 workspace 中必须使用精确版本 `=x.y.z` 且全部指向同一快照。
- `crates/component/Cargo.toml` — 典型子 crate 示例：通过 `workspace = true` 引用共享依赖，按 target/feature 条件编译引入平台相关依赖（macOS objc2、Windows windows crate、WASM 排除 native deps）。
- `website/package.json` — 文档站点的 JS 依赖入口。
- `flake.nix` — Nix 环境中的系统库与 Rust toolchain 声明。

## 3. 架构与约定

### 3.1 内部 crate 版本同步
根 `Cargo.toml` 的 `[workspace.package]` 将 `edition = "2024"`、`publish = false` 下沉给所有成员；各 crate 的 `version` 字段（如 `crates/component/Cargo.toml` 中的 `0.8.1`）与根 `[workspace.dependencies]` 中对 `gpui-kit`、`gpui-component`、`gpui-base`、`gpui-shell` 等的 `path = "...", version = "0.8.1"` 保持一致。发布时由外部流程统一 bump 版本号。

### 3.2 Zed/GPUI 快照锁定策略
仓库依赖的是 Zed 的预发布 crate（`gpui-pre-*`、`gpui-fast*`、`gpui-pre-reqwest-client`、`gpui-pre-sum-tree`）。根 `Cargo.toml` 注释明确说明：这些依赖被发布到 crates.io 时会被下游应用继承，因此必须用精确版本 `=0.3.10` 锁定，避免 caret 要求让下游意外升级到未测试过的快照。

`script/check-gpui-pin.ts` 会扫描 `workspace.dependencies` 中所有 `package` 名以 `gpui-pre` 开头的条目，断言：
- 每个都必须是 `=x.y.z` 形式（正则 `/^(\d+\.\d+\.\d+)$/`）；
- 除 `gpui-pre-reqwest` 外，所有快照 crate 必须指向同一个 `x.y.z`；
- 否则脚本以非零退出并在 CI 中输出 `::error file=Cargo.toml::...`。

### 3.3 条件依赖与 feature 门控
`crates/component/Cargo.toml` 展示了跨平台依赖的组织方式：
- 通用依赖放在顶层 `[dependencies]`；
- 非 WASM 依赖放入 `[target.'cfg(not(target_family = "wasm"))'.dependencies]`；
- macOS 专属依赖放入 `[target.'cfg(target_os = "macos")'.dependencies]`（objc2-*、core-text）；
- Windows 专属依赖放入 `[target.'cfg(target_os = "windows")'.dependencies]`（resvg、windows crate）；
- 可选功能（speech、syntect、tree-sitter-*、inspector、test-support）通过 `[features]` 组合，并转发到 `gpui-base`、`gpui-fast`、`gpui-kit-assets` 等同级 crate。

### 3.4 构建期优化
根 `Cargo.toml` 的 `[profile.dev.package]` 为 `resvg`、`rustybuzz`、`taffy`、`ttf-parser`、`smol`、`gpui-pre`、`quickjs-jit-*`、`cap-std` 等热路径 crate 设置 `opt-level = 3`，以加速增量构建；`[profile.release]` 开启 `strip = "symbols"` 以减小二进制体积。

## 4. 约定与约束

- **Workspace 集中化**：所有跨 crate 共享的第三方依赖必须在根 `Cargo.toml` 的 `[workspace.dependencies]` 中声明，子 crate 通过 `xxx.workspace = true` 引用（见 `crates/component/Cargo.toml` 中 `anyhow.workspace = true`、`notify.workspace = true` 等用法）。
- **Zed 快照精确锁定**：所有 `gpui-pre-*` crate 必须使用 `=x.y.z` 精确版本，且彼此一致；由 `script/check-gpui-pin.ts` 强制校验，CI 失败即阻止合并。
- **Dependabot 白名单**：只有 `gpui` 与 `wry` 允许 Dependabot 自动创建更新 PR，其他 Rust 依赖的升级需人工处理（见 `.github/dependabot.yml`）。
- **JS 依赖宽松范围**：`website/package.json` 大量使用 `^` 范围（如 `@tailwindcss/vite: ^4.1.15`、`vue: ^3.5.0`），无独立 lockfile 提交至仓库（仅有 `website/www/bun.lock` 属于独立的 `www/` 子项目）。
- **Nix 复现**：`flake.nix` 通过 `cargoLock.lockFile = ./Cargo.lock` 绑定 Rust 依赖，并通过 `nativeBuildInputs` / `buildInputs` 固定系统库版本（gtk3、vulkan-loader、wayland 等），保证可重现构建。
- **目标平台隔离**：平台相关依赖严格按 `cfg(target_os = ...)` 或 `cfg(not(target_family = "wasm"))` 分组，确保 WASM 构建不包含 native-only crate。
- **Windows 栈大小**：`.cargo/config.toml` 为 `x86_64-pc-windows-msvc` 设置 `link-arg=/STACK:8000000`，统一所有 crate 的 Windows 链接参数。