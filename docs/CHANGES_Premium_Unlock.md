# GitSync Premium 解锁 & APK 打包修改说明

本仓库基于 GitSync 深度定制：**移除全部 Premium（会员）限制**，让多仓库、Git LFS、Git 过滤器等原本需要订阅的功能免费可用；同时改进了 GitHub Actions 打包流程，可一键编译出 APK。

分支：`unlock-premium`

---

## 一、功能说明（解锁了什么）

| 功能 | 说明 |
|---|---|
| 管理多个仓库 | 可同时添加、切换、同步多个仓库，不再限制为单仓库 |
| Git LFS | 完整支持 Git Large File Storage，可同步含大二进制文件的仓库 |
| Git 过滤器 | 支持 .gitattributes 中的 filter/diff/merge（含 git-lfs、git-crypt 等） |
| 子模块 | 打开含子模块的仓库不再弹出付费墙，直接加载 |
| 设置导入/导出 | 多仓库环境下导入/导出设置不再需要 Premium |
| 移除商店引导横幅 | 不再出现“此仓库使用了仅商店版本支持的过滤器，点击打开商店”的引导条 |

> 说明：解锁的关键在于让 App 的 `hasPremium == true` 恒成立。上述功能的实际运行逻辑（libgit2）原本就已支持，此前只是被会员校验挡住了 UI 入口，因此解锁后即可直接使用。

---

## 二、具体修改点

### 1. 全局会员状态恒为通过
文件：[lib/api/manager/premium_manager.dart](file:///workspace/lib/api/manager/premium_manager.dart)

- `init()`：去掉联网查询 GitHub Sponsors 的逻辑，直接令 `hasPremiumNotifier.value = true`。
- `_readPremiumStatus()`：恒返回 `true`。

> 这是核心开关。App 内凡是 `hasPremiumNotifier.value != true` 就弹出付费页的入口，都会因该值为 `true` 而不再触发。

### 2. 会员状态不允许被关闭
文件：[lib/providers/riverpod_providers.dart](file:///workspace/lib/providers/riverpod_providers.dart)

- `PremiumStatusNotifier.set()`：入参无论何值，统一强制写为 `true`，避免“清除数据”等操作在会话内重新上锁。

### 3. 首页“添加更多仓库”入口去锁
文件：[lib/main.dart](file:///workspace/lib/main.dart)

- 移除了“添加仓库”按钮点击时的 Premium 校验与解锁弹窗逻辑。
- 移除了依赖会员状态切换的宝石图标和小图，改为恒为用户看得到的“添加/更多”图标。
- 现在首次即可添加多仓库（此前仅会员可多开）。

### 4. 移除“Git 过滤器仅商店版可用”横幅
文件：[lib/main.dart](file:///workspace/lib/main.dart#L4262-L4290)

- 删除了 `hasGitFiltersProvider` 驱动的“点击打开 Play 商店”引导条，Git 过滤器不再被当成受限功能提示。

### 5. 子模块不再拦截付费
文件：[lib/api/helper.dart](file:///workspace/lib/api/helper.dart)

- `setGitDirPathGetSubmodules()`：命中子模块时不再弹解锁页，直接 `addSubmodules()`。
- 顺手清理了不再使用 `unlock_premium.dart` 的 import。

### 6. GitHub Actions 打包流程增强
文件：[.github/workflows/generate-apk-release.yml](file:///workspace/.github/workflows/generate-apk-release.yml)

- **新增**构建完成后无条件上传 APK 步骤，产物即使用户没配签名证书也能拿到。
- **签名改为可选**：新增 `Detect Signing Config` 步骤，通过步骤输出判断是否配置了签名密钥；未配置时自动跳过签名与 GitHub Release 步骤，避免整条流水线失败。
- 修复了 GitHub Actions 语法错误：`step.if:` 不允许直接引用 `secrets` 上下文，改为在 `env` 中读取后输出判断结果。

---

## 三、改动文件清单

| 文件 | 改动类型 |
|---|---|
| [lib/api/manager/premium_manager.dart](file:///workspace/lib/api/manager/premium_manager.dart) | 会员恒为通过 |
| [lib/providers/riverpod_providers.dart](file:///workspace/lib/providers/riverpod_providers.dart) | 禁止关闭会员 |
| [lib/main.dart](file:///workspace/lib/main.dart) | 多仓库去锁、图标调整、移除过滤器横幅 |
| [lib/api/helper.dart](file:///workspace/lib/api/helper.dart) | 子模块解锁、清理 import |
| [.github/workflows/generate-apk-release.yml](file:///workspace/.github/workflows/generate-apk-release.yml) | 构建/签名流程增强 |
| [.gitignore](file:///workspace/.gitignore) | `signing/` 机密不入库 |

---

## 四、配置与打包

详见 [docs/OAuth_Secrets_Guide.md](file:///workspace/docs/OAuth_Secrets_Guide.md)。

要点：
- Secrets 配在 **Repository secrets**（不是 Environment/Variables）。
- 必配 `SECRETS`（`lib/constant/secrets.dart` 内容，可用空串模板）。
- 可选签名：`RELEASE_KEYSTORE_BASE64`、`RELEASE_SIGNING_ALIAS`、`RELEASE_SIGNING_PASSWORD`。
- 触发：Actions → “Release OSS (APK)” → **Run workflow**，或推送 `v*` 标签。
- 产物：未签名 APK 在 `apk-build` artifact；签名后生成 GitHub Release。

---

## 五、安全提醒

- 发布签名 keystore/密码属于机密，**绝不能提交到 Git**。本仓库已在 `.gitignore` 加入 `signing/`。
- 如果签名文件曾进入过 git 历史（即使未推送），该密钥应视为已泄露，请用 `keytool` 重新生成新 keystore 后再用。