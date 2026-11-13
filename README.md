# S805 armv7 云编译仓库

这个目录**本身就是一个独立的 GitHub 仓库骨架**：里面没有 workbuddy-manager 的任何源码，只有一个 workflow。它定时检查官方新版本，只在「上游 Go 源码变了」时交叉编译出 6 个 ARMv7 静态二进制，发布成本仓库的 Release，供 S805 盒子匿名拉取。

> **已落地实例（2026-11-13）**：本仓库 [`uwjtg/workbuddy-s805-build`](https://github.com/uwjtg/workbuddy-s805-build)，Public，Actions 已开启且 `GITHUB_TOKEN` 默认权限为 Read and write。
> - 固定清单：`https://github.com/uwjtg/workbuddy-s805-build/releases/latest/download/manifest.json`
> - 盒子端配置：`MANIFEST_URL=https://github.com/uwjtg/workbuddy-s805-build/releases/latest/download/manifest.json`
> - 首次构建由推送 workflow 自动触发；也可在 Actions → Build S805 armv7 upstream → Run workflow 手动重跑（可勾 `force`）。
> - 下文「三步搭建」是通用搭建说明，本实例已按此完成，无需重复操作。

- **不需要 fork 官方仓库**，不需要同步代码，官方一发新版这里自动跟上。
- **不需要在盒子上放任何 token**，盒子只做「主动拉取」。
- **不需要入站端口 / 内网穿透 / 公网 IP**。
- 公共仓库的 GitHub Actions 标准 runner **免费**；一次完整编译约 2~4 分钟。

---

## 一、三步搭建（需要做一次）

```bash
# 1. 在 GitHub 上新建一个空仓库，例如 workbuddy-s805-build（建议 Public）
#    Public 的 Release 资产可以匿名下载，盒子端零配置；
#    若选 Private，见下面「私有仓库变体」。

# 2. 把本目录推上去
cd cloud-build
git init -b main
git add .github/workflows/build-armv7.yml README.md
git commit -m "ci: armv7 cloud build for S805"
git remote add origin https://github.com/<你的用户名>/workbuddy-s805-build.git
git push -u origin main

# 3. 仓库 Settings → Actions → General → Workflow permissions
#    选 "Read and write permissions" 并保存
#    （否则 GITHUB_TOKEN 无法创建 Release）
```

然后到 **Actions → Build S805 armv7 upstream → Run workflow** 手动跑一次做首次构建。
首次会编译；之后只要 `upstream/.upstream-id` 没变，就复用上一版二进制、跳过 Go 编译。

> 定时：每天 UTC 03:23（北京时间 11:23）。故意避开整点——GitHub 官方文档说明整点是 Actions 高负载时段，定时任务可能被延迟甚至丢弃。想改时间就改 workflow 里的 `cron`。
>
> 官方提醒：**公共仓库 60 天无任何活动会自动停用 scheduled workflow**。但每次有新版本时本 workflow 会创建 Release（算仓库活动），所以只要项目还在发版就不会停；长期停更期间也不需要编译。若被停用，改一下 `cron` 提交一次即可重新激活（该提交者会成为后续 run 的 actor）。

---

## 二、产物契约（盒子端只认这一个 URL）

```
https://github.com/<OWNER>/<REPO>/releases/latest/download/manifest.json
```

`releases/latest/download/<资产名>` 是 GitHub 提供的**固定 URL**，会自动重定向到最新 Release 的同名资产（已在真机验证返回 200；不存在的资产返回 404）。因此盒子端**完全不调用 `api.github.com`**——那台 S805 的出口 IP 实测触发了匿名 API 限流（403 rate limit exceeded），但 `github.com` 直连下载 Release 资产正常。

`manifest.json` 结构：

```json
{
  "schema": 1,
  "package_version": "v2.11.14",
  "upstream_id": "801cebba…",
  "rebuilt": true,
  "goarm": "7",
  "built_at_utc": "2026-11-13T03:23:11Z",
  "builder": "github-actions",
  "asset_base": "https://github.com/<OWNER>/<REPO>/releases/download/v2.11.14",
  "assets": {
    "package":  { "name": "workbuddy-manager-v2.11.14.tar.gz", "sha256": "…", "url": "https://github.com/ithtelab/…" },
    "binaries": { "name": "s805-bin.tar.gz", "sha256": "…", "url": "https://github.com/<OWNER>/<REPO>/…" }
  }
}
```

字段含义：

| 字段 | 作用 |
| --- | --- |
| `package_version` | 与盒子 `/opt/workbuddy-manager/.version` 比较，决定是否要升 |
| `upstream_id` | 官方 `upstream/.upstream-id`。盒子拿它与 `/opt/workbuddy2api/.upstream-id` 比较，决定这次**要不要换二进制** |
| `rebuilt` | 本次云端是否真的跑过 Go 编译（用于审计/通知） |
| `assets.*.sha256` | 盒子下载后强校验，防传输损坏或篡改 |
| `assets.package.url` | **官方原地址**，刻意不镜像整个发布包，避免再分发整个项目 |

`s805-bin.tar.gz` 内含：6 个 ELF32/ARM 静态二进制 + `BUILD-INFO.txt`（版本、upstream-id、GOARM、Go 版本、编译时间）+ `LICENSE-upstream.txt`。盒子端 `upgrade-s805.sh` 会读 `BUILD-INFO.txt` 强校验二进制与源码配套，拒绝拿旧源码编出的产物。

---

## 三、盒子端接入

见 `../box/`。核心是把 `MANIFEST_URL` 指向上面的固定 URL：

```bash
MANIFEST_URL="https://github.com/<OWNER>/<REPO>/releases/latest/download/manifest.json"
```

同一个盒子脚本也兼容 VPS 产物源——只要 VPS 输出同样格式的 `manifest.json` 即可，见 `../vps/`。

---

## 四、私有仓库变体（介意公开分发二进制时）

Private 仓库的 Release 资产**不能匿名下载**，二选一：

1. **细粒度只读 PAT**：在盒子 `/etc/wbm-autoupdate.env` 里配 `CURL_AUTH_HEADER="Authorization: Bearer github_pat_…"`，并把 `MANIFEST_URL` 换成 GitHub API 的 asset 端点。注意认证请求按用户计额度（5000/h），不会撞上匿名共享 IP 的 403；但要调用 `api.github.com`。
2. **Actions 编译完同步到自己的 VPS / 对象存储**（推荐）：在 workflow 末尾加一步用 `scp`/`rclone` 把 `s805-bin.tar.gz` + `manifest.json` 推到静态目录，盒子只访问 VPS。这样 GitHub 侧完全私有，盒子也不需要 token。混合路线的脚本见 `../vps/`。

---

## 五、合规与边界

- 上游 `workbuddy2api` 源码为 **MIT**（Copyright (c) 2026 Sliverkiss），原仓库已由作者删除，源码由 workbuddy-manager 维护者留存。编译产物随包附带 `LICENSE-upstream.txt`，满足 MIT「保留版权声明」的要求。
- workbuddy-manager 主项目同为 MIT，但 README 有明确的**使用边界与免责声明**：只用于管理本人及亲属自愿授权的账号，禁止批量注册、账号交易、代注册、接码、盗号、绕过平台风控，以及任何商业转售或盈利性用途。本仓库只做自动化构建，不含任何账号凭据；请确认你的用途在该边界内。
- 产物按「现状」提供，无担保。

---

## 六、排错

| 现象 | 原因 / 处理 |
| --- | --- |
| `gh release create` 报 403 | 仓库 Settings → Actions → Workflow permissions 没开 Read and write |
| 卡在下载官方 tar.gz | 官方资产名是 `workbuddy-manager-<tag>.tar.gz`；若作者改了命名，用 `workflow_dispatch` 指定 tag 并核对 URL |
| 每次都重新编译 | 上一版 Release 缺 `manifest.json` 或 `s805-bin.tar.gz`；手动补齐后即恢复正常 |
| 盒子报 `manifest.json 404` | 还没跑过首次构建，或 Release 被标成了 draft / prerelease |
| 60 天后定时不跑 | 公共仓库无活动自动停用；改一次 cron 提交即可重新激活 |
