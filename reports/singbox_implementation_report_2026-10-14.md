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
