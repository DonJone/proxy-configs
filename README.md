# Proxy Configurations

本项目提供针对 Mihomo (Clash Meta)、Loon 及 Shadowrocket 优化的配置文件，旨在构建一个人工干预优先级高且具备自动冗余能力的代理环境。

以下是各平台客户端配置文件的 Raw 及 CDN 链接，可直接复制到客户端中作为订阅地址使用：


#### 🔹 Mihomo (Desktop & Mobile)
* **GitHub Raw:** 
```
https://raw.githubusercontent.com/DonJone/proxy-configs/main/mihomo/desktop%26mobile/mihomodeskmob.yaml
```
* **jsDelivr CDN:** 
```
https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@main/mihomo/desktop%26mobile/mihomodeskmob.yaml
```

#### 🔹 Loon
* **GitHub Raw:** 
```copy
https://raw.githubusercontent.com/DonJone/proxy-configs/main/loon/configs/loon.lcf
```
* **jsDelivr CDN:** 
```copy
https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@main/loon/configs/loon.lcf
```


#### 🔹 Mihomo (OpenClash)
* **GitHub Raw:** 
```
https://raw.githubusercontent.com/DonJone/proxy-configs/main/mihomo/OpenClash/openclash.yaml
```
* **jsDelivr CDN:** 
```
https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@main/mihomo/OpenClash/openclash.yaml
```

#### 🔹 Shadowrocket
* **GitHub Raw:** 
```
https://raw.githubusercontent.com/DonJone/proxy-configs/main/Shadowrocket/configs/Shadowrocket.conf
```
* **jsDelivr CDN:** 
```
https://cdn.jsdelivr.net/gh/DonJone/proxy-configs@main/Shadowrocket/configs/Shadowrocket.conf
```


##  核心架构：三大手选组与 Fallback 机制

本配置方案的核心逻辑为“手动指定出口 + 自动故障备份”。系统通过 手选节点 A、手选节点 B、手选节点 C 三个策略组实现精准控制，并将其嵌套在 Fallback 策略中。

1. 优先级定义：用户在“手选节点”组中主动锁定一个特定节点作为首选出口。
2. 故障监控：对应的 Fallback 组（如 线路 A）持续监测该手选节点的连通性。
3. 自动补位：一旦手选节点失效，流量将自动瞬移至隐藏的“全局自动”组（url-test 模式），确保网络连接不中断。
4. 自动回归：当手动选定的节点恢复正常后，Fallback 机制会自动将流量切回，无需人工二次干预。

## 相比地区自动分流（URL-Test）的优点

与传统的“香港自动”、“美国自动”等完全基于延迟频繁切换 IP 的模式相比，本方案具有显著优势：

1. IP 粘滞性与业务稳定性
有效避免因微小延迟波动导致出口 IP 频繁变动。对于 ChatGPT、Google、金融交易平台等对 IP 稳定性极其敏感的服务，可以极大降低因 IP 漂移触发的安全风控或登录失效风险。

2. 极高的操作可控性
用户可以明确感知并锁定当前的流量出口。在进行需要特定落地 IP 的业务操作时，手动选择具有绝对优先级，不会被自动算法意外覆盖或漂移到其他地区。

3. 灾备无感的弹性方案
在享受手动选择带来的 IP 稳定性时，Fallback 机制提供了兜底保障。即便手动指定的节点意外宕机，流量也会在毫秒级转移到备份节点池，实现了手动控制与高可用性的平衡。


