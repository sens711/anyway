# cfnew — Cloudflare Workers 部署包（无需买 VPS）

本包是 **cfnew 核心版（core-only）** 的可直接推送部署包。代码来自官方仓库
`byJoey/cfnew` 最新 Release 的 `_worker.js`（未改动，未植入任何 UUID / 密码 /
密钥，所有敏感值均为占位符，需你自己在 Cloudflare / GitHub 里填）。

## cfnew 是什么

cfnew 是运行在 Cloudflare Workers 上的 VLESS / Trojan / xhttp 代理面板 +
订阅生成器：

- 一个 Worker 即一个节点入口，客户端用 VLESS / Trojan 协议连接
- 访问 `https://<你的 workers.dev 域名>/<UUID>` 可获取订阅链接
- 支持 xhttp / WebSocket 等传输，支持优选 IP、地区筛选、订阅管理
- 本包只含核心 `_worker.js`，不含任何账号、UUID、密码

> 核心文件版本：下载时文件内标注 `2026.10.17.2`，37,806 字节，
> 以 `import{connect}from"cloudflare:sockets"` 开头，结尾 `export{Wn as default}`。

## 文件清单

| 文件 | 说明 |
|---|---|
| `_worker.js` | 官方 Release 下载的核心代码，不要手动改 |
| `wrangler.toml` | Worker 配置：name=`cfnew`，compatibility_date=`2026-01-20`，KV 绑定 `C`（ID 待替换） |
| `.github/workflows/deploy.yml` | push 到 `main` 分支时用 `cloudflare/wrangler-action@v4` 自动部署 |
| `.gitignore` | 忽略 `node_modules`、`.dev.vars`、`.wrangler/` 等本地文件 |
| `README.md` | 本说明 |

## 一、Cloudflare 侧准备（先做这步）

1. 登录 [dash.cloudflare.com](https://dash.cloudflare.com)，进入 **Workers & Pages**。
2. 创建一个 KV Namespace（**Workers & Pages > KV**）：
   - 名字任意，例如 `cfnew-kv`
   - 记下它的 **Namespace ID**，替换 `wrangler.toml` 里的
     `REPLACE_WITH_YOUR_KV_ID`。
3. 在 Worker 设置里确认：
   - **Compatibility date = 2026-01-20**（已写在 `wrangler.toml` 里）
   - KV 绑定名必须是 `C`，对应 `[[kv_namespaces]] binding = "C"`。

## 二、Worker 变量（在 Dashboard 里设，不要提交到 Git）

路径：**Workers & Pages > cfnew > Settings > Variables**，点 Add variable。

| 变量名 | 必填 | 说明 |
|---|---|---|
| `u` | 是 | UUID，VLESS / Trojan 的密码。自己生成一个，例如 `uuidgen` / 在线 UUID 生成器。不要用别人分享的 UUID。 |
| `s` | 否 | SOCKS5 家宽落地地址，见下面“住宅 IP 出口”一节。格式 `user:pass@host:port` 或 `host:port`，留空则不走落地。 |
| `p` | 否 | `proxyip`，Cloudflare 优选 IP / 反代地址，用于中转出站。 |
| `wk` | 否 | 地区筛选，例如 `HK,JP,SG,US`，用于过滤订阅节点地区。 |
| `d` | 否 | 自定义订阅路径，不填则用默认路径。 |

## 三、住宅 IP 出口是怎么回事（重要）

**只部署 Worker，出口 IP 是 Cloudflare 机房 IP，不是住宅 IP。**

链路是：

```
客户端 → Cloudflare Worker（cfnew，机房出口）
        → s 变量指向的 SOCKS5 家宽落地（你家宽带 / 家宽节点）
        → 目标网站（看到的是家宽住宅 IP）
```

- 不填 `s`：出口就是 Cloudflare 边缘节点 IP，会被识别为机房 / 数据中心 IP。
- 填了 `s`：Worker 把流量转发到你提供的 SOCKS5 家宽落地，再由家宽出口，
  目标网站看到的才是住宅 IP。
- 本包**没有**提供任何 SOCKS5 地址，`s` 必须由你自己提供（你已有的家宽
  节点 / 落地服务器）。不要把 SOCKS5 账号密码写进代码或提交到 Git，
  只填在 Cloudflare Dashboard 的变量里。

## 四、GitHub 私有仓 + 自动部署

1. 在 GitHub 新建一个 **私有仓库**（Private），例如 `cfnew-deploy`。
2. 把本目录全部文件 push 到该仓库的 `main` 分支：
   ```bash
   cd cfnew-deploy
   git init
   git add .
   git commit -m "cfnew worker deploy"
   git branch -M main
   git remote add origin https://github.com/<你的用户名>/cfnew-deploy.git
   git push -u origin main
   ```
3. 在 GitHub 仓库设置 Secrets（**Settings > Secrets and variables > Actions > New repository secret**）：
   - `CLOUDFLARE_API_TOKEN`：Cloudflare API Token，需有 Workers 编辑权限。
     建议在 Cloudflare **My Profile > API Tokens > Create Token** 用
     “Edit Cloudflare Workers” 模板创建。
   - `CLOUDFLARE_ACCOUNT_ID`：Cloudflare Account ID，在 Dashboard 右侧
     或 Workers 页面可复制。
4. 以后每次 `git push` 到 `main`，GitHub Actions 会自动跑
   `cloudflare/wrangler-action@v4` 部署到 Workers，无需本地装 Wrangler，
   也无需买 VPS。

首次部署后，在 Cloudflare Dashboard 的 Worker 页面可看到
`https://cfnew.<你的子域名>.workers.dev`，访问
`https://cfnew.<你的子域名>.workers.dev/<你的UUID>` 获取订阅。

## 五、安全提醒

- **不要**把 UUID、SOCKS5 账号密码、API Token、Account ID 写进代码、
  `wrangler.toml` 或提交到 Git（哪怕是私有仓）。
- UUID 泄露 = 任何人都能用你的节点，请定期更换。
- Cloudflare Workers 免费套餐有每日请求 / CPU 时间上限，超量会被限流，
  流量大请留意用量和账单。
- 本包未做任何部署、未推送 GitHub、未向 Cloudflare 提交任何信息。
