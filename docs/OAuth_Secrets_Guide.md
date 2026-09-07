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

## 二、`SECRETS` 的默认值

如果不使用 OAuth 登录，`SECRETS` 直接用空字符串模板即可，也能正常编译出 APK（App 内使用 PAT / SSH 登录）：

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
2. 回调 URL：`gitsync://auth`；Webhook 可留空；Permissions 给足仓库读写权限
3. **Client ID** → 填 `gitHubAppClientId`
4. Settings 里 **Client secrets → Generate** 的 secret → 填 `gitHubAppClientSecret`

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

需要签名 APK 时将以下 3 个 Secret 一起配好（配合 `RELEASE_KEYSTORE_BASE64`）：
- `RELEASE_KEYSTORE_BASE64`：keystore 文件的 base64（如 `base64 -w0 gitsync-release.jks` 的输出）
- `RELEASE_SIGNING_ALIAS`：keystore 别名
- `RELEASE_SIGNING_PASSWORD`：keystore 密码

未配置签名会得到未签名 APK（工作流仍会将 `*-release.apk` 上传到 `apk-build` artifact）。

## 五、触发编译

1. 推送分支：`git push -u origin <分支>`
2. 仓库 **Actions → “Release OSS (APK)” → Run workflow**（Branch 选对应分支）
3. 或推送 `v*` 标签（如 `v1.0.0`）自动触发
4. 成功后，在运行页 **Artifacts** 下载 `apk-build`（含 armeabi-v7a / arm64-v8a / x86_64 三个 `*-release.apk`）
   配置了签名时还会生成带签名的 GitHub Release 附件