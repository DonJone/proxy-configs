# Proxy Configurations

为 Mihomo (Clash Meta)、Loon、Shadowrocket 提供跨平台代理配置。

## 架构概览

三种策略组架构，共享同一套规则源和 DNS 设计：

| 架构 | 策略组 | 适用场景 |
|------|--------|----------|
| **Region** | 3 区手选池 → 7 业务组 | 推荐，极简稳定 |
| **Region SSID** | 业务组嵌套 SSID 策略 | 推荐软路由局域网环境下使用 (Loon 专用) |
| **Region Fallback** | 3 区手选 + url-test 备胎 + fallback 自动切换 | 需要自动故障转移 |
| **Country** | 9 区手选 + url-test 备胎 + fallback 自动切换 | 偏好特定国家/地区节点 |
| **ABC** | 3 线路手选 + 全局测速 + fallback 自动切换 | 全节点不分区域 |

### Region 纯手选 (推荐)

```
亚太 (FilterAsiaPacific + exclude 低倍)
欧美 (FilterEuAm + exclude 低倍)
低倍 (FilterLowRate)
  ↓
AI与Google / TikTok / Emby流媒体 / 通讯与Github / 游戏平台 / 微软苹果Nvidia / 加密货币与兜底
```

14 个策略组，无 url-test、无 fallback。手选池按 Filter 过滤节点，业务组直接引用。

### Fallback 备选

```
亚太 → 亚太备用 (url-test) → 亚太区 (fallback)
欧美 → 欧美备用 (url-test) → 欧美区 (fallback)
低倍 → 低倍备用 (url-test) → 低倍率 (fallback)
```

20 个策略组，手选池优先，故障自动切测速池，恢复后切回。

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
| `media`, `media_ip` | DustinWin | **Emby流媒体** |
| `telegram`, `telegram_ip` | MetaCubeX / echs-top | **通讯与Github** |
| `x`, `fcm` | MetaCubeX | **通讯与Github** |
| `github` + `DOMAIN-KEYWORD,github` | GinsRule-git | **通讯与Github** |
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

```
预留地址 → 私有地址 → 国内直连域名 → QUIC 拦截
→ AI / 通讯 / 流媒体 / TikTok / 开发者 / 微软苹果 / 游戏 / Google (域名层)
→ IP 级分流 (no-resolve) → 端口识别 → GeoIP CN → MATCH
```

> **关键顺序**: AI/YouTube 必须在 Google 之前 (google.mrs 含 youtube 域名)，GitHub 必须在 Microsoft 之前 (microsoft.mrs 含 github 域名)。

## DNS

- `enhanced-mode: fake-ip`，`fake-ip-filter-mode: blacklist`（ShellCrash 列表）
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

```
proxy-configs/
├── mihomo/
│   ├── desktop&mobile/
│   │   ├── mihomo_Region.yaml            # Region 纯手选 (推荐)
│   │   ├── mihomo_Region_fallback.yaml   # Region fallback 备选
│   │   ├── mihomo_country.yaml           # Country 9 区 fallback
│   │   └── mihomodeskmob.yaml            # ABC 线路 fallback
│   ├── OpenClash/
│   │   ├── openclash_Region.yaml         # = mihomo_Region (OpenClash 适配)
│   │   ├── openclash_Region_fallback.yaml
│   │   ├── openclash_country.yaml        # = mihomo_country (OpenClash 适配)
│   │   └── openclash.yaml                # = Region (默认入口)
│   └── Mobile_Modules/
│       └── Surfing/config_hybrid.yaml
├── loon/configs/
│   ├── loon_Region.lcf                   # Region 纯手选 (推荐)
│   ├── loon_Region_ssid.lcf              # Region 纯手选 + SSID 软路由直连
│   ├── loon_Region_fallback.lcf          # Region fallback 备选
│   ├── loon_country.lcf                  # Country
│   └── loon.lcf                          # 默认入口
├── Shadowrocket/configs/
│   ├── Shadowrocket_Region.conf          # Region 纯手选 (推荐)
│   ├── Shadowrocket_Region_fallback.conf # Region fallback 备选
│   ├── Shadowrocket_country.conf         # Country
│   └── Shadowrocket.conf                 # 默认入口
└── README.md
```

## 下载链接

### Mihomo Desktop

| 架构 | 链接 |
|------|------|
| Region (推荐) | [mihomo_Region.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomo_Region.yaml) |
| Region Fallback | [mihomo_Region_fallback.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomo_Region_fallback.yaml) |
| Country | [mihomo_country.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomo_country.yaml) |
| ABC | [mihomodeskmob.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomodeskmob.yaml) |

### Mihomo OpenClash

| 架构 | 链接 |
|------|------|
| Region (推荐) | [openclash_Region.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash_Region.yaml) |
| Region Fallback | [openclash_Region_fallback.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash_Region_fallback.yaml) |
| Country | [openclash_country.yaml](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash_country.yaml) |

### Loon

| 架构 | 链接 |
|------|------|
| Region (推荐) | [loon_Region.lcf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon_Region.lcf) |
| Region SSID (软路由) | [loon_Region_ssid.lcf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/loon_Region_ssid.lcf) |
| Region Fallback | [loon_Region_fallback.lcf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon_Region_fallback.lcf) |
| Country | [loon_country.lcf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon_country.lcf) |

### Shadowrocket

| 架构 | 链接 |
|------|------|
| Region (推荐) | [Shadowrocket_Region.conf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket_Region.conf) |
| Region Fallback | [Shadowrocket_Region_fallback.conf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket_Region_fallback.conf) |
| Country | [Shadowrocket_country.conf](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket_country.conf) |

## 参考

- [DustinWin/ruleset_geodata](https://github.com/DustinWin/ruleset_geodata) — Mihomo 规则源 (private/ai/media/trackerslist)
- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) — Mihomo 规则源 (geosite/geoip) + GeoIP 数据库
- [echs-top/proxy](https://github.com/echs-top/proxy) — Mihomo 规则源 (google/cn/captcha/proxy-ip)
- [GinsRule-git](https://github.com/DonJone/GinsRule-git) — Loon/Shadowrocket 规则 + Mihomo github.mrs
- [Koolson/Qure](https://github.com/Koolson/Qure) — 策略组图标
- [juewuy/ShellCrash](https://github.com/juewuy/ShellCrash) — fake-ip-filter 列表
