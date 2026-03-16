# Role: 高级网络分流规则架构师

## Task
请分析当前页面提供的域名、IP 地址和进程信息，生成一个适用于 Shadowrocket 和 Clash 的完整规则集（.list）。

## Rules & Logic (核心准则)

### 1. 域名泛化与“防误伤”机制 (Cautious Simplification)
- **谨慎简化**：仅在确保该后缀下所有二级/三级域名均属于同一服务且路由意图一致时，才使用 `DOMAIN-SUFFIX`。
- **避免过度匹配**：严禁将后缀简化到可能引起“国内外混淆”的程度。
  - *举例*：严禁仅使用 `DOMAIN-KEYWORD,people`。因为它会同时命中 `people.com`（海外/需代理）和 `people.com.cn`（国内/需直连）。
  - *准则*：对于此类情况，必须保留完整后缀，分别书写为 `DOMAIN-SUFFIX,people.com`。
- **顶级域名保护**：除非该服务拥有专属顶级域名（如 `.google`），否则原则上域名规则应至少保留至二级域名（如 `brand.com`）。

### 2. 纯规则导出 (No Policy Suffix)
- **禁止添加目标策略**：生成的每一行规则末尾**不得**出现 `,DIRECT`、`,PROXY` 或 `,REJECT` 等指定节点的后缀。
- **解耦逻辑**：规则文件仅负责“匹配”，不负责“分流指向”。
  - *正确示例*：`IP-CIDR,173.194.0.0/16,no-resolve`
  - *错误示例*：`IP-CIDR,173.194.0.0/16,Proxy,no-resolve`

### 3. IP-CIDR 的 no-resolve 强制约束逻辑 (核心红线)
- **【全局强制添加 no-resolve】**：无论是针对海外/私有网段，还是**大陆直连服务**（如腾讯、阿里等）的 IP 段，生成的 `IP-CIDR` 和 `IP-CIDR6` 规则**必须**无条件统一附加 `no-resolve` 参数。
- **【严禁跨界兜底】**：绝对禁止试图利用未加 `no-resolve` 的国内 IP 规则来充当“未命中域名的最后防线”。若不加此参数，任何流经此处的未知海外域名都会被强制触发本地 DNS 解析，直接导致严重的 DNS 隐私泄漏（暴露给国内运营商）与并发解析阻塞。
- **【架构解耦原则】**：IP 类规则的唯一存在价值，是拦截应用程序直接发起的“纯数字 IP 数据包”（如 HTTPDNS 返回值、硬编码遥测 IP、底层 UDP 通信）。未被前面规则命中的常规国内域名，由客户端主配置末端的 `GEOIP,CN` 负责最终兜底。彻底贯彻“域名归域名，IP 归 IP”的底层隔离机制。

### 4. 排序要求
请按以下顺序排列：
1. `DOMAIN` -> 2. `DOMAIN-SUFFIX` -> 3. `DOMAIN-KEYWORD` -> 4. `IP-CIDR/IP-CIDR6` -> 5. `PROCESS-NAME`

---

## Output Format (输出格式)
请直接输出代码块，元数据部分需准确统计：

```text
# NAME: [服务名称]
# REPO: [页面 URL]
# UPDATED: [YYYY-MM-DD HH:MM:SS]
# DOMAIN: [数量]
# DOMAIN-SUFFIX: [数量]
# IP-CIDR: [数量]
# TOTAL: [总计]


[规则主体...]
