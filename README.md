# dsh-daily-package

Daily automation: watch [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) upstream `master`, and build an **unsigned** Windows x64 desktop installer whenever there is a new commit.

> 第三方自动化，与 DeepSeek 官方无关；产物也不是官方发布。只是把上游代码按官方流程自动打包，方便自己用。

## 怎么工作

| 环节 | 说明 |
|---|---|
| 触发 | 每天 **UTC 18:00（北京 02:00）** 自动；也可以在 Actions 页手动 `Run workflow`（勾 `force` 可强制重打） |
| 判断有无新提交 | 取上游 `master` 的 SHA，看本仓库是否已有 tag 为 `build-<sha前12位>` 的 Release —— 有就跳过（所以大多数天只跑几秒，`package` job 显示 skipped 属正常） |
| 构建 | `actions/checkout` 直接拉上游代码（**不需要 fork**）→ `pnpm install --frozen-lockfile` → `pnpm run package:desktop:win:x64:unsigned` |
| 记录状态 | **构建成功后**才创建 `build-<sha12>` Release 并附上安装包；失败的构建不写 tag，下次会重试 |
| 耗时 | 约 15–25 分钟，峰值磁盘 ~7–8 GB（node_modules 2.3 GB + `.desktop-build` 2.8 GB） |

只有本仓库里的 workflow 会被执行：上游自带的那些 CI（Release / E2E / Sandbox …）只是被 checkout 成工作区里的普通文件，不会运行。

## 拿产物

- **Actions → 选一次 run → 页面底部 Artifacts**：`dsh-desktop-win-x64-<dsh版本>-<sha12>`
- 或 **Releases → `build-<sha12>`**：同一份安装包（NSIS，约 293 MB，当前用户安装、安装目录可改）

## 已知限制

- **未签名**：正式签名需要 EV 硬件 Token，托管 runner 没有 USB Token。首次运行 Windows 可能弹 SmartScreen；从网络拷来的文件需要 `Unblock-File`。
- **不含自动更新配置**（未签名构建会省略 `app-update.yml`）→ 应用内不会自动更新，升级 = 装新版本。
- **只有 Windows x64**：macOS 需要 Apple 开发者凭据（签名 + 公证），不在本仓库范围。
- 内嵌的强更策略是 `DSH_DESKTOP_AUTO_UPDATE_ENV=production`（`anonymous`），**不会**像 test 环境那样要求飞书登录。
- 产物顶着上游的产品名与图标，属于"非官方构建"，别当成官方发布分发。

## 常用改动

**换定时时间**（cron 用 UTC，北京 = UTC+8）：

| 北京时间 | cron |
|---|---|
| 凌晨 02:00（当前） | `0 18 * * *` |
| 凌晨 04:00 | `0 20 * * *` |
| 每 6 小时 | `0 */6 * * *` |

**带上本地补丁**（改 appId、去掉强更策略元数据等）：把 workflow 里 `actions/checkout` 的 `repository:` 换回你自己的 fork，其余不变。

**只出免安装目录版**（跳过 NSIS 压缩、少占磁盘）：在 `apps/desktop/package.json` 加一条脚本 `"package:win:x64:unsigned:dir": "tsx scripts/package-target.ts win-x64 --unsigned --dir"`，workflow 里改跑它，并上传 `unsigned-artifacts/win-unpacked/**`。

## 目录

```
.github/workflows/package-desktop.yml   # 唯一一条 workflow：解析上游 SHA → 判断 → 打包 → 记录 Release
```

## 注意

- 定时任务只从**默认分支**触发，当前默认分支为 `dev`；改默认分支时记得把 workflow 文件一起带过去。
- 仓库 60 天无活动时，GitHub 会自动停用定时任务（会发邮件提示），届时在 Actions 页点一下 Enable 即可。
- Actions 计费：本仓库 owner 是 `ai-written`。**Public** 仓库用标准 runner 免费；**Private** 为每月 2000 分钟，Windows runner 计 2×（一次构建 ≈ 40–50 分钟额度）。
- 上游代码为 MIT 许可（版权与 `LICENSE` 随上游仓库）。
