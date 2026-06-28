# RelayEmby

RelayEmby 是一个基于 Cloudflare Workers 与 D1 的 Emby 反向代理管理工具。项目提供网页管理面板，可用于维护多个 Emby 节点、生成代理访问地址、检测节点状态，并支持网盘播放直连、CNAME 更换、Telegram 每日报表等辅助能力。

项目地址：https://github.com/poouo/RelayEmby

## 功能概览

- Cloudflare Worker 单文件部署，入口文件为 `worker.js`
- D1 持久化存储节点、配置、播放统计和保号状态
- `/admin` 管理面板，使用 `ADMIN_TOKEN` 登录
- 每个反代节点使用独立二级域名访问，例如 `movie.example.com`
- 支持新增、编辑、删除、导入、导出和批量管理节点
- 支持节点健康检测、播放转发、网盘直连策略和图片/静态资源缓存
- 可选 Telegram 每日报表、保号提醒和 Cloudflare DNS 记录切换

## 部署准备

你需要准备：

- 一个 Cloudflare 账号
- 一个 Workers 项目
- 一个 D1 数据库
- 一个用于后台登录的 `ADMIN_TOKEN`
- 一个已接入 Cloudflare 的域名，并配置泛域名 DNS / Worker 路由

Worker 运行时会自动创建所需 D1 表结构，包括 `proxy_kv`、`keepalive_state`、`play_sessions` 和 `play_stats`。

## Cloudflare 控制台部署

1. 登录 Cloudflare Dashboard。
2. 进入 `Workers & Pages`，创建一个 Worker。
3. 将本仓库中的 `worker.js` 内容复制到 Worker 编辑器并保存。
4. 创建 D1 数据库，并在 Worker 的 `Settings` -> `Bindings` 中绑定数据库。
5. D1 绑定名称使用 `EMBY_D1`，也可以使用代码兼容的 `D1` 或 `DB`。
6. 为 Worker 绑定自定义域名或路由，例如：

```text
example.com/*
*.example.com/*
```

7. 在 DNS 中添加泛域名记录，例如 `*.example.com` 指向 Worker 可用的目标，并开启 Cloudflare 代理。使用 Workers Routes 时通常可以添加 `*` 或 `*.example.com` 的 proxied 记录；使用 Custom Domains 时请确保泛域名也能命中 Worker。
8. 在 `Settings` -> `Variables` 中添加环境变量：

```text
ADMIN_TOKEN=你的后台登录密码
```

9. 部署 Worker。
10. 打开 `https://example.com/admin`，输入 `ADMIN_TOKEN` 进入管理面板。
11. 如需使用后台的 CNAME/DNS 更换功能，进入菜单中的 `CNAME 更换`，填写 Cloudflare API Token；DNS 记录名可留空使用基础域名/当前后台域名，Zone 名称可留空自动推断。

访问 Worker 根路径 `/` 会自动跳转到 `/admin`。

## Wrangler 部署

如果你使用 Wrangler，可以在本地初始化 Cloudflare Worker 项目后，将 `worker.js` 作为 Worker 入口文件。

示例 `wrangler.toml`：

```toml
name = "relayemby"
main = "worker.js"
compatibility_date = "2024-06-01"

[[d1_databases]]
binding = "EMBY_D1"
database_name = "relayemby"
database_id = "你的 D1 database_id"

[vars]
ADMIN_TOKEN = "你的后台登录密码"
```

仓库已包含最小 `wrangler.toml`，用于告诉 Wrangler 入口文件是 `worker.js`。为了避免提交敏感信息，D1 数据库 ID 和 `ADMIN_TOKEN` 不写入仓库；请在 Cloudflare 控制台为 Worker 绑定 D1，并在变量中设置 `ADMIN_TOKEN`。

部署命令：

```bash
wrangler deploy
```

## 环境变量

必填变量：

| 变量名 | 说明 |
| --- | --- |
| `ADMIN_TOKEN` | 管理后台登录令牌 |
| `EMBY_D1` | 推荐的 D1 数据库绑定名，代码也兼容 `D1` 和 `DB` |

可选变量：

| 变量名 | 说明 |
| --- | --- |
| `RELAYEMBY_BASE_DOMAIN` | 反代使用的基础域名，例如 `example.com`；不填时使用当前后台访问域名 |
| `ENABLE_DIRECT_PROXY` | 设置为 `1` 后启用直接 URL 代理 |
| `CORS_ALLOW_ORIGIN` | 指定允许跨域的 Origin，多个值用英文逗号分隔 |
| `CF_API_TOKEN` | Cloudflare API Token，用于 DNS 记录管理；也可在后台 `CNAME 更换` 中保存 |
| `CF_ZONE_NAME` | Cloudflare Zone 名称；可不填，系统会从 DNS 记录名自动推断 |
| `CF_RECORD_NAME` | 需要管理的 DNS 记录名；可不填，系统会优先使用后台配置或基础域名 |
| `EMOS_PROXY_ID` | 转发时附加的 EMOS 代理 ID 请求头 |
| `EMOS_PROXY_NAME` | 转发时附加的 EMOS 代理名称请求头 |

## 使用方式

1. 访问 `/admin` 并使用 `ADMIN_TOKEN` 登录。
2. 点击新增按钮创建节点。
3. 填写显示名称、二级域名前缀和目标 Emby 地址。
4. 保存后，节点会生成独立代理域名，例如节点名前缀为 `movie`，基础域名为 `example.com` 时，代理地址为 `https://movie.example.com`。
5. 在 Emby 客户端中使用生成的地址进行访问。

节点名前缀必须是合法 DNS 标签：仅允许 `a-z`、`0-9` 和 `-`，长度 1~32，且不能以 `-` 开头或结尾。

## 路由说明

- `/`：自动跳转到 `/admin`
- `/admin`：管理后台
- `https://节点名.example.com/`：访问对应节点
- `https://节点名.example.com/emby/...`：按原始 Emby 路径转发到对应目标

旧的 `https://example.com/节点名` 路径模式已移除。

## 注意事项

- 请妥善保管 `ADMIN_TOKEN`，不要公开暴露后台令牌。
- 请确认 `*.基础域名` 能命中同一个 Worker，否则节点二级域名无法访问。
- 绑定 D1 后首次访问后台 API 会自动初始化表结构。
- 本项目仅供学习与技术测试使用，请遵守当地法律法规。使用者需自行承担配置、转发内容和访问行为产生的责任。
