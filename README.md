# Burrow

**Burrow** 是 [Mole](https://github.com/tw93/Mole) 命令行的非官方 macOS 桌面外壳。它把 Mole 的深度清理、卸载、优化、磁盘分析、项目产物清理、安装包清理和应用瘦身能力放进一个原生 SwiftUI 应用，并内置终端来处理需要交互或输入密码的步骤。

- 原生 macOS 应用，SwiftUI + SwiftTerm
- 通用二进制：Apple Silicon（arm64）与 Intel（x86_64）
- 内置 Mole 1.67.0 运行时，开箱即用
- 支持从 GitHub 更新 Mole 引擎，SHA-256 校验后再启用
- 支持稳定版与每日构建（main）通道
- 原生预览、原生进度、原生结果页：执行时不再显示终端画面
- 中英文界面，跟随系统语言或手动切换
- GPL-3.0 自由软件

> Burrow 与 Mole 项目没有隶属关系，也未获其背书。Mole 的名称和图标是 Mole 项目的商标；Burrow 使用自己的名称和图标。详见 [NOTICE](NOTICE) 和上游 [TRADEMARK.md](https://github.com/tw93/Mole/blob/main/TRADEMARK.md)。

---

## 系统要求

- macOS 14 Sonoma 或更高版本
- 推荐 Apple Silicon；Intel Mac 也支持
- 首次运行不需要安装 Mole，也不需要 Homebrew
- 应用未启用 App Sandbox（与 Mole CLI 的工作方式一致）。某些目录需要“完全磁盘访问权限”才能完整扫描

## 安装

构建完成后，安装包位于：

```text
dist/Burrow-2.5.dmg
dist/Burrow-2.5.zip
dist/Burrow.app
```

1. 打开 DMG。
2. 将 **Burrow.app** 拖到“应用程序”文件夹。
3. 首次启动时，由于当前构建是 ad-hoc 签名且未经过 Apple 公证，macOS 可能提示无法验证开发者。请按住 Control 点击 Burrow，选择“打开”，再确认一次。
4. 如果仍被拦截，可在终端执行：

   ```bash
   xattr -dr com.apple.quarantine /Applications/Burrow.app
   ```

应用会使用自己的 Mole 运行时，不会修改 Homebrew 安装的 Mole，也不会向 `/usr/local` 写入文件。

## 第一次使用

- 左侧“概览”会显示系统健康度、CPU、内存、磁盘和最近操作。
- 每个工具都是同一条流程：**扫描 → 原生清单 → 勾选 → 后台执行 → 原生结果**。执行期间看到的是进度条、当前条目和逐条结果，不是终端画面。
- 破坏性命令都提供预览。请先点“预览（安全）”，检查候选项后再运行。
- Mole 在后台私有会话中运行，确认提示会自动应答；需要管理员密码时会弹出密码框，也可以选择“跳过管理员任务”。
- 结果页的“已清理 / 已保留 / 失败”来自 Mole 的审计日志 `~/Library/Logs/mole/operations.log`，不是从终端输出里猜出来的。
- 需要看原始输出时，结果页和进度页都有“技术细节”；也可以点“在终端中打开”回退到完整终端。
- “实时状态”会每几秒刷新一次；也可以打开 Mole 自带的 TUI 状态面板。
- “操作历史”读取 `~/Library/Logs/mole/operations.log` 对应的 JSON 历史。
- “设置”中可以切换语言、更新通道、登录启动、运行时保留数量和诊断工具。

## 已知限制

**“应用瘦身”的扫描很慢，这是 Mole 自身的成本，不是界面卡住。**

`slim --dry-run` 在给出候选列表之前，会对每个已安装应用做两件重活：

1. `slim_skip_reason` 用 `find "$app" ! -user <你> -print -quit` 递归遍历整个应用包，确认里面没有别人的文件；
2. `slim_plan_app` 对 `Contents` 下每个文件跑 `file`，挑出通用二进制，再逐层校验代码签名封条。

实测（115 个应用，2026-11-01）：前 10 个应用就用了 185 秒——其中 CLion 单个应用 `slim_skip_reason` 46.0 秒、`slim_plan_app` 89.5 秒；`codesign` 取签名团队只要 0.2 秒，不是瓶颈。按 18.5 秒/应用外推，全量扫描约需 **35 分钟**。

Burrow 的处理方式：

- 扫描页会显示已用时间和一句说明，而不是只转圈；
- 提供“停止扫描”，点了会真的杀掉子进程（不是只关界面）；
- 内置 `--selftest` 在跑全量时**跳过 slim**，并在日志里写明跳过原因与影响；需要覆盖它时单独跑 `--selftest=slim` 并耐心等待。

不修改 Mole 运行时来“优化”这一步：运行时是按版本下载并用 SHA-256 校验的官方产物，改它会让升级失效。

Mole 数据位置：

```text
~/Library/Logs/mole/           日志
~/.config/mole/                Mole 设置
~/Library/Application Support/Burrow/Runtimes/   下载的运行时
~/Library/Application Support/Burrow/active-runtime.json   当前运行时指针
```

## 更新机制

Burrow 的“应用本体”和“Mole 引擎”是分开的：

- **应用本体**：从本仓库构建；升级应用需要安装新的 Burrow.app。
- **Mole 引擎**：在应用内检查并更新，不需要重新安装应用。

稳定版更新流程：

1. 请求 GitHub API 获取 `tw93/Mole` 最新 release。
2. 下载对应版本的源码包和 `binaries-darwin-<arch>.tar.gz`。
3. 下载 `SHA256SUMS`，校验二进制包的 SHA-256。
4. 组装运行时，重命名辅助二进制为 `bin/analyze-go`、`bin/status-go`。
5. 执行 `mole --version` 和 `mole status --json` 验证。
6. 校验通过后原子地切换 `active-runtime.json`。
7. 旧运行时可以回滚，也可以按保留数量自动清理。

每日构建（main 分支）：

- 通过 GitHub API 解析 main 最新 commit。
- 下载该 commit 的源码。
- 使用本机 Go 编译 `analyze` 和 `status` 辅助二进制。
- 因此 **每日构建需要本机安装 Go**：

  ```bash
  brew install go
  ```

更新失败时当前运行时保持不变。设置中提供“回滚到上一个运行时”“恢复内置版本”“导入运行时”和“清理旧运行时”。

GitHub API 未认证请求有速率限制；如果遇到限制，稍后再试即可。所有下载和校验逻辑位于 `app/Sources/Burrow/UpdateManager.swift`。

## 从源码构建

### 依赖

- macOS 14+
- Xcode 16 或更新版本（Swift 6 工具链）
- Go 1.26 或更新版本（构建 Mole 辅助二进制、每日构建）
- Xcode Command Line Tools（`lipo`、`iconutil`、`hdiutil`）
- 网络连接（首次构建需要下载 SwiftPM 依赖和 Mole 源码）

当前开发环境使用 Xcode 27.2、Swift 6.4、Go 1.27.3 构建通过。

### 构建全部产物

```bash
# 1. 构建固定的 Mole 运行时（arm64 + x86_64 辅助二进制）
./scripts/build-runtime.sh

# 2. 构建通用 Burrow.app，组装运行时并 ad-hoc 签名
./scripts/build-app.sh

# 3. 生成 DMG 和 ZIP
./scripts/make-dmg.sh
```

产物：

```text
vendor/runtime/1.67.0/     构建出的 Mole 运行时
vendor/runtime/current -> 1.67.0
dist/Burrow.app
dist/Burrow-2.5.dmg
dist/Burrow-2.5.zip
```

只构建当前架构：

```bash
ARCHS=arm64 ./scripts/build-app.sh
```

### 开发运行

```bash
BURROW_RUNTIME="$PWD/vendor/runtime/current" \
  swift run --package-path app
```

`BURROW_RUNTIME` 是开发时的覆盖变量；正式应用会优先使用 `Contents/Resources/Runtime`。

### 重新生成图标

```bash
./scripts/make-icon.swift build/AppIcon.iconset
iconutil -c icns build/AppIcon.iconset -o build/AppIcon.icns
```

图标是 Burrow 自己的隧道/洞穴图形，不使用 Mole 的名称或图标。

## 项目结构

```text
app/
  Package.swift                 SwiftPM 清单
  Resources/Info.plist          App 元数据
  Sources/Burrow/
    BurrowApp.swift             应用入口、菜单、设置场景
    AppDelegate.swift           点关闭按钮即退出（不留后台进程）
    RootView.swift              侧边栏与页面路由
    OverviewView.swift          概览仪表盘
    StatusView.swift            实时系统状态
    HistoryView.swift           操作历史
    ActionDetailView.swift      工具说明、预览和运行入口
    ToolRunView.swift           原生扫描 / 勾选 / 进度 / 结果界面
    ToolRunner.swift            PTY 驱动、扫描器、菜单行走、结果汇总
    RunActivity.swift           把 Mole 输出解析成结构化进度行
    OperationsLog.swift         读取审计日志，生成原生结果统计
    InstallerScanner.swift      原生安装包扫描（替代 TUI 扫描）
    AnalyzeBrowserView.swift    原生磁盘浏览器（mo analyze --json）
    TerminalView.swift          SwiftTerm 内嵌终端（兜底与自由终端）
    SettingsView.swift          设置、更新、运行时、诊断
    AboutView.swift             关于、许可证、组件
    AppModel.swift              应用状态和动作
    Runtime.swift               运行时发现、激活、导入、清理
    UpdateManager.swift         GitHub 更新、校验、安装
    ProcessRunner.swift         无 shell 子进程与 JSON 调用
    Models.swift                数据模型、版本和格式化
    L10n.swift                  中英文文案
    Theme.swift                 颜色、卡片、按钮等视觉组件
scripts/
  build-runtime.sh              固定 Mole 运行时构建
  build-app.sh                  通用 app 构建与签名
  make-dmg.sh                   DMG / ZIP 打包
  make-icon.swift               图标生成
vendor/
  SwiftTerm/                    内嵌终端组件（MIT）
  runtime/                      构建产物（不提交到源码仓库）
mole-upstream/                  上游源码参考（开发用）
LICENSE                         GPL-3.0
NOTICE                          上游、商标和第三方声明
```

## 安全说明

Mole 可以删除文件，Burrow 不会改变这一点：

- 先预览，再运行。
- 重要数据请提前备份。
- 不要用 `sudo` 运行 Burrow 或 Mole。
- Mole 只在需要时通过系统授权对话框请求管理员权限。
- 应用未启用 App Sandbox，因为 Mole 需要像 CLI 一样检查系统。请只从可信来源获取应用。
- 某些清理位置需要“完全磁盘访问权限”；设置中的诊断页可以打开隐私设置。

应用不会把密码写入日志。密码输入发生在内嵌终端或系统授权对话框中。

## 许可证与上游

Burrow 以 **GNU General Public License v3.0** 发布，完整文本见 [LICENSE](LICENSE)。

- 上游 Mole：<https://github.com/tw93/Mole>
- 固定版本：`V1.67.0`
- 内置源码包：`Contents/Resources/Runtime/SOURCE.tar.gz`
- 内置源码包 SHA-256：`f263992189c33e54dba12081e3455b1f25078e4e9806e5d6a1df67f7fa552645`
- 上游许可证：GPL-3.0
- 内嵌终端：SwiftTerm，MIT

Burrow 对 Mole 的修改仅限于打包、SwiftUI 外壳、运行时/更新管理器和集成代码；Mole 本身不被 Burrow 修改（组装运行时时会重命名辅助二进制）。

如果你重新分发修改版，请保留 `LICENSE`、`NOTICE`、Mole 源码包和 SwiftTerm 许可证，并遵守上游商标政策使用不同的名称和图标。

## 常见问题

**应用提示“已损坏”或无法打开**

这是未公证的 ad-hoc 签名构建。按住 Control 点击应用选择“打开”，或执行：

```bash
xattr -dr com.apple.quarantine /Applications/Burrow.app
```

**检查更新失败**

确认能访问 `api.github.com` 和 `github.com`。未认证 GitHub API 有速率限制；稍后再试，或在浏览器中查看 [Releases](https://github.com/tw93/Mole/releases)。

**每日构建失败**

确认已安装 Go，并且 `go version` 可用。每日构建会在本机编译辅助二进制，第一次可能需要下载 Go 模块。

**状态页显示错误**

在“设置 → 诊断”中运行诊断，确认运行时文件和辅助二进制可执行。也可以在终端中运行：

```bash
BURROW_RUNTIME="$PWD/vendor/runtime/current" ./vendor/runtime/current/mole status --json
```

**想换回内置 Mole**

设置 → 运行时 → “恢复内置版本”。下载的运行时仍保留在 Application Support 中，可手动清理或使用“清理旧运行时”。

## 致谢

- [Mole](https://github.com/tw93/Mole) — 出色的 macOS 清理与优化 CLI
- [SwiftTerm](https://github.com/migueldeicaza/SwiftTerm) — Swift 终端组件
- SwiftUI、AppKit 与 macOS 开发者工具

Burrow 是一个独立项目，与 Mole 项目没有隶属关系。
