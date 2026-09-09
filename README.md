# WireGuard Manager V2.0

**中文** ｜ [English](#english)

> 把 WireGuard 从"一个越来越大的 Bash 脚本"升级成"可以长期维护的管理平台"：
> Bash 核心负责系统级配置，Python 层提供 Web 面板 / JSON API / 状态采集 /
> 流量历史 / Health Check / 告警引擎。**面板挂了，WireGuard 照常运行。**

一个面向 VPS / Docker 宿主机 / PVE 宿主机 / 多机 Site-to-Site 场景的一键式
WireGuard 管理面板（CLI + Web）。V2.0 在 V1.3 纯命令行的基础上，加了一整层
可观测性，同时严格保持"状态层与配置层分离"——所有派生数据都可以丢，
WireGuard 隧道本身不受任何影响。

完整文档见 **[WireGuard-Manager-V2.0-面板搭建教程.md](WireGuard-Manager-V2.0-面板搭建教程.md)**（中文，17 节）。

---

## 特性

- **Web 面板 + JSON API** —— 结构化状态总览、Peer 详情、二维码/配置下载，
  不必再 SSH 进去敲 `wg show`。前端和第三方集成（手机 / Grafana / Bot）调同一套 `/api/v1`。
- **Peer 五态模型** —— `online` / `idle` / `offline` / `never` / `disabled`，
  比"在线/离线"两态直观得多。
- **流量历史** —— 采集进程周期采样，支持 24h / 7d / 30d 汇总、按 Peer 汇总、
  ASCII 趋势图，保留 90 天。
- **Health Check** —— 分级别（pass/info/warn/error）的结构化体检，
  每项告警都带"下一步该查什么"的具体建议。
- **告警引擎** —— Peer 掉线自动推送，支持 **Telegram / Bark / 企业微信 / 钉钉 /
  通用 Webhook** 五种渠道，带防抖（连续 N 次确认）与恢复通知。
- **纯标准库 Python** —— 不依赖 `pip` / venv，Python 3.8+ 即可。网络基础设施
  绝不能因为一个第三方包挂了而跟着挂。
- **继承 V1.3 的全部能力** —— 服务端/客户端/Site-to-Site、幂等防火墙重建、
  多跳路由、密钥轮换、自动快照 + 全量备份、IPv4/IPv6、NAT/NAT66、
  Docker/PVE 环境检测、flock 单实例锁、操作审计日志。

## 架构

```text
             wireguard-manager-v2.0.sh   （Bash 核心 = 配置真相来源）
                       │
         ┌─────────────┴─────────────┐
         │                           │
   Bash Core                    Python 层
 （配置/渲染/防火墙/           ┌────────┴────────┐
   路由/备份/CLI）        采集守护进程         Web 面板
         │               （root 运行）     （降权 wgmgr-web）
         ▼                     │                  │
  /etc/wireguard/              ▼                  ▼
  /etc/wireguard-manager/  state/*.json  ◀───读────┘
                               │
                               └──写操作经白名单 CLI + sudo -n──▶ /usr/local/bin/wgmgr
```

三条铁律：

1. **配置真相来源**永远是 Bash 核心管的 `manager.conf` + `clients/*/meta.conf`
   + `sites/*/meta.conf`；`wg0.conf` 是派生产物，删了能重新渲染。
2. **状态层（`state/*.json`）是纯派生数据**，丢了/坏了/陈旧了都不影响 WireGuard。
3. **Web 面板一行 `wg`/`nft`/`ip` 都不执行**：读来自 `state/`，写只走一份显式
   白名单 CLI。浏览器传来的字符串永远不会被拼进 shell。

## 安全模型

- **面板降权运行**：以 `wgmgr-web` 账号跑，不是 root。`server/` `clients/` `sites/`
  保持 `700 root:root`，面板**读不到任何私钥**。
- **sudoers 只放行一个绝对路径**：`/etc/sudoers.d/wireguard-manager-web` 里只有
  `NOPASSWD: /usr/local/bin/wgmgr`，安装前经 `visudo -cf` 校验。
- **双层命令白名单**：Python 侧 `ALLOWED_CLI` 限定可调子命令，`run_cli` 再拒绝
  含 shell 元字符的参数值。服务端级危险操作（`server restart` / `key rotate-server`
  / `fw clean` / `uninstall`）面板一律不提供。
- **Session Cookie**（`HttpOnly` + `SameSite=Strict`，非 URL token）、登录 IP 限速、
  恒定时间用户名比对、scrypt 加盐口令哈希、CSP 等安全响应头。
- **setgid 状态目录**：`state/` 为 `2750 root:wgmgr-web`，root 采集进程写的文件
  自动继承组属、组可读，面板无需提权即可展示。

## 环境要求

| 项目 | 要求 |
|---|---|
| 操作系统 | Debian 10+ / Ubuntu 20.04+ |
| 权限 | Bash 核心必须 root（或全程 sudo） |
| Python | 3.8+（仅标准库） |
| systemd | Web 面板与采集进程以 systemd 服务运行 |
| sudo | 降权面板通过 `sudo -n` 提权 |
| 依赖 | `wireguard-tools`、`iproute2`、`curl`、`qrencode`（可选，二维码用） |

**不适用**：CentOS/RHEL 系、OpenWrt 等无 systemd 的嵌入式系统（这些环境仍可用纯 CLI）。

## 快速开始

```bash
# 1. 把脚本连同同目录的 web/ 和 systemd/ 一起放到安装目录（三者相对位置不能变）
mkdir -p /opt/wg-manager && cd /opt/wg-manager
#    上传 wireguard-manager-v2.0.sh + web/ + systemd/
chmod +x wireguard-manager-v2.0.sh
ln -sf /opt/wg-manager/wireguard-manager-v2.0.sh /usr/local/bin/wgmgr

# 2. 进面板，先把 WireGuard 本身跑通
sudo wgmgr
#    → 2. 服务端 → 1. 初始化 / 重新配置服务端
#    → 3. 客户端 / 4. Site-to-Site …

# 3. 确认隧道能用之后，再装 Web 面板
#    → 12. Web 面板 → 1. 安装（选绑定范围 vpn/local/public、端口、账号）
```

绑定范围三选一：**`vpn`（推荐，只有连上 WireGuard 的设备能打开）** / `local`
（绑 127.0.0.1，配合 SSH 隧道或反代）/ `public`（绑 0.0.0.0，危险，需自备 HTTPS 反代）。

日常运维：

```bash
sudo systemctl status  wireguard-manager-collector   # 采集守护进程（root）
sudo systemctl status  wireguard-manager-web         # Web 面板（wgmgr-web）
sudo journalctl -u wireguard-manager-web -f

sudo wgmgr health --json                              # 结构化体检
sudo wgmgr traffic --range 7d                         # 流量历史
sudo wgmgr peer list --json | jq '.peers[] | select(.status=="offline")'
```

非交互式 CLI 与 Web 面板共用同一套核心逻辑：菜单里能做的事几乎都能用一条
`wgmgr <命令>` 做完，而面板只会调用这份白名单 CLI。完整命令见 `sudo wgmgr help`。

## 从 V1.3 升级

V1.3 与 V2.0 用**同一个配置目录、同一套元数据结构**，客户端/站点/密钥/路由/备份
全部原地保留，不用重建、不用重新分发配置。升级 = 换脚本 + 加 Web 层：

```bash
sudo wgmgr backup create                              # 保险起见先全量备份
sudo ln -sf /opt/wg-manager/wireguard-manager-v2.0.sh /usr/local/bin/wgmgr
sudo wgmgr                                            # WireGuard 数据照常认
#    → 12. Web 面板 → 1. 安装                          # 新增的一步
```

不想用 Web 面板也可以只换脚本、跳过 `web install`——V2.0 的命令行能力是 V1.3 的超集。
详见教程[第 16 节](WireGuard-Manager-V2.0-面板搭建教程.md)。

## 目录结构

```text
wireguard-manager-v2.0.sh          # Bash 核心（安装/渲染/防火墙/路由/备份/CLI/菜单）
web/                               # Python 层（仅标准库）
├── wgm_common.py                  #   路径 / 原子写 / CLI 白名单 / run_cli
├── wgm_collector.py               #   采集守护进程
├── wgm_web.py                     #   HTTP 服务 + JSON API + 静态面板
├── wgm_health.py                  #   Health Check 引擎
├── wgm_traffic.py                 #   流量采样 / 汇总 / 趋势图
├── wgm_alert.py                   #   告警引擎（5 渠道）
└── static/                        #   前端（index.html + app.js + style.css）
systemd/
├── wireguard-manager-collector.service   # 采集服务（root）
├── wireguard-manager-web.service         # 面板服务（降权 wgmgr-web）
└── wireguard-manager-web.sudoers         # 提权规则模板（只放行 wgmgr）
WireGuard-Manager-V2.0-面板搭建教程.md     # 完整中文教程（17 节）
```

## 已知限制

- IPv6 仍是"分一段 ULA + 出网 NAT66"的粗糙方案，未做 PD 委派 / RA / DNS64。
- Web 面板与采集进程依赖 systemd；无 systemd 环境只能用纯 CLI（或塞进 cron）。
- 单机单实例模型，未做"一个面板管多台机器"的聚合视图。
- 流量历史从装上采集进程那刻才开始积累，补不出之前的数据。
- API 认证为单用户 + Session Cookie，未做多用户 RBAC / OAuth。

详见教程[第 17 节](WireGuard-Manager-V2.0-面板搭建教程.md)。

## 许可证

[MIT](LICENSE) © 2026 plnl

---

<a id="english"></a>
# WireGuard Manager V2.0 (English)

> Turn WireGuard from "an ever-growing Bash script" into "a platform you can
> maintain long-term": a Bash core handles system-level configuration, while a
> Python layer provides a Web panel / JSON API / state collector / traffic
> history / health check / alert engine. **If the panel dies, WireGuard keeps running.**

A one-stop WireGuard management panel (CLI + Web) for VPS / Docker hosts / PVE
hosts / multi-machine Site-to-Site setups. On top of V1.3's pure-CLI foundation,
V2.0 adds a full observability layer while strictly keeping **state and
configuration separate** — every piece of derived data is disposable; the
WireGuard tunnel itself is never affected.

Full documentation: **[WireGuard-Manager-V2.0-面板搭建教程.md](WireGuard-Manager-V2.0-面板搭建教程.md)** (Chinese, 17 sections).

## Features

- **Web panel + JSON API** — structured status overview, peer details,
  QR-code/config download; no more SSHing in to run `wg show`. The frontend and
  third-party integrations (mobile / Grafana / bots) all call the same `/api/v1`.
- **5-state peer model** — `online` / `idle` / `offline` / `never` / `disabled`,
  far clearer than a binary online/offline.
- **Traffic history** — the collector samples periodically; 24h / 7d / 30d
  aggregates, per-peer breakdown, ASCII trend chart, 90-day retention.
- **Health check** — leveled (pass/info/warn/error) structured diagnostics; every
  warning ships with a concrete "what to check next".
- **Alert engine** — auto-push when a peer drops, via **Telegram / Bark / WeCom /
  DingTalk / generic Webhook**, with debounce (N consecutive confirmations) and
  recovery notices.
- **Standard-library-only Python** — no `pip` / venv, Python 3.8+ is enough.
  Network infrastructure must never break because a third-party package broke.
- **Everything from V1.3** — server/client/Site-to-Site, idempotent firewall
  rebuild, multi-hop routes, key rotation, auto snapshots + full backups,
  IPv4/IPv6, NAT/NAT66, Docker/PVE detection, flock single-instance lock,
  operation audit log.

## Architecture

```text
             wireguard-manager-v2.0.sh   (Bash core = source of truth)
                       │
         ┌─────────────┴─────────────┐
         │                           │
     Bash core                  Python layer
 (config/render/firewall/      ┌────────┴────────┐
  route/backup/CLI)        collector daemon     web panel
         │                 (runs as root)   (dropped to wgmgr-web)
         ▼                     │                  │
  /etc/wireguard/              ▼                  ▼
  /etc/wireguard-manager/  state/*.json  ◀───read───┘
                               │
                               └──writes via whitelisted CLI + sudo -n──▶ /usr/local/bin/wgmgr
```

Three iron rules:

1. The **source of truth** is always the Bash core's `manager.conf` +
   `clients/*/meta.conf` + `sites/*/meta.conf`; `wg0.conf` is derived and can be
   re-rendered if deleted.
2. The **state layer (`state/*.json`) is pure derived data** — losing it, corrupting
   it, or letting it go stale never affects WireGuard.
3. The **web panel never runs a single `wg`/`nft`/`ip`**: reads come from `state/`,
   writes go only through an explicit CLI whitelist. Strings from the browser are
   never concatenated into a shell.

## Security model

- **Dropped-privilege panel**: runs as `wgmgr-web`, not root. `server/`, `clients/`,
  `sites/` stay `700 root:root`, so the panel **cannot read any private key**.
- **sudoers allows a single absolute path**: `/etc/sudoers.d/wireguard-manager-web`
  contains only `NOPASSWD: /usr/local/bin/wgmgr`, validated with `visudo -cf` before install.
- **Two-layer command whitelist**: `ALLOWED_CLI` on the Python side restricts which
  subcommands may run; `run_cli` also rejects argument values containing shell
  metacharacters. Server-level destructive ops (`server restart` / `key rotate-server`
  / `fw clean` / `uninstall`) are never exposed to the panel.
- **Session cookie** (`HttpOnly` + `SameSite=Strict`, not a URL token), login IP
  throttling, constant-time username comparison, scrypt-salted password hash, CSP
  and other security headers.
- **setgid state dir**: `state/` is `2750 root:wgmgr-web`, so files written by the
  root collector inherit the group and become group-readable — the panel displays
  them without escalating.

## Requirements

| Item | Requirement |
|---|---|
| OS | Debian 10+ / Ubuntu 20.04+ |
| Privilege | Bash core must run as root (or fully via sudo) |
| Python | 3.8+ (standard library only) |
| systemd | Web panel and collector run as systemd services |
| sudo | The dropped-privilege panel escalates via `sudo -n` |
| Deps | `wireguard-tools`, `iproute2`, `curl`, `qrencode` (optional, for QR) |

**Not supported**: CentOS/RHEL, OpenWrt or other systemd-less embedded systems
(those can still use the pure CLI).

## Quick start

```bash
# 1. Place the script together with its sibling web/ and systemd/ dirs (keep relative layout)
mkdir -p /opt/wg-manager && cd /opt/wg-manager
#    upload wireguard-manager-v2.0.sh + web/ + systemd/
chmod +x wireguard-manager-v2.0.sh
ln -sf /opt/wg-manager/wireguard-manager-v2.0.sh /usr/local/bin/wgmgr

# 2. Open the menu and get WireGuard itself working first
sudo wgmgr
#    → 2. Server → 1. Initialize / reconfigure server
#    → 3. Clients / 4. Site-to-Site …

# 3. Once the tunnel works, install the web panel
#    → 12. Web panel → 1. Install (pick bind scope vpn/local/public, port, account)
```

Bind scope, pick one: **`vpn` (recommended — only devices on the WireGuard tunnel
can reach it)** / `local` (bind 127.0.0.1, use an SSH tunnel or reverse proxy) /
`public` (bind 0.0.0.0, risky, bring your own HTTPS reverse proxy).

Day-to-day ops:

```bash
sudo systemctl status  wireguard-manager-collector   # collector daemon (root)
sudo systemctl status  wireguard-manager-web         # web panel (wgmgr-web)
sudo journalctl -u wireguard-manager-web -f

sudo wgmgr health --json                              # structured health check
sudo wgmgr traffic --range 7d                         # traffic history
sudo wgmgr peer list --json | jq '.peers[] | select(.status=="offline")'
```

The non-interactive CLI and the web panel share one core: almost anything the menu
does can be done with a single `wgmgr <command>`, and the panel only ever calls this
whitelisted CLI. Full command list: `sudo wgmgr help`.

## Upgrading from V1.3

V1.3 and V2.0 use **the same config directory and the same metadata layout**, so
clients/sites/keys/routes/backups are all preserved in place — nothing to rebuild,
no configs to redistribute. Upgrading = swap the script + add the web layer:

```bash
sudo wgmgr backup create                              # full backup first, just in case
sudo ln -sf /opt/wg-manager/wireguard-manager-v2.0.sh /usr/local/bin/wgmgr
sudo wgmgr                                            # existing WireGuard data is picked up as-is
#    → 12. Web panel → 1. Install                      # the one new step
```

You can also just swap the script and skip `web install` — V2.0's CLI is a superset
of V1.3's. See tutorial section 16.

## Known limitations

- IPv6 is still a rough "assign a ULA range + NAT66 out" approach; no PD delegation / RA / DNS64.
- The web panel and collector depend on systemd; systemd-less environments can only use the pure CLI (or cron).
- Single-machine, single-instance model; no "one panel manages many machines" aggregated view.
- Traffic history only accumulates from the moment the collector is installed.
- API auth is single-user + session cookie; no multi-user RBAC / OAuth.

See tutorial section 17.

## License

[MIT](LICENSE) © 2026 plnl
