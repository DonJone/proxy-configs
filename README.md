# Proxy Configurations

为 Mihomo (Clash Meta)、Loon、Shadowrocket 提供跨平台代理配置。

## 推荐使用 Region

**Region 是主力架构**，三个平台通用，纯手选、极简、稳定。

```
节点按 Region Filter 分入三个手选池 → 业务组直接引用手选池 → 平铺规则分发流量
```

Country 和 ABC 为备选架构，分别适配「多国家偏好」和「全节点不分区域」场景。

## 架构：纯手选 Region

```
亚太 (select, FilterAsiaPacific + exclude 低倍)
欧美 (select, FilterEuAm + exclude 低倍)
低倍 (select, FilterLowRate)
  ↓ 业务组直接引用
AI与Google:       [欧美, 亚太, 低倍]
TikTok:           [亚太, 欧美, 低倍]
Emby流媒体:        [DIRECT, 亚太, 低倍, 欧美]
通讯与Github:      [亚太, 欧美, 低倍, DIRECT]
游戏平台:          [DIRECT, 亚太, 欧美, 低倍]
微软苹果Nvidia:    [DIRECT, 亚太, 欧美, 低倍]
加密货币与兜底:     [亚太, 欧美, 低倍, DIRECT]
  + 国内服务 [DIRECT, 亚太]  + 广告拦截 [REJECT]
  + 代理DNS [加密货币与兜底, 欧美, 亚太]  + 代理QUIC [REJECT, PASS]
```

14 个策略组，无 url-test 备胎、无 fallback 中转层。手选池过滤后用户直接挑节点，业务组直接走。

### 备选：Fallback 架构

如果偏好自动故障转移，可使用 `_fallback` 后缀的配置文件：

```
亚太 → 亚太备用 (url-test) → 亚太区 (fallback)
欧美 → 欧美备用 (url-test) → 欧美区 (fallback)
低倍 → 低倍备用 (url-test) → 低倍率 (fallback)
```

20 个策略组，手选池优先，节点离线自动切 url-test，恢复后自动切回。

## 规则映射

规则来源 [echs-top/proxy](https://github.com/echs-top/proxy)（27 个 MRS 规则集），平铺规则无 SUB-RULE：

| 规则集 | 目标组 |
|--------|--------|
| `ai`, `google`, `google-cn`, `google_ip` | **AI与Google** |
| `media`, `media_ip` | **Emby流媒体** |
| `telegram`, `telegram_ip`, `x`, `fcm`, `github` | **通讯与Github** |
| `captcha`, `trackerslist`, `proxy_domain`, `proxy`, `proxy_ip` | **加密货币与兜底** |
| `AWAvenue-Ads` | **广告拦截** |
| `apple-cn`, `microsoft-cn`, `games-cn`, `direct_domain`, `cn`, `dnsmasq-china-add`, `direct_ip`, `cn_ip`, `enhanced-FaaS-in-China_ip` | **DIRECT** |
| `private`, `private_ip` | **DIRECT** |

规则匹配顺序：

```
预留地址 → 私有地址 → 广告拦截 → 国内直连域名 → QUIC 拦截
→ AI/通讯/流媒体/Google/开发者/代理兜底 (域名层)
→ IP 级分流 (no-resolve) → 端口识别 → GeoIP → MATCH
```

## DNS

- `enhanced-mode: fake-ip`，`fake-ip-filter-mode: blacklist`（ShellCrash 列表）
- `respect-rules: true` — 规则匹配结果自动选择 DNS 服务器
- 直连域名 → `alidns`/`doh.pub`（国内 DoH）
- 代理域名 → `dns.google`/`quad9`（经 `#代理DNS` 策略组路由）
- 广告域名 → `rcode://name_error`
- `hosts` 硬编码 DNS 服务器 IP，避免冷启动死锁

## 平台支持

| 特性 | Mihomo | Loon | Shadowrocket |
|------|--------|------|-------------|
| 规则格式 | MRS (rule-providers) | Remote Rule URL (.lsr) | RULE-SET URL (.list) |
| 策略组过滤 | `filter` / `exclude-filter` | Remote Filter NameRegex | `policy-regex-filter` |
| DNS | fake-ip + nameserver-policy | `dns-server=system` | `dns-server` + `fallback-dns-server` |
| 嗅探 | sniffer 配置块 | `sni-sniffing=true` | 内置 |
| QUIC | AND 规则 → 代理QUIC | — | `block-quic=all-proxy` |

## 目录结构

```
proxy-configs/
├── mihomo/
│   ├── desktop&mobile/
│   │   ├── mihomo_Region.yaml           # Region 纯手选 (推荐)
│   │   ├── mihomo_Region_fallback.yaml  # Region fallback (备选)
│   │   ├── mihomo_country.yaml          # Country (待重建)
│   │   └── mihomodeskmob.yaml           # ABC (待重建)
│   ├── OpenClash/
│   │   ├── openclash_Region.yaml
│   │   ├── openclash_Region_fallback.yaml
│   │   ├── openclash_country.yaml
│   │   └── openclash.yaml
│   └── Mobile_Modules/
├── loon/configs/
│   ├── loon_Region.lcf                  # Region 纯手选 (推荐)
│   ├── loon_Region_fallback.lcf         # Region fallback (备选)
│   ├── loon_country.lcf
│   └── loon.lcf
├── Shadowrocket/configs/
│   ├── Shadowrocket_Region.conf         # Region 纯手选 (推荐)
│   ├── Shadowrocket_Region_fallback.conf # Region fallback (备选)
│   ├── Shadowrocket_country.conf
│   └── Shadowrocket.conf
└── README.md
```

## 链接

### Region（推荐）

| 平台 | 纯手选 | Fallback 备选 |
|------|--------|-------------|
| Mihomo Desktop | [Region](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomo_Region.yaml) | [Fallback](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomo_Region_fallback.yaml) |
| Mihomo OpenClash | [Region](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash_Region.yaml) | [Fallback](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash_Region_fallback.yaml) |
| Loon | [Region](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon_Region.lcf) | [Fallback](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon_Region_fallback.lcf) |
| Shadowrocket | [Region](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket_Region.conf) | [Fallback](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket_Region_fallback.conf) |

### Country / ABC

| 平台 | Country | ABC |
|------|---------|-----|
| Mihomo Desktop | [Country](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomo_country.yaml) | [ABC](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomodeskmob.yaml) |
| Mihomo OpenClash | [Country](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash_country.yaml) | [ABC](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash.yaml) |
| Loon | [Country](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon_country.lcf) | [默认](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon.lcf) |
| Shadowrocket | [Country](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket_country.conf) | [默认](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket.conf) |

## 参考

- [echs-top/proxy](https://github.com/echs-top/proxy) — 规则集来源
- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) — GeoIP 数据库 + x.mrs
- [Koolson/Qure](https://github.com/Koolson/Qure) — 策略组图标
- [fmz200/wool_scripts](https://github.com/fmz200/wool_scripts) — AI 规则合集 / Loon 插件
- [juewuy/ShellCrash](https://github.com/juewuy/ShellCrash) — fake-ip-filter 列表
