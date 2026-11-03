# iphone-client12（CI 打包仓）

本仓库**不存放明文源码**。加密包 `source.zip.gpg` 由 monorepo 里的 `ios-client12` 打包脚本写入；GitHub Actions 解密后用较旧的 Xcode 打 **iOS 12+** 侧载 IPA。

源工程：`ipadump-backend-rust/ios-client12`  
对齐流程：`apple-sign-desktop-action` ← `fast-sign/desktop/scripts/pack_encrypted_source.sh`

| 文档 | 说明 |
|------|------|
| [deploy.md](./deploy.md) | Secrets、加密源码更新、触发构建、Release |
| Releases | 带 tag 构建成功后的 IPA / `latest.json` |

## 最短路径

1. 本机（monorepo）：

   ```bash
   cd ios-client12
   ./scripts/pack_encrypted_source.sh
   ```

2. GitHub：Settings → Secrets → `SOURCE_GPG_PASSPHRASE` = 本机 `.secrets/SOURCE_GPG_PASSPHRASE` 内容

3. 本仓提交 push `source.zip.gpg`（及 `.github/`）→ Actions → **Build IPA**

4. 下载 artifact，或填 Release tag 直接发到 Releases

> 使用本项目或其产物，请遵守当地法律法规。
