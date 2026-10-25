# GitHub 仓库更新通知器（AstrBot 插件 / NapCat）

> 仓库：https://github.com/ltbjx/astrbot_plugin_github_notifier

监控 GitHub 仓库更新（**Release / Commit / Issue / PR**），自动推送到你指定的**白名单 QQ 群**。
所有配置都在 AstrBot 控制台的插件配置页里改，保存后插件自动热重载。

- 插件名：`astrbot_plugin_github_notifier`
- 适用：AstrBot 4.x + NapCat（aiocqhttp / OneBot v11）
- 依赖：`aiohttp`（AstrBot 自带，无需额外安装）

---

## 1. 功能

| 功能 | 说明 |
| --- | --- |
| 仓库监控 | `owner/repo`，也可写 `owner/repo#分支`、直接粘贴 GitHub 链接 |
| 更新类型 | Release（可过滤预发布）、Commit（聚合为一条，带 compare 链接）、Issue、PR，各自可开关 |
| 白名单群 | **只有 `notify_groups` 里的群会收到推送**，其他群一律不收 |
| 静默首检 | 装好后第一次只记录基线，不会把历史更新刷一遍；也可以关掉，直接推最新 Release |
| 状态持久化 | 上次检查到的 commit / release / issue 存在插件 KV 存储里，重载、重启都不会重复推送 |
| 消息分片 | 超过字数上限自动拆成多条，单条更新不会被切成两半 |
| @全体 | 可选，开启后每次推送带 `@全体成员`（需要机器人是群管理员） |
| 代理 | 可配置 `http_proxy` 访问 GitHub；本机默认走 Karing 的 `127.0.0.1:3065`（HTTP/SOCKS 混合端口） |
| Token | 配置后 API 额度 60 次/小时 → 5000 次/小时 |

---

## 2. 安装

### 方式 A：直接放进部署目录（推荐，配合 docker compose）

你的 `docker-compose.yml` 把 AstrBot 的 data 目录挂载为 `./astrbot/data`，所以插件放到：

```bash
# 在你放 astrbot-deploy 的机器上（宿主机）
mkdir -p astrbot/data/plugins
cp -r astrbot_plugin_github_notifier astrbot/data/plugins/
# 让容器重新加载插件
docker restart astrbot
```

也可以不重启，在控制台「插件管理」页点**重载插件**即可。

### 方式 B：控制台在线安装

在 AstrBot 控制台 → 插件管理 → 安装插件，粘贴插件目录的仓库地址或上传压缩包即可。
（本插件不依赖任何私有资源，纯本地代码。）

### 安装后检查

1. 控制台 → 插件管理，确认列表里出现「GitHub 仓库更新通知器」，状态正常；
2. 控制台 → 插件管理 → 该插件 → 配置，按下面第 3 节填写；
3. 群里发送 `/ghn status`，能看到配置摘要就说明跑起来了。

---

## 3. 配置说明（控制台插件配置页）

| 配置项 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `enable` | bool | `true` | 总开关，关掉后不检查也不推送 |
| `repos` | list | `["AstrBotDevs/AstrBot"]` | 监控的仓库。`owner/repo`、`owner/repo#dev`、`https://github.com/owner/repo` 都支持 |
| `notify_groups` | list | `[]` | **白名单群**。填群号（`123456789`）；多账号/多平台时可写 `平台ID:群号` |
| `check_interval_minutes` | int | `10` | 检查间隔（分钟），建议 5~30 |
| `monitor_releases` | bool | `true` | 是否监控 Release |
| `monitor_commits` | bool | `true` | 是否监控 Commit |
| `monitor_issues` | bool | `false` | 是否监控 Issue（会多耗 API 额度） |
| `monitor_pull_requests` | bool | `false` | 是否监控 PR |
| `include_prerelease` | bool | `true` | 是否推送预发布版本 |
| `github_token` | string | `""` | GitHub Token，强烈建议填写（见第 5 节） |
| `http_proxy` | string | `http://127.0.0.1:3065` | Karing 混合端口；不需要代理就清空（只支持 http://，不支持 socks5://） |
| `first_run_silent` | bool | `true` | 首次检查静默，只建基线 |
| `check_on_startup` | bool | `true` | 插件加载后立即检查一次 |
| `max_items_per_repo` | int | `5` | 单仓库每次最多推送几条（commit 列表只展示前几条） |
| `max_message_chars` | int | `2800` | 单条消息最大字数，超出自动分片 |
| `mention_all` | bool | `false` | 推送时 @全体成员 |
| `header_text` | string | `🔔 GitHub 仓库更新播报` | 推送消息标题，可随意改 |

### 群号怎么填

`notify_groups` 里填的就是 **QQ 群号**（NapCat 的 `group_id` 就是群号），例如：

```json
["123456789", "987654321"]
```

如果机器人接了多个 QQ 号（多个 aiocqhttp 平台适配器），想只推送到其中一个，就在控制台该平台配置里找到**适配器 ID**（如 `napcat`），写成：

```json
["napcat:123456789"]
```

> 注意：`notify_groups` 只决定「推送到哪些群」，不影响别人在别的群里用 `/ghn` 查看状态。

---

## 4. 指令

| 指令 | 权限 | 说明 |
| --- | --- | --- |
| `/ghn` 或 `/ghn status` | 所有人 | 查看运行状态（开关、间隔、仓库、白名单群、下次检查时间） |
| `/ghn repos` | 所有人 | 查看正在监控的仓库和分支 |
| `/ghn check` | **管理员** | 立刻检查一次，命中更新会照常推送到白名单群 |
| `/ghn test` | **管理员** | 向所有白名单群发一条测试推送，用来验证配置 |
| `/ghn state` | **管理员** | 查看内部状态（最近一次 commit / release 基线），排查问题用 |
| `/ghn help` | 所有人 | 全部指令列表（**不带斜杠直接发「ghn help」也行**） |

别名：`/github_notifier`、`/仓库提醒` 等价于 `/ghn`。

> 提示：群里直接发「ghn help」（不带 `/`）也会输出完整指令表；如果装了群问答插件，列表里会自动附上 `/faq` 的指令。

> 「管理员」指 AstrBot 的管理员身份（`event.is_admin()`，通常是群主/群管理或在控制台配置的管理员账号）。

---

## 5. 建议：配置 GitHub Token

不填 Token 时，GitHub 匿名限额是 **60 次/小时**，按默认 10 分钟检查 1 个仓库（约 3~4 次请求/轮）会接近限额。
建议创建一个 **只读** 的 fine-grained token：

1. GitHub → Settings → Developer settings → Personal access tokens → **Fine-grained tokens** → Generate new token；
2. Repository access 选 **Public Repositories (read-only)**；
3. 把 token 粘贴到插件配置的 `github_token`。

配置好后限额提升到 5000 次/小时，监控多个仓库也没问题。

---

## 6. 常见问题

**Q：加载插件报 `AttributeError: 'GithubNotifierPlugin' object has no attribute 'logger'`？**
已修复（v1.0.1）。原因是旧版 AstrBot 的 `Star` 基类没有插件专属 `logger` 属性，本插件现在统一使用
`from astrbot.api import logger`，所有 4.x 版本通用。如果你的版本还报这个错，说明用的是修复前的旧压缩包，
替换成新版 `main.py` / 重新安装即可。

**Q：插件加载后状态没有持久化 / 每次重启都重新推送？**
新版优先用 AstrBot 插件 KV 存储；如果你的 AstrBot 版本较老、没有该能力，会自动把状态落到
`data/plugin_data/<插件ID>/state.json`（找不到 data 目录时退化到插件目录下的 `state.json`），日志里会提示。
可以发 `/ghn state` 查看当前的基线记录。

**Q：群里一条消息都没收到？**
1. `/ghn test` 看是否报错；报错信息里会提示「AstrBot 是否已连接 NapCat」「群号是否正确」；
2. 确认 `notify_groups` 里填的是群号，不是 QQ 号；
3. 确认配置保存后插件确实重载了（控制台提示「正在热重载插件」）；
4. 到控制台看 AstrBot 日志，搜 `github-notifier` 或 `GitHub 通知器`。

**Q：为什么不推送历史更新？**
`first_run_silent` 默认开启：第一次检查只记录当前状态作为基线。想立刻看到最新 Release，把它关掉并保存（会推送仓库最新的一个 Release）。

**Q：为什么 commit 推送只显示几条？**
一次推送里同一仓库最多展示 `max_items_per_repo` 条，其余显示「另有 N 条未展示」，点 compare 链接可以看完整 diff。

**Q：`@全体成员` 不生效？**
需要机器人是该群管理员，且 `mention_all` 已开启。

**Q：代理怎么配？连不上 GitHub / 超时？**

看 AstrBot 跑在哪：

*A. 跑在本机 macOS（当前情况）* —— 本机由 Karing 全局代理（TUN 模式），出口是香港节点
`38.76.172.221`，实测 `api.github.com` 直连 200。所以：

| 场景 | 配置 |
| --- | --- |
| 代理软件开着（TUN 或系统代理模式都行），想更稳更快 | `http_proxy = http://127.0.0.1:3065`（默认值） |
| 想省掉一层转发 | 清空 `http_proxy`，走 TUN 直连（实测可用，只是首次握手稍慢） |
| 代理软件换端口了 | 改成实际端口；Karing 的 HTTP/SOCKS 混合端口一般是 3065 |

实测过 Karing 的三个端口，都能用（HTTP 代理模式）：

| 端口 | 结果 |
| --- | --- |
| `127.0.0.1:3065` | ✅ 200，0.36s（推荐） |
| `127.0.0.1:3066` | ✅ 200，1.03s |
| `127.0.0.1:3067` | ✅ 200，2.36s |

> 只支持 `http://` / `https://` 代理（可带账号密码：`http://user:pass@host:port`）。
> 填 `socks5://` 会被忽略并退回直连（aiohttp 不支持 socks5），日志里会有警告。

*B. 把 AstrBot 部署到服务器上* —— 那台机器需要能访问 GitHub，两种办法：
1. 在服务器上自己跑一个 HTTP 代理（如 `tinyproxy`），监听一个端口，插件里填 `http://服务器IP:端口`，
   并只允许你自己的 IP 访问；
2. 或在服务器上 `ssh -L` 到自己另一台能上 GitHub 的机器做端口转发，插件填 `http://127.0.0.1:转发端口`。

**Q：同一个仓库想监控多个分支？**
`repos` 里写两条：`owner/repo#main` 和 `owner/repo#dev`，两者的状态互相独立。

**Q：Force push / 分支被重建会怎样？**
compare 失败时会自动降级为「最近 N 条提交」，不会报错中断，基线会重置到最新。

---

## 7. 文件结构

```
astrbot_plugin_github_notifier/
├── metadata.yaml         # 插件元信息
├── _conf_schema.json     # 控制台配置项定义（控制台就是按它渲染的）
├── main.py               # 插件主体：后台轮询、推送、指令
├── github_client.py      # GitHub REST API 客户端（aiohttp）
├── logic.py              # 纯逻辑：配置解析、状态 diff、消息渲染、分片
├── requirements.txt      # 依赖（aiohttp，AstrBot 自带）
├── tests/
│   ├── test_logic.py         # 纯逻辑单测（python3 tests/test_logic.py）
│   ├── test_plugin_import.py # 插件端到端测试（stub astrbot，不联网）
│   └── live_github_smoke.py  # 真实 GitHub API 冒烟（可选，联网）
└── README.md
```

本地自测：

```bash
python3 tests/test_logic.py
python3 tests/test_plugin_import.py
python3 tests/live_github_smoke.py     # 可选，需要能访问 api.github.com
```

> 小提示：macOS 自带的 Python 有时缺根证书，跑联网冒烟时可以
> `SSL_CERT_FILE=$(python3 -c "import certifi;print(certifi.where())") python3 tests/live_github_smoke.py`。
> 服务器 / Docker 环境不受影响。

---

## 8. 工作原理（一句话版）

插件加载 → 起一个后台协程 → 每 N 分钟拉一次仓库的 Release/Commit/Issue/PR → 和 KV 里存的基线比较 → 有新的就渲染成文本 → 通过 `context.send_message("<平台ID>:GroupMessage:<群号>", ...)` 推到白名单群 → 更新基线。
