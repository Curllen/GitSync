# GitHub Actions Secrets 配置指南

本仓库通过 `.github/workflows/generate-apk-release.yml` 在 GitHub Actions 上编译出 APK。
编译时需要用 `lib/constant/secrets.dart` 生成 OAuth 配置文件，该文件内容由仓库级 Secret `SECRETS` 提供。

## 一、Secrets 配置到哪

在仓库页 **Settings → Secrets and variables → Actions → Secrets**（Repository secrets）中新增：

| Secret 名称 | 说明 | 是否必须 |
|---|---|---|
| `SECRETS` | `lib/constant/secrets.dart` 的**整体内容** | 是 |
| `RELEASE_KEYSTORE_BASE64` | 发布签名 keystore 的 base64 | 否（想得到签名 APK 才需要） |
| `RELEASE_SIGNING_ALIAS` | 签名别名 | 否 |
| `RELEASE_SIGNING_PASSWORD` | 签名密码 | 否 |

> 不要配置为 Environment secrets，也不要放进 Repository variables——工作流使用 `secrets.*`，且未声明 `environment:`。

## 二、`SECRETS` 的含义

`SECRETS` 的值就是 **`lib/constant/secrets.dart` 的完整内容**（8 行 `const` 声明）。工作流会把这段内容原样写入构建时的 `secrets.dart`，App 初始化时引用这些常量来决定**内置的一键 OAuth 登录按钮**能否使用。

> 关键澄清：`SECRETS` **不决定**“能连哪些用户/仓库”。无论填不填，用户都可用 HTTPS(用户名+token) / SSH 登录并管理自己有权访问的仓库；多仓库、LFS、Git 过滤器等解锁功能也不受影响。它只影响 App 里 GitHub / Gitea / GitLab / Codeberg 那几个 **OAuth 登录按钮**是否可点。

### 2.1 每个常量的何时需值 / 何时可空

| 常量 | 作用（App 内对应功能） | 何时需要值 | 何时可为空 |
|---|---|---|---|
| `oauthRedirectUrl` | 预留的 OAuth 回调地址 | **暂不需要**：源码当前未引用该常量 | 始终为空 |
| `gitHubClientId`<br>`gitHubClientSecret` | GitHub **OAuth** 登录按钮 | 想让你用户能点 GitHub 按钮一键登录时 | 用不到 OAuth 时可空 |
| `gitHubAppClientId`<br>`gitHubAppClientSecret` | GitHub **App**（免 token 安装式）登录按钮 | 想让你用户能用 GitHub App 方式授权登录时 | 不用此登录项时可空 |
| `giteaClientId` | Gitea OAuth 登录按钮 | 想让用户能一键登录自建 Gitea 时（需有自建实例） | 用不到时可空 |
| `gitlabClientId` | GitLab OAuth 登录按钮 | 想让用户一键登录 GitLab 时 | 用不到时可空 |
| `codebergClientId` | Codeberg OAuth 登录按钮 | 想让用户一键登录 Codeberg 时 | 用不到时可空 |

**一句话规则：**
- 只想得到可用的 APK → **8 个都填空字符串**（编译通过，用户用 HTTPS/SSH 登录）。
- 想让某些**平台的一键 OAuth 登录按钮**可用 → 只填对应平台的 `ClientId`/`ClientSecret`（须在[第三节](#三各字段的申请位置与填法)各平台注册自己的 OAuth 应用），其余保持空串。
- 8 个常量**缺一不可**：`secrets.dart` 必须含全部 8 个声明，空文件或缺任一常量会导致编译报错。

### 2.2 空字符串模板（可直接作为 `SECRETS` 值）

如果不使用 OAuth 登录，`SECRETS` 直接用空字符串模板即可，也能正常编译出 APK（App 内使用 HTTPS / SSH 登录）：

```dart
const oauthRedirectUrl = "";
const gitHubClientId = "";
const gitHubClientSecret = "";
const gitHubAppClientId = "";
const gitHubAppClientSecret = "";
const giteaClientId = "";
const gitlabClientId = "";
const codebergClientId = "";
```

> 所有 OAuth 平台的回调地址统一填 `gitsync://auth`（App 通过自定义协议 `gitsync://` 回跳）。

## 三、各字段的申请位置与填法

### `oauthRedirectUrl`
当前代码中并未引用该常量（仅存在于模板），可一直留空，无需申请。

### `gitHubClientId` / `gitHubClientSecret`（GitHub OAuth 应用，App 内 GitHub OAuth 登录）
1. GitHub → 右上头像 → **Settings → Developer settings → OAuth Apps → New OAuth App**
2. Application name 任意；Homepage URL 任意；**Authorization callback URL：`gitsync://auth`**
3. 创建后 **Client ID** → 填 `gitHubClientId`
4. 点 *Generate a new client secret* 得到的 **Client secret** → 填 `gitHubClientSecret`

### `gitHubAppClientId` / `gitHubAppClientSecret`（GitHub App，免 token 安装式登录）
1. 开发者后台 → **GitHub Apps → New GitHub App**（[创建地址](https://github.com/settings/apps/new)）
2. 回调 URL：`gitsync://auth`；**Webhook Active 可关掉**（GitSync 用不到 webhook）
3. 按下述“权限勾选”设置 Permissions
4. **Client ID** → 填 `gitHubAppClientId`
5. Settings 里 **Client secrets → Generate** 的 secret → 填 `gitHubAppClientSecret`

#### GitHub App 权限勾选（依据 GitSync 实际调用接口，见 `lib/api/manager/auth/github_manager.dart`、`github_app_manager.dart`）

**必选：**

| 界面行 | 官方权限 | 建议 | 用途 |
|---|---|---|---|
| Contents | Contents | **Read & write** | clone/fetch/push、列仓库、读分支 |
| Issues | Issues | **Read & write** | 议题/评论/标签/表情、issue 模板 |
| Pull requests | Pull requests | **Read & write** | PR 列表、PR 文件变动 |
| Metadata | Metadata | 只读（强制） | 基础仓库元信息 |

**推荐（可选）：**

| 界面行 | 官方权限 | 建议 | 用途 |
|---|---|---|---|
| 工作流程 / Workflows, workflow runs and artifacts | Actions | **Read** | 显示工作流运行记录 |
| Administration | Administration | **Read** | 显示协作者列表 |

**其余**（Checks、Codespaces、Dependabot、Secrets、Variables、Webhooks、Deployments、Discussions、Pages、Projects、Merge queues、Security advisories 等）GitSync 均未调用，保持 **None** 避免过度授权。

**安装仓库（关键）：** 创建后把 App **安装到你想要 GitSync 管理的仓库**（Install → 选仓库或 “All repositories”）。`github_app_manager` 通过 `/user/installations` 查找已安装仓库，未安装任何仓库将列不出仓库。

### `giteaClientId`（Gitea OAuth）
1. 在自建 Gitea 实例：右上头像 → **设置 → 应用 → 管理 OAuth2 应用 → 创建新应用**
2. 重定向 URI：`gitsync://auth`
3. 创建后 **Client ID** → 填 `giteaClientId`（代码只用 id，secret 不填）

### `gitlabClientId`（GitLab OAuth）
1. GitLab → 头像 → **Preferences/Settings → Applications**（直链：[`gitlab.com/-/user_settings/applications`](https://gitlab.com/-/user_settings/applications)）
2. Redirect URI：`gitsync://auth`；Scopes 建议勾 `api`（或 `write_repository` + `read_repository`）
3. **Application ID** → 填 `gitlabClientId`（代码只用 id）

### `codebergClientId`（Codeberg OAuth）
1. Codeberg → 右上头像 → **Settings → Applications → 新建 OAuth2 应用 / 创建令牌**
2. Redirect URI：`gitsync://auth`；勾选仓库读写权限
3. **Client ID** → 填 `codebergClientId`

## 四、发布签名（可选）

需要**已签名** APK 时，先自行生成签名 keystore，再配置以下 3 个 Secret：
- `RELEASE_KEYSTORE_BASE64`：keystore 文件的 base64（keystore 的 base64 内容）
- `RELEASE_SIGNING_ALIAS`：keystore 别名（默认 `gitsync`）
- `RELEASE_SIGNING_PASSWORD`：keystore 密码

未配置签名会得到未签名 APK（工作流仍会将 `*-release.apk` 上传到 `apk-build` artifact）。

### 4.1 用 keytool 生成 keystore（Linux/macOS）

```bash
mkdir -p signing && cd signing

# 生成一个随机密码
PASS=$(openssl rand -base64 24 | tr -dc 'A-Za-z0-9' | head -c 24)

# 生成 keystore（别名 gitsync，RSA 2048，有效期 10000 天）
keytool -genkeypair -v -keystore gitsync-release.jks -alias gitsync \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -storepass "$PASS" -keypass "$PASS" \
  -dname "CN=GitSync OSS, OU=OSS Build, O=GitSync, L=Internet, ST=Internet, C=US"

# 打印并保存密码（务必自行备份，丢了将无法更新已发布应用）
echo "$PASS" > keystore_pass.txt
echo "你的密码：$PASS"
```

### 4.2 生成 base64 并配置

```bash
# 得到 RELEASE_KEYSTORE_BASE64 的值（keystore 的 base64 内容）
base64 -w0 signing/gitsync-release.jks

# 得到 RELEASE_SIGNING_PASSWORD 的值
cat signing/keystore_pass.txt
```

- `RELEASE_SIGNING_ALIAS`：`gitsync`
- `RELEASE_SIGNING_PASSWORD`：`keystore_pass.txt` 内容
- `RELEASE_KEYSTORE_BASE64`：上面 base64 的**整段输出**（内容可能很长，粘贴完整）

### 4.3 安全提醒

- `signing/` 已加入 `.gitignore`，**不要提交** keystore 与密码到 Git。
- 若 keystore 曾进入过 git 历史（即使未推送），应视为已泄露，请按 4.1 重新生成新 keystore。

## 五、触发编译

1. 推送分支：`git push -u origin <分支>`
2. 仓库 **Actions → “Release OSS (APK)” → Run workflow**（Branch 选对应分支）
3. 或推送 `v*` 标签（如 `v1.0.0`）自动触发
4. 成功后，在运行页 **Artifacts** 下载 `apk-build`（含 armeabi-v7a / arm64-v8a / x86_64 三个 `*-release.apk`）
   配置了签名时还会生成带签名的 GitHub Release 附件