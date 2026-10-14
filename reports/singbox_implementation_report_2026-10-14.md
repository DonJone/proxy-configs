# Sing-box 1.14+ 官方最新稳定版生产级配置实现报告

## 1. 交付概况

基于全新构建的规则编译产物与四层流向收敛架构，现已在工作区完成 Sing-box 官方最新发布版（Latest Release v1.14.3+）统一生产级配置的落地与端到端严格校验。

- 物理配置文件：[singbox_Region.json](file:///Users/don/work/proxy/proxy-configs/singbox/singbox_Region.json)
- 规则源仓库：[DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets)
- 远程发布分支：`release`
- 标定内核：Sing-box 最新稳定版 (Latest Release v1.14.3+ darwin/arm64)
- 官方规范参考：本地 Sing-box 官方文档与 JSON Schema ([sing-box/docs/schema.json](file:///Users/don/work/proxy/sing-box/docs/schema.json))

---

## 2. 核心架构设计与最新规范演进

### 2.1 规则收敛与流量四分法

为消除以往几十个分散规则集之间的交叉污染与顺序竞争，全量流量被严格收敛为四大核心规则集：

| 规则集标签 | 远程 SRS 地址 | 目标出站 | 包含核心内容 |
|---|---|---|---|
| `rule-direct` | `https://fastly.jsdelivr.net/gh/DonJone/proxy-rulesets@release/rules/singbox/direct.srs` | `DIRECT` | 国内 IP (16,896 条 CIDR)、国内全量域名、Steam/Epic 等游戏切片下载 CDN |
| `rule-apac` | `https://fastly.jsdelivr.net/gh/DonJone/proxy-rulesets@release/rules/singbox/apac.srs` | `亚太` | Telegram 专属 IP/域名、Twitter/X、Steam/Epic 商城与社交社区 |
| `rule-ai` | `https://fastly.jsdelivr.net/gh/DonJone/proxy-rulesets@release/rules/singbox/ai.srs` | `AI` | OpenAI、Claude、Gemini、Copilot、Grok、Mistral 等海外核心 AI 矩阵 |
| `rule-lowrate` | `https://fastly.jsdelivr.net/gh/DonJone/proxy-rulesets@release/rules/singbox/lowrate.srs` | `低倍` | YouTube、Netflix、Disney+、Emby、GitHub、Google Drive/Photos、海外大盘 |

### 2.2 游戏平台动静物理级解耦

针对此前游戏平台商城选亚太后下载游戏误走亚太的问题，通过在规则生成器层面对游戏流量进行物理拆分：
* 商城/社交/社区域名（如 `steampowered.com`, `steamcommunity.com`, `epicgames.com`）进入 `rule-apac`，走亚太低延迟节点。
* 安装包/切片下载 CDN（如 `steamcontent.com`, `download.epicgames.com`）进入 `rule-direct`，强制直连走本地中国 CDN，满速下载且零代理流量损耗。

### 2.3 最新标准下的 HTTP 客户端与规则集分流 (`http_clients`)

Sing-box 1.14.0 废弃了远程规则集选项中的 `download_detour`，全面引入基于标签解耦的 `route.rule_set.http_client` 架构。在根级定义强隔离的 HTTP 客户端：
* `http-direct`：空出站，直连下载直连类规则集。
* `http-proxy`：绑定 `detour: "兜底"`，走代理出站下载海外受阻或加速节点资源。
* 彻底消除了 v1.14+ 启动时的 `WARN legacy download_detour remote rule-set option is deprecated in 1.14.0` 警告。

### 2.4 强类型 DNS 架构与底层防回环

严格适配 Sing-box 1.12+ / 1.14+ 强类型 DNS 对象体系：
* `dns-direct`：`{"type": "https", "server": "223.5.5.5"}`，直连 DNS 显式省去 `detour`（杜绝 1.14+ 中 `detour to an empty direct outbound makes no sense` 的致命报错）。
* `dns-proxy`：`{"type": "https", "server": "dns.google", "domain_resolver": "dns-direct", "detour": "兜底"}`，采用 1.14+ 新规范 `domain_resolver` 直连引导 Google DoH。
* `dns-fakeip`：内置 Fake-IP 服务器，网段分配 `198.18.0.0/15` 与 `fc00::/18`，彻底移除根级废弃的 `dns.fakeip` 块及性能损耗字段 `independent_cache` / `store_rdrc`。
* 路由引导：配置 `route.default_domain_resolver: "dns-direct"`，彻底取代旧版已废弃的 DNS 规则 `outbound: "any"`。

### 2.5 MTProto 避让与 QUIC 智能拦截

遵循项目核心避让规范：
1. 基础地址防探测：`169.254.169.254/32` 立即执行 `action: reject`。
2. NTP 时间防偏移：端口 123 与 NTP 域名强制走 `DIRECT`。
3. FCM 推送保护：端口 `5228-5230` 强制走 `DIRECT`，防止推送握手超时。
4. **规则顺序**：
   - 优先匹配 `rule-direct`（保证国内应用的 HTTP/3 QUIC 畅通）；
   - 其次匹配 `rule-apac`（保证 Telegram MTProto UDP 443 语音与握手不被误伤）；
   - 然后拦截海外 QUIC：`action: reject, network: udp, port: 443`（阻止海外网页与 YouTube QUIC 假死卡顿）；
   - 最后分流 `rule-ai` 与 `rule-lowrate`，最终兜底至 `兜底`。

### 2.6 协议嗅探与高性能 Tun 协议栈

* **路由首行嗅探**：入站监听器中的 `sniff: true` 在 1.14.0 被标记为废弃；现代标准在 `route.rules` 首行定义 `{"action": "sniff"}` 进行全量流量的精准协议与域名提取。
* **Tun 协议栈**：移除已废弃的 `stack: "system"` 及自 1.11.0 起已失效的 `endpoint_independent_nat`，默认启用 Sing-box 官方自研高性能 `sing-tun` 网络栈。

### 2.7 Web 控制面板 (Zashboard 集成)

* 配置 Clash API 扩展模块，监听地址 `0.0.0.0:9090`，支持局域网设备与本地浏览器访问。
* 静态资源路径指定为 `zashboard`，自动拉取上游发布包 `https://github.com/Zephyruso/zashboard/releases/latest/download/dist.zip`。
* 首次启动时内核自动解压至本地目录，访问 `http://<ip>:9090/ui` 即可体验与 Mihomo 一致的现代可视化控制面板。

---

## 3. 严格校验结果

### 3.1 官方二进制校验 (sing-box check)

使用官方最新稳定版内核 `sing-box version 1.14.3 darwin/arm64` 进行审查：
```bash
/Users/don/work/proxy/proxy-rulesets/bin/sing-box check -c /Users/don/work/proxy/proxy-configs/singbox/singbox_Region.json
```
结果：
- 退出码：`0`
- 标准输出 / 标准错误：完全纯净，**0 警告、0 废弃项提示、0 错误**。

### 3.2 官方 JSON Schema 严格比对

通过 Python `jsonschema` 针对 `/Users/don/work/proxy/sing-box/docs/schema.json` 对 [singbox_Region.json](file:///Users/don/work/proxy/proxy-configs/singbox/singbox_Region.json) 进行全结构递归审查：
- 校验结果：`Validation Succeeded with 0 errors!`
- 完全契合 Sing-box 官方仓库最新数据模式定义。

### 3.3 规则集命中逆校验 (sing-box rule-set match)

针对编译好的 `.srs` 二进制产物进行抽样靶向断言：

| 目标域名 / IP | 预期规则集 | 匹配状态 | 规则匹配明细 |
|---|---|---|---|
| `steampowered.com` | `apac.srs` | 成功 | match rules.[0]: domain/domain_suffix |
| `steamcontent.com` | `direct.srs` | 成功 | match rules.[0]: domain/domain_suffix |
| `epicgames.com` | `apac.srs` | 成功 | match rules.[0]: domain/domain_suffix |
| `download.epicgames.com` | `direct.srs` | 成功 | match rules.[0]: domain/domain_suffix |
| `api.openai.com` | `ai.srs` | 成功 | match rules.[0]: domain/domain_suffix |
| `youtube.com` | `lowrate.srs` | 成功 | match rules.[0]: domain/domain_suffix |
| `baidu.com` | `direct.srs` | 成功 | match rules.[0]: domain/domain_suffix |
| `114.114.114.114` | `direct.srs` | 成功 | match rules.[0]: ip_cidr |
| `91.108.4.1` (Telegram) | `apac.srs` | 成功 | match rules.[0]: ip_cidr |

全部测试靶点 100% 精准归流，动静解耦机制与 Telegram 避让逻辑完全生效。
