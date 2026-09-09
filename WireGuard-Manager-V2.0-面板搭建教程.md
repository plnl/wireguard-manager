# WireGuard Manager 面板搭建教程（V2.0）

本教程面向已经在用 VPS / Docker 宿主机 / PVE 宿主机 / 多台不同发行版机器
之间折腾网络、并且**准备长期使用**这套 WireGuard 管理的场景。

V1.3 是一个"命令行脚本"：所有能力都塞在一个 3000 多行的 `wg-manager.sh` 里，
出问题只能 SSH 进去敲菜单。V2.0 的目标是把它升级成一个"管理平台"——
在保留原来全部命令行能力的基础上，加了一层 **Web 面板 + JSON API + 状态采集
+ 流量历史 + Health Check + 告警引擎**，让你日常查看状态时不必再 SSH。

> **一句话概括 V2.0：**
> Bash 负责系统级的事（配置、渲染、防火墙、路由、备份），
> Python 负责 Web / API / 状态采集 / 流量历史 / 健康检查 / 告警。
> 两层之间只通过 `state/*.json` 和一套白名单 CLI 通信。

如果你手上还是 V1.3，想平滑升级，直接看[第 16 节：从 V1.3 升级](#16-从-v13-升级)。

---

## 目录

1. [环境要求](#1-环境要求)
2. [V2.0 三层架构与"状态 / 配置分离"原则](#2-v20-三层架构与状态--配置分离原则)
3. [安装与首次运行](#3-安装与首次运行)
4. [目录结构说明](#4-目录结构说明)
5. [安装 Web 面板（`wgmgr web install`）](#5-安装-web-面板wgmgr-web-install)
6. [Web 面板日常使用](#6-web-面板日常使用)
7. [采集守护进程（collector）](#7-采集守护进程collector)
8. [流量历史统计](#8-流量历史统计)
9. [Health Check](#9-health-check)
10. [告警引擎（Telegram / Bark / 企业微信 / 钉钉 / Webhook）](#10-告警引擎telegram--bark--企业微信--钉钉--webhook)
11. [非交互式 CLI 与 `--json`](#11-非交互式-cli-与---json)
12. [JSON API 一览](#12-json-api-一览)
13. [服务运维（systemctl / journalctl）](#13-服务运维systemctl--journalctl)
14. [安全加固](#14-安全加固)
15. [故障排查 Checklist](#15-故障排查-checklist)
16. [从 V1.3 升级](#16-从-v13-升级)
17. [已知限制与老实话](#17-已知限制与老实话)

---

## 1. 环境要求

| 项目 | 要求 |
|---|---|
| 操作系统 | Debian 10+ / Ubuntu 20.04+（脚本内 `check_os` 强制校验） |
| 权限 | 必须 root（或全程 sudo）运行 Bash 核心 |
| 内核 | 建议 5.6+（自带 WireGuard 内核模块），低版本退回 `wireguard-dkms` |
| **Python** | **3.8+**（Debian 10 / Ubuntu 20.04 自带的 `python3` 即可） |
| systemd | Web 面板与采集进程都以 systemd 服务运行；无 systemd 的环境见[第 17 节](#17-已知限制与老实话) |
| sudo | 降权面板通过 `sudo -n` 提权，需要系统装有 `sudo`（安装向导会检测并询问是否 `apt install`） |
| 依赖 | `wireguard-tools`、`iproute2`、`curl`、`qrencode`（二维码用，可选） |
| 网络 | 出网可达 `api.ipify.org` / `ifconfig.me`（自动探测公网 IP，失败可手填） |

**关于 Python 依赖，这是一条刻意的设计约束：** V2.0 的 Python 层**只用标准库**，
不需要 `pip install`、不需要 venv。原因很直接——这台机器上跑的是网络基础设施，
面板绝不能因为 `pip install` 失败、venv 被 apt 升级冲掉、或者某个第三方包改了 API
就跟着一起挂。你能 `python3 --version` 看到 3.8 以上，Web 层就能跑。

**不适用场景**：CentOS/RHEL 系（包管理器与 firewalld 不同，脚本目前不识别）、
OpenWrt 等嵌入式系统（没有 systemd）。这两类如需支持要单独适配。

---

## 2. V2.0 三层架构与"状态 / 配置分离"原则

这是理解整个 V2.0 的关键，也是它区别于 V1.3 的根本。

```text
                 wireguard-manager-v2.0.sh   （Bash 核心 = 配置真相来源）
                           │
             ┌─────────────┴─────────────┐
             │                           │
       Bash Core                    Python 层
   （安装/渲染/防火墙/              ┌────────┴────────┐
     路由/备份/CLI）           采集守护进程         Web 面板
             │                （root 运行）      （降权 wgmgr-web）
             │                     │                  │
             ▼                     ▼                  ▼
      /etc/wireguard/         state/*.json  ◀────读────┘
      /etc/wireguard-manager/      │
                                   └─────写操作经白名单 CLI + sudo -n─────┐
                                                                         ▼
                                                              /usr/local/bin/wgmgr
```

三条铁律，请务必记住：

1. **配置层（真相来源）永远是 Bash 核心管的 `manager.conf` + `clients/*/meta.conf`
   + `sites/*/meta.conf`。** `/etc/wireguard/wg0.conf` 是"派生产物"，不是真相；
   删掉它随时能重新渲染出来。这一点从 V1.3 继承下来，V2.0 完全没变。

2. **状态层（`state/*.json`）是纯派生数据。** 它由 root 的采集进程周期性写入，
   内容全部来自 `wg show` / `ip route` / 防火墙规则 / 系统指标。状态层丢了、
   坏了、陈旧了，**都不影响 WireGuard 本身**，最多是面板显示"数据已过期"。

3. **Web 面板一行 `wg` / `nft` / `ip` 都不执行。** 它的所有读操作来自 `state/*.json`，
   所有写操作都映射到 `wgmgr` CLI 里一份**显式白名单**（`wgm_common.ALLOWED_CLI`）
   上的固定命令。浏览器传进来的字符串永远不会被拼进 shell。

这套设计的直接好处：**面板挂了，采集照常；采集挂了，WireGuard 照常。** 三层互不
拖累。你在手机上打开面板看到一片"离线"，先别慌着以为网断了——很可能是采集进程
或面板本身出了问题，WireGuard 隧道其实还好好的（用[第 9 节](#9-health-check)的
Health Check 或 SSH 进去 `wg show` 一秒就能确认）。

---

## 3. 安装与首次运行

### 方式 A：直接下载（最简单，适合先在测试机上跑通）

```bash
mkdir -p /opt/wg-manager
cd /opt/wg-manager
# 把 wireguard-manager-v2.0.sh（连同同目录的 web/ 和 systemd/ 两个子目录）
# 上传或用你自己的分发方式放到这里
chmod +x wireguard-manager-v2.0.sh
sudo ./wireguard-manager-v2.0.sh
```

> **重要：** V2.0 的脚本会去它自己所在目录找 `web/`（Python 层）和 `systemd/`
> （服务单元 + sudoers 模板）两个子目录。分发时**三个东西要一起放**，
> 保持相对位置不变，否则 `wgmgr web install` 会找不到文件。

### 方式 B：做成一键安装（推荐，方便在多台机器上复用）

和 V1.3 一样，自建一个极简 `install.sh` 放在你自己可控的地址。区别是 V2.0
要连同 `web/` 和 `systemd/` 一起拉下来。示例（假设你的分发根目录里同时有
脚本和这两个子目录）：

```bash
#!/usr/bin/env bash
set -e

INSTALL_DIR="/opt/wg-manager"
BASE_URL="https://your-domain.example.com/wgm"   # 换成你自己的地址

mkdir -p "$INSTALL_DIR"
curl -fsSL "$BASE_URL/wireguard-manager-v2.0.sh" -o "${INSTALL_DIR}/wireguard-manager-v2.0.sh"
chmod +x "${INSTALL_DIR}/wireguard-manager-v2.0.sh"

# web/ 和 systemd/ 用 rsync 或 tar 整目录拉，别一个个文件 curl
rsync -a --delete "$BASE_URL/web/"     "${INSTALL_DIR}/web/"
rsync -a --delete "$BASE_URL/systemd/" "${INSTALL_DIR}/systemd/"

ln -sf "${INSTALL_DIR}/wireguard-manager-v2.0.sh" /usr/local/bin/wgmgr
echo "安装完成，运行：sudo wgmgr"
```

> **安全提示**：`curl | bash` 的前提是你完全信任脚本来源（通常是你自己的仓库/服务器）。
> 分发给别人用时，务必在安装脚本里加 sha256 校验，防 CDN 缓存投毒或中间人篡改。

### 首次运行

```bash
sudo wgmgr        # 或 sudo ./wireguard-manager-v2.0.sh
```

进入 V2.0 主菜单（14 项）：

```text
╔══════════════════════════════════════════════════════╗
║          WireGuard Manager v2.0                       ║
╠══════════════════════════════════════════════════════╣
║   1. Dashboard / 实时状态                             ║
║   2. 服务端                                           ║
║   3. 客户端                                           ║
║   4. Site-to-Site                                     ║
║   5. Peer / 连接                                      ║
║   6. 路由                                             ║
║   7. 防火墙 / NAT                                     ║
║   8. 密钥                                             ║
║   9. 流量统计                                         ║
║  10. 诊断 / Health Check                              ║
║  11. 备份 / 恢复                                      ║
║  12. Web 面板                                         ║
║  13. 系统 / 日志                                      ║
║  14. 高级设置                                         ║
║   0. 退出                                             ║
╚══════════════════════════════════════════════════════╝
```

**建议的上手顺序：** 先 `2. 服务端` → `1. 初始化 / 重新配置服务端` 把骨架搭好
（和 V1.3 完全一致，问端口 / VPN 网段 / DNS / Endpoint / IPv6），再加客户端、
建 Site-to-Site，把 VPN 本身跑通。**确认 WireGuard 能用之后**，再回头做
`12. Web 面板` → `1. 安装`。不要倒过来——面板是观测层，底下没有能用的隧道，
面板也只是显示一片空白。

---

## 4. 目录结构说明

脚本首次运行会自动创建下面这套目录。相比 V1.3，**新增了 `state/`（状态层）、
`web.conf`（面板配置）**，以及装在系统别处的两个 systemd 服务和一份 sudoers：

```text
/etc/wireguard/
└── wg0.conf                     # 真正被 wg-quick 读取的配置，完全由脚本生成，不要手改

/etc/wireguard-manager/          # 权限 710 root:wgmgr-web（面板可 traverse 进来，但列不出目录）
├── manager.conf                 # 全局配置：端口/网段/DNS/IPv6/采集间隔/告警… key=value，640
├── web.conf                     # 面板专属配置：绑定范围/端口/账号/口令哈希/TLS…，640
├── routes.conf                  # 手动加的额外静态路由
├── server/                      # 700 root:root —— 面板读不到
│   ├── private.key              #   服务端私钥，权限 600
│   └── public.key
├── clients/                     # 700 root:root —— 面板读不到
│   └── <客户端名>/
│       ├── meta.conf            #   该客户端的元数据（IP/公钥/是否启用…）
│       ├── private.key
│       ├── public.key
│       └── <客户端名>.conf      #   可直接发给用户导入的完整配置
├── sites/                       # 700 root:root —— 面板读不到
│   └── <站点名>/meta.conf
├── state/                       # 【V2.0 新增】2750 setgid root:wgmgr-web —— 面板可读
│   ├── status.json              #   接口 + 全部 Peer 的实时状态（采集进程写）
│   ├── health.json              #   Health Check 结构化结论
│   ├── alerts.json              #   告警评估状态 + 历史记录
│   ├── traffic.json             #   流量汇总缓存
│   ├── firewall.json            #   防火墙规则快照（root 才能读 nft/iptables，落这里给面板看）
│   ├── manager-log.json         #   manager.log 尾部快照（原文件 root 600，落这里给面板看）
│   └── traffic/                 #   2750 setgid，按天分片的流量采样
│       └── YYYY-MM-DD.jsonl
├── backups/                     # 自动快照 + 手动全量备份（同 V1.3）
└── logs/
    └── manager.log              # 操作审计日志，600 root:root

/opt/wireguard-manager/          # 【V2.0 新增】Web 层运行目录，chgrp wgmgr-web
└── web/
    ├── wgm_common.py            #   共享基础：路径、原子写、CLI 白名单、run_cli
    ├── wgm_collector.py         #   采集守护进程入口
    ├── wgm_web.py               #   HTTP 服务 + JSON API + 静态面板
    ├── wgm_health.py            #   Health Check 引擎
    ├── wgm_traffic.py           #   流量采样 / 汇总 / 趋势图
    ├── wgm_alert.py             #   告警引擎（5 种渠道）
    └── static/                  #   前端页面（index.html + JS/CSS）

/usr/local/bin/wgmgr             # -> 你的 wireguard-manager-v2.0.sh（软链）

/etc/systemd/system/
├── wireguard-manager-collector.service   # 采集服务，root 运行
└── wireguard-manager-web.service         # 面板服务，降权 wgmgr-web 运行

/etc/sudoers.d/wireguard-manager-web      # 0440 root:root，只放行 /usr/local/bin/wgmgr
```

**权限设计的精髓（也是 V2.0 最该讲清楚的一点）：**

- `state/` 和 `state/traffic/` 用了 **setgid 目录（`chmod 2750`）**。采集进程是
  root，它写出来的 JSON 文件属主是 root，但因为目录带 setgid 且属组是
  `wgmgr-web`，**新文件会自动继承 `wgmgr-web` 组**，配合组可读，降权的面板
  进程就读得到了。这个技巧避免了"root 写完文件面板读不了"的经典坑。
- `server/` `clients/` `sites/` 三个目录**死死保持 700 root:root**。面板账号
  `wgmgr-web` 虽然能 traverse 进 `/etc/wireguard-manager`（因为它是 710，给了
  组 `x` 但没给 `r`，进得去但列不出里面有啥），却**读不到任何私钥目录**。
  这是"面板被攻破也偷不走私钥"这条安全底线的物理保证。
- `manager.conf` / `web.conf` 是 640 root:wgmgr-web——面板要读里面的采集间隔、
  告警配置、自己的登录信息，但不需要写（写操作走 sudo 调 CLI）。

> **为什么权限维护放在 `init_directories()` 里、而不是只在安装时 chmod 一次？**
> 因为 `init_directories()` 在脚本**每次被调用时都会跑**（采集进程每 30 秒
> `wgmgr collect` 一次也会触发）。如果只在 `web install` 里 chmod 一次，采集
> 进程下一次跑就可能把权限重置掉。所以 V2.0 把"当 `WEB_ENABLED=yes` 且
> `wgmgr-web` 组存在时，维持这套最小权限"做成了幂等逻辑，每轮都刷一遍，
> 永远不会漂。同理 `set_kv()` 也被改成**保留文件原有的属组和权限位**，
> 面板通过 sudo 重写 `web.conf` 时不会把它打回 600 root:root。

---

## 5. 安装 Web 面板（`wgmgr web install`）

先确认 WireGuard 本身已经能用，然后 `12. Web 面板` → `1. 安装 / 重新安装`
（等价于命令行 `sudo wgmgr web install`）。向导会依次问你：

### 5.1 绑定范围（最关键的一步）

```text
面板监听范围：
  1. 仅 VPN 内可达  —— 绑 <你的 VPN IP>，只有连上 WireGuard 的设备能打开（推荐）
  2. 仅本机         —— 绑 127.0.0.1，配合 SSH 隧道或 Nginx/Cloudflare 反代
  3. 公网可达       —— 绑 0.0.0.0，任何人扫到端口就能访问登录页
```

| 选项 | `WEB_BIND_SCOPE` | 监听地址 | 适用 |
|---|---|---|---|
| 1（推荐） | `vpn` | `<VPN IP>:端口` | 你自己/团队都已经连了 WireGuard，面板只在内网可见，最省心 |
| 2 | `local` | `127.0.0.1:端口` | 前面挂 SSH 隧道、或 Nginx Proxy Manager / Caddy / Cloudflare Tunnel 反代 |
| 3 | `public` | `0.0.0.0:端口` | **危险**，会被全网扫描器持续爆破登录页 |

选 3 时脚本会红字警告并要求你二次确认（默认 N）。**只有在你确实挂了 HTTPS
反向代理、并且限制了来源时才该选 3。** 绑定范围还会连带影响一件事：
`public` 模式下，"从面板下载含私钥的客户端配置"这个功能**默认关闭**
（见[第 6.4 节](#64-下载配置--二维码)）。

> 如果选了"仅 VPN"但服务端还没初始化（拿不到 VPN IP），脚本会自动降级成
> "仅本机"并提示你初始化后重新执行一次安装改绑定。

### 5.2 端口与账号

```text
面板端口 [8443]:            # 默认 8443，写进 WEB_LISTEN
状态采集间隔秒数 [30]:      # 默认 30，写进 manager.conf 的 COLLECT_INTERVAL
```

账号密码在向导里设置：用户名默认 `admin`（写进 `WEB_USER`），口令**不会明文
存储**——脚本用 scrypt 加盐哈希后写进 `WEB_PASS_HASH`，格式是
`scrypt$N$r$p$salt$hash`。之后想改口令，用 `12. Web 面板` → `7. 重置面板密码`
（或 `sudo wgmgr web passwd`）。

### 5.3 安装向导实际做了什么

一次 `web install` 会：

1. 把脚本同目录的 `web/` 整份复制到 `/opt/wireguard-manager/web/`（先删旧的再拷，
   所以**重新安装=升级 Python 层**，见[第 16 节](#16-从-v13-升级)）。
2. 建立软链 `/usr/local/bin/wgmgr -> 你的脚本`。
3. 创建系统账号 `wgmgr-web`（`--system --no-create-home --shell /usr/sbin/nologin`），
   并放好[第 4 节](#4-目录结构说明)讲的那套最小权限（setgid state、710 manager 目录、
   640 两个 conf、chgrp `/opt/wireguard-manager`）。
4. **安装 sudoers 提权规则**：先用 `visudo -cf` 校验
   `systemd/wireguard-manager-web.sudoers` 的语法，通过了才 `install -m 0440 -o root
   -g root` 到 `/etc/sudoers.d/wireguard-manager-web`。**这一步绝不用 `cp` 了事**——
   一份语法错的 sudoers 落到 `/etc/sudoers.d/` 会让整机所有 sudo 直接瘫痪。
   如果系统没装 `sudo`，向导会问你要不要现在 `apt install sudo`。
5. 安装两个 systemd 单元（`install -m 644` + `daemon-reload`），并放行面板端口的
   防火墙规则（按绑定范围决定放行方式）。

装完用 `12. Web 面板` → `5. 查看状态` 或 `9. 显示访问方式` 确认服务起来了、
拿到访问地址。

---

## 6. Web 面板日常使用

### 6.1 登录

浏览器打开 `http://<监听地址>:8443/`（按你的绑定范围替换），输入 `web install`
时设的用户名口令。登录成功后服务端下发一个 **Session Cookie**：

- `HttpOnly`——JS 读不到，挡 XSS 偷 cookie；
- `SameSite=Strict`——挡 CSRF；
- 配了 TLS 或 `WEB_COOKIE_SECURE=yes` 时才加 `Secure` 标志。

**V2.0 刻意不用 `?token=xxxx` 这种 URL 传令牌的方式**——URL 会进浏览器历史、
进反代日志、进 Referer，泄露面太大。登录失败有 IP 级限速（连续失败会被临时封），
用户名比对用 `hmac.compare_digest` 恒定时间比较，避免通过响应时间猜有效用户名。

### 6.2 Dashboard

首页是结构化状态总览（不是把 `wg show` 的文本塞进网页），数据全部来自
`state/status.json`：接口状态、监听端口、VPN 网段、在线/离线 Peer 数、
收发包总量、Site-to-Site 拓扑。Peer 用**五态模型**显示，比 V1.3 的"在线/离线"
两态直观得多：

| 状态 | 判定 | 图标 |
|---|---|---|
| `online` | 最近握手 < 180 秒 | ● 绿 |
| `idle` | 握手在 180 秒 ~ 15 分钟之间 | ● 黄 |
| `offline` | 握手 > 15 分钟 | ● 红 |
| `never` | 从未握手成功过 | ○ 灰 |
| `disabled` | 被禁用（Peer 已从 wg0.conf 移除） | — |
| `pending` | 已创建但尚未启用 | · |

### 6.3 Peer 详情

点任意 Peer 进详情页：名称、类型（client/site）、VPN IP、公钥、Endpoint、
最近握手、PersistentKeepalive、AllowedIPs、RX/TX 流量，以及可用的操作按钮
（启用/禁用、轮换密钥、下载配置、二维码、删除）。

**这里能做的写操作，就是[第 12 节](#12-json-api-一览)白名单里那些**——都是
"只影响单个 Peer"的动作。**服务端级别的危险操作（`server up/down/restart/rebuild`、
`key rotate-server`、`fw clean`、`uninstall`、`web *`）面板一律不提供**，必须
SSH 上去在交互菜单里由人做。这是白名单的核心思想：即便面板口令被爆破，
攻击者能造成的损失上限也是"骚扰某一个客户端"，没法把整张网掀掉。

### 6.4 下载配置 / 二维码

详情页可以下载 `.conf` 或直接看二维码，方便手机扫码导入。但客户端配置里
**含私钥**，所以能不能从面板下载取决于面板暴露得多宽（`allow_key_export()`）：

- `WEB_BIND_SCOPE=vpn` 或 `local`：**默认允许**——这两种情况下能打开面板的人
  本来就能 SSH 上来 `cat` 这个文件，风险等级一样。
- `WEB_BIND_SCOPE=public`：**默认禁止**——公网可达的面板一旦被爆破，第一个
  被拿走的就是私钥。确有需要必须在 `web.conf` 里显式写 `WEB_ALLOW_KEY_EXPORT=yes`
  自己表态。

### 6.5 只读模式

如果你想把面板开给"只能看不能改"的人（比如给同事看状态），在 `web.conf` 里设
`WEB_READONLY=yes`，面板会拒绝一切写操作（POST/DELETE 直接返回权限错误）。

> **注意：** `WEB_READONLY` / `WEB_COOKIE_SECURE` / `WEB_ALLOW_KEY_EXPORT` /
> `WEB_TLS_CERT` / `WEB_TLS_KEY` 这几项**没有交互菜单入口**，需要手动编辑
> `/etc/wireguard-manager/web.conf` 后 `sudo systemctl restart wireguard-manager-web`
> 生效。菜单只覆盖了最常用的绑定范围/端口/账号/密码/对外 URL。

---

## 7. 采集守护进程（collector）

`wireguard-manager-collector.service` 以 **root** 运行（它要 `wg show`、读 nft/iptables、
读 root 拥有的 `manager.log`），默认每 30 秒（`COLLECT_INTERVAL`，最小 5 秒）跑一轮。
**每一轮做五件事**（`wgm_collector.py` 的 `cycle()`）：

1. `wgmgr collect` —— 刷新 `state/status.json`（Bash 侧，root）。
2. 追加一个流量采样点到 `state/traffic/YYYY-MM-DD.jsonl`。
3. 跑一遍 Health Check，写 `state/health.json`。
4. 跑一遍告警评估，写 `state/alerts.json`，命中就实际推送。
5. 低频快照：每 10 轮抓一次防火墙规则（`state/firewall.json`），每轮抓一次
   `manager.log` 尾部（`state/manager-log.json`，只在日志真变了时才重写）；
   每天清一次过期的流量采样文件（保留 90 天）。

几个值得知道的设计点：

- **采集进程不参与 Bash 侧的 flock 排它锁。** 它只读 `wg show` + 写 `state/`，
  不碰任何配置。否则你人在菜单里停着不动（交互模式全程持锁），采集就全停了。
  它自己用 `/run/wireguard-manager-collector.lock` 防止两个采集实例并跑。
- **写入全是 tmp + rename 原子操作**，面板随时在读也不会读到半截 JSON。
- **任何一步失败都不影响后面的步骤**，一轮异常也绝不会让守护进程退出
  （外层有兜底 try/except）。告警失败尤其不能影响采集——宁可少推一条，
  也不能让状态层停更。
- **从"本轮结束"开始算下一次间隔**，不是从"本轮开始"。否则一轮跑了 40 秒
  而间隔是 30 秒时，会变成背靠背连轴转。

手动立即采集一次：`12. Web 面板` → `10. 立即采集一次状态`，或 `sudo wgmgr collect`。

---

## 8. 流量历史统计

WireGuard 本身只提供瞬时的 `rx_bytes` / `tx_bytes`（接口重启就清零），看不到
"今天用了多少、这个月趋势如何"。V2.0 的采集进程每轮把每个 Peer 的累计字节数
打一个采样点存进 `state/traffic/YYYY-MM-DD.jsonl`（按天分片），再由 `wgm_traffic.py`
做差值汇总，就有了历史。

- **查看：** `9. 流量统计` 菜单，或命令行 `sudo wgmgr traffic`。
- **时间范围：** `24h` / `7d` / `30d`（`--range`）。
- **维度：** 全局汇总，或 `--by-peer` 按 Peer 汇总（本月）。
- **趋势图：** `wgmgr traffic chart --range 24h` 在终端画 ASCII 趋势图；
  面板上也有对应图表。
- **保留期：** 90 天（`TRAFFIC_RETENTION_DAYS`），采集进程每天自动清理过期分片。
  30 秒一次、20 个 Peer，一天约 1MB，90 天不到 100MB，不用担心占盘。

菜单对应关系：`1. 最近 24 小时` / `2. 最近 7 天` / `3. 最近 30 天` /
`4. 按 Peer 汇总（本月）` / `5. 趋势图（最近 24 小时）`。

---

## 9. Health Check

V1.3 的"系统诊断"是一份 ✓/✗ 清单，只告诉你"哪一项挂了"。V2.0 的 Health Check
（`wgm_health.py`）在此之上做了两件事：**分级别** + **给出下一步该查什么**。

- **查看：** `10. 诊断 / Health Check` → `1. Health Check`，或 `sudo wgmgr health`。
  （同一菜单里 `2. 传统系统诊断` 就是 V1.3 那份不依赖 Python 层的 ✓/✗ 清单，
  应急时即使 Python 层坏了也能用。）
- **级别：** `pass`（✓ 绿）/ `info`（· 灰）/ `warn`（⚠ 黄）/ `error`（✗ 红）。
  顶部有一句总结论（headline），比如"能跑，但有 3 项需要注意"。

它检查的东西分两块：

**系统级：** 服务端是否初始化、出网 NAT 规则是否存在、IPv6 全局转发有没有加固、
内核 IPv4/IPv6 转发、防火墙后端是否检测到、根分区剩余空间（< 512MB 告警）、
可用内存（< 10% 告警）、`state/status.json` 是否存在/陈旧、Web 面板是否在跑、
采集服务是否在跑。

**Peer 级（逐个）：** 握手是否正常、创建很久却从未握手、Site-to-Site 有没有设
PersistentKeepalive、是不是"网段冲突"降级模式、两端 LAN 网段是否完全相同。

每一项 warn/error 都带**具体的排查建议**，比如某站点两小时没握手，它会直接列出
"1. 对端 WireGuard 是否运行 2. UDP 端口是否被防火墙拦 3. PersistentKeepalive
4. Endpoint 是否变化"。采集进程每轮都会跑一次 Health Check 写进 `state/health.json`，
面板首页的健康区块读的就是它。

---

## 10. 告警引擎（Telegram / Bark / 企业微信 / 钉钉 / Webhook）

这是 V2.0 最适合"长期维护 VPS / 家庭网络"场景的功能：某个该一直在线的 Peer
掉线了，自动推送到你手机，不用你自己盯着面板。

### 10.1 配置入口

`10. 诊断 / Health Check` → `3. 告警设置`（等价命令行见下）。这个菜单
（`alert_menu`）覆盖：

```text
  总开关     : 已开启 / 关闭          （ALERT_ENABLED，默认 no）
  监控范围   : all                    （ALERT_WATCH：all / sites / clients / 逗号分隔名单）
  确认次数   : 3 次判定离线后才告警    （ALERT_THRESHOLD，防抖，默认 3）
  渠道       : telegram,bark,…        （ALERT_CHANNELS，逗号分隔）

  1. 开关告警
  2. 设置监控范围
  3. 设置确认次数（防抖，避免一次采集抖动就推送）
  4. 配置 Telegram        （ALERT_TELEGRAM_TOKEN + ALERT_TELEGRAM_CHAT_ID）
  5. 配置 Bark            （ALERT_BARK_URL，如 https://api.day.app/xxxx）
  6. 配置企业微信          （ALERT_WECOM_KEY，机器人 Webhook Key）
  7. 配置钉钉             （ALERT_DINGTALK_TOKEN + 可选 ALERT_DINGTALK_SECRET 加签）
  8. 配置通用 Webhook      （ALERT_WEBHOOK_URL，POST JSON）
  9. 发一条测试告警
 10. 查看最近告警记录
```

### 10.2 防抖与恢复通知

- **防抖（`ALERT_THRESHOLD`）：** 一个 Peer 要**连续 N 次**采集都被判定离线
  才真正推送，避免一次网络抖动/采集毛刺就给你发一条假告警。默认 3 次
  （30 秒间隔 ≈ 90 秒确认）。
- **恢复通知（`ALERT_RECOVERY`，默认 yes）：** Peer 恢复上线时再推一条"已恢复"。
  这一项**没有菜单入口**，要关掉得手动在 `manager.conf` 里设 `ALERT_RECOVERY=no`。

### 10.3 告警是怎么被触发的

告警评估**由采集进程每轮自动跑**（`cycle()` 第 4 步调 `wgm_alert.evaluate`），
不需要你手动触发。评估结果和推送历史都记在 `state/alerts.json`，面板的告警
区块和 `10. 诊断` → `4. 最近告警记录`（`sudo wgmgr alert history`）读的就是它。
手动发测试告警：`sudo wgmgr alert test`（或菜单 `9. 发一条测试告警`，或面板上
的"测试告警"按钮——它调 `POST /api/v1/alerts/test`）。

> 所有告警渠道的配置键都写在 `manager.conf` 里（不是 `web.conf`）。测试告警时
> 如果一个渠道都没配，面板会明确告诉你"还没配置任何告警渠道，先去 SSH 里配"，
> 而不是含糊地报"发送失败"——这两件事的处置方式完全不同。

---

## 11. 非交互式 CLI 与 `--json`

V2.0 的 Bash 核心有两张脸：**不带参数=交互式菜单，带参数=非交互式 CLI**。
这不是两个入口，而是同一套核心逻辑——菜单里能做的事几乎都能在 CLI 里用一条
命令做完，而 **Web 面板只会调用 CLI**，它自己一行 `wg`/`iptables` 都不碰。
这样任何一条配置变更路径最终都收敛到同一批函数上，不会出现"网页改了但脚本
不知道"的分叉。

约定：

- 带 `--json` 时 **stdout 只有 JSON**，所有彩色提示走 stderr（方便 `jq` 处理）。
- 会改配置的子命令拿排他锁；只读/采集类不加锁。
- 成功返回 0，失败返回非 0（方便 shell 里判断）。

完整用法（`sudo wgmgr help`）：

```text
状态 / 采集
  collect [--print]                刷新 state/status.json
  status [--json]                  接口与 Peer 概览
  dashboard [--refresh N]          终端实时仪表盘
  health [--json]                  Health Check（结构化结论 + 排查建议）
  traffic [--range 24h|7d|30d] [--by-peer] [--json]
  diagnose                         传统 ✓/✗ 系统诊断

Peer
  peer list [--kind client|site] [--status online|idle|offline|never|pending|disabled] [--json]
  peer show <name> [--json]

客户端
  client list [--json]
  client add <name> [--ip4 <a.b.c.d>] [--ip6 <addr>] [--allowed <cidr,...>] [--print-conf] [--json]
  client enable|disable|delete <name>
  client conf <name> [--qrcode|--png]
  client rotate-key <name> [--print-conf]

Site-to-Site
  site list [--json]
  site create <name> --remote-lan <cidr> --remote-wg-ip <ip> --remote-pubkey <key>
                     [--local-lan <cidr>] [--remote-endpoint <ip:port>]
                     [--mode routing|nat] [--keepalive <sec>] [--force]
  site enable|disable|delete <name>
  site test <name>

服务端 / 路由 / 防火墙 / 密钥
  server info [--json]
  server up|down|restart|rebuild
  route list [--json] | route add <name> <subnet> <via> [comment] | route delete <name>
  fw backend|show|sync|clean
  key rotate-server | key export-pubkeys

告警 / 备份 / 面板
  alert test | alert history
  backup create|list
  web install|start|stop|restart|status|logs|passwd [--user <name>] [--stdin]|url|uninstall
  uninstall

其他
  version | help
```

给 cron / 监控 / Telegram Bot 用的例子：

```bash
# 把所有离线 Peer 挑出来（配合 jq）
wgmgr peer list --json | jq '.peers[] | select(.status=="offline")'

# 批量给 20 台设备建客户端
for n in $(seq 1 20); do wgmgr client add "dev-$n" --allowed 0.0.0.0/0; done

# 导出当前状态给别的系统消费
wgmgr status --json > /tmp/wg.json
```

---

## 12. JSON API 一览

面板前端和第三方集成（手机 App、Grafana、Uptime Kuma、你自己的 Bot）调的是
同一套 HTTP JSON API，前缀 `/api/v1`。除 `ping` / `login` 外**全部需要登录**
（带 Session Cookie）。

```text
认证
  GET    /api/v1/ping                      探活（无需登录）
  POST   /api/v1/login                     登录，下发 Session Cookie
  POST   /api/v1/logout                    登出

只读
  GET    /api/v1/status                    接口 + 全部 Peer 概览
  GET    /api/v1/interface                 接口详情
  GET    /api/v1/system                    系统指标（CPU/内存/磁盘/转发…）
  GET    /api/v1/peers                     Peer 列表
  GET    /api/v1/peers/:name               单个 Peer 详情
  GET    /api/v1/peers/:name/config        下载客户端配置（受 key_export 策略约束）
  GET    /api/v1/peers/:name/qrcode.png    二维码图片
  GET    /api/v1/sites                     Site-to-Site 列表
  GET    /api/v1/routes                    路由（含内核路由表）
  GET    /api/v1/firewall                  防火墙快照
  GET    /api/v1/traffic                   流量历史
  GET    /api/v1/logs                      操作日志尾部
  GET    /api/v1/health                    Health Check 结论
  GET    /api/v1/alerts                    告警配置 + 历史

写操作（全部映射到白名单 CLI，单 Peer 级别）
  POST   /api/v1/collect                   立即采集一次
  POST   /api/v1/clients                   新建客户端
  POST   /api/v1/sites                     新建站点
  POST   /api/v1/backup                    立即全量备份
  POST   /api/v1/firewall/sync             重新同步防火墙
  POST   /api/v1/alerts/test               发测试告警
  POST   /api/v1/peers/:name/enable        启用
  POST   /api/v1/peers/:name/disable       禁用
  POST   /api/v1/peers/:name/rotate-key    轮换该 Peer 密钥
  POST   /api/v1/sites/:name/test          站点连通性测试
  DELETE /api/v1/peers/:name               删除
```

**为什么写操作这么克制？** API 的白名单边界是"影响范围"而不是"危险程度"：
允许的（增删/启停/导出/轮换单个 Peer）最坏情况是那一个设备连不上、重新导入
即可，而且轮换前 Bash 侧自己会先打快照；禁止的（`server up/down/restart/rebuild`、
`key rotate-server`、`fw clean`、`uninstall`、`web *`）会让**所有** Peer 一起掉线
或干脆不可逆，必须人在交互菜单里做。白名单在 Python 侧（`wgm_common.ALLOWED_CLI`）
把关，`run_cli` 还会拒绝含 shell 元字符（`; & | ` $ < > ( ) { } \ 换行` 等）的参数值，
两层纵深防御。

---

## 13. 服务运维（systemctl / journalctl）

两个服务：

```bash
# 采集守护进程（root）
sudo systemctl status  wireguard-manager-collector
sudo systemctl restart wireguard-manager-collector
sudo journalctl -u     wireguard-manager-collector -f

# Web 面板（降权 wgmgr-web）
sudo systemctl status  wireguard-manager-web
sudo systemctl restart wireguard-manager-web
sudo journalctl -u     wireguard-manager-web -f
```

也可以全在菜单里做：`12. Web 面板` → `2/3/4/5/6`（启动/停止/重启/状态/日志），
或 `13. 系统 / 日志` 里集中看两个服务活着没有、状态层有没有在正常刷新、最近改过
什么（不用在 journalctl / ls / tail 之间来回切）。

命令行等价：`sudo wgmgr web start|stop|restart|status|logs`。

采集进程每 20 轮（默认约 10 分钟）打一次心跳日志，方便你从 `journalctl` 确认
它还活着：`心跳 seq=… ✗0 ⚠2 告警=-`。

**两个服务都是 `Restart=always` + `RestartSec=5`**，崩了会自动拉起。它们对
`wg-quick@wg0` **只有排序依赖、没有 Require/Wants**——这是刻意的：采集挂了
面板照常起（只是数据显示陈旧），面板挂了采集照常跑，两者都不拖累 WireGuard
本身。这正是[第 2 节](#2-v20-三层架构与状态--配置分离原则)"状态与配置分离"
在 systemd 层的落地。

---

## 14. 安全加固

这套面板管理的是网络层配置，出问题影响面比一般应用大。V2.0 在 V1.3 的基础上
多加了一整层"降权 + 提权隔离"，建议至少做到：

1. **面板降权运行（V2.0 默认已做）。** 面板服务以 `wgmgr-web` 跑，不是 root。
   它读不到 `server/` `clients/` `sites/` 里的任何私钥（那些目录 700 root:root），
   写操作只能通过 sudoers 白名单调 `wgmgr`。**不要图省事把面板改回 root 跑。**

2. **sudoers 只放行一个绝对路径。** `/etc/sudoers.d/wireguard-manager-web` 里
   只有 `wgmgr-web ALL=(root) NOPASSWD: /usr/local/bin/wgmgr`——不是 shell、
   不是 `ALL`。哪些子命令能调由 Python 侧白名单二次把关。**别手动往这份
   sudoers 里加东西**，尤其是别加 `wgmgr-web ALL=(ALL) ALL` 这种。改完记得
   `sudo visudo -cf /etc/sudoers.d/wireguard-manager-web` 校验。

3. **优先用 `vpn` / `local` 绑定，别用 `public`。** 真要公网可达，前面务必挂
   HTTPS 反向代理（Nginx Proxy Manager / Caddy / Cloudflare Tunnel），并在
   `web.conf` 里设 `WEB_COOKIE_SECURE=yes`。`public` + 无 TLS 时登录口令是明文
   过公网，脚本启动时会红字警告。也可以让面板自己跑 TLS：在 `web.conf` 里配
   `WEB_TLS_CERT` / `WEB_TLS_KEY` 指向证书和私钥。

4. **口令用强随机值。** 存的是 scrypt 哈希不是明文，但弱口令照样能被爆破。
   登录有 IP 限速兜底，但别依赖它。

5. **WireGuard 端口不要用默认 51820**（同 V1.3），减少被自动化扫描的噪音。

6. **私钥权限别放宽。** 脚本已把所有私钥设 600、`/etc/wireguard-manager` 710、
   `server/clients/sites` 700。不要手动改宽。

7. **备份文件本身也要保护。** `backups/*.tar.gz` 含所有私钥，异地同步走加密传输
   （scp/rsync over ssh），别丢明文私钥到公共存储。

8. **定期核对 Peer 列表。** 面板或 `wgmgr peer list` 里出现不认识的 Peer，或
   `wgmgr key export-pubkeys` 里有对不上号的公钥，立刻 `key rotate-server` 强制
   所有旧配置失效，再逐个重新分发给可信设备。

---

## 15. 故障排查 Checklist

按顺序排查，大部分问题能在前几步定位：

1. **面板打不开 / 一片空白，但 VPN 其实是通的** —— 这是 V2.0 最常见的"假故障"。
   先分清是哪一层：
   ```bash
   sudo systemctl status wireguard-manager-web        # 面板活着吗
   sudo systemctl status wireguard-manager-collector  # 采集活着吗
   sudo wgmgr health                                  # 状态层陈旧吗
   ```
   面板显示"数据已过期"通常是**采集进程**挂了，不是 WireGuard 挂了。SSH 进去
   `wg show` 一秒确认隧道本身好不好。

2. **面板能打开但所有写操作失败（加客户端/启停报错）** —— 多半是 sudoers 没装好：
   ```bash
   sudo visudo -cf /etc/sudoers.d/wireguard-manager-web   # 语法对吗
   sudo -u wgmgr-web sudo -n /usr/local/bin/wgmgr status   # 降权账号能免密 sudo 吗
   ```
   如果第二条报 "a password is required" 或 permission denied，说明规则没生效——
   重新跑一次 `sudo wgmgr web install`，或检查 `/etc/sudoers` 里默认的
   `#includedir /etc/sudoers.d` 有没有被你删掉。

3. **面板报"读不到私钥/配置"或页面缺数据** —— 检查 setgid 权限有没有漂：
   ```bash
   stat -c '%A %U:%G %n' /etc/wireguard-manager/state /etc/wireguard-manager/state/status.json
   # state 目录应是 drwxr-sr-x root:wgmgr-web，里面的 json 应属组 wgmgr-web 且组可读
   ```
   不对就重跑 `sudo wgmgr web install`（它会重放那套最小权限），或手动
   `sudo wgmgr collect` 触发一次 `init_directories` 幂等修复。

4. **WireGuard 连接层问题（Peer 连不上、握手停滞、流量为 0）** —— 和 V1.3 一样：
   `10. 诊断` → `1. Health Check` 看结构化结论和建议；`5. Peer / 连接` 看目标
   Peer 的 Latest Handshake；服务端 `tcpdump -ni any udp port <你的端口>` 抓包
   确认包有没有到；**别忘了云厂商控制台那一层安全组，脚本管不到它**。

5. **告警没推出来** —— `sudo wgmgr alert test` 手动发一条看报什么错；确认
   `ALERT_ENABLED=yes`、`ALERT_CHANNELS` 里有你配好的渠道、对应渠道的 token/url
   填对了；注意防抖——`ALERT_THRESHOLD=3` 意味着要连续 3 轮判定离线才推，
   刚掉线一两分钟不推是正常的。

6. **这次改动之后才出的问题** —— 直接 `11. 备份 / 恢复` → 从自动快照回滚，
   选最近一次改动之前的快照，先恢复到能用再慢慢排查差异（同 V1.3）。

---

## 16. 从 V1.3 升级

**好消息：V1.3 和 V2.0 用的是同一个配置目录 `/etc/wireguard-manager/`、同一套
`manager.conf` + `clients/*/meta.conf` + `sites/*/meta.conf` 结构。** 所以从 V1.3
升 V2.0 本质上是"换脚本 + 加 Web 层"，你的客户端、站点、密钥、路由、备份
**全部原地保留，不用重建、不用重新分发配置**。

步骤：

```bash
# 1. 先做一次全量备份（保险，虽然升级不动你的配置数据）
sudo wgmgr backup create        # 或在 V1.3 菜单里手动全量备份

# 2. 把 V2.0 的脚本 + web/ + systemd/ 放到 /opt/wg-manager（三者相对位置不变）
#    然后用 V2.0 脚本重新建一次全局命令软链
sudo ln -sf /opt/wg-manager/wireguard-manager-v2.0.sh /usr/local/bin/wgmgr

# 3. 进 V2.0 菜单，WireGuard 本身照常能用（服务端/客户端/站点数据都在）
sudo wgmgr

# 4. 新增的一步：装 Web 面板
#    12. Web 面板 → 1. 安装（选绑定范围、端口、账号）
```

要点：

- **`wg0.conf` 不用管。** 它是派生产物，V2.0 第一次跑会按现有元数据重新渲染，
  内容和 V1.3 生成的一致。
- **V2.0 新增的 `state/`、`web.conf`、两个 systemd 服务、sudoers 都是"加"出来的**，
  不覆盖你任何现有配置。`web install` 之前面板相关的一切都不存在，WireGuard
  纯命令行用法和 V1.3 完全一样。
- **如果你不想用 Web 面板**，完全可以只把脚本换成 V2.0、`web install` 那步跳过。
  V2.0 的命令行能力是 V1.3 的超集（多了 `health` / `traffic` / `--json` / 告警等），
  纯 CLI 用法照样跑。
- **升级 Python 层**（以后出了新版 `web/`）：重跑一次 `sudo wgmgr web install`
  即可——它会先 `rm -rf` 旧的 `/opt/wireguard-manager/web` 再拷新的，然后
  `systemctl restart` 两个服务。配置数据（`/etc/wireguard-manager/`）不受影响。

> 注：脚本里还有一段 `migrate_legacy_v1`，那是给**更早的 V1.0**（用的是单文件
> DB 格式，不是目录结构）准备的自动迁移，和 V1.3→V2.0 无关。V1.3 的数据 V2.0
> 直接认，不会触发那段迁移。

---

## 17. 已知限制与老实话

和 V1.3 教程一样，这里把 V2.0 **刻意没做**或**做不了**的事讲清楚，免得你有
错误预期：

1. **IPv6 仍然是"能用但粗糙"的方案。** V2.0 完全继承了 V1.3 对 IPv6 的处理
   （分一段 ULA/自定义网段 + 出网 NAT66），**没有**做 PD 委派、没有识别上游
   分配方式（SLAAC / DHCPv6-PD / 厂商路由）、不配 RA、不做 DNS64/NAT64。
   Health Check 里那条 "IPv6 全局转发已开启且没有加固" 的 warn 就是在提醒
   你这个风险。对 IPv6 有严格要求的场景，建议单独设计，别指望这个脚本全覆盖。

2. **Web 面板和采集进程依赖 systemd。** 没有 systemd 的环境（OpenWrt、部分
   极简容器）跑不了这两个服务。这种情况下你仍然可以用**纯 CLI**（`wgmgr` 的
   所有非交互子命令都能手动跑，或塞进 cron），只是没有常驻面板和自动采集。
   想要状态页可以退化成"cron 定期 `wgmgr status --json > 某静态文件`"。

3. **面板是单机单实例模型。** 一台机器一份 `/etc/wireguard-manager/`、一个
   `wg0`。V2.0 **没有**做"一个面板管多台机器"的聚合视图——跨机器批量操作
   一旦哪步失败很难保证各机状态一致，风险大于收益。多机管理仍建议每台独立
   跑面板 + 你自己维护一份拓扑登记表（同 V1.3 第 14 节的思路）。

4. **流量历史从"装了采集进程"那一刻才开始积累。** 它靠周期性采样做差值，
   补不出安装之前的历史。刚装完头几天趋势图数据少是正常的。

5. **`WEB_READONLY` / `WEB_COOKIE_SECURE` / `WEB_ALLOW_KEY_EXPORT` /
   `WEB_TLS_CERT` / `WEB_TLS_KEY` / `ALERT_RECOVERY` 这几项没有交互菜单入口**，
   要手动编辑 `web.conf`（前五项，改完 restart 面板服务）或 `manager.conf`
   （`ALERT_RECOVERY`，采集进程下一轮自动生效）。菜单只覆盖了最常用的那几项。

6. **面板不提供服务端级危险操作。** `server up/down/restart/rebuild`、
   `key rotate-server`、`fw clean`、`uninstall`、`web *` 这些"会让所有 Peer
   一起掉线或不可逆"的动作，白名单刻意排除，必须 SSH 进交互菜单由人做。
   这不是没做完，是安全边界的设计选择。

7. **API 没有做分页/限流的企业级完备性。** 它有登录限速、有 CSP/安全响应头、
   有原子读写，够个人和小团队长期用；但如果你要拿它做大规模多租户平台，
   认证（目前是单用户 + Session Cookie，不是 OAuth/多用户 RBAC）、审计、
   配额这些都需要再单独设计。

---

*本教程配套脚本：`wireguard-manager-v2.0.sh` V2.0.0，以及同目录的 `web/`
（Python 层）和 `systemd/`（服务单元 + sudoers 模板）。菜单编号/行为如有后续
调整，以脚本内 `wgmgr version` 或主菜单标题为准。*
