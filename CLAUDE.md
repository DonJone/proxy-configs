# CLAUDE.md — Proxy Configs 项目维护指南

## 项目概述

为 Mihomo (Clash Meta)、Loon、Shadowrocket、Sing-box 提供跨平台代理配置。

- **Mihomo / Loon / Sing-box**: 规则来源 [DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets) (四大核心规则集: .mrs / .lsr / .srs)
- **Shadowrocket**: 规则来源 [GinsRule-git](https://github.com/DonJone/GinsRule-git) (.list)

## 文件命名约定

```
mihomo_Region.yaml            → Mihomo 通用客户端四大块架构 (推荐)
mihomo_Region_openclash.yaml  → OpenClash 路由四大块架构
mihomo_Region_core.yaml       → 路由器 / 嵌入式 Linux 裸核专用 (四大块架构 + 零泄露纯密 DoH3 + 秒启 + 假死免疫)
mihomo_country.yaml           → Country (待重建)
mihomodeskmob.yaml            → ABC 线路 (待重建)
```

Loon 文件命名约定：
```
loon_Region_ssid.lcf          → Loon iPhone / iOS 移动端四大块架构 (含移动去广告插件与任务脚本)
loon_Region_mac.lcf           → Loon macOS 桌面端四大块架构 (纯净网络栈，DoH 加密，零移动插件)
```
Shadowrocket 文件（`.conf`）遵循相同命名。

Sing-box 文件命名约定：
```
singbox_Region.json           → Sing-box 1.14.3+ 官方最新标准配置 (强类型 DNS/FakeIP + http_clients + Tun + 四大 .srs 规则集 + Zashboard)
```

## 策略组命名字典

以下组名跨越所有文件和平台，修改时需全部同步：

| 组名 | 类型 | 职责 |
|------|------|------|
| `亚太` | select (手选池) | FilterAsiaPacific, exclude 低倍 |
| `欧美` | select (手选池) | FilterEuAm, exclude 低倍 |
| `AI` | select (手选池) | FilterAI (日/美/英/欧 + 新加坡), exclude 低倍 |
| `低倍自动` | url-test (自动池) | FilterLowRate, interval=180, tolerance=30, lazy |
| `低倍` | select (手选池) | 包含 低倍自动 (url-test) + FilterLowRate |
| `AI与Google` | select (业务组) | ai + google + google-cn + google_ip + fcm (遗留组) |
| `TikTok` | select (业务组) | tiktok 规则集 (遗留组) |
| `Emby流媒体Github` | select (业务组) | youtube + media + media_ip + github 规则集 (默认低倍) (遗留组) |
| `通讯` | select (业务组) | telegram + telegram_ip + x (遗留组) |
| `游戏平台` | select (业务组) | games-cn + steam 规则集 (遗留组) |
| `微软苹果Nvidia` | select (业务组) | apple + microsoft + apple-cn + microsoft-cn (遗留组) |
| `兜底` | select (业务组) | trackerslist + proxy_domain + proxy-lite, MATCH 兜底 (默认低倍) |
| `国内服务` | select (静默组) | 国内流量手动兜底 (默认 DIRECT), hidden |
| `代理DNS` | select (基础设施) | DoH 代理链 (proxy-doh 经此组路由), hidden |
| `代理QUIC` | select (基础设施) | UDP 443 控制 (REJECT 强制 TCP 回退), hidden |

## 架构

### Region — 纯手选 (Shadowrocket, 15 个策略组)

```
3 hand-select pools (亚太/欧美/AI) + 1 select pool (低倍) → 7 business groups → 直接引用池
```

亚太/欧美/AI 保持手选，低倍为 select 手选组（包含低倍自动 url-test 测速池及低倍节点）。业务组直指 `[亚太, 欧美, AI, 低倍, DIRECT]`，其中 Emby流媒体Github 与 兜底 默认优选 `低倍`。

> **注**: Fallback 备选架构（`_fallback` 后缀文件）已废弃移除。Loon 与 Mihomo 原 15 策略组旧架构已分别归档至 `archive_loon_legacy` 与 `archive_mihomo_legacy` 分支。

### 四大块收敛架构 (Mihomo, Loon, Sing-box: Direct / APAC / AI / LowRate)

在 Mihomo、Loon 与 Sing-box 客户端中，依托 `DonJone/proxy-rulesets` 将庞杂规则收敛为四大规则集：
- `direct.mrs` / `direct.lsr` / `direct.srs` -> `DIRECT`（国内 IP、国内域名、Steam/Epic 等游戏切片下载 CDN 100% 直连）
- `apac.mrs` / `apac.lsr` / `apac.srs` -> `亚太`（Telegram 核心 IP/域名、Twitter/X、Steam/Epic 商城与社区）
- `ai.mrs` / `ai.lsr` / `ai.srs` -> `AI`（OpenAI、Claude、Gemini、Copilot、Grok、Mistral 等）
- `lowrate.mrs` / `lowrate.lsr` / `lowrate.srs` -> `低倍`（YouTube、Media/Netflix/Disney+/Emby、GitHub、海外大盘）
- `MATCH` / `FINAL` / `final` -> `兜底`（包含低倍、亚太、AI、DIRECT）

## 规则路由映射

平铺规则，无 SUB-RULE。域名规则先于 IP 规则，IP 规则带 `no-resolve`。

| 规则集 | 目标组 | 平台 |
|--------|--------|------|
| `direct` / `direct_ip` | **DIRECT** | Mihomo / Loon / Sing-box |
| `apac` / `apac_ip` | **亚太** | Mihomo / Loon / Sing-box |
| `ai` / `ai_ip` | **AI** | Mihomo / Loon / Sing-box |
| `lowrate` / `lowrate_ip` | **低倍** | Mihomo / Loon / Sing-box |

规则匹配顺序：`预留地址 → 私有地址 → 国内直连 → QUIC → 代理业务 → IP 分流 → 端口识别 → GEOIP → MATCH`

Shadowrocket 规则通过 Remote Rule URL 逐个引用，沿用 GinsRule-git 的 `.list` 格式规则。Mihomo 使用 .mrs 二进制规则集，Sing-box 使用 .srs 二进制规则集，Loon 使用 .lsr 规则集。组名与内核保持一致。

## DNS 架构

- `enhanced-mode: fake-ip`, `fake-ip-filter-mode: blacklist` (ShellCrash 列表)
- `respect-rules: true` — 规则匹配结果自动选择 DNS 服务器
- `nameserver: *direct-doh` (alidns/doh.pub) — 直连域名用国内 DoH
- `proxy-server-nameserver: *proxy-doh` (dns.google/dns.quad9.net via `#代理DNS`)
- `nameserver-policy`: 仅 `AWAvenue-Ads → rcode://name_error`
- `hosts`: DNS 服务器域名硬编码 IP，避免冷启动死锁
- `geodata-mode: true` + `GEOIP,CN,DIRECT` 兜底

Sing-box 采用双 DoH 架构：国内流量经由 `dns-direct` (Alidns DoH) 直连解析，代理流量通过 `dns-fakeip` 分配 Fake-IP，未收录类型及兜底流量由 `dns-proxy` (Google DoH via 兜底) 解析。

## 平台差异

| 特性 | Mihomo | Loon | Shadowrocket | Sing-box |
|------|--------|------|-------------|----------|
| 规则来源 | echs-top/proxy (MRS) | GinsRule-git (.lsr) | GinsRule-git (.list) | proxy-rulesets (.srs) |
| 策略组过滤 | `filter` / `exclude-filter` | Remote Filter NameRegex | `policy-regex-filter` | 手选池 (selector) + 测速 (urltest) |
| DNS | fake-ip + nameserver-policy | `dns-server` (阿里/腾讯/system) | `dns-server` + `fallback-dns-server` | fake-ip (198.18/15) + 双 DoH 路由 |
| 嗅探 | sniffer 配置块 | `sni-sniffing=true` | 内置 | `action: sniff` (首行路由嗅探) |
| QUIC | AND 规则 → 代理QUIC | — | `block-quic=all-proxy` | `action: reject` (UDP 443 拦截) |

Loon 和 Shadowrocket 不支持 mihomo 的三系统联动（规则集驱动 DNS/嗅探）。Sing-box 采用 1.14+ 现代 Action 语法与强类型 DNS / http_clients 架构。

## 修改指南

### 同步要求
- **每次对配置进行任何修改完成后，都必须同步推送（Push）到 GitHub (gh) 仓库。**

### 检修与检修报告规范
- 定期周检与巡检报告统一归档在 `reports/` 专用目录下（命名格式：`weekly_inspection_report_YYYY-MM-DD.md`）。
- 检修发现的配置缺陷在全平台修复后，报告同步归档至 `reports/` 并一同推送至 GitHub 仓库。

### 添加/删除策略组
1. 修改 `mihomo_Region.yaml`（主文件）的 `proxy-groups` 节
2. 同步到 OpenClash、Loon、Shadowrocket、Sing-box 对应文件
3. 如需 fallback 版本，同步到 `_fallback` 文件
4. 更新 README.md 和本文件

### 修改规则集来源 (Mihomo)
1. 修改 `rule-providers` 节对应条目的 `url`
2. 同步更新 `rules` / `dns.nameserver-policy` / `dns.fake-ip-filter` / `sniffer` 中引用
3. 所有 mihomo 文件（desktop/OpenClash/Mobile_Modules）规则集名称必须一致

### 修改规则集来源 (Loon/Shadowrocket/Sing-box)
1. 修改 `[Remote Rule]` 节对应的 URL 或 Sing-box `route.rule_set` URL
2. Loon 和 Shadowrocket 不共享 rule-providers，需分别修改

### 修改组名
1. 跨全部配置文件搜索旧组名
2. 同步更新 README.md 和本文件的命名字典

### RegEx Filter 注意事项
- Mihomo `filter` 不支持负向零宽断言 `(?<!...)`，使用 `exclude-filter` 代替
- Loon `NameRegex` 支持 `(?<!...)` 负向回顾
- Shadowrocket `policy-regex-filter` 使用 `^(?i)(?!...).*` 格式

### 规则顺序约束 (Mihomo & Sing-box)
- **域名规则必须在 IP 规则之前**，否则 IP 规则（无 `no-resolve` 时）会解析 fake-IP 并劫持域名流量
- IP 规则统一加 `no-resolve`，避免对域名连接做多余的 DNS 解析
- QUIC 拦截置于国内直连之后、代理规则之前
- **Telegram MTProto 避让铁律**：Telegram 纯 IP 规则 (`telegram_ip` 或 `apac.srs`) 必须置于 QUIC 拦截之前，确保 MTProto UDP 443 握手流量不被误拦截；裸核/路由器 TUN 模式必须接管 Telegram 纯 IP 流量。

## 外部参考

```
proxy-rulesets:     https://github.com/DonJone/proxy-rulesets  — 四端规则编译管线与 Sing-box SRS
zashboard:          https://github.com/Zephyruso/zashboard    — 现代 Web 控制面板
echs-top/proxy:     https://github.com/echs-top/proxy          — Mihomo MRS 规则
GinsRule-git:       https://github.com/DonJone/GinsRule-git    — Loon/SR 规则
MetaCubeX:          https://github.com/MetaCubeX/meta-rules-dat — GeoIP + x.mrs
Qure icons:         https://github.com/Koolson/Qure             — 策略组图标
wool_scripts:       https://github.com/fmz200/wool_scripts      — AI 规则 / Loon 插件
ShellCrash:         https://github.com/juewuy/ShellCrash        — fake-ip-filter
```

