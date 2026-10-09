# Fork 维护与 Release Runbook

本文记录本 fork 的长期维护流程，以及 `Chat Completions + encrypted_content` 修复从定位到发布的完整经验。目标是在官方未修复、或 fork 后续与上游分叉时，能够稳定复现这套流程。

## 1. 当前问题背景

- 现象：Codex++ 1.7.1 + 只支持 Chat Completions 的上游时报：
  `unsupported_encrypted_agent_content: Chat Completions 上游无法处理加密的 agent 消息内容，请使用支持该协议的 Responses 上游`
- 本质：这不是上游 API 返回的错误，而是 Codex++ 在 Responses -> Chat Completions 转换层主动拒绝。
- 1.6.0 不报错，是因为它会静默丢弃 `encrypted_content`，但多代理 v2 的任务正文也会一起丢失。
- 1.7.1 改为显式报错，因此问题从「静默丢正文」变成「请求直接失败」。
- 相关上游讨论：
  - issue #2424：`本地 relay 静默丢弃 agent 间消息 payload（encrypted_content 未转发）`
  - PR #2425：`fix(core): 透传 encrypted_content，修复 agent 间消息 payload 丢失`

## 2. 本 fork 的修复策略

修复只处理 `multi_agent v2` 的真实投递形态：

```json
{
  "type": "agent_message",
  "content": [
    { "type": "input_text", "text": "Message Type: NEW_TASK ... Payload:" },
    { "type": "encrypted_content", "encrypted_content": "<明文任务正文>" }
  ]
}
```

转换规则：

1. `agent_message` 的 `content` 数组内，明文 `encrypted_content` 片段按原顺序转换成 Chat 文本片段。
2. 无法判定为明文的 opaque 内容继续明确报错，不静默丢弃。
3. 顶层独立 `encrypted_content` 项继续明确报错；真实流量不使用该形态。
4. Responses 上游路径不参与转换，始终原样透传。

核心落点：

- `crates/codex-plus-core/src/protocol_proxy.rs`
  - `responses_to_chat_completions_with_options`
  - `append_responses_input`
  - `append_responses_item`
  - `responses_content_to_chat_content`
  - `encrypted_content_value_is_opaque`
- `crates/codex-plus-core/tests/protocol_proxy.rs`
  - 明文 agent payload 顺序透传
  - opaque 内容拒绝
  - 拒绝发生在联系 Chat 上游之前

## 3. 改动规模

以首次修复 `d0d2dcc` 为准：

| 范围 | 改动 |
| --- | --- |
| 核心协议文件 | `+103 / -49`，152 行变更 |
| 协议测试 | `+42 / -9`，51 行变更 |
| 版本元数据 | `Cargo.toml`、`Cargo.lock`、`package.json`、`package-lock.json` |
| 发布说明 | `CHANGELOG.md`、`docs/release/v1.7.2-notes.md` |

结论：

- 逻辑改动集中，不是大重构。
- 风险集中在两处判断：明文判定是否准确、opaque 是否保持 fail loud。
- 后续与上游同步时，应优先检查官方是否已修复同一路径；若已修复，删除本 fork 的重复补丁。

## 4. 本地开发与验证

### 4.1 工具链

本机首次使用时，Rust 可能不在 PATH：

```powershell
$env:Path = "$env:USERPROFILE\.cargo\bin;$env:Path"
cargo --version
rustc --version
```

如系统代理失效，Cargo 会卡在 `127.0.0.1` 代理错误。实践中可直接清空 Cargo 代理后重试：

```powershell
$env:CARGO_HTTP_PROXY = ''
$env:HTTPS_PROXY = ''
$env:HTTP_PROXY = ''
$env:ALL_PROXY = ''
```

### 4.2 必跑测试

```powershell
cargo test -p codex-plus-core --test protocol_proxy
cargo test -p codex-plus-core --lib helper_rejects_encrypted_agent_content_as_bad_request_before_upstream_send
```

验证标准：

- `protocol_proxy` 149 例全部通过。
- launcher 错误映射测试通过。
- 明文 payload 出现在转换后的 Chat 消息中，且顺序为 header -> payload。
- opaque 内容仍在请求发往上游前失败。

### 4.3 格式检查

仓库当前存在上游遗留的全局 `cargo fmt --check` 漂移，不能把全仓格式失败当作本次改动失败。对本次改动应至少确认：

```powershell
git diff --check
```

## 5. Fork 与远端

当前 fork：

- `origin`：`https://github.com/yhan1203/CodexPlusPlus.git`
- 上游：`https://github.com/BigPizzaV3/CodexPlusPlus.git`
- 默认分支：`main`
- 发布 tag：`v1.7.2`
- 修复提交：`d0d2dcc fix: support plaintext encrypted agent payload on chat upstreams`

### 5.1 Git 网络与认证

本机曾出现：

- Git 直连 GitHub 失败；
- `gh` API 正常；
- 系统代理 `127.0.0.1:11808` 可用；
- 默认凭据不会自动进入 Git push。

可靠流程：

```powershell
gh auth status
gh auth setup-git --hostname github.com

git -c http.proxy=http://127.0.0.1:11808 `
    -c https.proxy=http://127.0.0.1:11808 `
    push origin main
```

如果本机代理端口变化，以 `HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings` 中的 `ProxyServer` 为准。不要把任何 token 写入配置文件；认证统一交给 `gh auth setup-git`。

## 6. GitHub Actions 发布流程

### 6.1 关键前提

`Release assets` workflow 的 checkout 使用 tag ref。只创建 draft Release 不等于 tag 已存在。必须先推送 tag：

```powershell
git tag -f v1.7.2 d0d2dcc
git -c http.proxy=http://127.0.0.1:11808 `
    -c https.proxy=http://127.0.0.1:11808 `
    push origin refs/tags/v1.7.2
```

否则 Windows job 会在 `Checkout release tag` 阶段失败。

### 6.2 Windows Release 构建

先创建 draft Release，再手动派发 workflow：

```powershell
gh release create v1.7.2 `
  --repo yhan1203/CodexPlusPlus `
  --draft `
  --title "v1.7.2 · Chat Completions 加密 agent 消息修复" `
  --notes-file docs/release/v1.7.2-notes.md `
  --target main

gh workflow run "Release assets" `
  --repo yhan1203/CodexPlusPlus `
  --ref main `
  -f tag=v1.7.2 `
  -f platform=windows `
  -f draft=true `
  -f publish=false
```

Windows job 产出：

- `CodexPlusPlus-<version>-windows-x64-setup.exe`
- `CodexPlusPlus-<version>-windows-x64.zip`

### 6.3 macOS 限制

本 fork 没有上游的 Apple 签名与公证 secrets：

- `MACOS_CERT_P12`
- `MACOS_CERT_PASSWORD`
- `MACOS_KEYCHAIN_PASSWORD`
- `APPLE_API_KEY_P8`
- `APPLE_API_KEY_ID`
- `APPLE_API_ISSUER_ID`
- `MACOS_SIGNING_IDENTITY`

因此不要在这个 fork 上声称 macOS 正式包可用。`platform=windows` 时 macOS job 会 skip。

### 6.4 latest.json 的已知坑

上游 `latest-json` job 会强制要求：

- Windows setup
- Windows zip
- macOS universal DMG

Windows-only 发布时该 job 必然失败，这是预期的，不代表 Windows 构建失败。

手动生成 Windows-only `latest.json` 时，必须注意 PowerShell 会把命令输出拆成数组，导致 `body` 变成 JSON 数组。更新器期望 `body` 是字符串：

```powershell
$body = (
  gh release view v1.7.2 `
    --repo yhan1203/CodexPlusPlus `
    --json body --jq .body | Out-String
).Trim()
```

生成并上传：

```powershell
$latest = [ordered]@{
  version = 'v1.7.2'
  url = 'https://github.com/yhan1203/CodexPlusPlus/releases/tag/v1.7.2'
  body = $body
  assets = @(
    [ordered]@{
      name = 'CodexPlusPlus-1.7.2-windows-x64-setup.exe'
      url = 'https://github.com/yhan1203/CodexPlusPlus/releases/download/v1.7.2/CodexPlusPlus-1.7.2-windows-x64-setup.exe'
    },
    [ordered]@{
      name = 'CodexPlusPlus-1.7.2-windows-x64.zip'
      url = 'https://github.com/yhan1203/CodexPlusPlus/releases/download/v1.7.2/CodexPlusPlus-1.7.2-windows-x64.zip'
    }
  )
} | ConvertTo-Json -Depth 6

[IO.File]::WriteAllText(
  (Join-Path (Get-Location) 'latest.json'),
  $latest,
  [Text.UTF8Encoding]::new($false)
)

gh release upload v1.7.2 latest.json `
  --clobber `
  --repo yhan1203/CodexPlusPlus
```

### 6.5 发布

资产与 `latest.json` 都齐备后再发布：

```powershell
gh release edit v1.7.2 `
  --repo yhan1203/CodexPlusPlus `
  --draft=false `
  --latest `
  --verify-tag
```

## 7. 发布后核验

必须重新从 Release 下载并核验，不要只看构建日志：

```powershell
gh release download v1.7.2 `
  --repo yhan1203/CodexPlusPlus `
  --pattern "*.exe" `
  --pattern "*.zip" `
  --pattern "latest.json" `
  --dir work/release-verify `
  --clobber

Get-FileHash work/release-verify/* -Algorithm SHA256
tar -tf work/release-verify/CodexPlusPlus-1.7.2-windows-x64.zip
```

`v1.7.2` 已核验值：

- setup.exe
  - size：25,272,190
  - SHA256：`AE7A4029E4595D8120E49601A3B598232C2F25AE33A5FC7FFEBE91D7D1E75F05`
- zip
  - size：32,925,882
  - SHA256：`63B4D75A9EA69C89A5CF5418B8886086119E3351D91B2C7A60FC30A4D6FA4BFD`

注意：Windows 安装包未做代码签名，SmartScreen 可能提示未知发布者；这属于 fork 无证书的限制。

## 8. 与上游同步策略

建议流程：

```powershell
git remote add upstream https://github.com/BigPizzaV3/CodexPlusPlus.git
git fetch upstream
git rebase upstream/main
```

处理冲突时：

1. 先检查上游是否已实现等价修复。
2. 若已合并，删除本 fork 重复实现，只保留 fork 自有 feature。
3. 若未合并，优先保留最小补丁：明文片段透传 + opaque fail loud。
4. 不要为了同步上游而改动与任务无关的文件。
5. 每次修正都要在 `CHANGELOG.md`、`docs/release/` 或本文件的修订记录中留下痕迹。

## 9. 将来如何升级和使用

### 9.1 当前 v1.7.2 的更新行为

当前更新器常量仍指向官方仓库：

```text
https://github.com/BigPizzaV3/CodexPlusPlus/releases/latest/download/latest.json
```

因此 v1.7.2 的应用内“检查更新”仍会检查官方版本，不会把本 fork 的 Release 当作更新来源。

现在可靠的升级方式是：

1. 打开 fork Release 页面；
2. 下载最新 Windows setup 或 ZIP；
3. 覆盖安装或替换目录；
4. 用本文第 7 节的方法核验 SHA256。

### 9.2 推荐的长期方案

如果官方长期不修，建议让 fork 自己维护更新源，而不是每次手动下载。

需要修改：

```text
crates/codex-plus-core/src/update.rs
```

将 `DEFAULT_LATEST_JSON_URL` 指向本 fork：

```text
https://github.com/yhan1203/CodexPlusPlus/releases/latest/download/latest.json
```

修改后需要：

1. 提升 fork 版本号；
2. 重新跑 `cargo test`；
3. 重新按第 6 节构建并发布；
4. 确认 Release 的 `latest.json` 是公开可读且指向 fork。

完成后，以后在 Codex++ 内点“检查更新”即可看到 fork 自己的新版本。

### 9.3 什么时候回到官方

出现以下任一情况时，应优先回到官方版本：

- issue #2424 / PR #2425 已被官方正式合并；
- 官方代码已包含明文 `encrypted_content` 透传；
- fork 剩余变化只剩版本号或发布元数据；
- 上游新版本已经修复同一错误。

回官方前先备份配置和会话数据；不要在不了解差异的情况下来回覆盖安装。

## 10. 修订记录

### 2026-10-09

- 定位 1.6.0 与 1.7.1 行为差异：前者静默丢弃，后者显式拒绝。
- 确认 `multi_agent v2` 真实 payload 形态为 `agent_message.content[]` 中的明文 `encrypted_content`。
- 落地 fork 修复，提交 `d0d2dcc`，版本 `1.7.2`。
- 本地 `protocol_proxy` 149 例、launcher 错误映射 1 例通过。
- GitHub Actions `PR build artifacts` 通过。
- Release assets Windows job 通过；`latest-json` job 因 fork 无 macOS DMG 按预期失败，改为手动生成 Windows-only `latest.json`。
- 发布：https://github.com/yhan1203/CodexPlusPlus/releases/tag/v1.7.2
