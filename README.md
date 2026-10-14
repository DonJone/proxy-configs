# Proxy Configurations

为 Mihomo (Clash Meta)、Loon、Shadowrocket、Sing-box 提供跨平台代理配置。

## 架构概览

目前本仓库主推 **Region (按地区/节点手选)** 策略组架构，共享同一套规则源和 DNS 设计：

| 架构 | 策略组 | 适用场景 | 平台支持 |
|------|--------|----------|----------|
| **Region** | 4 区手选池 (含低倍自动选优) → 7 业务组 | 极简稳定，低倍节点支持手选与自动测速 | Mihomo, Shadowrocket |
| **Region SSID** | 业务组嵌套 SSID 策略 | 推荐软路由局域网环境下使用 | Loon 专用 |
| **Region (Sing-box)** | 4 大块规则收敛 (直连/亚太/AI/低倍) + 兜底 | 现代标准 Tun/Mixed 协议栈，零冗余秒级流转 | Sing-box 1.11+ |

### Region 纯手选

```text
亚太 (FilterAsiaPacific + exclude 低倍)
欧美 (FilterEuAm + exclude 低倍)
AI (FilterAI + exclude 低倍)
低倍 (FilterLowRate, 手选池，包含 低倍自动 url-test)
  ↓
AI与Google / TikTok / Emby流媒体Github / 通讯 / 游戏平台 / 微软苹果Nvidia / 兜底
```

核心理念：极致精简的策略组设计。底层通过正则过滤出【亚太】、【欧美】、【AI】、【低倍】四大手选节点池（低倍池内置【低倍自动】url-test 测速优选，180s），上层具体业务组直接引用对应池。全部区域与低倍保持手选掌控感，低倍内嵌自动测速保障可用性。其中【Emby流媒体Github】与【兜底】默认优选【低倍】。

### Sing-box 四大块架构 (Direct / APAC / AI / LowRate)

在 Sing-box 客户端中，依托编译管线 `DonJone/proxy-rulesets` 将庞杂规则收敛为四大规则集：
- **DIRECT (`direct.srs`)**：私有地址、国内 IP CIDR、国内域名大盘、Steam/Epic 等游戏切片下载 CDN。
- **亚太 (`apac.srs`)**：Telegram 全量 IP/域名、Twitter/X、Steam/Epic 商城与社交、亚太直连节点。
- **AI (`ai.srs`)**：OpenAI、Claude、Gemini、Copilot、Grok、Mistral 等海外核心 AI 矩阵。
- **低倍 (`lowrate.srs`)**：YouTube、Netflix、Disney+、Emby 流媒体、GitHub、FCM 原生直连推送及海外大盘。
- **兜底 (`final`)**：未收录的海外小众域名及动态 IP，默认交由【兜底】策略组（优选低倍/欧美/亚太）。

## 规则来源

| 平台 | 规则源 | 格式 |
|------|--------|------|
| Mihomo | [DustinWin](https://github.com/DustinWin/ruleset_geodata) + [MetaCubeX](https://github.com/MetaCubeX/meta-rules-dat) + [echs-top](https://github.com/echs-top/proxy) + [GinsRule-git](https://github.com/DonJone/GinsRule-git) (github.mrs) | MRS |
| Loon | [GinsRule-git](https://github.com/DonJone/GinsRule-git) | .lsr Remote Rule |
| Shadowrocket | [GinsRule-git](https://github.com/DonJone/GinsRule-git) | .list RULE-SET |
| Sing-box | [DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets) | .srs Binary Rule-set |

### Mihomo 规则映射

| 规则集 | 来源 | 目标组 |
|--------|------|--------|
| `ai` | DustinWin | **AI与Google** |
| `google`, `google-cn`, `google_ip`, `fcm` | echs-top / MetaCubeX | **AI与Google** |
| `youtube` | MetaCubeX | **Emby流媒体Github** |
| `media`, `media_ip` | DustinWin | **Emby流媒体Github** |
| `github` + `DOMAIN-KEYWORD,github` | GinsRule-git | **Emby流媒体Github** |
| `telegram`, `telegram_ip`, `x` | MetaCubeX / echs-top | **通讯** |
| `tiktok` | MetaCubeX | **TikTok** |
| `apple`, `microsoft` | MetaCubeX | **微软苹果Nvidia** |
| `apple-cn`, `microsoft-cn` | MetaCubeX | **微软苹果Nvidia** |
| `games-cn` | MetaCubeX | **游戏平台** |
| `captcha` | echs-top | **兜底** |
| `trackerslist` | DustinWin | **兜底** |
| `proxy_domain`, `proxy-lite`, `proxy_ip` | MetaCubeX / echs-top | **兜底** |
| `cn`, `dnsmasq-china-lite`, `cn_ip` | echs-top | **DIRECT** |
| `private`, `private_ip` | DustinWin | **DIRECT** |

规则匹配顺序：

```text
预留地址 → 私有地址 → 国内直连域名 → QUIC 拦截
→ AI / 通讯 / 流媒体 / TikTok / 开发者 / 微软苹果 / 游戏 / Google (域名层)
→ IP 级分流 (no-resolve) → 端口识别 → GeoIP CN → MATCH
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
| 规则来源 | DustinWin + MetaCubeX + echs-top (.mrs) | GinsRule-git (.lsr) | GinsRule-git (.list) | proxy-rulesets (.srs) |
| 策略组过滤 | `filter` / `exclude-filter` | Remote Filter NameRegex | `policy-regex-filter` | 手选池 (selector) + 测速 (urltest) |
| DNS | fake-ip + respect-rules | `dns-server` (阿里/腾讯/system) | `dns-server` + `fallback-dns-server` | fake-ip (198.18/15) + 双 DoH 路由 |
| 嗅探 | sniffer 配置块 | `sni-sniffing=true` | 内置 | `sniff: true` (系统级嗅探) |
| QUIC | AND 规则 → 代理QUIC | — | `block-quic=all-proxy` | `action: reject` (UDP 443 拦截) |

Loon 和 Shadowrocket 不支持 mihomo 的规则集联动 DNS/嗅探，架构相对简单。Sing-box 使用现代化 1.11+ Action 语法和独立的二进制规则集路由。

## 目录结构

```text
proxy-configs/
├── mihomo/
│   ├── mihomo_Region.yaml            # Region 纯手选 (推荐)
│   ├── mihomo_Region_openclash.yaml  # Region (OpenClash 适配)
│   └── mihomo_Region_core.yaml       # Region 裸核专用 (Mihomo Standalone Core / OpenWrt 生产级)
├── loon/
│   └── loon_Region_ssid.lcf          # Region 纯手选 + SSID 软路由直连
├── Shadowrocket/
│   └── Shadowrocket_Region.conf      # Region 纯手选 (推荐)
├── singbox/
│   └── singbox_Region.json           # Sing-box 1.11+ 生产级配置 (Tun + Mixed + 4 大块 .srs 远程分流)
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
| 通用 (Desktop/Mobile) | Region | [mihomo_Region.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/mihomo_Region.yaml) |
| OpenClash 路由 | Region | [mihomo_Region_openclash.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/mihomo_Region_openclash.yaml) |
| 嵌入式 Linux / OpenWrt 裸核 | Region Core | [mihomo_Region_core.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/mihomo_Region_core.yaml) |

### Loon

| 架构 | 链接 |
|------|------|
| Region SSID | [loon_Region_ssid.lcf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/loon_Region_ssid.lcf) |

> 注：Loon 相关的其他配置备份及其 Fallback 等版本在本仓库的 `minor` 分支中维护。

### Shadowrocket

| 架构 | 链接 |
|------|------|
| Region | [Shadowrocket_Region.conf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/Shadowrocket_Region.conf) |

### Sing-box

| 适用环境 | 架构 | 链接 |
|----------|------|------|
| 通用客户端 / 裸核 / 路由 (v1.11+) | Region (4 大块) | [singbox_Region.json](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/singbox/singbox_Region.json) |

## 参考

- [DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets) — 四端规则编译管线与 Sing-box .srs 二进制规则集
- [DustinWin/ruleset_geodata](https://github.com/DustinWin/ruleset_geodata) — Mihomo 规则源 (private/ai/media/trackerslist)
- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) — Mihomo 规则源 (geosite/geoip) + GeoIP 数据库
- [echs-top/proxy](https://github.com/echs-top/proxy) — Mihomo 规则源 (google/cn/captcha/proxy-ip)
- [GinsRule-git](https://github.com/DonJone/GinsRule-git) — Loon/Shadowrocket 规则 + Mihomo github.mrs
- [Koolson/Qure](https://github.com/Koolson/Qure) — 策略组图标
- [juewuy/ShellCrash](https://github.com/juewuy/ShellCrash) — fake-ip-filter 列表

