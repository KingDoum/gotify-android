# CHEATSHEET — Gotify Android 个人构建

> 一页纸速查卡。事实以本仓库代码 + GitHub Actions 实际运行结果为准。
> 建立：2026-10-20

## 1. 仓库关系

| 角色 | 地址 |
|---|---|
| 上游（只读参考） | `https://github.com/gotify/android`（MIT） |
| 本仓库（自有 fork 形态） | `https://github.com/KingDoum/gotify-android`（public） |
| NAS 工作副本 | `/vol1/1000/docker/Services/gotify/android` |
| 签名密钥（不在仓库内） | `/vol1/1000/docker/Services/gotify/keys/gotify-release-key.jks` |
| 已下载 APK | `/vol1/1000/docker/Services/gotify/apk/` |

远端约定：`origin` = 上游，`mine` = 本仓库（SSH）。

## 2. 构建链路

CI 沿用上游 `.github/workflows/main.yml`，**未做任何修改**：

| 触发 | 行为 |
|---|---|
| push 到 `master` / PR | `./gradlew build`（编译 + lint，不签名） |
| push tag `v*` | `./gradlew -Psign build` → 用 secrets 签名 → 上传 APK 到同名 GitHub Release |

签名所需的 3 个 secrets（已配置，值来自本地 keystore）：

| Secret | 内容 |
|---|---|
| `RELEASE_KEY` | keystore 的 base64（单行） |
| `STOREPASS` | keystore 口令 |
| `KEYPASS` | key 口令 |

## 3. 发一个新版本

```bash
cd /vol1/1000/docker/Services/gotify/android
# ...改代码...
git add -A && git commit -m "feat: 改动说明"
git push mine master

git tag -a v2.10.1-custom.2 -m "改动说明"
git push mine v2.10.1-custom.2
# 约 5 分钟出 APK（首次含依赖下载约 4-5 分钟，后续更快）
```

APK 下载地址（public，手机浏览器可直接打开）：

```
https://github.com/KingDoum/gotify-android/releases/download/<tag>/Gotify.apk
```

## 4. 验证改动真的生效（不要只看 CI 绿）

```bash
# 1) 运行结论
curl -s -H "Authorization: token $TOKEN" \
  https://api.github.com/repos/KingDoum/gotify-android/actions/runs?per_page=3 \
  | python3 -c "import sys,json;[print(r['id'],r['status'],r['conclusion']) for r in json.load(sys.stdin)['workflow_runs']]"

# 2) 签名指纹必须等于本地 keystore（否则无法覆盖升级）
python3 /tmp/apk_sig.py Gotify.apk   # cert_sha256 应 = 7F:C5:8D:1D:...

# 3) 安装后核对版本号 / 行为
```

签名证书 SHA-256（唯一可信基准）：

```
7F:C5:8D:1D:2A:EA:2E:E2:AA:4F:00:7F:9B:65:CA:36:9C:8B:76:65:A2:6B:36:20:68:4C:28:0A:69:BC:09:16
```

## 5. 已知坑

| 症状 | 根因 | 处理 |
|---|---|---|
| Release 步骤报 `Failed to create release ... Not Found` | 新仓库 `GITHUB_TOKEN` 默认 `read`，无 `contents: write`；GitHub 对无权限的 Release 接口返回 404 而非 403 | 仓库 Settings → Actions → General 设为 Read and write，或 `PUT /repos/{o}/{r}/actions/permissions/workflow` 传 `default_workflow_permissions=write` |
| 改完权限 API 返回 204 但 `GET` 仍是旧值 | 最终一致延迟 | 等约 20 秒再读，勿重复 PUT |
| fork 仓库 push 不触发构建 / `workflow_dispatch` 返回 422 | 平台侧对 fork 的运行层禁用（配置层显示 enabled） | 在网页 Actions 页点一次启用；本仓库是新建仓而非 fork，未遇到 |
| `github.com` 直连失败 | 网络 | 走 `ssh.github.com:443`（本机 `~/.ssh/config` 已配）；或 Git Data API |

## 6. 上游同步

```bash
git fetch origin --tags
git log --oneline <旧tag>..<新tag>     # 看上游改了什么
git rebase <新tag>                     # 我们的提交在 master 上，rebase 后重新打 tag
```

> 注意：**不要**把 `v*` 上游 tag 直接推到 `mine` 再手动改——上游 tag 会触发签名构建并生成与官方同名的 Release。发版一律用 `v<上游版本>-custom.<N>`。

## 7. 回退

| 场景 | 做法 |
|---|---|
| 新版 APK 有问题 | 装回上一个 release 的 APK（同签名可覆盖） |
| 代码改坏 | `git revert <sha>` 或 `git reset --hard <好提交>`（仅本地，勿动已发布 tag） |
| CI 挂了 | 看 run 日志；签名相关先查 secrets 是否被删 |

## 8. 密钥备份（重要）

`keys/gotify-release-key.jks` + `keys/keystore-passwords.txt` 一旦丢失，**已安装的 App 将无法再覆盖升级**（必须卸载重装）。
建议异地备份该目录（含口令），不要提交进仓库。
