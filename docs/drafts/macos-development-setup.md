# macOS 本地开发环境搭建与构建排障

本文补充 [README 开发者指南](../../README.md#开发者指南)，记录在 Apple Silicon macOS 上搭建 Reveric Debug 环境、首次构建和启动时需要处理的原生工具链与网络问题。正常流程仍以仓库的 `Makefile` 和 `.fvmrc` 为准。

## 现有文档覆盖范围

README 已列出 `make deps`、`make codegen` 和 `make run-macos` 的基础顺序；[Codemode 方案](../solutions/ai-chat-codemode.md)记录了 FJS 的 Rust 1.97.1 recipe，[插件验证说明](../solutions/plugin-system/verification.md)列出本机 FJS 动态库的构建入口。这些内容没有说明完整 Xcode 的选择、许可与首次初始化，也没有覆盖 FVP 的准备、Cargo 代理与 shell 代理不一致、依赖解析改写锁文件等首次搭建问题，因此补充本 runbook。

## 快速开始

需要完整 Xcode、Homebrew、FVM、Rustup，以及能够访问 pub.dev、GitHub 和 Rust crate 源的网络。当前代码通过 FVM 固定 Flutter commit；不要用 PATH 中的裸 `flutter` 执行本地项目命令。

```sh
# 首次安装 FVM；已安装时跳过
brew install fvm

# 安装 .fvmrc 指定的 Flutter SDK，并让项目使用它
fvm install
fvm flutter --version

# 从仓库根目录运行
make deps
make codegen
make run-macos
```

`make run-macos` 会依次准备 FJS、FVP 原生依赖，再执行 `fvm flutter run -d macos`。首次构建可能需要下载并编译多个原生依赖，时间会明显长于之后的增量构建。

## Xcode 检查与首次初始化

确认活动开发目录指向完整 Xcode，而不是仅有编译工具的 Command Line Tools：

```sh
xcode-select -p
xcodebuild -version
xcodebuild -license check
```

如果机器已安装 Xcode，但 `xcode-select -p` 显示 `/Library/Developer/CommandLineTools`，切换到实际安装路径。以下以 `/Applications/Xcode.app` 为例：

```sh
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -license
xcodebuild -runFirstLaunch
```

许可协议需要由本机用户阅读并接受。`xcodebuild -runFirstLaunch` 会安装 Xcode 首次运行所需组件。若 `xcodebuild -create-xcframework` 报 `A required plugin failed to load`、`IDESimulatorFoundation` 符号缺失，或提示运行 `-runFirstLaunch`，先完成这一步，再重试原命令。

检查结果：本次环境最初选中了 Command Line Tools；切换至 Xcode 后发现许可尚未接受。接受许可并运行首次初始化后，FVP 的 `xcodebuild -create-xcframework` 成功。首次初始化还输出过 CoreSimulator 旧服务版本提示，随后清理旧服务并继续。

## 网络代理分层

常规 HTTP 下载可按本机代理设置导出环境变量，例如：

```sh
export https_proxy=http://127.0.0.1:7897
export http_proxy=http://127.0.0.1:7897
export all_proxy=socks5://127.0.0.1:7897
```

这能覆盖 FVM、curl 和多数 HTTP 下载。Git 与 Cargo 使用各自的配置；可分别检查：

```sh
git config --show-origin --get-regexp 'http\..*proxy|https\..*proxy'
cat "$HOME/.cargo/config.toml"
```

本次机器的 Git 已走 `127.0.0.1:7897`，但 `~/.cargo/config.toml` 的 Cargo 代理仍指向未运行的 `127.0.0.1:9981`。FJS 构建因此在下载 crates 时失败，并报告无法连接该地址。直接调用 FJS 脚本时，`CARGO_HTTP_PROXY=http://127.0.0.1:7897` 可以覆盖 Cargo 设置；但 Flutter 的 macOS 原生 build hook 由 Xcode 启动，hook 进程没有继承 shell 导出的 Cargo 代理变量。此时只 export 环境变量不足以修复 Xcode 内的 Rust 构建。

如果 hook 仍连接旧端口：

1. 备份 `~/.cargo/config.toml`。
2. 将 `[http]` 下的 `proxy` 更新为当前可用的 HTTP 代理 URL；若文件中也有 `[https] proxy`，同步更新。
3. 保持代理可用，重试 `make run-macos` 或 `make build-macos`。
4. 构建完成后，如不希望保留该全局设置，将备份恢复到原路径。

Cargo 的 HTTP 代理值应使用 `http://主机:端口` 形式，即使目标地址是 HTTPS。不要把代理端口固定写进项目配置；代理地址属于本机网络环境。

## 首次原生构建会做什么

| 步骤 | 仓库入口 | 首次运行行为 |
| --- | --- | --- |
| FJS 插件运行时 | `third/fjs/tool/build_runtime.sh` | 校验本机归档；缺失或与源码、Rust/Xcode 工具链不匹配时，从本地补丁源码生成 XCFramework。Rust recipe 固定为 1.97.1，目标包含 `aarch64-apple-ios`、`aarch64-apple-ios-sim` 和 `aarch64-apple-darwin`。Rustup 会按需安装工具链和 target。 |
| FVP 视频运行时 | `third/fvp/macos/setup.sh` | 下载 MDK 0.39.0 macOS SDK，校验 SHA-256，准备 framework 和 SwiftPM XCFramework。 |
| macOS Swift 包 | Flutter/Xcode 构建 | 根据 `macos/Runner.xcworkspace/xcshareddata/swiftpm/Package.resolved` 获取锁定的 CocoaAsyncSocket 等依赖。 |

首次 FJS 编译会下载 Cargo Git 依赖和 crates，并对三个 Apple arm64 target 做 Release 编译；它可能长时间没有 Flutter 层输出。确认进程仍在运行后再判断是否卡住，不要只因终端暂时安静就中断。FJS 的最终归档会校验源码/工具链身份和 checksum；MDK 下载也会校验仓库脚本中固定的 SHA-256。

## 依赖解析和代码生成差异

本次 `make deps` 成功，但 pubspec 解析重写了 `pubspec.lock` 中 267 个依赖项；hosted 包 URL 从 `pub.flutter-io.cn` 转为 `pub.dev`，多个版本也随 SDK 兼容性更新。之后 `make codegen` 成功完成，运行约 92 秒，并对 `ai_providers.g.dart` 产生了 Riverpod hash 变化。`fvm install` 还移除了 `.fvmrc` 末尾换行，形成纯格式 diff。生成过程提示 `--delete-conflicting-outputs` 已移除并被忽略，但 build_runner 仍以退出码 0 完成；Drift 和 JSON 序列化的既有警告没有阻止生成。

安装后先检查：

```sh
git status --short
git diff -- .fvmrc pubspec.lock
git diff -- 'lib/**/*.g.dart' 'lib/**/*.drift.dart'
```

只有在依赖升级或生成变化属于当前任务时，才保留并提交这些差异。不要把环境安装带来的锁文件更新或单独的生成 hash 变化悄悄混入无关改动；恢复前先确认差异中没有自己的业务改动。

## 构建与启动验证

```sh
make run-macos
```

Debug app 产物位于：

```text
build/macos/Build/Products/Debug/Reveric.app
```

若需要在构建完成后单独打开：

```sh
open -a "$PWD/build/macos/Build/Products/Debug/Reveric.app"
```

确认应用进程和 Debug 启动日志：

```sh
ps -Ao pid,etime,%cpu,command | rg 'Reveric.app/Contents/MacOS/Reveric'
tail -80 "$HOME/Library/Application Support/ai.reveric.debug/logs/latest.log"
```

Debug 数据库和日志位于 `~/Library/Application Support/ai.reveric.debug/`。首次启动会初始化 Debug 数据库和演示作品；这不是 Release 数据目录。

## 本次验证记录

2026-11-01 在 Apple Silicon macOS 26.6.2 上验证，工具链为 FVM 4.3.2、仓库锁定的 Flutter commit `0b771a584214b44aa1f94f9707f2756207356f41`（Flutter 3.50.0-0.3.pre / Dart 3.14.0）、Xcode 27.2、FJS Rust 1.97.1。`make deps`、`make codegen`、FJS/FVP 原生准备和 macOS Debug 构建均完成；应用进程保持运行，启动日志显示数据库、默认初始化任务和首次使用流程已加载。

README 仍是推荐入口；本文记录的是原生构建准备和本次排障经验，不替代跨平台构建说明。
