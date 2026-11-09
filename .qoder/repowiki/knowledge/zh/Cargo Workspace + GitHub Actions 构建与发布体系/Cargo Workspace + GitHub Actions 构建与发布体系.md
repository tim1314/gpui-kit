---
kind: build_system
name: Cargo Workspace + GitHub Actions 构建与发布体系
category: build_system
scope:
    - '**'
source_files:
    - Cargo.toml
    - Makefile
    - .cargo/config.toml
    - flake.nix
    - .github/workflows/ci.yml
    - .github/workflows/release.yml
    - .github/workflows/release-shell.yml
    - .github/workflows/release-website.yml
    - script/bump-version.sh
    - script/publish-crates
    - script/check-gpui-pin.ts
    - crates/story-web/Makefile
---

## 1. 构建系统与工具链

仓库采用 **Rust Cargo workspace**（`Cargo.toml`）作为核心构建系统，聚合 `crates/` 下的 8 个核心 crate（kit、base、component、component-macros、assets、fps、shell、webview）、`examples/` 下 20+ 示例 crate 以及 `crates/base/examples/{native,wasm}`。默认成员为 `crates/story`。

- **Rust 版本**：通过 `.cargo/config.toml` 指定 Windows 目标栈大小；Nix flake (`flake.nix`) 使用 `rust-bin.beta.latest` 并启用 `rust-src`。
- **依赖锁定**：所有 CI 步骤均加 `--locked`，禁止未提交变更的 `Cargo.lock`。
- **WASM 支持**：额外安装 `nightly` toolchain 与 `wasm32-unknown-unknown` target，并通过 `wasm-bindgen-cli`（版本从 `Cargo.lock` 解析）编译 WASM。
- **链接器加速**：CI 中统一通过 `rui314/setup-mold@v1` 替换系统 `ld`。
- **系统依赖**：Linux/macOS 通过 `script/bootstrap` 安装；Windows 不执行此步骤。

## 2. 关键文件与职责

| 文件 | 职责 |
|---|---|
| `Cargo.toml` | workspace 定义、workspace.dependencies 集中管理 gpui-pre/gpui-fast 快照版本、Clippy lint 规则、dev/release profile 优化 |
| `Makefile` | 顶层入口：`dev-web`、`dev-gallery`、`dev:website`、`build:website` |
| `crates/story-web/Makefile` | Story Web 子项目：WASM 构建、前端 bun build、开发服务器 |
| `flake.nix` | Nix 开发环境：声明 GTK3/Vulkan/fontconfig 等动态库依赖，设置 `FONTCONFIG_FILE` 与 `LD_LIBRARY_PATH` |
| `.cargo/config.toml` | Windows 目标栈大小 `-C link-arg=/STACK:8000000` |
| `script/bump-version.sh` | 发布流程脚本：`cargo set-version` → git add/commit/tag/push |
| `script/publish-crates` | crates.io 发布：按依赖顺序依次发布 gpui-kit-assets → component-macros → base → fps → component → kit → webview → shell，已存在包跳过 |
| `script/check-gpui-pin.ts` | 校验 gpui-pre 快照版本精确 pin（不允许 caret） |
| `script/install-linux.sh` / `script/install-window.ps1` | 平台安装辅助脚本 |

## 3. CI 架构（`.github/workflows/`）

### CI 流水线（`ci.yml`）
- **checks**：typos、AI recipe 验证、gpui-pre pin 校验、`cargo machete`。
- **lint**：macOS runner 上对全 workspace clippy（排除 gpui-shell/webview），WebView 单独以 `gpui-fast` feature 运行。
- **no-default-features**：Ubuntu 上检查 `gpui-component --no-default-features`。
- **test**：矩阵覆盖 `aarch64-apple-darwin` / `x86_64-linux-gnu` / `x86_64-windows-msvc`，排除 examples 目录（由 lint job 类型检查），启用 `gpui-component-story/test-support` 跑 headless UI 集成测试。
- **gpui-fast**：三平台测试 `gpui-kit`、`gpui-component tooltip::tests`、`gpui-webview`。
- **gpui-fast-web**：nightly + wasm32 target check。
- **gpui-shell-core**：Ubuntu 上测试运行时 core（跳过 standard_runtime/network/fs/process/benchmark），clippy，无 components feature 编译检查。
- **gpui-shell-components**：组件绑定测试 + TypeScript 契约校验 + story 覆盖率校验。
- **gpui-shell-standard-runtime**：三平台 LLRT 模块集成测试。

缓存策略：仅 `main` 分支保存 rust-cache；PR 恢复最近 main 缓存；不同 feature 集使用独立 `cache-shared-key`（如 `gpui-fast`、`gpui-fast-web`、`gpui-shell-release`）避免互相污染。

### 发布流水线
- **release.yml**：push `v*` tag 触发，调用 `script/publish-crates` 发布到 crates.io。
- **release-shell.yml**：构建 gpui-shell 预编译二进制（不在 crates.io），产物上传至 GitHub Release，附带 sha256 校验；支持 `workflow_dispatch` 重打已有 release、PR 加 `release-shell` label 预览构建；非 push 事件需 release 已存在否则报错；记录 `shell-<tag>` 标签关联构建 commit。
- **release-website.yml**：在 `website-versioned-docs` 分支推送或 `Release Crate` workflow 成功后，构建 Astro 站点 + Story Web WASM + Base WASM 示例，部署到 GitHub Pages；通过 `script/build-website-versions` 生成历史版本站点缓存。
- **test-docs.yml**：文档站点测试。

## 4. 约定与约束

- **GPUI 快照版本必须精确 pin**：`Cargo.toml` 中 `gpui-pre` 系列依赖使用 `=0.3.10` 形式，注释明确说明“caret requirement let the weekly release move applications onto a newer snapshot that gpui-component did not compile with”；`script/check-gpui-pin.ts` 在 CI checks job 中强制校验，失败即中断。
- **发布顺序固定**：`script/publish-crates` 硬编码依赖顺序（assets → macros → base → fps → component → kit → webview → shell），确保先发布被依赖方。
- **版本号单一来源**：`script/bump-version.sh` 通过 `cargo set-version` 统一更新所有 crate 版本，随后 git add/commit/tag/push，形成 `vX.Y.Z` 标签后触发 release workflow。
- **Shell 二进制命名不含版本**：`release-shell.yml` 注释说明资产名不带版本以便 Homebrew/mise 通过 `releases/latest/download/<asset>` 下载。
- **CI 禁用 debug info**：`CARGO_PROFILE_DEV_DEBUG=0` 环境变量使依赖缓存保持在 10 GB 限制内。
- **Profile 优化**：`profile.dev` 关闭调试信息、拆分 debuginfo；`profile.release` strip symbols；`profile.dev.package` 对 resvg/rustybuzz/taffy/ttf-parser/smol/gpui-pre/quickjs-jit-* 等热路径 crate 显式 `opt-level = 3`。
- **Clippy 规则**：workspace.lints.clippy 中 `dbg_macro = "deny"`、`todo = "deny"`，其余多数 allow；WebView 单独 clippy 也 `--deny warnings`。
- **Typos 配置**：`[workspace.metadata.typos]` 排除 fixtures，扩展标识符 `consts`。
- **Cargo-machete 忽略**：log、anyhow、serde 三个 crate 被标记为 ignored。
- **Nix 开发环境**：`flake.nix` 固定 GTK3/Vulkan/fontconfig/libxcb/libxkbcommon/atk/gio-sharp/glib 等动态库，并通过 `wrapProgram` 注入 `LD_LIBRARY_PATH` 与 `FONTCONFIG_FILE`。
- **Website 构建**：顶层 `Makefile dev:website` 通过 `bun run --cwd website dev` 启动 Astro dev server，保持仓库根 Rust-only 语义。