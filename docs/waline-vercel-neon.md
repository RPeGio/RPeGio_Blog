# Waline：Vercel + Neon 部署清单

目标架构：

```text
Astro 静态站（Cloudflare Pages，https://rpegio.top）
        │  @waline/client
        ▼
Waline 服务端（Vercel，https://waline.rpegio.top）
        │  PostgreSQL
        ▼
Neon（免费版即可）
```

前端已经按这个域名接好；如果最终不用 `waline.rpegio.top`，只需要改
`src/site.config.ts` 里的 `server` 一行。

## 0. 前置条件

- Vercel 账号（用 GitHub 登录最方便）
- Neon 账号（在 Vercel 的 Storage 里创建即可，不用单独注册）
- `rpegio.top` 的 DNS 管理权限（当前在 Cloudflare）
- 本机有 Node 18+（当前是 Node 24）和 bun

## 1. 部署 Waline 服务端

打开官方一键模板：

<https://vercel.com/new/clone?repository-url=https%3A%2F%2Fgithub.com%2Fwalinejs%2Fwaline%2Ftree%2Fmain%2Fexample>

1. 项目名填 `rpegio-waline`（或你喜欢的名字）。
2. 点 `Create`，Vercel 会新建一个仓库并开始部署。
3. 第一次部署可能报 `500 This Serverless Function has crashed`，这是还没连数据库，正常。继续下一步。

> 这个模板的内容就是 `index.cjs` + `vercel.json`，入口是
> `@waline/vercel`；不需要自己写服务端代码。

## 2. 创建并初始化 Neon

1. 进入 Vercel 项目 → 左侧 `Storage` → `Create Database`。
2. 在 `Marketplace Database Providers` 里选 `Neon` → `Continue`。
3. 按提示接受并创建 Neon 账号，区域尽量选新加坡/香港（离读者近）。
4. 创建完成后点 `Connect`，把数据库连接到 `rpegio-waline` 项目。
5. 点 `Open in Neon` → 左侧 `SQL Editor`。
6. 把 Waline 的建表 SQL 粘进去执行：

   <https://github.com/walinejs/waline/blob/main/assets/waline.pgsql>

   这会创建 `wl_comment`、`wl_counter`、`wl_users` 三张表。
7. 回到 Vercel → `Deployments` → 最新一次部署 → `Redeploy`，让数据库环境变量生效。
8. 等状态变成 `Ready`，点 `Visit`，能看到 Waline 页面就说明服务端起来了。

Neon 连接成功后，Vercel 会自动注入数据库变量，一般不需要手填 `PG_*`。
如果你是手动把 Neon 连接串填到 Vercel，则按 Waline 文档配置
`PG_HOST` / `PG_PORT` / `PG_DB` / `PG_USER` / `PG_PASSWORD` / `PG_SSL`。

## 3. 配置环境变量

Vercel 项目 → `Settings` → `Environment Variables`，至少加这些：

| 变量 | 值 | 说明 |
|---|---|---|
| `SITE_NAME` | `RPeGio的小站` | 邮件/页面标题 |
| `SITE_URL` | `https://rpegio.top` | 前端站点地址，不要结尾斜杠 |
| `SERVER_URL` | `https://waline.rpegio.top` | 服务端地址 |
| `SECURE_DOMAINS` | `rpegio.top,www.rpegio.top,waline.rpegio.top` | 允许的前端来源 |
| `COMMENT_AUDIT` | `true` | 新评论先待审核，防垃圾评论 |
| `AKISMET_KEY` | `false` | 不用默认共享 key；要反垃圾可换自己的 key |
| `IPQPS` | `60` | 单 IP 每分钟请求上限，默认值即可 |

可选：

- `DISABLE_REGION=true`：不在评论区显示 IP 归属地。
- `LOGIN=force`：强制登录后才能评论；一般不要设，保留匿名评论。
- `SMTP_*` / `SENDER_*`：邮件通知，见第 7 节。

加完变量后要再 `Redeploy` 一次。环境变量建议同时勾选
Production / Preview / Development，避免预览环境 CORS 不一致。

## 4. 绑定自定义域名

`*.vercel.app` 在国内不可靠，必须绑自己的域名。

1. Vercel 项目 → `Settings` → `Domains` → 添加 `waline.rpegio.top`。
2. 到 Cloudflare DNS 加记录：

   | Type | Name | Value | Proxy |
   |---|---|---|---|
   | CNAME | `waline` | `cname.vercel-dns.com` | DNS only（灰云） |

3. 等 Vercel 显示证书 `Valid`。第一次验证时不要开 Cloudflare 代理，否则可能证书验证失败。
4. 访问 `https://waline.rpegio.top/`，能看到 Waline 页面即可。

## 5. 立刻注册管理员

部署完成后第一时间打开：

```text
https://waline.rpegio.top/ui/register
```

**第一个注册的用户会自动成为管理员。** 注册后：

- 用管理员账号登录 `/ui`，确认能进评论管理。
- 在 `/ui` 里把匿名评论默认状态设为待审核（或依赖 `COMMENT_AUDIT=true`）。
- 测试一条评论：前台提交 → 后台通过 → 前台显示；再测试回复、删除。

不要先把服务端地址公开很久再注册，避免被别人抢注管理员。

## 6. 前端构建与部署

前端改动已经完成：

- `src/site.config.ts`：`server` 已指向 `https://waline.rpegio.top/`；
  `lang: 'zh-CN'`；中文 placeholder；`reaction0` 改为「喜欢」。
- `src/components/waline/PageInfo.astro`：阅读/评论中文文案；把服务端地址写到
  `data-server`，供独立浏览量脚本使用。
- `src/components/waline/Pageview.astro`：改为本地导入
  `@waline/client/pageview`，不再从 jsDelivr 运行时拉取。
- `src/components/waline/Comment.astro`：修正 `showMeta` 判断取反的问题；
  加载占位文案改为中文。

构建并部署：

```bash
bun run build
```

把 `dist` 部署到 Cloudflare Pages（当前站点已有 Pages 项目，按原来的方式
push 或上传即可）。如果最终域名不是 `waline.rpegio.top`，先改
`src/site.config.ts` 的 `server` 再构建。

## 7. 可选：邮件通知

Waline 官方支持 SMTP。要“有人回复时发邮件”，在 Vercel 环境变量里加：

```text
SMTP_SERVICE=QQ
SMTP_USER=你的邮箱
SMTP_PASS=邮箱授权码
SENDER_NAME=RPeGio
SENDER_EMAIL=你的邮箱
```

也可以改用 `SMTP_HOST` / `SMTP_PORT` / `SMTP_SECURE`。具体字段以
<https://waline.js.org/reference/server/env.html> 为准。

不配 SMTP 也能正常评论，只是没有邮件提醒；后台仍能看到待审核评论。

## 8. 验证清单

> **PowerShell 注意**：`\` 换行是 Bash 语法，PowerShell 会把它当成参数，
> 导致 `curl: (3) URL rejected: Bad hostname`。下面每条命令都写成一行；
> 在 PowerShell 里建议显式使用 `curl.exe`。

```bash
# 服务端是否起来
curl.exe -I https://waline.rpegio.top/

# 评论接口是否正常（不带 Origin 会被 Waline 拒绝，这是正常的）
curl.exe -s -H "Origin: https://rpegio.top" "https://waline.rpegio.top/api/comment?path=/&pageSize=1"

# 浏览量接口是否正常
curl.exe -s -H "Origin: https://rpegio.top" "https://waline.rpegio.top/api/article?path=/&type=time"

# CORS 是否允许站点来源
curl.exe -i -H "Origin: https://rpegio.top" "https://waline.rpegio.top/api/comment?path=/&pageSize=1"

# 不允许的来源应被拒绝
curl.exe -i -H "Origin: https://evil.example" "https://waline.rpegio.top/api/comment?path=/&pageSize=1"
```

前三条应返回 `200`。CORS 这条要看到 `200`；当前 Waline 1.43.6 对允许来源
返回的是 `access-control-allow-origin: *`（同时带 `Vary: Origin`），不是
具体 origin，所以不能只看这个头判断限制是否生效。最后一条应返回 `403`，
才说明 `SECURE_DOMAINS` 的白名单在起作用。

如果允许来源仍是 `403`，检查 `SECURE_DOMAINS` 是否包含 `rpegio.top` 和
`www.rpegio.top`，以及是否在改完变量后重新部署。

页面验收：

- [ ] 文章页显示「阅读 / 评论」，数字来自新服务端
- [ ] 匿名可以提交评论，提交后进入待审核
- [ ] 管理员在 `/ui` 通过后，前台能看到
- [ ] 回复、删除、点赞正常
- [ ] 手机端和暗色模式正常
- [ ] 断网/服务端挂掉时，评论区有错误提示而不是一直转圈

## 9. 运维与迁移

- **备份**：Neon 控制台有备份/分支/恢复能力；也可以本机用连接串
  `pg_dump "$DATABASE_URL" > waline.sql` 定期导出。
- **区域**：Vercel 项目 `Settings → Functions → Region` 如果允许，选
  `hkg1`（香港）或 `sin1`（新加坡）；默认美东会明显慢。Neon 也选同一区域。
- **国内访问**：即使绑了自定义域，Vercel 函数也不在大陆；比 `*.vercel.app`
  稳定，但延迟仍高于国内 ECS。以后评论量大或要大陆低延迟，可以把同一套
  Waline + PostgreSQL 迁到你的 ECS，数据用 `pg_dump` 搬过去。
- **旧数据**：旧模板用的是公共 demo，没有管理员权限，评论无法完整导出；
  旧浏览量是多个站点共享的计数，不可信。能确认的真实评论只有文章下那条
  「不赖」，需要的话手动重发。新库从 0 开始。
- **隐私政策**：上线前建议补 `/terms/privacy-policy`，说明评论会收集昵称、
  邮箱、IP、浏览器信息，用途是展示和反垃圾，数据存放在 Vercel/Neon，以及
  如何申请删除。

## 10. 最小验收顺序

1. Vercel 服务端 `Ready`
2. Neon 建表完成
3. `waline.rpegio.top` 证书有效
4. `/ui/register` 管理员已注册
5. `COMMENT_AUDIT=true`
6. 前端构建并部署
7. CORS `curl` 通过
8. 前台发一条评论 → 后台审核 → 前台显示
9. 配好备份
