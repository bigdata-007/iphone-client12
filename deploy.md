# 部署与 CI 指南（ios-client12 → iphone-client12）

源码本身不落明文；解密口令通过 GitHub Actions Secret 注入。对齐 `apple-sign-desktop-action` 的加密源码流程。

## Secrets

在仓库 **Settings → Secrets and variables → Actions** 添加：

| Name | 说明 |
|------|------|
| `SOURCE_GPG_PASSPHRASE` | 与打包脚本加密时相同的 GPG 对称口令（本机 `action/iphone-client12/.secrets/SOURCE_GPG_PASSPHRASE`） |

本机口令文件**不要**提交（已在 `.gitignore`）。

```bash
# 查看本机口令（复制到 GitHub Secret）
cat action/iphone-client12/.secrets/SOURCE_GPG_PASSPHRASE
# 或用 gh：
gh secret set SOURCE_GPG_PASSPHRASE < action/iphone-client12/.secrets/SOURCE_GPG_PASSPHRASE \
  --repo bigdata-007/iphone-client12
```

## 更新加密源码

在 **ipadump-backend-rust** 仓库执行：

```bash
cd ios-client12
./scripts/pack_encrypted_source.sh
# 默认写出到 action/iphone-client12/source.zip.gpg
```

然后在 **iphone-client12** 仓库提交并 push `source.zip.gpg`（及 workflow 变更）：

```bash
cd action/iphone-client12
git add source.zip.gpg .github README.md deploy.md .gitignore
git status
git commit -m "更新加密源码 / CI"
git push
```

## 触发构建

### 手动触发

Actions → **Build IPA** → Run workflow。

| 输入 | 说明 |
|------|------|
| **Release tag** | 如 `v0.0.1`：构建后自动建 GitHub Release；留空只留 artifact（14 天） |
| **native_ios12** | `true`（默认）：用 **Xcode 16** 把 DT 改成 12.0（iOS 14 真机需要；勿只靠 LOWER_MINOS） |
| **unsigned** | `true`（默认）：`UNSIGNED=1` 无签名包，给侧载工具自签 |

### 推送 tag 自动发布

推送符合 `v*` 的 tag（如 `v0.0.1`）也会触发构建，并在成功后发布到 GitHub Release。

### 推送加密源码

提交并 push `source.zip.gpg` 会触发构建；未带 Release tag 时只保留 workflow artifacts。

## 产物

- `dist/IPADump_<version>.ipa`
- `dist/client.ipa`（同内容副本）
- `dist/latest.json`（仅自动改 version / size / ipaUrl）

默认 **UNSIGNED + 原生 DT=12（Xcode 16）+ LOWER_MINOS 兜底**；artifact / Release **只含一个** `IPADump_*.ipa`（`ONLY_IPA=1`，无 `client.ipa` / `latest.json`）。  
**注意**：本机 Xcode 27 只改 minos 的包在 iOS 14 上会秒闪退；请用本仓 Actions 产物测旧机。CDN 上传仍本机 `ios-client12/upload-qiniu.sh`（会生成 client.ipa + latest.json）。

## 本地解密试跑

```bash
cd action/iphone-client12
gpg --batch --yes --pinentry-mode loopback \
  --passphrase "$(cat .secrets/SOURCE_GPG_PASSPHRASE)" \
  --decrypt -o source.zip source.zip.gpg
unzip -q source.zip -d /tmp/ios-client12-src
cd /tmp/ios-client12-src
UNSIGNED=1 ./scripts/build-release-ipa.sh
```

## Workflow 权限

| 范围 | 用途 |
|------|------|
| `contents: read` | 检出代码（build job） |
| `actions: write` | 上传构建产物 artifact |
| `contents: write` | 创建 Release、上传 IPA（release job） |
| `actions: read` | 下载 artifact（release job） |

若仓库 **Settings → Actions → General → Workflow permissions** 设为「Read repository contents and packages permissions only」，以上 job 级权限声明仍会生效。

## Runner / Xcode（旧机能跑的关键）

| 方案 | 说明 |
|------|------|
| **推荐：本仓 CI** | `macos-15` + **强制 Xcode 16.x** + `native_ios12=true` → 真正按 DT=12 编译 |
| 本机 Xcode 27 + `LOWER_MINOS` | 只能骗过**安装**；**iOS 14 启动常秒闪退**（Swift/ABI 按 15+） |
| `macos-14` | 已于 2026-11-02 退役，勿再用 |
| `macos-26` / `macos-latest` | 基本只有 Xcode 26+，不适合打旧机包 |

若 GitHub 某天去掉 Xcode 16：只能改 **self-hosted Mac**（自装 Xcode 16）或第三方 Mac CI（Codemagic / MacStadium 等）。
