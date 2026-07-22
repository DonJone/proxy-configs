# Proxy Configurations

为 Mihomo (Clash Meta)、Loon、Shadowrocket 提供跨平台代理配置。

## 架构概览

目前本仓库主推 **Region (按地区/节点手选)** 策略组架构，共享同一套规则源和 DNS 设计：

| 架构 | 策略组 | 适用场景 | 平台支持 |
|------|--------|----------|----------|
| **Region** | 3 区手选池 → 7 业务组 | 极简稳定，掌控感强，无自动测速开销 | Mihomo, Shadowrocket |
| **Region SSID** | 业务组嵌套 SSID 策略 | 推荐软路由局域网环境下使用 | Loon 专用 |

### Region 纯手选

```text
亚太 (FilterAsiaPacific + exclude 低倍)
欧美 (FilterEuAm + exclude 低倍)
低倍 (FilterLowRate)
  ↓
AI与Google / TikTok / Emby流媒体Github / 通讯 / 游戏平台 / 微软苹果Nvidia / 加密货币与兜底
```

核心理念：无 `url-test`、无 `fallback`，极致精简的策略组设计。底层通过正则过滤出【亚太】、【欧美】、【低倍】三大手选节点池，上层具体业务组直接引用对应的手选池。此方案没有自动测速负担，适合机场稳定、清楚自身节点分布的用户。

## 规则来源

| 平台 | 规则源 | 格式 |
|------|--------|------|
| Mihomo | [DustinWin](https://github.com/DustinWin/ruleset_geodata) + [MetaCubeX](https://github.com/MetaCubeX/meta-rules-dat) + [echs-top](https://github.com/echs-top/proxy) + [GinsRule-git](https://github.com/DonJone/GinsRule-git) (github.mrs) | MRS |
| Loon | [GinsRule-git](https://github.com/DonJone/GinsRule-git) | .lsr Remote Rule |
| Shadowrocket | [GinsRule-git](https://github.com/DonJone/GinsRule-git) | .list RULE-SET |

### Mihomo 规则映射

| 规则集 | 来源 | 目标组 |
|--------|------|--------|
| `ai` | DustinWin | **AI与Google** |
| `google`, `google-cn`, `google_ip` | echs-top / MetaCubeX | **AI与Google** |
| `media`, `media_ip` | DustinWin | **Emby流媒体Github** |
| `telegram`, `telegram_ip` | MetaCubeX / echs-top | **通讯** |
| `x`, `fcm` | MetaCubeX | **通讯** |
| `github` + `DOMAIN-KEYWORD,github` | GinsRule-git | **通讯** |
| `tiktok` | MetaCubeX | **TikTok** |
| `apple`, `microsoft` | MetaCubeX | **微软苹果Nvidia** |
| `apple-cn`, `microsoft-cn` | MetaCubeX | **微软苹果Nvidia** |
| `games-cn` | MetaCubeX | **游戏平台** |
| `captcha` | echs-top | **加密货币与兜底** |
| `trackerslist` | DustinWin | **加密货币与兜底** |
| `proxy_domain`, `proxy-lite`, `proxy_ip` | MetaCubeX / echs-top | **加密货币与兜底** |
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

| 特性 | Mihomo | Loon | Shadowrocket |
|------|--------|------|-------------|
| 规则来源 | DustinWin + MetaCubeX + echs-top (.mrs) | GinsRule-git (.lsr) | GinsRule-git (.list) |
| 策略组过滤 | `filter` / `exclude-filter` | Remote Filter NameRegex | `policy-regex-filter` |
| DNS | fake-ip + respect-rules | `dns-server=system` | `dns-server` + `fallback-dns-server` |
| 嗅探 | sniffer 配置块 | `sni-sniffing=true` | 内置 |
| QUIC | AND 规则 → 代理QUIC | — | `block-quic=all-proxy` |

Loon 和 Shadowrocket 不支持 mihomo 的规则集联动 DNS/嗅探，架构相对简单。

## 目录结构

```text
proxy-configs/
├── mihomo/
│   ├── mihomo_Region.yaml            # Region 纯手选 (推荐)
│   └── mihomo_Region_openclash.yaml  # Region (OpenClash 适配)
├── loon/
│   └── loon_Region_ssid.lcf          # Region 纯手选 + SSID 软路由直连
├── Shadowrocket/
│   └── Shadowrocket_Region.conf      # Region 纯手选 (推荐)
└── README.md
```

## 下载链接

基于 jsDelivr CDN 加速的远程配置文件链接：

### Mihomo / OpenClash

| 适用环境 | 架构 | 链接 |
|----------|------|------|
| 通用 (Desktop/Mobile) | Region | [mihomo_Region.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/mihomo_Region.yaml) |
| OpenClash 路由 | Region | [mihomo_Region_openclash.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/mihomo_Region_openclash.yaml) |

### Loon

| 架构 | 链接 |
|------|------|
| Region SSID | [loon_Region_ssid.lcf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/loon_Region_ssid.lcf) |

> 注：Loon 相关的其他配置备份及其 Fallback 等版本在本仓库的 `minor` 分支中维护。

### Shadowrocket

| 架构 | 链接 |
|------|------|
| Region | [Shadowrocket_Region.conf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/Shadowrocket_Region.conf) |

## 参考

- [DustinWin/ruleset_geodata](https://github.com/DustinWin/ruleset_geodata) — Mihomo 规则源 (private/ai/media/trackerslist)
- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) — Mihomo 规则源 (geosite/geoip) + GeoIP 数据库
- [echs-top/proxy](https://github.com/echs-top/proxy) — Mihomo 规则源 (google/cn/captcha/proxy-ip)
- [GinsRule-git](https://github.com/DonJone/GinsRule-git) — Loon/Shadowrocket 规则 + Mihomo github.mrs
- [Koolson/Qure](https://github.com/Koolson/Qure) — 策略组图标
- [juewuy/ShellCrash](https://github.com/juewuy/ShellCrash) — fake-ip-filter 列表
