# GitSync 定制文档

本分支（`unlock-premium`）定制内容与构建说明。

## 文档列表

- [CHANGES_Premium_Unlock.md](CHANGES_Premium_Unlock.md) — **修改说明**：Premium 解锁的具体改动点与功能说明、改动文件清单、打包流程。
- [OAuth_Secrets_Guide.md](OAuth_Secrets_Guide.md) — **配置文档**：GitHub Actions 的 Secrets 配置、各 OAuth 凭据申请位置、如何触发编译。

## 快速开始

1. 按 [OAuth_Secrets_Guide.md](OAuth_Secrets_Guide.md) 在 **Repository secrets** 配置 `SECRETS`（及可选签名项）。
2. `git push -u origin unlock-premium`
3. 仓库 **Actions → “Release OSS (APK)” → Run workflow**。
4. 成功后在运行页 Artifacts 下载 `apk-build` 中的 `*-release.apk`。

详细改动见 [CHANGES_Premium_Unlock.md](CHANGES_Premium_Unlock.md)。