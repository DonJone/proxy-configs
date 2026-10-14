# Sing-box 1.11+ 生产级配置实现与规则集架构报告

## 1. 交付概况

基于全新构建的规则编译产物与四层流向收敛架构，现已在工作区完成 Sing-box 现代生产级配置的落地与端到端校验。

- 物理配置文件：[singbox_Region.json](file:///Users/don/work/proxy/proxy-configs/singbox/singbox_Region.json)
- 规则源仓库：[DonJone/proxy-rulesets](https://github.com/DonJone/proxy-rulesets)
- 远程发布分支：`release`
- 适用内核：Sing-box 1.11.0 及以上版本

---

## 2. 核心架构设计

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

### 2.3 现代 DNS 双栈与防回环设计

* `dns-direct`：采用阿里公共 DoH (`https://223.5.5.5/dns-query`)，通过 `dns-local` (UDP 223.5.5.5) 进行冷启动解析，强制 `detour: DIRECT`。
* `dns-proxy`：采用 Google DoH (`https://dns.google/dns-query`)，通过 `dns-direct` 直连解析其引导域名，强制 `detour: 兜底`。
* `dns-fakeip`：网段分配 `198.18.0.0/15` (IPv4) 与 `fc00::/18` (IPv6)，启用 `independent_cache: true`。
* 解析路由顺序：
  1. `outbound: any` -> `dns-direct`（内核自身出站连接直连解析，杜绝死锁）
  2. `rule_set: rule-direct` -> `dns-direct`（国内与直连域名获取真实 IP）
  3. `rule_set: [rule-apac, rule-ai, rule-lowrate]` -> `dns-fakeip`（代理域名秒级响应 Fake-IP）
  4. `query_type: [A, AAAA]` -> `dns-fakeip`（其余所有标准域名分配 Fake-IP，交由路由层逆向还原本体进行策略判定）

### 2.4 MTProto 避让与 QUIC 智能拦截

遵循项目核心避让规范：
1. 基础地址防探测：`169.254.169.254/32` 立即执行 `action: reject`。
2. NTP 时间防偏移：端口 123 与 NTP 域名强制走 `DIRECT`。
3. FCM 推送保护：端口 `5228-5230` 强制走 `DIRECT`，防止推送握手超时。
4. **规则顺序**：
   - 优先匹配 `rule-direct`（保证国内 Bilibili、字节跳动等应用的 HTTP/3 QUIC 畅通）；
   - 其次匹配 `rule-apac`（保证 Telegram MTProto UDP 443 语音与握手不被误伤）；
   - 然后拦截海外 QUIC：`action: reject, network: udp, port: 443`（阻止海外网页与 YouTube QUIC 假死卡顿）；
   - 最后分流 `rule-ai` 与 `rule-lowrate`，最终兜底至 `兜底`。

### 2.5 Web 控制面板 (Zashboard 集成)

* 配置 Clash API 扩展模块，监听地址 `0.0.0.0:9090`，支持局域网设备与本地浏览器访问。
* 静态资源路径指定为 `zashboard`，自动拉取上游发布包 `https://github.com/Zephyruso/zashboard/releases/latest/download/dist.zip`。
* 首次启动时内核自动解压至本地目录，访问 `http://<ip>:9090/ui` 即可体验与 Mihomo 一致的现代可视化控制面板。

---

## 3. 校验与测试结果

### 3.1 语法校验 (sing-box check)

执行 Sing-box 1.11.4 严格语法审查：
```bash
/Users/don/work/proxy/proxy-rulesets/bin/sing-box check -c /Users/don/work/proxy/proxy-configs/singbox/singbox_Region.json
```
结果：Return code 0，零 Warning，零 Error，完全符合 Sing-box 1.11+ Action 语法标准。

### 3.2 规则集命中逆校验 (sing-box rule-set match)

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

---

## 4. 基于官方仓库 docs/schema.json 的严格审查与版本演进报告

对照用户拉取的 Sing-box 官方仓库最新文档 (`/Users/don/work/proxy/sing-box/docs`) 及 `schema.json`，配置文件的版本语法差异如下：

### 4.1 核心语法版本分水岭

| 模块 / 字段 | Sing-box 1.11.x (当前生产内核) | Sing-box 1.14+ ~ 1.15+ (官方最新标准) | 规范变更与处理依据 |
|---|---|---|---|
| **DNS 服务器声明** | `address: "https://..."`, `address: "fakeip"` | `type: "https", server: "..."`, `type: "fakeip"` | 1.12.0 重构 DNS 服务器为类型化对象，1.14.0 正式移除旧版 `address` 字符串字段。 |
| **Fake-IP 作用域** | 独立于根级 `dns.fakeip: {...}` | 内置于 `dns.servers` 的 `type: "fakeip"` 中 | 1.14.0 彻底移除根级 `dns.fakeip` 字段。 |
| **出站 DNS 引导** | DNS 规则 `outbound: any -> dns-direct` | 路由层 `route.default_domain_resolver: "dns-direct"` | 1.12.0 废弃 `outbound` DNS 规则项，改为显式配置域解析器。 |
| **DNS 缓存隔离** | `independent_cache: true` | 已废弃并删除 | 官方文档指出启用后会降低性能，1.14.0 废弃，已从两套配置中移除。 |
| **拒绝响应缓存** | `store_rdrc: true` | 已废弃并删除 | 1.14.0 废弃（由 `store_dns` 取代），已从配置中移除。 |
| **TUN 协议栈** | `stack: "system"` | 字段废弃，移除以启用自研 stack | 1.15.0 废弃 `stack`，采用自研高性能 sing-tun TCP/IP 栈，移除即可自动启用。 |
| **TUN NAT** | `endpoint_independent_nat: true` | 已废弃并删除 | 官方文档明确注明自 1.11.0 起已不再生效，已从两套配置中彻底移除。 |
| **协议与域名嗅探** | 入站 `sniff: true` | 路由规则动作 `{"action": "sniff"}` | 1.14.0 废弃入站监听级嗅探字段，推荐作为路由首条规则执行。两套配置均已加入路由首行 `action: sniff`。 |

### 4.2 双轨落地成果

针对不同环境的客户端，工作区现同时提供两套经严格校验的实现方案：
1. [singbox_Region.json](file:///Users/don/work/proxy/proxy-configs/singbox/singbox_Region.json)：面向当前生产运行环境（Sing-box 1.11.x ~ 1.13.x），已清洗全部废弃及失效参数，`sing-box check` 校验通过，Exit code 0。
2. [singbox_Region_next.json](file:///Users/don/work/proxy/proxy-configs/singbox/singbox_Region_next.json)：面向 Sing-box 1.14+ 及 1.15+ 最新官方规范，严格对齐 `/Users/don/work/proxy/sing-box/docs/schema.json`，Python 强校验 0 错误。

