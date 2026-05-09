# CLAUDE.md — Proxy Configs 项目维护指南

## 项目概述

为 Mihomo (Clash Meta)、Loon、Shadowrocket 提供跨平台代理配置。规则来自 [GinsRule-git](https://github.com/DonJone/GinsRule-git)，策略组架构自建。

## 文件命名约定

```
mihomo_Region.yaml    → 区域聚合（3区：亚太/欧美/低倍）
mihomo_country.yaml   → 国家拆分（9国：港/澳/台/日/新/美/欧/其他/低倍）
mihomodeskmob.yaml    → ABC线路（3条：A/B/C + 全局自动）
```

Loon 和 Shadowrocket 的文件（`.lcf` / `.conf`）遵循相同命名规则。

## 策略组命名字典

以下组名跨越所有文件，修改时需全部同步：

| 组名 | 职责 |
|---|---|
| `AI与Google` | ai + google + google-cn 规则集 |
| `Emby流媒体` | media + emby 规则集（YouTube/Netflix/Disney/Spotify/HBO/Emby） |
| `通讯与Github` | telegram + telegram_ip + github 规则集 |
| `加密货币与兜底` | Region/Country 架构的 MATCH 兜底组 |
| `漏网之鱼` | ABC 线路架构的 MATCH 兜底组 |
| `代理DNS` | DoH 代理链，DNS 查询经此组路由 |
| `代理QUIC` | UDP 443 控制，REJECT 强制 TCP 回退 |
| `广告拦截` | AWAvenue-Ads 规则集出口 |
| `国内服务` | 国内流量手动兜底 |

## 架构对照

### Region（主力，14 个策略组 + 2 基础设施）
```
3 hand-select pools   → 3 url-test backups     → 3 fallback zones
亚太/欧美/低倍          亚太备用/欧美备用/低倍备用     亚太区/欧美区/低倍区

7 business groups     → all reference fallback zones
4 silent/infra        → 国内服务, 广告拦截, 代理DNS, 代理QUIC
```

### Country（36 个策略组 + 2 基础设施）
```
9 hand-select pools   → 9 url-test backups     → 9 fallback zones
港/澳/台/日/新/美/欧/其他/低倍  港备用...低倍备用    港区/澳区/台区/日区/新区/美区/欧区/其他区/低倍区

7 business groups     → all include 9 country zones
```
欧 包含了加/澳/南美等非亚洲非美国的节点。其他 是不匹配任何特定地区且排除低倍的兜底池。

### ABC 线路（10 个策略组 + 2 基础设施）
```
3 full-node selects   → 3 fallback lines (each → 全局自动)
全局自动 = url-test, include-all, exclude dialer
7 business groups     → all reference 线路 A/B/C
漏网之鱼 = MATCH 兜底
```

## 规则路由映射

GinsRule-git 规则集通过 SUB-RULE 分发：

| 规则集 | SUB-RULE | MATCH 出口 |
|---|---|---|
| fcm | — (直连规则) | 国内服务 |
| captcha | captcha_rules | 加密货币与兜底（或漏网之鱼） |
| ai | ai_rules | AI与Google |
| telegram / telegram_ip | telegram_rules | 通讯与Github |
| media / media_ip | media_rules | Emby流媒体 |
| google / google_ip | google_rules | AI与Google |
| trackerslist | trackerslist_rules | 加密货币与兜底（或漏网之鱼） |
| proxy_domain / github / proxy / proxy_ip | proxy_rules | 加密货币与兜底（或漏网之鱼） |
| google-cn | — (直连规则) | AI与Google |
| apple-cn / microsoft-cn / games-cn / cn / direct_domain / private / private_ip | — (直连规则) | DIRECT |
| AWAvenue-Ads | — (直连规则) | 广告拦截 |

每个 SUB-RULE 第一行处理 `UDP 443 → 代理QUIC`，第二行 `MATCH → 业务组`。

## DNS 架构

- **fake-ip 模式**，blacklist：国内规则集 real-ip，其余 fake-ip
- **nameserver-policy**：逐规则集指定 DoH，国外 → proxy-doh（dns.google/quad9 走代理），国内 → direct-doh（alidns/doh.pub），广告 → `rcode://name_error`
- **hosts**：DNS 服务器域名硬编码 IP，避免 DNS 查询死锁
- **DoH 代理链**：proxy-doh URL 带 `#代理DNS`，经策略组路由

## 修改指南

### 添加/删除策略组
1. 修改 `mihomo_Region.yaml`（主文件）的 `proxy-groups` 节
2. 如果影响规则路由，同步修改 `rules` / `sub-rules` 中的 MATCH 目标
3. 同步到 `mihomo_country.yaml`、`mihomodeskmob.yaml`、`OpenClash/` 下同名文件
4. 同步到 `loon/configs/` 和 `Shadowrocket/configs/` 下对应架构文件
5. 更新 README.md 中的表格

### 修改规则集来源
1. 修改 `rule-providers` 节中对应条目的 `url`
2. 如果更改了规则集名称，同步更新 `rules` / `sub-rules` / `dns.nameserver-policy` / `dns.fake-ip-filter` / `sniffer` 中的引用
3. 所有 mihomo 文件（Region/Country/ABC/OpenClash/Mobile_Modules）中的规则集名称必须一致

### 修改组名
1. 跨全部 12 个配置文件搜索旧组名
2. 排除旧线基文件（`mihomodeskmob.yaml` / `openclash.yaml` 如果未启用）中的旧引用
3. 更新 README.md 中对应的表格

### RegEx Filter 注意事项
- Mihomo 的 `filter` 语法不支持负向零宽断言 `(?<!...)`，使用 `exclude-filter` 代替
- Loon 的 `NameRegex` 支持 `(?<!...)` 负向回顾
- Shadowrocket 的 `policy-regex-filter` 使用 `^(?i)(?!...).*` 格式

## 外部参考

```
GinsRule-git rules: https://github.com/DonJone/GinsRule-git
Qure icons:        https://github.com/Koolson/Qure
Loon rules:        https://github.com/DonJone/ios_rule_script
AI rule bundle:    https://github.com/fmz200/wool_scripts
```

## 平台差异

| 特性 | Mihomo | Loon | Shadowrocket |
|---|---|---|---|
| 规则集格式 | MRS (rule-providers) | Remote Rule URLs | RULE-SET URLs |
| 策略组过滤 | `filter` / `exclude-filter` | Remote Filter NameRegex | `policy-regex-filter` |
| DNS 架构 | fake-ip + nameserver-policy | `dns-server=system` | `dns-server` + `fallback-dns-server` |
| 嗅探 | sniffer 配置块 | `sni-sniffing=true` | 内置 |
| SUB-RULE | 支持 | 不支持 | 不支持 |

Loon 和 Shadowrocket 不支持 mihomo 的三系统联动（规则集驱动 DNS/嗅探）。它们的规则通过远程 URL 列表逐个引用，组名必须与 mihomo 版保持一致。
