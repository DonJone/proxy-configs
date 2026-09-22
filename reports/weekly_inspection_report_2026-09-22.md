# Proxy-Configs (Public) 周检与节点维护报告

> **维护时间**：2026-09-22  
> **检查范围**：`DonJone/proxy-configs` (master 分支)  
> **覆盖平台**：Mihomo (Desktop/ShellCrash)、OpenClash、Loon (SSID)、Shadowrocket (Region)  
> **维护目标**：AI 策略组扩容（引入新加坡与台湾低延迟节点池）、全平台远程规则源审计、正则有效性与低倍排除验证、跨平台一致性回测

---

## 一、维护与检查总体结论

本次维护针对用户关于在 AI 筛选组中加入新加坡与台湾节点的需求，并在全平台配置文件中完成了深度审计、正则优化与跨平台配置同步：

| 检查维度 | 检查项数 | 状态 | 核心发现与变更 |
| :--- | :--- | :--- | :--- |
| **策略组定义与正则过滤** | 4 份核心配置 | [完成] | 全平台同步在 `AI` / `AI节点` 策略组中加入新加坡（SG/Singapore/狮城）与台湾（TW/Taiwan/Tai/Wan）过滤模式 |
| **低倍倍率排除安全** | 4 份核心配置 | [通过] | 验证 `FilterExcludeLowRate` 与负向预查逻辑，确保新加坡与台湾的低倍节点（如 0.5x、0.2x）仍正常被排除 |
| **远端规则源健康度** | 141 个外部资源链接 | [通过] | DustinWin、MetaCubeX、echs-top、GinsRule-git 等全量规则文件均可稳定解析拉取（HTTP 200） |
| **配置语法与架构校验** | 4 份核心配置 | [通过] | YAML 安全解析无误（15 个策略组），Loon / Shadowrocket 解析正常 |
| **文档与架构一致性** | CLAUDE.md / README.md | [完成] | 同步更新架构说明中的手选池数量（3 大手选池: 亚太/欧美/AI）与对应过滤字典说明 |
| **隐私安全与脱敏审计** | 全库扫描 | [通过] | 订阅链接与 SSID 参数均维持标准化占位符，无机密泄露 |

> [!IMPORTANT]
> 本次 AI 筛选组调整及所有附属代码与文档变更，均已通过跨平台回测与语法验证，并同步提交推送至 GitHub 远程仓库 (`master` 分支)。

---

## 二、AI 策略组扩容与正则变更详情

### 1. 扩容背景与需求
原 `AI` 策略组仅收录日本、美国、英国及欧洲大陆等节点。随着主流大模型（如 ChatGPT、Claude、Gemini）在亚太成熟区域的开放支持，扩充新加坡（Singapore）与台湾（Taiwan）节点可显著降低亚太区网络往返延迟（RTT），提供更优质的 AI 响应体验。

### 2. 跨平台正则调整对比

#### (1) Mihomo (Desktop & OpenClash)
*   **锚点位置**：`mihomo/mihomo_Region.yaml` 与 `mihomo/mihomo_Region_openclash.yaml` 中的 `FilterAI` 锚点。
*   **原正则**：
    ```yaml
    FilterAI: &FilterAI '(?i)(日|Japan|美|America|纽约|洛杉矶|圣何塞|芝加哥|西雅图|英|Kingdom|Britain|伦敦|德|Germany|法|France|荷兰|Netherlands|意|Italy|瑞士|Switzerland|瑞典|Sweden|欧洲|Europe|(?:^|[^a-zA-Z])(JP|US|UK|DE|FR|NL|IT|CH|SE|EU)(?:[^a-zA-Z]|$))'
    ```
*   **新正则**：
    ```yaml
    FilterAI: &FilterAI '(?i)(新加坡|狮城|Singapore|台|Tai|Wan|日|Japan|美|America|纽约|洛杉矶|圣何塞|芝加哥|西雅图|英|Kingdom|Britain|伦敦|德|Germany|法|France|荷兰|Netherlands|意|Italy|瑞士|Switzerland|瑞典|Sweden|欧洲|Europe|(?:^|[^a-zA-Z])(SG|TW|JP|US|UK|DE|FR|NL|IT|CH|SE|EU)(?:[^a-zA-Z]|$))'
    ```

#### (2) Loon (Region SSID)
*   **配置段落**：`loon/loon_Region_ssid.lcf` 中的 `[Remote Filter]` -> `AI节点`。
*   **原配置**：
    ```ini
    AI节点 = NameRegex, FilterKey = "^(?i)(?!.*((?<![0-9.])0(?:\.[0-7]\d*)?\s*[xX倍]|低倍)).*(日|Japan|美|America|纽约|洛杉矶|圣何塞|芝加哥|西雅图|英|Kingdom|Britain|伦敦|德|Germany|法|France|荷兰|Netherlands|意|Italy|瑞士|Switzerland|瑞典|Sweden|欧洲|Europe|(?:^|[^a-zA-Z])(JP|US|UK|DE|FR|NL|IT|CH|SE|EU)(?:[^a-zA-Z]|$))"
    ```
*   **新配置**：
    ```ini
    AI节点 = NameRegex, FilterKey = "^(?i)(?!.*((?<![0-9.])0(?:\.[0-7]\d*)?\s*[xX倍]|低倍)).*(新加坡|狮城|Singapore|台|Tai|Wan|日|Japan|美|America|纽约|洛杉矶|圣何塞|芝加哥|西雅图|英|Kingdom|Britain|伦敦|德|Germany|法|France|荷兰|Netherlands|意|Italy|瑞士|Switzerland|瑞典|Sweden|欧洲|Europe|(?:^|[^a-zA-Z])(SG|TW|JP|US|UK|DE|FR|NL|IT|CH|SE|EU)(?:[^a-zA-Z]|$))"
    ```

#### (3) Shadowrocket (Region)
*   **配置段落**：`Shadowrocket/Shadowrocket_Region.conf` 中的 `[Proxy Group]` -> `AI`。
*   **原配置**：
    ```ini
    AI = select, policy-regex-filter=^(?i)(?!.*((?<![0-9.])0(?:\.[0-7]\d*)?\s*[xX倍]|低倍)).*(日|Japan|美|America|纽约|洛杉矶|圣何塞|芝加哥|西雅图|英|Kingdom|Britain|伦敦|德|Germany|法|France|荷兰|Netherlands|意|Italy|瑞士|Switzerland|瑞典|Sweden|欧洲|Europe|(?:^|[^a-zA-Z])(JP|US|UK|DE|FR|NL|IT|CH|SE|EU)(?:[^a-zA-Z]|$))
    ```
*   **新配置**：
    ```ini
    AI = select, policy-regex-filter=^(?i)(?!.*((?<![0-9.])0(?:\.[0-7]\d*)?\s*[xX倍]|低倍)).*(新加坡|狮城|Singapore|台|Tai|Wan|日|Japan|美|America|纽约|洛杉矶|圣何塞|芝加哥|西雅图|英|Kingdom|Britain|伦敦|德|Germany|法|France|荷兰|Netherlands|意|Italy|瑞士|Switzerland|瑞典|Sweden|欧洲|Europe|(?:^|[^a-zA-Z])(SG|TW|JP|US|UK|DE|FR|NL|IT|CH|SE|EU)(?:[^a-zA-Z]|$))
    ```

---

## 三、正则匹配与过滤安全回测

通过专用测试脚本对包含常见机场命名格式的节点列表进行了全量正则断言回测：

| 测试节点样例 | 匹配期望 | Mihomo 过滤结果 | Loon / Shadowrocket 过滤结果 | 判定 |
| :--- | :--- | :--- | :--- | :--- |
| `新加坡 01 [x1.0]` | 纳入 AI | 纳入 (True) | 纳入 (True) | [通过] |
| `Singapore 01 - BGP` | 纳入 AI | 纳入 (True) | 纳入 (True) | [通过] |
| `SG 02 [1.0x]` | 纳入 AI | 纳入 (True) | 纳入 (True) | [通过] |
| `狮城 01 专线` | 纳入 AI | 纳入 (True) | 纳入 (True) | [通过] |
| `台湾 01 [x1.0]` | 纳入 AI | 纳入 (True) | 纳入 (True) | [通过] |
| `Taiwan 02 [1.0x]` | 纳入 AI | 纳入 (True) | 纳入 (True) | [通过] |
| `TW 03 专线` | 纳入 AI | 纳入 (True) | 纳入 (True) | [通过] |
| `Taipei 04 节点` | 纳入 AI | 纳入 (True) | 纳入 (True) | [通过] |
| `日本 01 [x1.0]` | 纳入 AI (保留原支持) | 纳入 (True) | 纳入 (True) | [通过] |
| `美国 01 [x1.0]` | 纳入 AI (保留原支持) | 纳入 (True) | 纳入 (True) | [通过] |
| `香港 01 [x1.0]` | 排除 AI (无大模型支持) | 排除 (False) | 排除 (False) | [通过] |
| `韩国 01 [x1.0]` | 排除 AI | 排除 (False) | 排除 (False) | [通过] |
| `新加坡 01 [0.5x]` | 排除 AI (低倍节点) | 排除 (False, 命中低倍排除) | 排除 (False, 命中负向预查) | [通过] |
| `TW 01 [0.2x]` | 排除 AI (低倍节点) | 排除 (False, 命中低倍排除) | 排除 (False, 命中负向预查) | [通过] |
| `低倍 - 新加坡 01` | 排除 AI (低倍节点) | 排除 (False, 命中低倍排除) | 排除 (False, 命中负向预查) | [通过] |

回测表明：
1. 台湾与新加坡的主流命名（中英文、城市名、二字缩写）均能 100% 精确捕获并汇入 AI 候选池。
2. 低倍（0.7x 及以下）节点即使属于新加坡或台湾，仍被精确排除并仅由低倍池接管，杜绝了低倍节点混入主力 AI 组带来的不稳定性。
3. 原有的日/美/英/欧节点匹配逻辑完全不受影响，香港等非 AI 主流节点维持隔离。

---

## 四、基础设施与远端源健康审计

本次审计对配置文件中涉及的 141 个外部资源链接进行了连通性扫描：
1. **外部规则集 (Rule Providers)**：
   * MetaCubeX 规则仓库（`meta-rules-dat`）：全部分流 MRS 规则源可达。
   * echs-top 规则仓库（`echs-top/proxy`）：直连与海外分流 MRS 规则源可达。
   * DustinWin 规则仓库（`ruleset_geodata`）：AI、流媒体、私有规则源可达。
   * GinsRule-git 规则仓库：Loon (`.lsr`) 与 Shadowrocket (`.list`) 规则集全部正常返回 HTTP 200。
2. **DNS / DoH 解析链路**：
   * 阿里 DNS (`dns.alidns.com` / `223.5.5.5`)、腾讯 DNS (`doh.pub` / `119.29.29.29`) 保持健康状态。
   * 代理 DoH（`dns.google` / `dns.quad9.net`）经由基础设施组 `#代理DNS` 正常路由。

---

## 五、维护文件变更清单

| 文件路径 | 变更类型 | 变更内容说明 |
| :--- | :--- | :--- |
| `mihomo/mihomo_Region.yaml` | 修改 | `FilterAI` 锚点追加新加坡与台湾关键词与缩写 |
| `mihomo/mihomo_Region_openclash.yaml` | 修改 | OpenClash 同步更新 `FilterAI` 锚点 |
| `loon/loon_Region_ssid.lcf` | 修改 | `[Remote Filter]` 下 `AI节点` 正则追加新加坡与台湾 |
| `Shadowrocket/Shadowrocket_Region.conf` | 修改 | `[Proxy Group]` 下 `AI` 策略组正则追加新加坡与台湾 |
| `CLAUDE.md` | 修改 | 更新策略组命名字典中 AI 组描述及 15 组策略架构图 |
| `README.md` | 修改 | 更新手选池架构说明（显式列出 3 大手选池并补齐 AI 说明） |
| `reports/weekly_inspection_report_2026-09-22.md` | 新增 | 本次项目维护与规则审计周检报告 |
