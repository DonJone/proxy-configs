# Proxy Configurations

本项目为 Mihomo (Clash Meta)、Loon、Shadowrocket 提供跨平台代理配置。
---

## 区域聚合架构（Region

核心思路：将节点按地理位置和倍率分为三个池，每池独立运作 select → url-test → fallback 链路，通过业务入口组将七大类流量分发到不同区域的 fallback。

### 节点池

```
亚太               →  亚太备用 (url-test)  →  亚太区 (fallback)
欧美               →  欧美备用 (url-test)  →  欧美区 (fallback)
低倍               →  低倍备用 (url-test)  →  低倍区 (fallback)
```

- **手选池**（亚太 / 欧美 / 低倍）— `select`，`include-all: true`，按正则 filter 过滤节点，exclude-filter 排除伪低倍（如 4.0x）
- **测速池**（亚太备用 / 欧美备用 / 低倍备用）— `url-test`，`hidden: true`，tolerance=50，30 秒间隔
- **故障转移池**（亚太区 / 欧美区 / 低倍区）— `fallback`，手选池优先，宕机自动切测速池，恢复自动切回

### 业务入口与规则路由

七个业务入口组将 echs-top 的 27 套规则集分发到不同区域：

| 规则集 | 出口 | 覆盖范围 |
|---|---|---|
| `ai` `google` `google-cn` | **AI与Google** | ChatGPT, Claude, Gemini, Copilot, Grok, Perplexity, Google 全系 |
| `media` `emby` | **Emby流媒体** | YouTube, Netflix, Disney+, HBO, Spotify, Emby/Jellyfin |
| `telegram` `telegram_ip` `github` | **通讯与Github** | Telegram, Twitter, Reddit, GitHub |
| `fcm` `private` `cn` `apple-cn` `microsoft-cn` `games-cn` `direct_domain` | **DIRECT** | FCM 推送、私有网络、国内域名/CDN |
| `AWAvenue-Ads` | **广告拦截** | 广告/跟踪域名（DNS 层 NXDOMAIN + 规则层 REJECT） |
| `proxy` `proxy_domain` `captcha` `trackerslist` | **加密货币与兜底** | 其余代理流量 + MATCH 全局兜底 |

业务入口组的可选出口：

| 业务组 | 可选区域 |
|---|---|
| AI与Google | 欧美区, 亚太区, 低倍率 |
| TikTok | 亚太区, 欧美区, 低倍率 |
| Emby流媒体 | DIRECT, 亚太区, 低倍率, 欧美区 |
| 通讯与Github | 亚太区, 欧美区, 低倍率, DIRECT |
| 游戏平台 | DIRECT, 亚太区, 欧美区, 低倍率 |
| 微软苹果Nvidia | DIRECT, 亚太区, 欧美区, 低倍率 |
| 加密货币与兜底 | 亚太区, 欧美区, 低倍率, DIRECT |

### 静默与基础设施组

| 组名 | 出口 | 用途 |
|---|---|---|
| 国内服务 | DIRECT, 亚太区 | 国内流量兜底（规则未覆盖时的手动切换） |
| 广告拦截 | REJECT | 广告域名规则出口 |
| 代理DNS | 加密货币与兜底, 欧美区, 亚太区 | DoH 代理链（dns.google/quad9 走代理） |
| 代理QUIC | REJECT, PASS | UDP 443 强制 TCP 回退开关 |

---

## ABC 线路架构 — 灵感来源

区域聚合的前身，极致简洁。不按地区筛选节点，三个全节点手选组 + 一个全局 url-test 池：

```
手选节点 A  →  线路 A (fallback: A → 全局自动)
手选节点 B  →  线路 B (fallback: B → 全局自动)
手选节点 C  →  线路 C (fallback: C → 全局自动)
全局自动 (url-test, 全节点, hidden)
```

适合节点数 < 50 或不需要按地区筛选的场景。

---

## 国家拆分架构（Country）

区域聚合的细粒度版本。将节点拆分为 9 个独立国家/地区池：

`港` `澳` `台` `日` `新` `美` `欧` `其他` `低倍`

每个池拥有独立的 select → url-test → fallback 链路，业务入口包含全部 9 个区。适合节点数 200+、对特定国家线路有明确偏好的场景。

---

## 技术骨架（Mihomo 平台）

配置文件共用 GinsRule-git 的三系统联动设计，规则集同时驱动 DNS、分流、嗅探：

### DNS — 四层架构

```
hosts (DNS 引导 IP 绑定)
  → default-nameserver
    → nameserver-policy (按规则集选择 DoH)
      → nameserver (兜底)
```

- `enhanced-mode: fake-ip`，`fake-ip-filter-mode: blacklist`（国内规则集 real-ip，其余 fake-ip）
- `nameserver-policy`：国外规则集 → `dns.google`/`quad9`（走代理），国内规则集 → `alidns`/`doh.pub`（直连），广告规则集 → `rcode://name_error`（NXDOMAIN）
- DoH 代理链：`dns.google/dns-query#代理DNS`，DNS 查询经策略组路由

### 规则引擎 — 三层递进

31 条主规则 + 8 个 SUB-RULE 子规则，按端口 → 域名 → IP 三层递进匹配：

```
端口层  → DST-PORT 5228-5230 (FCM), 1337-7777 (BT)
域名层  → RULE-SET + SUB-RULE (ai, google, telegram, media...)
IP 层   → RULE-SET (cn_ip, telegram_ip, google_ip...)
兜底层  → DST-PORT 10000-65535 (BT), MATCH
```

每个 SUB-RULE 统一处理 UDP 443 → 代理QUIC（强制 TCP），其余 MATCH → 对应业务组。

### 嗅探

HTTP (80/8080-8880) / TLS (443/8443) / QUIC (443/8443)，按规则集三层跳过（skip-domain / skip-src-address / skip-dst-address），HTTP 开启 override-destination。

### 27 套规则集

全部 MRS 格式（`behavior: domain / ipcidr`），来源 [GinsRule-git](https://github.com/DonJone/GinsRule-git)：

```
域名: private, AWAvenue-Ads, fcm, captcha, ai, telegram, media, google-cn,
      google, trackerslist, apple-cn, microsoft-cn, games-cn, proxy_domain,
      direct_domain, github, proxy, cn, dnsmasq-china-add
IP:   private_ip, telegram_ip, media_ip, google_ip, proxy_ip,
      enhanced-FaaS-in-China_ip, direct_ip, cn_ip
```

---

## 目录结构

```
proxy-configs/
├── mihomo/
│   ├── desktop&mobile/           # Mihomo 桌面/移动端内核
│   │   ├── mihomo_Region.yaml    # 区域聚合 (主力)
│   │   ├── mihomo_country.yaml   # 国家拆分
│   │   └── mihomodeskmob.yaml    # ABC 线路
│   ├── OpenClash/                # OpenClash 路由器插件
│   │   ├── openclash_Region.yaml
│   │   ├── openclash_country.yaml
│   │   └── openclash.yaml
│   └── Mobile_Modules/           # 社区模块适配
│       ├── AkashaProxy/          # config.yaml + config_hybrid.yaml
│       ├── BoxProxy/             # config.yaml + config_hybrid.yaml
│       ├── ClashMix/             # config.yaml + config_hybrid.yaml
│       └── Surfing/              # config.yaml + config_hybrid.yaml
├── loon/configs/
│   ├── loon_Region.lcf
│   ├── loon_country.lcf
│   └── loon.lcf
├── Shadowrocket/configs/
│   ├── Shadowrocket_Region.conf
│   ├── Shadowrocket_country.conf
│   └── Shadowrocket.conf
└── README.md
```

---

## 链接

### 区域聚合（Region）

| 平台 | Raw | CDN |
|---|---|---|
| Mihomo Desktop | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/mihomo/desktop%26mobile/mihomo_Region.yaml) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomo_Region.yaml) |
| Mihomo OpenClash | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/mihomo/OpenClash/openclash_Region.yaml) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash_Region.yaml) |
| Loon | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/loon/configs/loon_Region.lcf) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon_Region.lcf) |
| Shadowrocket | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/Shadowrocket/configs/Shadowrocket_Region.conf) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket_Region.conf) |

### 国家拆分（Country）

| 平台 | Raw | CDN |
|---|---|---|
| Mihomo Desktop | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/mihomo/desktop%26mobile/mihomo_country.yaml) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomo_country.yaml) |
| Mihomo OpenClash | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/mihomo/OpenClash/openclash_country.yaml) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash_country.yaml) |
| Loon | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/loon/configs/loon_country.lcf) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon_country.lcf) |
| Shadowrocket | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/Shadowrocket/configs/Shadowrocket_country.conf) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket_country.conf) |

### ABC 线路

| 平台 | Raw | CDN |
|---|---|---|
| Mihomo Desktop | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/mihomo/desktop%26mobile/mihomodeskmob.yaml) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/desktop%26mobile/mihomodeskmob.yaml) |
| Mihomo OpenClash | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/mihomo/OpenClash/openclash.yaml) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/mihomo/OpenClash/openclash.yaml) |
| Loon | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/loon/configs/loon.lcf) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/loon/configs/loon.lcf) |
| Shadowrocket | [Raw](https://raw.githubusercontent.com/DonJone/proxy-configs/master/Shadowrocket/configs/Shadowrocket.conf) | [CDN](https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@master/Shadowrocket/configs/Shadowrocket.conf) |

---

## 社区模块适配（Mobile_Modules）

`mihomo/Mobile_Modules/` 下为四个 Android 模块的适配配置，每个模块提供两套文件：`config.yaml`（模块原生骨架 + 本项目策略组）和 `config_hybrid.yaml`（echs-top 骨架 + 本项目策略组 + 模块设备参数）。

| 模块 | 特点 |
|---|---|
| AkashaProxy | GEOSITE/GEOIP 分流，redir-host DNS，NTP 时间同步，strict 进程匹配 |
| BoxProxy | 极简 MRS 规则，5 条规则覆盖全部分流，gvisor TUN |
| ClashMix | 文件自定义规则 + HTTP 规则集混合，gvisor TUN，三星 VoLTE 豁免 |
| Surfing | 28 规则集精细分流，UA 伪装，每应用独立图标，DNS_Hijack |

---

## 参考

- [GinsRule-git](https://github.com/DonJone/GinsRule-git) — 规则集 / DNS / 嗅探骨架
- [Koolson/Qure](https://github.com/Koolson/Qure) — 策略组图标
- [fmz200/wool_scripts](https://github.com/fmz200/wool_scripts) — AI 规则合集
- [DonJone/ios_rule_script](https://github.com/DonJone/ios_rule_script) — 应用分流规则
