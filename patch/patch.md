# d2736965-patch-v0.7 分支笔记

本分支基于 Dioxus 官方 v0.7 分支（v0.7.10 = `57d6794a`），包含针对个人开发环境的定制修改。

crates.io 发布的版本对应 `v0.x` 版本号分支，`v0.7` 是当前最新稳定版。

## 维护指引

- **何时更新**：每次新增、修改或删除与上游有差异的代码时，必须同步更新本文件的对应条目
- **更新内容**：改动描述要包含具体文件和关键代码位置（函数名、行号范围），方便后续对照
- **删除条目**：如果某个改动被合入上游或被回退，删除对应条目并注明原因
- **改动提交**：日常修改用 `git commit --amend` 合入分支顶部的提交
- **合并上游后**：检查改动清单是否仍然准确，合并可能使某些 diff 消失或产生新的差异
- **编译前检查**：每次 `cargo check` 或 `cargo build` 前，先执行 `cargo clippy --package dioxus-cli -- --no-deps` 并核对输出。**lint 只约束本分支的补丁代码，官方代码的警告一律不管**：全量跑完后，只要警告不落在我们的补丁块（android.rs 的 Termux 分支 ~1204 行、res_dir/write_default_res ~296-401 行、cli/Cargo.toml、gradle-wrapper.properties、web 三个 html 模板）即 PASS；官方文件的警告（如 clippy 暴露的 `useless_borrows_in_formatting`、`result_large_err`，确认无一在补丁块内）不处理、不豁免、不记录
  - 不添加 `#![deny(clippy::unwrap_used)]` 等全局 lint —— 上游有大量 anyhow / Mutex / unwrap，全局 deny 会导致大量误报

## 补丁清单（与上游 v0.7.10 的差异）

### Gradle 分发镜像

**修改文件**：
- `packages/cli/assets/android/gen/gradle/wrapper/gradle-wrapper.properties`

**问题**：默认的 Gradle 分发 URL `https://services.gradle.org` 在中国网络环境下下载速度很慢。

**处理方式**：
- distributionUrl 改用腾讯云镜像
  - 原：`https\://services.gradle.org/distributions/gradle-9.1.0-bin.zip`
  - 新：`https\://mirrors.cloud.tencent.com/gradle/gradle-9.1.0-bin.zip`

### NDK 工具链路径支持 Termux

**修改文件**：
- `packages/cli/src/build/android.rs` 第 1204 行附近（`cfg!(target_os = "android")` 分支）

**问题**：交叉编译到 `aarch64-linux-android` 的 CLI 二进制在 Termux 上运行时，`cfg!(target_os = "android")` 为 true，但源码中没有处理 android 分支，导致读取 toolchains 目录失败。

**处理方式**：
- 在 `cfg!(target_os = "windows")` 分支后添加 `target_os = "android"` 分支
- Android/Termux 的 NDK prebuilt 目录名为 `android`（符号链接指向 `linux-x86_64`）
- 返回 `prebuilt/android/bin` 而不是尝试读取整个目录

### git2 为 musl 启用 vendored-openssl

**修改文件**：
- `packages/cli/Cargo.toml` 第 175 行

**问题**：musl 交叉编译时 git2 的 OpenSSL 链接失败，原版仅对 musl 启用 `vendored-libgit2`，缺少 `vendored-openssl`。

**处理方式**：
- 在 `[target.'cfg(target_env = "musl")'.dependencies]` 中为 git2 添加 `vendored-openssl` feature
  - 原：`features = ["vendored-libgit2"]`
  - 新：`features = ["vendored-libgit2", "vendored-openssl"]`

### Web 模板移除 Google Fonts，使用系统字体

**修改文件**：
- `packages/cli/assets/web/dev.index.html`
- `packages/cli/assets/web/dev.loading.html`
- `packages/cli/assets/web/prod.index.html`

**问题**：默认模板引入了 `https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap` 和 `https://fonts.googleapis.com/css2?family=Fira+Mono:wght@400;500;700&display=swap`，在中国网络环境下该请求加载缓慢或无法加载，导致页面加载超时。

**处理方式**：
- 删除 Google Fonts @import 语句
- 将 font-family 从自定义字体改为系统字体
  - Inter → `system-ui, -apple-system, "Segoe UI", Roboto, sans-serif`
  - Fira Mono → `system-ui, -apple-system, "Segoe UI", Roboto, "Courier New", monospace`
- 在三个模板中添加 `lang="zh-CN"` 属性

### Android 自定义 res_dir 支持

**修改文件**：
- `packages/cli/src/build/android.rs` 第 296 行起（`build_android_app_dir` 方法中的 res_dir 分支）
- `packages/cli/src/build/android.rs` 第 318 行起（`write_default_res` 方法）
- `packages/cli/src/config/manifest.rs` 第 631-636 行（`AndroidConfig.res_dir` 字段，已存在）

**问题**：CLI 硬编码默认 Android 图标（`include_bytes!`），不支持自定义 Adaptive Icon 或其他 res 资源。`[bundle] icon` 字段仅对 Windows 生效，Android 完全忽略。

**处理方式**：
- 在 `AndroidConfig` 中启用已有的 `res_dir: Option<PathBuf>` 字段
- `build_android_app_dir` 中，若 `[android] res_dir` 有值：
  - 跳过整个默认 res 写入（图标、values、xml 等）
  - 从自定义 res 目录递归复制到构建输出的 `res/`
  - 自定义 res 目录不存在时 `bail!` 报错（而非静默跳过）
- 若 `res_dir` 未配置，行为与原版一致
- 将默认 res 写入逻辑提取为 `write_default_res` 方法

**Dioxus.toml 用法**：
```toml
[android]
min_sdk = 26
res_dir = "android/res"   # 相对于 Dioxus.toml 所在目录
```

**自定义 res 目录需包含的必要文件**：
- `values/strings.xml`（app_name）
- `values/colors.xml`
- `values/styles.xml`
- `xml/network_security_config.xml`
- `drawable/ic_launcher_foreground.xml`（自定义前景）
- `drawable/ic_launcher_background.xml`（自定义背景）
- `mipmap-anydpi-v26/ic_launcher.xml`（Adaptive Icon 定义）

**注意**：配置 `res_dir` 后，用户必须完整提供所有必要资源文件，CLI 不再生成任何默认 res。

### 清理 0 字节占位文件

**修改文件**：
- `packages/cli/assets/android/gen/app/src/main/kotlin/.gitignore`（删除）
- `packages/manganis/manganis-macro/assets/asset.txt`（删除）
- `packages/manganis/manganis/assets/style.css`（删除）

**说明**：三个文件均为 0 字节占位/测试夹具，无功能引用。

## Android/Termux 交叉编译 dioxus-cli (aarch64-linux-android)

> 本节只涉及「把 dx 本身编译到安卓、在 Termux 里运行」这一可选目标。日常编译项目（dx 跑在宿主、产出安卓 app）不涉及本节任何配置——宿主版 CLI 的 git2/OpenSSL 按宿主目标解析。

### 状态：Ubuntu 待配置

CLI 交叉编译成功后可在 Termux 上运行。

### 编译

```bash
cargo build --release --target aarch64-linux-android -p dioxus-cli
```

### Ubuntu 环境现状

- NDK 27.1.12297006 已装于 `~/Android/Sdk/ndk/27.1.12297006`
- aarch64 目标普通 Rust 项目的编译链接已端到端验证通过
- **待配置**：dioxus-cli 依赖 git2 + OpenSSL，需要先构建 aarch64-linux-android 的 OpenSSL 3.x 静态库，并通过全局 `~/.cargo/config.toml` 的 `[env]` 段配置 `AARCH64_LINUX_ANDROID_OPENSSL_LIB_DIR` / `AARCH64_LINUX_ANDROID_OPENSSL_INCLUDE_DIR` 等 target-specific 环境变量（Ubuntu 上尚未构建该 OpenSSL 库）

### 关键文件

- `~/.cargo/config.toml` — 镜像源配置；Android OpenSSL 环境变量在待配置项完成后加到这里的 `[env]` 段

## 分支管理

```bash
# 同步官方 v0.7 更新
git fetch origin v0.7
git switch d2736965-patch-v0.7
git rebase v0.7

# 修改后更新分支顶部提交
git add -A && git commit --amend --no-edit

# 安装更新后的 dioxus-cli
cargo install --path packages/cli --force
```
