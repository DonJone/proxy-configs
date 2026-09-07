# Proxy-Configs (Public) 周检报告

> **周检时间**：2026-09-07  
> **检查范围**：`DonJone/proxy-configs` (master 分支)  
> **覆盖平台**：Mihomo (Desktop/ShellCrash)、OpenClash、Loon (SSID)、Shadowrocket (Region)  
> **检查维度**：远端规则源健康度、跨平台分流规则一致性、DNS 与 DoH 基础设施、语法与正则校验、敏感信息与隐私合规

---

## 一、检查总体结论

本次周检对 `proxy-configs` 仓库的 4 份核心配置文件及文档进行了深度静态分析与全量网络自动化审计：

| 检查维度 | 检查项数 | 状态 | 核心发现 |
| :--- | :--- | :--- | :--- |
| **远端规则源健康度** | 45 个外部资源链接 | ⚠️ 存在缺陷 (已修复) | 发现 1 处 404 规则（Loon FCM）与 1 处 404 图标（Spotify） |
| **跨平台规则一致性** | 143 条内联自定义规则 | ⚠️ 存在遗漏 (已修复) | Mihomo 遗漏主关键词 `emby` 与 `douyin`；Loon 规则顺序错位 |
| **配置语法与正则安全** | 4 个配置文件 | ✅ 100% 通过 | YAML 解析无误，节点过滤正则与负向断言测试全部通过 |
| **文档与代码一致性** | CLAUDE.md / README.md | ⚠️ 描述陈旧 (已修正) | 文档表格中 `github` 与 `fcm` 的路由组与代码实际行为脱节 |
| **隐私安全与脱敏审计** | 全库扫描 | ✅ 100% 合规 | 机场链接、SSID 均采用标准占位符，无凭据或私有 IP 泄露 |

> [!IMPORTANT]
> 发现的问题已完成**全平台代码修复、文档订正与自动化回测**，并已按照规范提交同步推送至 GitHub 远程仓库 (`master` 分支)。

---

## 二、远端规则源与网络基础设施可用性检查

使用自动化探测脚本对全平台引用的规则源、GeoIP 数据库、图标集、脚本与插件链接进行了 HTTP 状态检查与负载验证：

### 1. 规则集提供者（Rule Providers）
*   **MetaCubeX / meta-rules-dat**（GeoSite / GeoIP MRS）：
    *   `googlefcm`, `telegram`, `x`, `tiktok`, `apple`, `microsoft`, `apple@cn`, `microsoft@cn`, `category-games@cn`, `google@cn`, `geolocation-!cn`, `google` (geoip) 全部返回 **HTTP 200**。
*   **DustinWin / ruleset_geodata**（MRS）：
    *   `private`, `privateip`, `ai`, `media`, `mediaip`, `trackerslist` 全部返回 **HTTP 200**。
*   **echs-top / proxy**（MRS）：
    *   `proxy-ltsc`, `cn`, `dnsmasq-china-lite`, `telegram` (ip), `cn` (ip) 全部返回 **HTTP 200**。
*   **ShellCrash**：
    *   `fake_ip_filter.list` 规则文件返回 **HTTP 200**。
*   **GinsRule-git**（Loon `.lsr` / Shadowrocket `.list`）：
    *   **异常拦截**：`rules/loon/proxy/googlefcm.lsr` 返回 **HTTP 404 Not Found**（原因分析见下文第三节）。
    *   其余所有 Loon/Shadowrocket 规则源（YouTube, Netflix, Disney, HBO, Spotify, Telegram, Twitter, GitHub, TikTok, Apple, Microsoft, Steam, GFWList 等）均返回 **HTTP 200**。

### 2. DNS 与 DoH 基础设施验证
*   对配置中引用的 DoH 解析节点（`dns.google`, `dns.quad9.net`, `dns.alidns.com`, `cloudflare-dns.com`, `doh.pub`）发起标准 DNS wireformat 查询报文测试，**全部正常返回 200 响应与有效解析结果**。
*   `mihomo` 中的 DoH 代理链（`#代理DNS` 经由海外代理分流解析）逻辑完整。

### 3. 插件、脚本与图标资源
*   **kelee.one 防泄漏与工具插件**：在模拟 Loon 专用 User-Agent 环境下，所有 11 个插件及 MMDB 离线库（`Country-Masaiki.mmdb`, `GeoLite2-ASN-P3TERX.mmdb`）均正常拉取（HTTP 200）。
*   **第三方脚本与插件**：Sub-Store 解析脚本、FMZ200 脚本合集、去广告插件等 16 个 jsDelivr 托管资源均全部返回 HTTP 200。
*   **异常拦截**：Loon 中引用的 Spotify 插件图标 `fmz200/wool_scripts@main/icons/apps/spotify.png` 返回 **HTTP 404**。

---

## 三、跨平台一致性与规则审计（发现的问题）

### 1. 【致命缺陷】Loon FCM 规则 404 错误
*   **问题描述**：在先前的一次提交 (`910017d`) 中，误将 `loon/loon_Region_ssid.lcf` 中的 `googlefcm.list` 批量重命名为了 `googlefcm.lsr`。但远端 `DonJone/GinsRule-git` 仓库中实际保存的文件名为 `googlefcm.list`，导致 Loon 客户端在更新规则时遭遇 404 错误，FCM 规则加载失败。
*   **修复动作**：订正为 `googlefcm.list`，经回测 HTTP 状态恢复为 200 OK。

### 2. 【功能缺陷】Mihomo 遗漏核心主关键词 `emby` 与 `douyin`
*   **问题描述**：
    *   Loon 和 Shadowrocket 均显式配置了 `DOMAIN-KEYWORD, emby, Emby流媒体Github` 与 `DOMAIN-KEYWORD, douyin, 国内服务`。
    *   但在 `mihomo/mihomo_Region.yaml` 与 `mihomo_Region_openclash.yaml` 中，虽然拥有 100+ 个具体的私服关键词，却唯独缺失了统领性的主关键词 `DOMAIN-KEYWORD,emby,Emby流媒体Github`。这会导致未经单独收录的新建/自建 Emby 域名无法被该规则兜底。
    *   同时 Mihomo 缺少 `douyin` 直连关键词。
*   **修复动作**：在 Mihomo 两份配置文件中统一补全 `DOMAIN-KEYWORD,douyin,DIRECT` 与 `DOMAIN-KEYWORD,emby,Emby流媒体Github`。

### 3. 【规范优化】Loon 规则顺序组织错位
*   **问题描述**：在 `loon/loon_Region_ssid.lcf` 中，`DOMAIN-KEYWORD, libvio, DIRECT` 和 `DOMAIN, train.suuwu.de, DIRECT` 被遗漏在 100+ 个 Emby 规则之后、`[Remote Rule]` 之前的 `# 拦截` 区块下方，与顶部 `# 优先直连` 区块割裂。
*   **修复动作**：将这两个直连规则上移归入顶部 `# 优先直连` 区块，保持三端规则组织完全对称。

### 4. 【资源失效】Loon Spotify 图标 404
*   **问题描述**：`loon/loon_Region_ssid.lcf` 第 295 行的 `spotify.png` 图标链接失效 (404)。
*   **修复动作**：替换为仓库统一使用的 Qure 官方图标源 `https://cdn.jsdelivr.net/gh/Koolson/Qure@master/IconSet/Color/Spotify.png` (200 OK)。

### 5. 【文档陈旧】`CLAUDE.md` 与 `README.md` 分流映射表偏差
*   **问题描述**：
    *   在之前的提交 `b5c3f0b` 中，`github` 规则集已从 `通讯` 全面改入 `Emby流媒体Github`，且 `fcm` 规则全平台均指向 `AI与Google`。
    *   但 `CLAUDE.md` 和 `README.md` 的路由映射表中仍记录 `github -> 通讯` 和 `fcm -> 通讯`，与代码实际行为不符。
    *   `mihomo_Region_openclash.yaml` 头部注释仍残留“3 区纯手选”，未同步当前低倍测速架构。
*   **修复动作**：全量更新文档与头部注释中的策略组归属描述。

---

## 四、配置语法与正则表达式验证

### 1. YAML 语法验证
*   对 [mihomo_Region.yaml](file:///Users/don/work/proxy/proxy-configs/mihomo/mihomo_Region.yaml) 和 [mihomo_Region_openclash.yaml](file:///Users/don/work/proxy/proxy-configs/mihomo/mihomo_Region_openclash.yaml) 进行了完整的 YAML Schema 解析，锚点引用（`*domain`, `*ip`, `*Select`, `*FilterAsiaPacific` 等）解析正常，键值结构 100% 合法。

### 2. 正则表达式有效性验证
对三端所使用的节点过滤正则进行了真实节点名称仿真测试：
*   **亚太节点过滤** (`FilterAsiaPacific`)：港/台/日/韩/新/马/印/澳等国家代码及中文别名匹配率 100%，且精确通过 `exclude-filter` 排除低倍节点。
*   **欧美节点过滤** (`FilterEuAm`)：美/加/英/德/法/荷/俄/意/瑞等匹配正常，低倍节点成功剥离。
*   **低倍节点识别** (`FilterLowRate`)：`0.1x`、`0.5x`、`0.2倍`、`低倍` 均能被准确识别并归入自动测速容灾池。
*   **Shadowrocket / Loon 负向前瞻**：`^(?i)(?!.*((?<![0-9.])0(?:\.[0-7]\d*)?\s*[xX倍]|低倍)).*` 语法符合 iOS/macOS 客户端正则引擎要求。

---

## 五、已落地的修复清单

本次周检执行并完成提交的代码变更如下：

```
 CLAUDE.md                           | 6 +++---
 README.md                           | 7 +++----
 loon/loon_Region_ssid.lcf           | 8 ++++----
 mihomo/mihomo_Region.yaml           | 2 ++
 mihomo/mihomo_Region_openclash.yaml | 8 +++++---
 5 files changed, 17 insertions(+), 14 deletions(-)
```

*   **Git 提交信息**：`fix(weekly-check): 修复规则源404/跨平台Emby分流遗漏与文档同步`
*   **提交哈希**：`37b9e28`
*   **远端状态**：已推送到 GitHub `origin/master` (`https://github.com/DonJone/proxy-configs.git`)。

---

## 六、维护建议

1.  **GinsRule-git 规则扩展名规范**：GinsRule-git 中 Loon 规则多数为 `.lsr`，但极少数（如 `googlefcm.list`）仍保留 `.list` 扩展名。后续若对规则源做扩展名重构，需先确认远端仓库文件是否存在再行替换。
2.  **jsDelivr 缓存刷新**：由于已推送到 `master` 分支，通过 jsDelivr 获取新配置的用户通常在 CDN 缓存失效（最长 12 小时）后自动生效，紧急情况下可通过 `https://purge.jsdelivr.net/` 刷新特定链接缓存。
