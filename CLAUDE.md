# CLAUDE.md — Proxy Configs 项目维护指南

## 项目概述

为 Mihomo (Clash Meta)、Loon、Shadowrocket 提供跨平台代理配置。

- **Mihomo**: 规则来源 [echs-top/proxy](https://github.com/echs-top/proxy) (MRS) + [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) (GeoIP + x.mrs)
- **Loon / Shadowrocket**: 规则来源 [GinsRule-git](https://github.com/DonJone/GinsRule-git) (.lsr / .list)

## 文件命名约定

```
mihomo_Region.yaml            → Region 纯手选 (推荐)
mihomo_Region_fallback.yaml   → Region fallback 备选
mihomo_country.yaml           → Country (待重建)
mihomodeskmob.yaml            → ABC 线路 (待重建)
```

Loon 和 Shadowrocket 文件（`.lcf` / `.conf`）遵循相同命名。

## 策略组命名字典

以下组名跨越所有文件和平台，修改时需全部同步：

| 组名 | 类型 | 职责 |
|------|------|------|
| `亚太` | select (手选池) | FilterAsiaPacific, exclude 低倍 |
| `欧美` | select (手选池) | FilterEuAm, exclude 低倍 |
| `AI` | select (手选池) | FilterAI, exclude 低倍 |
| `低倍自动` | fallback (自动池) | FilterLowRate, interval=180 |
| `低倍` | select (手选池) | 包含 低倍自动 + FilterLowRate |
| `AI与Google` | select (业务组) | ai + google + google-cn + google_ip + fcm |
| `TikTok` | select (业务组) | tiktok 规则集 |
| `Emby流媒体Github` | select (业务组) | media + media_ip + github 规则集 |
| `通讯` | select (业务组) | telegram + telegram_ip + x |
| `游戏平台` | select (业务组) | games-cn + steam 规则集 |
| `微软苹果Nvidia` | select (业务组) | apple + microsoft + apple-cn + microsoft-cn |
| `加密货币与兜底` | select (业务组) | trackerslist + proxy_domain + proxy-lite, MATCH 兜底 |
| `国内服务` | select (静默组) | 国内流量手动兜底 (默认 DIRECT), hidden |
| `代理DNS` | select (基础设施) | DoH 代理链 (proxy-doh 经此组路由), hidden |
| `代理QUIC` | select (基础设施) | UDP 443 控制 (REJECT 强制 TCP 回退), hidden |

## 架构

### Region — 纯手选 (主力, 14 个策略组)

```
2 hand-select pools (亚太/欧美) + 1 select-fallback pool (低倍) → 7 business groups → 直接引用池
```

亚太/欧美/AI 保持手选，低倍改为手选并默认包含低倍自动 fallback 组。业务组直指 `[亚太, 欧美, AI, 低倍, DIRECT]`。

> **注**: Fallback 备选架构（`_fallback` 后缀文件）已废弃移除。

## 规则路由映射

平铺规则，无 SUB-RULE。域名规则先于 IP 规则，IP 规则带 `no-resolve`。

| 规则集 | 目标组 | 平台 |
|--------|--------|------|
| `ai`, `google`, `google-cn`, `google_ip` | **AI与Google** | Mihomo |
| `media`, `media_ip` | **Emby流媒体Github** | Mihomo |
| `telegram`, `telegram_ip`, `x`, `fcm`, `github` | **通讯** | Mihomo |
| `captcha`, `trackerslist`, `proxy_domain`, `proxy`, `proxy_ip` | **加密货币与兜底** | Mihomo |
| `AWAvenue-Ads` | **广告拦截** | Mihomo |
| `apple-cn`, `microsoft-cn`, `games-cn`, `direct_domain`, `cn`, `dnsmasq-china-add` | **DIRECT** | Mihomo |
| `direct_ip`, `cn_ip`, `enhanced-FaaS-in-China_ip` | **DIRECT** (no-resolve) | Mihomo |
| `private`, `private_ip` | **DIRECT** | Mihomo |

规则匹配顺序：`预留地址 → 私有地址 → 广告拦截 → 国内直连 → QUIC → 代理业务 → IP 分流 → 端口识别 → GEOIP → MATCH`

Loon / Shadowrocket 规则通过 Remote Rule URL 逐个引用，内核不支持 MRS，沿用 GinsRule-git 的 `.lsr` / `.list` 格式规则。组名与 Mihomo 保持一致。

## DNS 架构

- `enhanced-mode: fake-ip`, `fake-ip-filter-mode: blacklist` (ShellCrash 列表)
- `respect-rules: true` — 规则匹配结果自动选择 DNS 服务器
- `nameserver: *direct-doh` (alidns/doh.pub) — 直连域名用国内 DoH
- `proxy-server-nameserver: *proxy-doh` (dns.google/dns.quad9.net via `#代理DNS`)
- `nameserver-policy`: 仅 `AWAvenue-Ads → rcode://name_error`
- `hosts`: DNS 服务器域名硬编码 IP，避免冷启动死锁
- `geodata-mode: true` + `GEOIP,CN,DIRECT` 兜底

## 平台差异

| 特性 | Mihomo | Loon | Shadowrocket |
|------|--------|------|-------------|
| 规则来源 | echs-top/proxy (MRS) | GinsRule-git (.lsr) | GinsRule-git (.list) |
| 策略组过滤 | `filter` / `exclude-filter` | Remote Filter NameRegex | `policy-regex-filter` |
| DNS | fake-ip + nameserver-policy | `dns-server=system` | `dns-server` + `fallback-dns-server` |
| 嗅探 | sniffer 配置块 | `sni-sniffing=true` | 内置 |
| QUIC | AND 规则 → 代理QUIC | — | `block-quic=all-proxy` |

Loon 和 Shadowrocket 不支持 mihomo 的三系统联动（规则集驱动 DNS/嗅探）。

## 修改指南

### 同步要求
- **每次对配置进行任何修改完成后，都必须同步推送（Push）到 GitHub (gh) 仓库。**

### 添加/删除策略组
1. 修改 `mihomo_Region.yaml`（主文件）的 `proxy-groups` 节
2. 同步到 OpenClash、Loon、Shadowrocket 对应文件
3. 如需 fallback 版本，同步到 `_fallback` 文件
4. 更新 README.md 和本文件

### 修改规则集来源 (Mihomo)
1. 修改 `rule-providers` 节对应条目的 `url`
2. 同步更新 `rules` / `dns.nameserver-policy` / `dns.fake-ip-filter` / `sniffer` 中引用
3. 所有 mihomo 文件（desktop/OpenClash/Mobile_Modules）规则集名称必须一致

### 修改规则集来源 (Loon/Shadowrocket)
1. 修改 `[Remote Rule]` 节对应的 URL
2. Loon 和 Shadowrocket 不共享 rule-providers，需分别修改

### 修改组名
1. 跨全部配置文件搜索旧组名
2. 同步更新 README.md 和本文件的命名字典

### RegEx Filter 注意事项
- Mihomo `filter` 不支持负向零宽断言 `(?<!...)`，使用 `exclude-filter` 代替
- Loon `NameRegex` 支持 `(?<!...)` 负向回顾
- Shadowrocket `policy-regex-filter` 使用 `^(?i)(?!...).*` 格式

### 规则顺序约束 (Mihomo)
- **域名规则必须在 IP 规则之前**，否则 IP 规则（无 `no-resolve` 时）会解析 fake-IP 并劫持域名流量
- IP 规则统一加 `no-resolve`，避免对域名连接做多余的 DNS 解析
- QUIC 拦截置于国内直连之后、代理规则之前

## 外部参考

```
echs-top/proxy:     https://github.com/echs-top/proxy          — Mihomo MRS 规则
GinsRule-git:       https://github.com/DonJone/GinsRule-git    — Loon/SR 规则
MetaCubeX:          https://github.com/MetaCubeX/meta-rules-dat — GeoIP + x.mrs
Qure icons:         https://github.com/Koolson/Qure             — 策略组图标
wool_scripts:       https://github.com/fmz200/wool_scripts      — AI 规则 / Loon 插件
ShellCrash:         https://github.com/juewuy/ShellCrash        — fake-ip-filter
```
