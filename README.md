# Proxy Configurations

为 Mihomo (Clash Meta)、Loon、Shadowrocket、Sing-box 提供跨平台代理配置。

## 架构概览

目前本仓库主推 **Region (按地区/节点手选)** 策略组架构，共享同一套规则源和 DNS 设计：

| 架构 | 策略组 | 适用场景 | 平台支持 |
|------|--------|----------|----------|
| **Region (4大块收敛)** | 4 大块规则收敛 (直连/亚太/AI/低倍) + 兜底 | 极简零冗余秒级流转，全平台统一分流 | Mihomo, Loon, Shadowrocket, Sing-box 1.14+ |

### 四大块收敛架构 (Mihomo, Loon, Shadowrocket, Sing-box: Direct / APAC / AI / LowRate)

在全平台客户端（Mihomo、Loon、Shadowrocket、Sing-box）中，依托编译管线 `DonJone/proxy-rulesets` 将庞杂规则收敛为四大规则集：
- **DIRECT (`direct.mrs` / `direct.lsr` / `direct.list` / `direct.srs`)**：私有地址、国内 IP CIDR、国内域名大盘、Steam/Epic 等游戏切片下载 CDN。
- **亚太 (`apac.mrs` / `apac.lsr` / `apac.list` / `apac.srs`)**：Telegram 全量 IP/域名、Twitter/X、Steam/Epic 商城与社交、亚太直连节点。
- **AI (`ai.mrs` / `ai.lsr` / `ai.list` / `ai.srs`)**：OpenAI、Claude、Gemini、Copilot、Grok、Mistral 等海外核心 AI 矩阵。
- **低倍 (`lowrate.mrs` / `lowrate.lsr` / `lowrate.list` / `lowrate.srs`)**：YouTube、Netflix、Disney+、Emby 流媒体、GitHub、海外 Apple 服务及大吞吐兜底。
- **兜底 (`MATCH` / `FINAL` / `final`)**：未收录的海外小众域名及动态 IP，默认交由【兜底】策略组（包含低倍/亚太/AI/DIRECT）。

## 规则来源

| 平台 | 规则源 | 格式 |
|------|--------|------|
| Mihomo | [DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets) | .mrs Binary Rule-set (四大块收敛架构) |
| Loon | [DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets) | .lsr Remote Rule (四大块收敛架构) |
| Shadowrocket | [DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets) | .list RULE-SET (四大块收敛架构) |
| Sing-box | [DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets) | .srs Binary Rule-set (四大块收敛架构) |

### 规则映射与匹配顺序

| 规则集 | 目标组 | 平台 |
|--------|--------|------|
| `direct` / `direct_ip` | **DIRECT** | Mihomo / Loon / Shadowrocket / Sing-box |
| `apac` / `apac_ip` | **亚太** | Mihomo / Loon / Shadowrocket / Sing-box |
| `ai` / `ai_ip` | **AI** | Mihomo / Loon / Shadowrocket / Sing-box |
| `lowrate` / `lowrate_ip` | **低倍** | Mihomo / Loon / Shadowrocket / Sing-box |

规则匹配顺序：
```text
预留地址 → 私有地址 → NTP 时间保护 → FCM 推送保护 → 国内直连 (direct)
→ 亚太通讯 (apac: Telegram 专属防 UDP 443 误伤) → QUIC 拦截 (UDP 443)
→ AI 核心业务 (ai) → 低倍流媒体/海外大盘 (lowrate) → GeoIP CN 直连 → MATCH / FINAL 兜底
```

> **关键顺序**: AI/YouTube 必须在 Google 之前 (google.mrs 含 youtube 域名)，GitHub 必须在 Microsoft 之前 (microsoft.mrs 含 github 域名)。

## DNS 设计

- `enhanced-mode: fake-ip`，`fake-ip-filter-mode: blacklist`（使用 ShellCrash 列表）
- `respect-rules: true` — 规则匹配结果自动选择 DNS 服务器（代理域名走 proxy-doh，直连域名走 direct-doh）
- `nameserver-policy: {}` — 无额外策略，完全依赖 respect-rules
- `hosts` 硬编码 DNS 服务器 IP，避免冷启动死锁

## 平台差异

| 特性 | Mihomo | Loon | Shadowrocket | Sing-box |
|------|--------|------|-------------|----------|
| 规则来源 | proxy-rulesets (.mrs) | proxy-rulesets (.lsr) | proxy-rulesets (.list) | proxy-rulesets (.srs) |
| 策略组过滤 | `filter` / `exclude-filter` | Remote Filter NameRegex | `policy-regex-filter` | 手选池 (selector) + 测速 (urltest) |
| DNS | fake-ip + respect-rules | `dns-server` (阿里/腾讯/system) | `dns-server` + `fallback-dns-server` | fake-ip (198.18/15) + 双 DoH 路由 |
| 嗅探 | sniffer 配置块 | `sni-sniffing=true` | 内置 | `sniff: true` (系统级嗅探) |
| QUIC | AND 规则 → 代理QUIC | — | `block-quic=all-proxy` | `action: reject` (UDP 443 拦截) |

全平台客户端（Mihomo, Loon, Shadowrocket, Sing-box）现已全量接入 `DonJone/proxy-rulesets` 四大块收敛架构。Sing-box 使用现代化 1.14+ 强类型 DNS 与 http_clients 架构。

## 目录结构

```text
proxy-configs/
├── mihomo/
│   ├── mihomo_Region.yaml            # Mihomo 通用客户端 (四大块收敛架构)
│   ├── mihomo_Region_openclash.yaml  # OpenClash 路由 (四大块收敛架构)
│   └── mihomo_Region_core.yaml       # Mihomo 裸核专用 (四大块收敛架构 + Zashboard + DoH3)
├── loon/
│   ├── loon_Region_mac.lcf           # Loon macOS 桌面端 (四大块收敛架构 + SSID 直连)
│   └── loon_Region_ssid.lcf          # Loon iOS 移动端全功能 (四大块收敛架构 + 去广告插件 + SSID 直连)
├── Shadowrocket/
│   └── Shadowrocket_Region.conf      # Region (四大块收敛架构)
├── singbox/
│   └── singbox_Region.json           # Sing-box 1.14+ 生产级配置 (Tun + Mixed + 4 大块 .srs 远程分流 + Zashboard)
├── reports/                          # 检修与检修报告专用文件夹
│   ├── weekly_inspection_report_2026-09-07.md
│   └── bytedance_latency_inspection_report_2026-09-17.md
└── README.md
```

## 下载链接

基于 jsDelivr CDN 加速的远程配置文件链接：

### Mihomo / OpenClash

| 适用环境 | 架构 | 链接 |
|----------|------|------|
| 通用 (Desktop/Mobile) | Region (四大块) | [mihomo_Region.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/mihomo_Region.yaml) |
| OpenClash 路由 | Region (四大块) | [mihomo_Region_openclash.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/mihomo_Region_openclash.yaml) |
| 嵌入式 Linux / OpenWrt 裸核 | Region Core (四大块 + Zashboard) | [mihomo_Region_core.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/mihomo_Region_core.yaml) |

> 注：Mihomo 原 15 策略组旧架构已归档至本仓库的 `archive_mihomo_legacy` 分支中保存。

### Loon

| 适用环境 | 架构 | 链接 |
|----------|------|------|
| macOS 桌面端专属 | Region (四大块 + SSID) | [loon_Region_mac.lcf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/loon_Region_mac.lcf) |
| iPhone / iOS 移动端全功能 | Region (四大块 + 移动插件 + SSID) | [loon_Region_ssid.lcf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/loon_Region_ssid.lcf) |

> 注：Loon 原 15 策略组旧架构已归档至本仓库的 `archive_loon_legacy` 分支中保存。

### Shadowrocket

| 适用环境 | 架构 | 链接 |
|----------|------|------|
| 通用客户端 (iOS) | Region (四大块收敛架构) | [Shadowrocket_Region.conf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/Shadowrocket_Region.conf) |

> 注：Shadowrocket 原多策略组旧架构已归档至本仓库的 `archive_shadowrocket_legacy` 分支中保存。

### Sing-box

| 适用环境 | 架构 | 链接 |
|----------|------|------|
| 通用客户端 / 裸核 / 路由 (v1.14+ Latest Release) | Region (4 大块 + Zashboard) | [singbox_Region.json](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/singbox/singbox_Region.json) |

## 校园网与私有专线配置指南

本套配置预置了对高校校园网及内部局域网的隐私隔离与专线分流支持（在配置中统一采用通用的占位符 `campus.edu.cn` 与 `1.2.3.4` 表示）。

### 1. 对于不需要校园网的普通用户 (家庭宽带 / 移动蜂窝网络)
* **默认无需任何修改，完全开箱即用**：
  配置中内置的 `campus.edu.cn`、`1.2.3.4` 仅为静态占位符（Placeholder）。在非校园网环境下，这些规则不会命中任何流量，对公网浏览、4K 流媒体播放及各场景代理体验没有任何性能损耗或负面影响。
* **如果追求配置文件绝对纯净**：
  可在所用的配置文件中全局搜索并删除以下占位行（可选，不删也完全不影响使用）：
  * `DOMAIN-SUFFIX,campus.edu.cn,DIRECT`
  * `"+.campus.edu.cn": "1.2.3.4"`
  * `campus.edu.cn: 10.0.0.1`

### 2. 对于需要接入校园网/内网专线的用户
公共加密 DNS（如 AliDNS DoH、DNSPod DoH）通常无法解析高校/企业内网专有的教务系统、图书馆外文学术数据库、WebVPN 或内部 OA 域名；而如果直接将全部网络交给校园网明文 DNS，又会导致外网访问被劫持监听或发生严重 DNS 泄露。
因此推荐通过 **“专线定向分流 (Split DNS) + 内网直连白名单”** 的方式实现双赢：

#### 各客户端修改方法

* **Mihomo (Clash Meta / OpenClash)**：
  1. `dns.nameserver-policy`：将 `"+.campus.edu.cn": "1.2.3.4"` 中的域名替换为您学校的顶级域名，IP 替换为您学校的内网 DNS 服务器（例如 `"+.your-univ.edu.cn": "10.x.x.x"`）。
  2. `dns.fake-ip-filter`：将 `"+.campus.edu.cn"` 替换为学校域名，确保内网域名获得真实 IP 直连。
  3. `rules`：确保 `DOMAIN-SUFFIX,您的学校域名,DIRECT` 位于直连规则首位。

* **Loon (macOS / iOS)**：
  1. `[Host]` 专线绑定：添加学校内网专线映射，如 `*.your-univ.edu.cn = server: 10.x.x.x`。
  2. 软路由 Wi-Fi 触发器：将 `[Host]` 与策略组中的 `ssid: 你的软路由wifi名 = server: 10.0.0.1` 替换为您家里的 Wi-Fi 名称（在家自动直连软路由，出门自动切换代理）。
  3. `[Rule]`：将 `DOMAIN-SUFFIX, campus.edu.cn, DIRECT` 替换为学校域名。

* **Shadowrocket (小火箭)**：
  1. `[Host]`：添加行 `*.your-univ.edu.cn = server: 10.x.x.x`。
  2. `[Rule]`：将 `DOMAIN-SUFFIX, campus.edu.cn, DIRECT` 替换为学校域名。

* **Sing-box (1.14+)**：
  1. `dns.servers`：添加内网专线 DNS 节点，如 `{"type": "udp", "tag": "dns-campus", "server": "10.x.x.x"}`。
  2. `dns.rules`：添加定向解析规则 `{"domain_suffix": ["your-univ.edu.cn"], "server": "dns-campus"}`。
  3. `route.rules`：添加直连路由 `{"domain_suffix": ["your-univ.edu.cn"], "outbound": "DIRECT"}`。

#### 自动化覆写 (推荐)
如果您使用本地的 `local/sync/synconfig.sh` 脚本进行自动化构建与订阅同步，只需在脚本开头的变量区填入学校参数：
```bash
CAMPUS_DOMAIN="+.your-univ.edu.cn"
CAMPUS_RAW_DOMAIN="your-univ.edu.cn"
CAMPUS_DNS="10.x.x.x"
CAMPUS_ROUTER_IP="10.0.0.1"
```
脚本执行时将自动把校园网专线规则无缝注入到全部平台的输出配置中；若将 `CAMPUS_DOMAIN=""` 置空，则会自动生成完全不含校园网规则的纯净通用版本。

## 参考

- [Zephyruso/zashboard](https://github.com/Zephyruso/zashboard) — 现代 Web 控制面板 (Mihomo & Sing-box)
- [DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets) — 四端规则编译管线与 Sing-box .srs 二进制规则集
- [DustinWin/ruleset_geodata](https://github.com/DustinWin/ruleset_geodata) — Mihomo 规则源 (private/ai/media/trackerslist)
- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) — Mihomo 规则源 (geosite/geoip) + GeoIP 数据库
- [echs-top/proxy](https://github.com/echs-top/proxy) — Mihomo 规则源 (google/cn/captcha/proxy-ip)
- [GinsRule-git](https://github.com/DonJone/GinsRule-git) — Loon/Shadowrocket 规则 + Mihomo github.mrs
- [Koolson/Qure](https://github.com/Koolson/Qure) — 策略组图标
- [juewuy/ShellCrash](https://github.com/juewuy/ShellCrash) — fake-ip-filter 列表

