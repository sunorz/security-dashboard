# 安全看板迁移指南

[English edition](README.md)

### 引言 

这是一个vibe coding项目，完全*黑盒开发*，助手是MuseAI和Codex。看板管两件事：网站安全（攻击、封禁）和主机安全（登录、密钥）。

#### 演示地址

[https://kkii.site/dashboard](https://kkii.site/dashboard)

#### Github仓库地址

[https://github.com/sunorz/security-dashboard](https://github.com/sunorz/security-dashboard)

### 新机部署

1. 这个仓库里有什么

```
security-dashboard/
├── README.md                          # 本指南（英文版），与 README-zh.md 内容相同
├── README-zh.md                       # 本指南
├── dashboard-init.py.template               # 生成器模板（看板的核心）→ dashboard-init.py
├── push-host-data.py.template               # host 2 推送脚本模板 → push-host-data.py
├── refresh-security-data.py.template  # 30 分钟刷新脚本模板 → refresh-security-data.py
├── dashboard.cron.template            # 两个定时任务模板 → 装进 cron
├── nginx-dashboard.conf.template      # nginx 配置模板：封禁名单 include + 看板站点 server 段
├── index.html.template                # 看板页面模板 → __DASHBOARD_DIR__ 里的 index.html
├── .dashboard-rules.json.template     # 规则文件模板 → __RULES_PATH__ 的 .dashboard-rules.json
└── assets/                            # 看板图片（6 张：金鱼、空缸红叉、奖牌）→ __DASHBOARD_DIR__
    ├── fish-sprite.webp
    ├── fish-zero.webp
    ├── ie-redx.png
    ├── medal-gold.png
    ├── medal-silver.png
    └── medal-bronze.png
```

模板里的占位符都要填成新主机的真实值，见第 3 节。填完之前这些文件都不能直接跑。模板里没有任何真实密码和真实路径，可以放心放着。

状态文件（`.autoban-state.json`、`.autoban-7d.json`、`.dashboard-resolutions.json`）不在这个目录里：全新部署时它们会自动重建（见第 8 节）；从旧主机迁移时再从旧主机拷过来。看板页 `index.html`本身就是这个目录里的 `index.html.template`（见第 5 节第 3 步）。图片已在 `assets/` 里，不用从别处拷。

注意：`dashboard-init.py.template` 是旧主机 `dashboard-init.py` 在某一时刻的快照。如果生成器后来改了，要重新生成本模板（或直接拷新版 `dashboard-init.py`， 按第 3 节的表把路径常量改掉）。

2. 架构速览

```
nginx 访问日志（每站点一个 <site>.log）──┐
SSH 日志（/var/log/secure 或 auth.log）──┼─→ dashboard-init.py ─→ s-<key>.json ─→ index.html（静态页）
fail2ban 日志（可选）───────────────┘         │                 │
                                              ├─→ nginx blocklist.conf（autoban，改完自己 nginx -t 再 reload）
                                              └─→ 日报邮件（Sohu SMTP，07:00 全量时发）
```

- `index.html` 是纯静态页，凭 `?key=` 读 `s-<key>.json`。没有 key 或 key 不对时显示演示数据；`?key=0` 显示出错页。
- `dashboard-init.py` 是生成器，只用 Python 标准库（Python 3.6+ 都能跑）。直接跑它就是 07:00 全量：换 key、重新生成数据、发日报邮件。
- `refresh-security-data.py` 每 30 分钟跑一次，只刷新 `s-<key>.json`，不换 key、不发邮件。

3. 占位符表

填模板时按这个表换。占位符按所在文件分组。**必填**是程序必须有真实值；**可选**是没值也可以留着不填——程序照样跑，那一项只是永远匹配不上（网段占位符不填会退到一个不承载真实流量的文档网段）。没有某个值时就留着，别凭空编一个。

#### 安全看板生成脚本

**dashboard-init.py.template**（此文件部署在要展示安全看板的主机`host1`上）

| 占位符 | 是什么 | 必填？ | 新主机填什么 |
|---|---|---|---|
| `__DASHBOARD_DIR__` | 看板目录（放 `index.html` 和 `s-<key>.json` 的地方） | 必填 | 如 `/www/wwwroot/example.com/dashboard` |
| `__LOG_DIR__` | nginx 访问日志目录 | 必填 | 新主机放站点日志的目录；每个站点一个 `<site>.log`，不要 `.error.log` |
| `__RULES_PATH__` | 判定规则文件 | 必填 | 规则文件的实际路径（建议放 root 家目录或同级位置） |
| `__BLOCKLIST_CONF__` | nginx 封禁名单（生成器这边看到的） | 必填 | blocklist.conf 的路径；必须和下面 nginx 配置里的 `__BLOCKLIST_PATH__` 是同一个文件 |
| `__HOST2_JSON_PATH__` | 另一台主机的推送数据 | 可选 | 还监控第二台主机时才填：推送文件在本机的路径，如 `/home/<接收账号>/host-host2.json`。只监控本机时删掉生成器里读它的调用（见第 9 节） |
| `__PUSH_USER__` | host 1 上接收账号的用户名 | 可选 | host 2 推送数据时用的 SSH 用户（接收账号）。只在多主机推送时需要；只监控一台主机时留着 |
| `__HOSTSEC_STATE_PATH__` | 主机安全状态 | 必填 | 实际路径（建议跟生成器放一起）；全新部署时会重建（见第 8 节） |
| `__HOSTSEC_IP_BASELINE_PATH__` | 主机 IP 基线 | 必填 | 同上（全新部署时会重建，见第 8 节） |
| `__USUALLY_KEY_PATH__` | 常用密钥归属地基线 | 必填 | 同上（全新部署时会重建，见第 8 节） |
| `__RESOLUTIONS_PATH__` | 已处理事件登记 | 必填 | 实际路径（建议跟生成器放一起） |
| `__AUTOBAN_STATE_PATH__` | 封禁记录 | 必填 | 实际路径（建议跟生成器放一起） |
| `__AUTOBAN_7D_STATE_PATH__` | 7 天封禁计数 | 必填 | 同上 |
| `__SMTP_HOST__` | 邮件服务器地址 | 必填 | 日报邮件的 SMTP 服务器（模板默认 Sohu SMTP，换别的邮箱服务商时改这里） |
| `__SMTP_SENDER__` | 发件邮箱 | 必填 | 新主机发日报邮件用的邮箱（SMTP 账号） |
| `__SMTP_RECEIVER__` | 收件邮箱 | 必填 | 接收日报邮件的邮箱 |
| `__SMTP_PASSWORD__` | 发件邮箱密码 | 必填 | 发件邮箱的密码，不要外泄 |
| `__HOME_IP__` | 站主本人 IP | 可选 | 站主常用的公网 IP（单个地址），主机栏“已知来源”用。没有固定 IP 就留着 |
| `__HOME_NET__` | 站主家庭网段 | 可选 | 站主家庭宽带网段（CIDR），面板登录“已知来源”用。这是 `__HOME_IP__` 的 /24；没有固定家庭 IP 时两个都留着 |
| `__HOST_PUBLIC_IP__` | 本机公网 IP | 必填 | 本机自己的公网 IP（单个地址）；面板/SSH 有时会记下本机自己从公网 IP 登录，那不是陌生人 |
| `__EGRESS_NET_1__` | 助手出口网段 1 | 可选 | 助手工具出口的网段（CIDR）；有多个按顺序填，`__EGRESS_NET_2__` 同理。没有助手工具连这台主机时两个都留着 |
| `__EGRESS_NET_2__` | 助手出口网段 2 | 可选 | 同上 |
| `__LAN_NET__` | 站主局域网段 | 可选 | 站主局域网网段（CIDR），永不封禁的安全底线。这台主机没有局域网时留着 |
| `__LAN_PREFIX__` | 局域网 IP 前缀 | 可选 | 站主局域网的 IP 前缀（如 `172.32.`），生成器用它识别内网 IP、不做归属地查询。这台主机没有局域网时留着 |
| `__TAILSCALE_NET__` | Tailscale 组网网段 | 可选 | Tailscale 的网段（CIDR）。不用 Tailscale 时留着 |
| `__ZEROTIER_NET__` | ZeroTier 组网网段 | 可选 | 组网（ZeroTier）的网段（CIDR）。不用组网时留着 |
| `__YANG_TAILSCALE_IP__` | Yang 的 Tailscale IP | 可选 | Yang 节点的 Tailscale IP；把它发来的 SSH 登录归到 "yang" 密钥标记。没有这个节点时留着 |
| `__PEPPER_TAILSCALE_IP__` | Pepper 的 Tailscale IP | 可选 | Pepper 节点的 Tailscale IP；把它发来的 SSH 登录归到 "pepper" 密钥标记。没有这个节点时留着 |

#### 其他机器主机安全推送脚本

**push-host-data.py.template**（放置在`host2`）

| 占位符 | 是什么 | 必填？ | 新主机填什么 |
|---|---|---|---|
| `__PUSH_SSH_KEY__` | 推送主机数据的 SSH 密钥 | 必填 | host 2 上能 scp 到看板主机（host 1）的私钥；对应的公钥放到 host 1 的接收账号里 |
| `__SSH_PORT__` | SSH 端口 | 必填 | 看板主机（host 1）的 SSH 端口 |
| `__HOST2_PUSH_DEST__` | 推送目的地 | 必填 | host 2 推送数据的位置：`user@host1主机:文件名`（scp 到 host 1 的接收账号家目录） |
| `__PUSH_USER__` | host 1 上接收账号的用户名 | 可选 | 同上；只监控一台主机时留着 |
| `__HOME_NET__` | 站主家庭网段 | 可选 | 同上；没有固定家庭 IP 时留着 |
| `__LAN_NET__` | 站主局域网段 | 可选 | 同上；host 2 没有局域网时留着 |
| `__LAN_PREFIX__` | 局域网 IP 前缀 | 可选 | 同上 |
| `__TAILSCALE_NET__` | Tailscale 组网网段 | 可选 | 不用 Tailscale 时留着 |
| `__ZEROTIER_NET__` | ZeroTier 组网网段 | 可选 | 不用组网时留着 |
| `__YANG_TAILSCALE_IP__` | Yang 的 Tailscale IP | 可选 | 没有这个节点时留着 |
| `__PEPPER_TAILSCALE_IP__` | Pepper 的 Tailscale IP | 可选 | 没有这个节点时留着 |

#### 负责定时轮换看板数据的脚本

**refresh-security-data.py.template** （此文件部署在要展示安全看板的主机`host1`上）

| 占位符 | 是什么 | 必填？ | 新主机填什么 |
|---|---|---|---|
| `__DASHBOARD_DIR__` | 看板目录 | 必填 | 同上 |
| `__DASHBOARD_INIT_PY__` | 生成器路径 | 必填 | 新主机上 `dashboard-init.py` 的实际路径，如 `/root/dashboard-init.py` |
| `__REFRESH_SCRIPT__` | 本脚本的路径 | 必填 | 新主机上 `refresh-security-data.py` 的实际路径 |

#### 定时任务

**dashboard.cron.template** （此文件部署在要展示安全看板的主机`host1`上）

| 占位符 | 是什么 | 必填？ | 新主机填什么 |
|---|---|---|---|
| `__SCRIPT_DIR__` | 生成器所在目录 | 必填 | 新主机上放 `dashboard-init.py` 的目录，如 `/root` |
| `__REFRESH_SCRIPT__` | 30 分钟刷新脚本路径 | 必填 | 同上 |
| `__BAN_WATCH_LOG__` | 30 分钟刷新的日志 | 必填 | 任意可写路径 |
| `__DAILY_LOG__` | 07:00 全量的日志 | 必填 | 任意可写路径 |

#### 看板设置文件

**nginx-dashboard.conf.template** （此文件部署在要展示安全看板的主机`host1`上）

| 占位符 | 是什么 | 必填？ | 新主机填什么 |
|---|---|---|---|
| `__BLOCKLIST_PATH__` | nginx 封禁名单（nginx 这边看到的） | 必填 | blocklist.conf 的路径；必须和上面生成器里的 `__BLOCKLIST_CONF__` 是同一个文件 |
| `__DASHBOARD_DOMAIN__` | 看板域名 | 必填 | 新主机的看板域名，如 `example.com` |
| `__SITE_ROOT__` | 看板所在站点的根目录 | 必填 | 新主机该站点的根目录（dashboard 是其下的子目录） |

#### 网段白名单

**.dashboard-rules.json.template** （此文件部署在要展示安全看板的主机`host1`上）

| 占位符 | 是什么 | 必填？ | 新主机填什么 |
|---|---|---|---|
| `__LAN_NET__` | 站主局域网段 | 可选 | 同上 |
| `__HOME_IP__` | 站主本人 IP | 可选 | 同上；没有固定 IP 就留着 |
| `__EGRESS_NET_1__` | 助手出口网段 1 | 可选 | 同上 |
| `__EGRESS_NET_2__` | 助手出口网段 2 | 可选 | 同上 |
| `__TAILSCALE_NET__` | Tailscale 组网网段 | 可选 | 不用 Tailscale 时留着 |

### index.html.template

唯一的占位符是 `__SAMPLE__`（演示数据），不用改：演示用的是 `example.com` 系列假数据和随机私有 IP，不含真实地址。

### 写死的路径（不是占位符）

下面这些路径是直接写在模板里的，不是占位符。只有主机情况不一样时 才去文件里改：

- SSH 日志：两个脚本都读 `/var/log/secure`（RHEL 系）。Debian/Ubuntu 上把两处都改成 `/var/log/auth.log`。
- 面板登录库：生成器读 `/www/server/panel/data/db/log.db`（宝塔）。没有宝塔 时代码能容忍缺失，面板登录那栏会是空的。
- nginx 可执行文件：生成器调 `/www/server/nginx/sbin/nginx`。nginx 在别处 时改 `_NGINX_BIN`。

邮件服务器端口不用改，模板里已经是现成的（Sohu SMTP 465）。换用别的 邮箱服务商时，再改模板里的 `SMTP_SERVER` / `SMTP_PORT` / `IMAP_SERVER`。

### index.html.template 里要改的地方

模板里的站点和主机名都是示例，部署时按下面改成自己的：

- `SITE_DOTS`（站点颜色表）：把 `example.com`、`shop.example.com`、`api.example.com`、`blog.example.com`、`docs.example.com`、`support.example.com` 换成自己要监控的站点域名，颜色可留着或自己改。
- `SITE_LABELS`（站点短名表）：把 `api.example.com`、`docs.example.com` 换成自己站点的域名和短名。
- 主机名：模板里主机显示名是 `Host 1` 和 `Host 2`（`hostSection` 里的 `var name`）。改成自己的主机名，不要用真实主机名以外的信息（如 IP）。
- 演示数据不用改：模板里的演示（`__SAMPLE__`）用的是 `example.com` 系列假数据和随机私有 IP，不含真实地址。

### 监控多个站点

看板显示的是 nginx 访问日志里出现过的所有站点（见 `__LOG_DIR__`），生成器里没有要配的站点列表。想给某个站点配颜色和短名，把它的域名加到 `index.html.template` 的 `SITE_DOTS` / `SITE_LABELS` 里。

4. 新主机先决条件

- Python 3（3.6+ 都行；代码是按 3.6 写的，新版 Python 也能跑）。
- nginx（或任何能 serving 静态文件的 web 服务器）+ 一个看板域名。
- nginx 访问日志：每个站点一个文件，命名 `<site>.log`（站点名就是 文件名去掉 `.log`），放在 `__LOG_DIR__` 里。日志格式必须是 nginx combined（`_LOG_RE` 要的字段：IP、时间、方法、路径、状态码、 大小、UA）。如果新主机的日志格式不一样，要改生成器里的 `_LOG_RE`。
- SSH 日志（主机安全栏要用）：脚本读 `/var/log/secure`（见第 3 节 “写死的路径”）。
- fail2ban（可选）：没有它主机栏的 fail2ban 统计就是 0，不影响其他。
- 生成器和刷新脚本要以能读日志、能写看板目录和 blocklist 的身份跑 （旧主机上是 root 跑 cron）。

5. 部署步骤（按顺序）

全新主机上按下面一步步走。

1. 新主机装好 Python 3、nginx，按第 4 节配好 vhost 和日志格式。
2. 从这个目录把模板拷到新主机，填好第 3 节的占位符，去掉 `.template` 后缀：
   - `dashboard-init.py.template` → `__SCRIPT_DIR__` 当 `dashboard-init.py` （如 `/root/dashboard-init.py`）
   - `refresh-security-data.py.template` → `__REFRESH_SCRIPT__`
   - `dashboard.cron.template` 的两行 → root 的 crontab（`crontab -e`）
   - `nginx-dashboard.conf.template` 的两段 → `nginx.conf` 的 http 段 和站点配置
3. 从本目录拷看板页模板 `index.html.template`，按第 3 节末尾改好站点和主机名， 去掉 `.template` 后缀放到 `__DASHBOARD_DIR__` 当 `index.html`；再从本目录 `assets/` 拷 6 张图片（`fish-sprite.webp`、`fish-zero.webp`、`ie-redx.png`、 `medal-gold.png`、`medal-silver.png`、`medal-bronze.png`）到同一目录。 `s-<key>.json` 不用拷，新主机会自己生成。
4. 配规则文件。从本目录拷 `.dashboard-rules.json.template`，填好里面的 占位符（`__LAN_NET__`、`__HOME_IP__`、`__EGRESS_NET_1__`、 `__EGRESS_NET_2__`、`__TAILSCALE_NET__`），去掉 `.template` 后缀放到 `__RULES_PATH__` 当 `.dashboard-rules.json`。注意：如果这个文件没配好或坏了， 生成器会写未知看板（这是 2026-11-26 定下的无兜底规则），但封禁不停。 所以先确认规则文件就位再验证。
5. 状态文件：不用拷，生成器会重建（见第 8 节）。
6. nginx 里 include 封禁名单（模板第 1 段），`nginx -t` 通过后 reload。
7. 手动跑一次全量验证：`cd __SCRIPT_DIR__ && python3 dashboard-init.py`， 确认 `__DASHBOARD_DIR__` 里生成了新的 `s-<key>.json`，用新 key 的 链接打开看板有数据，并且收到日报邮件。
8. 手动跑一次 30 分钟刷新验证：`__REFRESH_SCRIPT__`，确认 `generated_at` 变了。
9. 有第二台主机时接跨主机线（第 9 节）。
10. 观察一个完整统计时段（07:00→22:00），确认评级、封禁、主机三栏 都正常，再切 DNS。

6. 部署后检查清单

- [ ] 新 key 的链接能打开看板，评级、攻击数、攻击者、事件都有数据。
- [ ] `s-<key>.json` 的 `generated_at` 在变（30 分钟刷新在工作）。
- [ ] 07:00 收到日报邮件，邮件里的链接能打开。
- [ ] IP 封禁列表与 nginx `blocklist.conf` 一致。
- [ ] 主机栏有数据；头一两天的陌生登录提示逐个确认后不再出现。
- [ ] 规则文件就位（看板不是未知页）。

### 从旧主机迁移

7. 迁移步骤（从旧主机迁过来）

把看板从旧主机迁到新主机。按第 5 节走，不同的地方如下：

1. 先决条件和模板：跟第 5 节第 1–3 步一样。
2. 规则文件：把旧主机的 `.dashboard-rules.json` 拷到 `__RULES_PATH__` （不用这个目录里的 `.dashboard-rules.json.template`）。
3. 状态文件：把旧主机上第 8 节里写着“拷”的那三个拷过来（不用等 生成器重建）。
4. 邮箱密码：把旧主机 `dashboard-init.py` 里的 `SENDER_PASSWORD` 原样 拷进 `__SMTP_PASSWORD__`，不要重设、不要外泄。
5. nginx、验证跑一遍、跨主机线：跟第 5 节第 6–9 步一样。
6. 观察一个完整统计时段（07:00→22:00），确认评级、封禁、主机三栏 都正常，再切 DNS、停旧主机。旧主机不要停太早。

8. 状态文件：拷哪些、哪些要重建

| 文件 | 处理 | 为什么 |
|---|---|---|
| `.autoban-state.json` | 拷 | 封禁记录；不拷则封禁历史丢失 |
| `.autoban-7d.json` | 拷 | 7 天累计计数；不拷则计数清零，重复攻击者要重新累计到 100 才封 |
| `.dashboard-resolutions.json` | 拷 | 已处理事件登记；不拷则已处理的事件会重新冒出来当“需留意” |
| `.hostsec-state.json` | 不拷，重建 | 机器自身 SSH 状态的基线；新机器的 SSH 主机密钥/指纹不同，直接拷会误报 |
| `.hostsec-ip-baseline.json` | 不拷，重建 | 同上 |
| `usually-key.json` | 不拷，重建 | 常用密钥归属地基线；新主机上跑几次自然重建（旧主机上目前没有这个文件） |
| `.dashboard-init.lock` | 不拷 | 锁文件，新主机会自己建 |
| `s-<key>.json` | 不拷 | 每次只留一个，轮换时删掉；新主机会自己生成 |

主机基线重建期间，头一两天可能出现陌生登录提示，逐个确认即可， 之后就正常了。

9. 两台主机：host 2 怎么把数据推给 host 1

看板跑在 host 1 上，但也可以同时显示第二台主机（host 2）。设计上就是两台主机，第三台要改代码才能加。两台主机跑 **不同的脚本**：host 1 跑 `dashboard-init.py`（生成器），host 2 跑 `push-host-data.py`（一个更小的脚本，只采集 host 2 自己本机的主机安全 数据、推给 host 1）。

**推送是怎么跑的。** host 2 上 root 的 crontab 每天 06:30 跑一次 推送脚本：

```
30 6 * * * /usr/bin/python3 /root/push-host-data.py --push-host >> /root/host-push.log 2>&1
```

`--push-host` 只采集 host 2 本机的主机安全数据（SSH 登录、面板登录、 密钥变动），走接收账号的 SSH 通道 scp 到 host 1 的 `__HOST2_JSON_PATH__`（如 `/home/<接收账号>/host-host2.json`，在接收 账号的家目录里）。不发邮件。host 1 早上 07:00 生成看板时读这个文件， 把 host 2 那行合并进主机栏。推送日志在 host 2 的 `/root/host-push.log`。 同一个 `--push-host` 在 30 分钟刷新脚本里（第 5 节第 2 步）每 30 分钟 （06:30–22:00）也会跑一次，所以 host 2 的数据全天都保持新鲜。

**新部署时怎么接。** host 2 上拷 `push-host-data.py.template`，把占位符 （网段占位符和推送占位符 `__PUSH_SSH_KEY__`、`__SSH_PORT__`、 `__HOST2_PUSH_DEST__`）填好，再加上面那行 06:30 的 cron。host 1 上把 `__HOST2_JSON_PATH__` 填成推送文件的路径。只监控一台主机时，推送整个 不用接，把生成器里读 `__HOST2_JSON_PATH__` 的调用（`_read_host2`）删掉， 主机栏就只剩本机。

- **host 2 → 看板主机（30 分钟刷新触发）**：host 2 每 30 分钟远程触发 host 1 刷新。新主机上直接用本地 cron 跑 30 分钟刷新（第 5 节第 2 步） 即可，不必再走 host 2 远程触发。

10. 迁移后检查清单

- [ ] 新 key 的链接能打开看板，评级、攻击数、攻击者、事件都有数据。
- [ ] `s-<key>.json` 的 `generated_at` 在变（30 分钟刷新在工作）。
- [ ] 07:00 收到日报邮件，邮件里的链接能打开。
- [ ] IP 封禁列表与 nginx `blocklist.conf` 一致。
- [ ] 主机栏有数据；头一两天的陌生登录提示逐个确认后不再出现。
- [ ] 规则文件就位（看板不是未知页）。

11. 注意事项

- key 每天 07:00 轮换，旧 key 的链接轮换后打不开（会显示演示数据）， 这是正常的；日报邮件里的链接每天都是新的。
- `s-<key>.json` 每次只留一个，旧的轮换时删掉，不要手动去删/拷它。
- nginx 444 保持原样，不要改成 403（2026-11-26 已定）；444 的探测 会继续算事件，结论是未得手。
- 规则文件丢了或坏了时看板显示未知，但封禁不停（只停 7 天计数 更新），主机评级也不受影响。
- 模板是快照：`dashboard-init.py` 改了之后要重新生成模板（或直接拷新版 再改路径常量），不要拿旧模板去套新版生成器。
