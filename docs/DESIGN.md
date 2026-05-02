# 方案设计文档

> 项目：Mihomo 多网段精细分流配置  
> 版本：v7
> 最后更新：2026-05-02

---

## 1. 整体架构

```
                        ┌──────────────────────────────┐
                        │        OpenWrt 路由器          │
                        │    Mihomo 内核 (Clash Meta)    │
                        └──────────┬───────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
    ┌─────────▼──────┐  ┌─────────▼──────┐  ┌─────────▼──────┐
    │  ap-direct     │  │  ap-rule       │  │  ap-Global     │
    │  192.168.31.x  │  │  192.168.32.x  │  │  192.168.33.x  │
    │  → DIRECT      │  │  → 规则分流    │  │  → 全局代理    │
    └────────────────┘  └───────┬────────┘  └────────────────┘
                                │
                    ┌───────────┼───────────┐
                    │           │           │
              ┌─────▼────┐ ┌───▼─────┐ ┌───▼──────┐
              │ AI 平台  │ │ 视频    │ │ 社交/    │  ...更多分类
              │ (★稳定)  │ │ (省流)  │ │ 开发等   │
              └─────┬────┘ └───┬─────┘ └───┬──────┘
                    │          │           │
              ┌─────▼──────────▼───────────▼──────┐
              │         地区选择层                  │
              │  🇭🇰香港  🇯🇵日本  🇸🇬新加坡  🇺🇸美国  │
              └─────────────┬──────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
        ┌─────▼─────┐ ┌────▼─────┐ ┌─────▼─────┐
        │ AirportA  │ │ AirportB │ │ AirportC  │
        │ (最好)    │ │ (次要)   │ │ (最便宜)  │
        └───────────┘ └──────────┘ └───────────┘
```

### 1.1 SOCKS5 入站架构（v7 新增）

```
                        ┌──────────────────────────────┐
                        │        OpenWrt 路由器          │
                        │    Mihomo 内核 (Clash Meta)    │
                        └──────┬──────────┬─────────────┘
                               │          │
                 ┌─── TUN 透明代理 ──┐  ┌── SOCKS5 入站 (port 7891) ──┐
                 │                   │  │                              │
        ┌────────▼──────────────────▼┐ │  listeners:                  │
        │  ap-direct / ap-rule /     │ │    - socks5://:7891           │
        │  ap-Global / residential   │ │      rule: socks5-rules      │
        │  (按源网段分流)             │ │                              │
        └────────────────────────────┘ └──────────┬───────────────────┘
                                                 │
                                      ┌──────────▼───────────┐
                                      │  sub-rules:socks5    │
                                      │  socks5-rules 子链   │
                                      └──────────┬───────────┘
                                                 │
                          ┌──────────────────────┼──────────────────────┐
                          │                      │                      │
                    ┌─────▼──────┐         ┌─────▼──────┐        ┌─────▼──────┐
                    │  高级层    │         │  中级层    │        │  基础层    │
                    │  AI 类    │         │  搜索/电商 │        │  社交/视频 │
                    │  ★稳定版  │         │  ★稳定版   │        │  省流版    │
                    └───────────┘         └────────────┘        └────────────┘
```

**与 TUN 透明代理的关系**：SOCKS5 入站与 TUN 透明代理完全隔离。TUN 按 `SRC-IP-CIDR` / `SUB-RULE` 拆分网段，SOCKS5 按 `listeners` + `rule` 独立绑定 `socks5-rules` 子链，两者互不干扰。

## 2. 网段路由设计

### 2.1 实现方式

使用 `SRC-IP-CIDR` / `SUB-RULE` 等规则，放在所有 RULE-SET 之前（最高优先级）。**v6 起 34.x 住宅网段**不再使用「单行 SRC 直达住宅」，改为 **SUB-RULE + 子链**：白名单直连，其余 `MATCH` 住宅出口（见下）。

```yaml
# 子规则（与 rules 同级，见 Mihomo sub-rule 文档）
sub-rules:
  residential34:
    - RULE-SET,direct_34_relays,DIRECT
    - RULE-SET,direct_34,DIRECT
    - MATCH,🏠 住宅IP 34.x

rules:
  - SRC-IP-CIDR,192.168.31.0/24,DIRECT,no-resolve
  - SRC-IP-CIDR,192.168.33.0/24,🌐 全局代理 33.x,no-resolve
  - SUB-RULE,(SRC-IP-CIDR,192.168.34.0/24),residential34
  # 192.168.32.x → 不拦截，继续向下走全部分流规则
```

### 2.2 设计决策

- 31.x 用 `DIRECT`，不走任何代理逻辑
- 32.x **不写任何 SRC-IP-CIDR 规则**，让流量自然落入后续的 RULE-SET 匹配
- 33.x 指向一个 `select` 策略组，用户可在面板切换全局出口地区
- **34.x（v6）**：`SUB-RULE,(SRC-IP-CIDR,192.168.34.0/24),residential34` 进入子链；**先** `RULE-SET,direct_34_relays`（仅 `DST-PORT`/`IP-CIDR`），**再** `RULE-SET,direct_34`（域名）。Tun+UDP 无 Sniff 时，端口/IP 与域名**分文件**可避免单 RULE-SET 不命中；未命中则 **`MATCH` → 住宅**
- `DIRECT-34-relays.yaml` / `DIRECT-34.yaml` 均**不写源 IP**；源网段由 `SUB-RULE` 限定

### 2.3 扩展方式

**普通网段**（无住宅白名单）：在 `rules` 顶部增加 `SRC-IP-CIDR` 即可；`proxies` / `proxy-groups` 按需补充。

**住宅类网段且需白名单（推荐与 v6 一致）**：

1. 在 `proxies` / `proxy-groups` 中增加住宅节点与出口组（同前）
2. 若需 Tun/UDP 下稳定命中：新增 **`DIRECT-{网段}-relays.yaml`**（仅端口/IP）与 **`DIRECT-{网段}.yaml`**（域名），均为 classical `payload`
3. 增加 `rule-providers.direct_{网段}_relays` 与 `direct_{网段}` 指向对应 raw URL
4. 增加 `sub-rules.residential{网段}`：`RULE-SET,direct_{网段}_relays,DIRECT` → `RULE-SET,direct_{网段},DIRECT` → `MATCH,🏠 住宅IP …`
5. 在 `rules` 顶部用 `SUB-RULE,(SRC-IP-CIDR,…),residential{网段}` 作为该网段入口

### 2.4 本仓 `configs/rulesets` 命名约定（v6）

| 文件名模式 | 含义 | 主配置中的典型用法 |
|-----------|------|-------------------|
| `DIRECT-32.yaml` | 与 **32.x 分流场景**相关的个人直连目的列表 | `RULE-SET,direct_32,DIRECT`（与 v5 前 `prdiy` 等价，全局命中即直连，语义上服务「规则分流网段」维护） |
| `DIRECT-34-relays.yaml` | 34.x 白名单：**端口与落地 IP**（先于域名 RULE-SET） | `RULE-SET,direct_34_relays,DIRECT` |
| `DIRECT-34.yaml` | 34.x 白名单：**域名** | `RULE-SET,direct_34,DIRECT`；与上一行同处 `sub-rules.residential34` |

**rule-provider 键名**：小写 + 下划线，与文件名对应，如 `direct_32`、`direct_34_relays`、`direct_34`。同类住宅网段可增加 `DIRECT-35-relays.yaml` 等。

### 2.5 SOCKS5 入站路由设计（v7 新增）

SOCKS5 入站不经过 TUN 透明代理的网段路由链，而是通过 `listeners` 独立配置，绑定专属 `socks5-rules` 子链实现隔离分流。

```yaml
# listeners 配置（与 dns 同级）
listeners:
  - name: socks5-in
    type: socks
    port: 7891
    rule: socks5-rules      # 绑定专用 sub-rules

# sub-rules 定义
sub-rules:
  socks5-rules:
    - RULE-SET,private_domain,DIRECT
    - RULE-SET,ai_domain,🤖 SOCKS5-AI
    - RULE-SET,search_engine_domain,🔍 SOCKS5-搜索
    - RULE-SET,ecommerce_domain,🛒 SOCKS5-电商
    - RULE-SET,social_domain,💬 SOCKS5-社交
    - RULE-SET,video_domain,📹 SOCKS5-视频
    - RULE-SET,cn_domain,DIRECT
    - RULE-SET,geolocation-!cn,🌍 SOCKS5-国外
    - MATCH,🐟 SOCKS5-漏网之鱼
```

**设计要点**：

- **完全隔离**：SOCKS5 入站流量不经过 `rules` 主链，不匹配任何 `SRC-IP-CIDR` 网段规则，与 TUN 透明代理零耦合
- **独立子链**：`socks5-rules` 是完整的规则链，从 private 兜底到 MATCH，无需回退到主 rules
- **端口 7891**：默认 SOCKS5 监听端口，可按需修改；仅接受 SOCKS5 协议连接
- **优先级与主链一致**：private → 分类应用 → 国内直连 → 国外代理 → MATCH，保证逻辑完整性

## 3. 机场阶梯式分层体系设计

### 3.1 三梯队多机场架构

每个梯队包含 1~2 个机场，梯队内 url-test 跨机场选最快节点，梯队间 fallback 故障转移。

```
┌─────────────────────────────────────────────────────────┐
│ 第1层：梯队子组（url-test，跨该梯队所有机场选最快）      │
│                                                          │
│  [C] 香港 = CrossWall+DuoBaoYiYuan 的香港节点选最快      │
│  [B] 香港 = YiYuan+Kitty 的香港节点选最快                │
│  [A] 香港 = YunTu 的香港节点选最快                       │
├─────────────────────────────────────────────────────────┤
│ 第2层：地区组（fallback，按梯队优先级故障转移）          │
│                                                          │
│  🇭🇰 香港   = fallback [C]→[B]→[A]（省流版）            │
│  🇭🇰 香港★  = fallback [A]→[C]→[B]（稳定版）           │
├─────────────────────────────────────────────────────────┤
│ 第3层：应用策略组（select，用户手动选地区）              │
│                                                          │
│  📹 YouTube 32.x = select [美国, 香港, 日本...]          │
│  🤖 Claude 32.x  = select [日本★, 新加坡★...]           │
└─────────────────────────────────────────────────────────┘
```

### 3.2 梯队定义

| 梯队 | 机场 | 定位 | 适用场景 |
|------|------|------|---------|
| 主力 C | CrossWall + DuoBaoYiYuan | 速度适中价格合理 | 视频/社交/日常大流量 |
| 保底 B | YiYuan + Kitty | 最便宜，兜底用 | 主力挂了时备用 |
| 优质 A | YunTu | 最贵最稳 | AI/默认出口/全局代理 |

### 3.3 省流版 vs 稳定版

| 属性 | 省流版（无★） | 稳定版（带★） |
|------|-------------|-------------|
| 故障转移顺序 | C→B→A | A→C→B |
| 设计意图 | 主力先上，保底兜底 | 优质先上，主力次之 |
| 适用场景 | 视频/社交/日常 | AI/默认出口/全局代理 |
| 为什么这样设计 | 看视频流量大，优先省钱 | AI 长对话不能断，优先稳定 |

### 3.4 调整梯队归属

机场的梯队归属只由锚点的 `use:` 列表控制。调整时只需移动机场名：

```yaml
# 例：把 Kitty 从保底移到主力
sub_ut_c: use: [CrossWall, DuoBaoYiYuan, Kitty]  # 加入主力
sub_ut_b: use: [YiYuan]                           # 保底只剩一元
```

无需改动策略组、规则、规则集等任何其他部分。

### 3.5 次要地区处理

英国/德国/法国/韩国节点数量少，不值得拆成 3 个梯队子组。直接用 `url-test + include-all-providers + filter` 从全部 5 个机场选最快节点。

## 4. 策略组分区设计

### 4.1 面板布局（从上到下）

```
┌─ 第1区：出口组（4个）──────────────────────────┐
│  🚀 默认出口 32.x    → 未命中规则的海外流量（可选 ★稳定 / 无★省流）│
│  🌐 全局代理 33.x    → 33.x 网段全部流量        │
│  🏠 住宅IP 34.x      → 34.x 网段全部流量        │
│  🐟 漏网之鱼         → MATCH 兜底               │
├─ 第2区：应用策略组（24个，多数含 ★稳定+省流双轨）───────┤
│  AI: Claude, ChatGPT, Gemini, DeepSeek          │
│  视频: YouTube, Netflix, TikTok, Spotify, Disney│
│  社交: Telegram, X, Instagram, Facebook, Discord│
│  开发: Google, GitHub, Cloudflare, Figma, Notion│
│  系统: Microsoft, Apple                          │
│  金融: 加密货币, PayPal                          │
│  游戏: Steam                                     │
├─ 第3区：地区组（18个）─────────────────────────┤
│  5主要地区 × 2版本 + 4次要地区 + 全部 + 故转    │
├─ 第4区：机场子组（15个）───────────────────────┤
│  5主要地区 × 3机场 = 15个 url-test 子组          │
└────────────────────────────────────────────────┘
```

### 4.2 设计原则

- **日常操作区在上**：用户最常切换的应用组排最前
- **基础设施沉底**：机场子组几乎不需要手动操作
- **名称即语义**：看到 `Claude 32.x` 就知道控制哪个网段的什么服务

## 5. 规则匹配链设计

```
请求进入
  │
  ├─ SRC-IP-CIDR 31.x → DIRECT（直链网段，立即返回）
  ├─ SRC-IP-CIDR 33.x → 全局代理（全局网段，立即返回）
  ├─ SUB-RULE 源为 34.x → 子链：direct_34_relays → direct_34 → DIRECT；否则 MATCH → 🏠 住宅IP 34.x
  │
  │  ↓ 32.x 网段继续向下
  │
  ├─ private_ip / private_domain → DIRECT
  │
  ├─ AI: claude → claude_domain → ChatGPT → gemini → deepseek → ai_catchall
  ├─ 视频: youtube → netflix(domain+ip) → tiktok → spotify → disney
  ├─ 社交: telegram(domain+ip) → twitter(domain+ip) → instagram → facebook(domain+ip) → discord
  ├─ 开发: google(domain+ip) → github → cloudflare(domain+ip) → figma → notion
  ├─ 系统: onedrive → microsoft → apple(domain+ip)
  ├─ 金融: crypto → paypal → steam
  │
  ├─ geolocation-!cn → 默认出口（非中国域名兜底代理）
  ├─ cn_domain → DIRECT
  ├─ cn_ip → DIRECT
  │
  └─ MATCH → 漏网之鱼
```

### 5.1 规则排列原则

1. **网段路由最先**：31.x、33.x 单行定向；**34.x 为 SUB-RULE 入口**，子链内再区分直连白名单与住宅默认
2. **具体应用在前**：YouTube 规则在 Google 之前（YouTube 是 Google 子集）
3. **domain + ip 配对**：域名规则匹配域名，IP 规则补漏 CDN/直连 IP
4. **兜底在后**：`geolocation-!cn` → `cn` → `MATCH`

## 6. DNS 防泄漏设计

```
                    DNS 查询
                       │
                       ▼
              ┌────────────────┐
              │  fake-ip 模式  │ ← 为域名分配假 IP，不暴露真实 DNS 查询
              └───────┬────────┘
                      │
          ┌───────────┼───────────┐
          │           │           │
    ┌─────▼─────┐ ┌───▼───┐ ┌────▼─────┐
    │ default   │ │ proxy │ │ direct   │
    │ nameserver│ │ server│ │ nameserver│
    │ 223.5.5.5 │ │ Ali+  │ │ Ali+Pub  │
    │ (解析DNS  │ │ Pub   │ │ (直连域名│
    │  服务器)  │ │ DOH   │ │  解析)   │
    └───────────┘ └───────┘ └──────────┘
```

- `default-nameserver`：纯 IP，用于解析 DOH 服务器本身的域名
- `proxy-server-nameserver`：解析代理节点域名（防鸡蛋问题）
- `direct-nameserver`：直连流量的域名解析
- `nameserver`：海外域名解析，走 8.8.8.8 DOH 并附加 ECS 优化
- `respect-rules: true`：DNS 查询也遵守路由规则

## 7. 扩展指南

### 7.1 新增平台

```yaml
# 1. 在 proxy-groups 的第2区添加策略组
- name: "🎯 NewApp 32.x"
  type: select
  proxies: ["🇺🇸 美国", "🇯🇵 日本", ...]

# 2. 在 rules 对应分类处添加规则
- RULE-SET,newapp_domain,🎯 NewApp 32.x

# 3. 在 rule-providers 添加规则集
newapp_domain:
  <<: *domain
  url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/newapp.mrs"
```

### 7.2 新增网段

```yaml
# 1. 在 proxies 添加住宅 IP 节点
- name: "🏠 英国住宅-35"
  type: socks5
  server: "xx.xx.xx.xx"
  port: 8001
  username: "xxx"
  password: "xxx"

# 2. 在 proxy-groups 添加策略组
- name: "🏠 住宅IP 35.x"
  type: select
  proxies: ["🏠 英国住宅-35"]

# 3a. 若 35.x 为「全屋住宅、无白名单」：一行 SRC 即可
- SRC-IP-CIDR,192.168.35.0/24,🏠 住宅IP 35.x,no-resolve

# 3b. 若与 34.x 相同需求（白名单直连 + 默认住宅）：复制 v6 的
#     sub-rules + DIRECT-35.yaml + direct_35 + SUB-RULE 模式（见 §2.3）
```

### 7.3 新增机场（加入已有梯队）

```yaml
# 1. 在 proxy-providers 添加新机场
NewAirport:
  type: http
  url: "新订阅链接"
  ...

# 2. 在对应梯队的锚点 use: 列表中加入机场名
sub_ut_c: &sub_ut_c
  use: [CrossWall, DuoBaoYiYuan, NewAirport]  # 加入主力梯队
```

无需修改策略组、规则或规则集。

### 7.4 调整机场梯队归属

```yaml
# 把 Kitty 从保底梯队移到主力梯队
sub_ut_c: use: [CrossWall, DuoBaoYiYuan, Kitty]
sub_ut_b: use: [YiYuan]
```

### 7.5 住宅网段白名单与文档索引（v6）

- **需求与验收**：`REQUIREMENTS.md`（NET-04、RES-*、NFR-05）
- **落地配置**：`configs/v6.yaml`、`configs/rulesets/DIRECT-34-relays.yaml`、`configs/rulesets/DIRECT-34.yaml`
- **内核说明**：[Route Rules](https://wiki.metacubex.one/en/config/rules/)（`SUB-RULE`）、[sub-rule](https://wiki.metacubex.one/en/config/sub-rule/)

### 7.6 SOCKS5 扩展指南（v7 新增）

**新增 SOCKS5 应用分类**：

```yaml
# 1. 在 proxy-groups 中添加 SOCKS5 专用策略组
- name: "🧠 SOCKS5-新应用"
  type: select
  proxies: ["🇯🇵 日本★", "🇸🇬 新加坡★", "🇭🇰 香港★"]

# 2. 在 sub-rules.socks5-rules 中对应位置插入规则
sub-rules:
  socks5-rules:
    - RULE-SET,private_domain,DIRECT
    - RULE-SET,ai_domain,🤖 SOCKS5-AI
    - RULE-SET,newapp_domain,🧠 SOCKS5-新应用    # ← 插入此处
    - RULE-SET,search_engine_domain,🔍 SOCKS5-搜索
    # ... 其余规则不变
```

**新增 SOCKS5 监听端口**：

```yaml
# 如需第二个 SOCKS5 入站（例如绑定不同规则链）
listeners:
  - name: socks5-in
    type: socks
    port: 7891
    rule: socks5-rules
  - name: socks5-in-alt
    type: socks
    port: 7892
    rule: socks5-alt-rules     # 绑定另一条子链
```

**调整分层归属**：将某个应用从基础层提升到高级层，只需在 `socks5-rules` 中调整规则顺序，使其指向高级层策略组即可。无需修改代理节点或机场配置。

## 8. SOCKS5 分层代理池设计（v7 新增）

### 8.1 三层架构

SOCKS5 入站将流量按业务敏感度分为三层，每层对应不同的代理质量和成本策略：

```
┌─────────────────────────────────────────────────────────┐
│ 高级层（★稳定版）                                        │
│  🤖 SOCKS5-AI    → Claude / ChatGPT / Gemini / DeepSeek │
│  策略：优质机场优先，故障转移 A→C→B                      │
│  原因：AI 长对话不能断，优先稳定                         │
├─────────────────────────────────────────────────────────┤
│ 中级层（★稳定版）                                        │
│  🔍 SOCKS5-搜索   → Google / Bing / DuckDuckGo          │
│  🛒 SOCKS5-电商   → Amazon / eBay / Shopify             │
│  策略：优质机场优先，故障转移 A→C→B                      │
│  原因：搜索结果与电商价格敏感于 IP 归属地，需稳定出口     │
├─────────────────────────────────────────────────────────┤
│ 基础层（省流版）                                         │
│  💬 SOCKS5-社交   → Telegram / X / Instagram / Discord  │
│  📹 SOCKS5-视频   → YouTube / Netflix / TikTok          │
│  策略：主力机场优先，故障转移 C→B→A                      │
│  原因：社交/视频流量大，优先省钱；断连重连即可           │
└─────────────────────────────────────────────────────────┘
```

### 8.2 负载均衡策略

🌍 SOCKS5-国外 使用 `load-balance` + `consistent-hashing`，确保同一目标域名始终路由到同一出口节点，避免频繁切换 IP 触发风控。

```yaml
proxy-groups:
  - name: "🌍 SOCKS5-国外"
    type: load-balance
    strategy: consistent-hashing    # 一致性哈希，同域名同节点
    proxies: &socks5-foreign
      - "🇭🇰 香港★"
      - "🇯🇵 日本★"
      - "🇸🇬 新加坡★"
      - "🇺🇸 美国★"
    url: "http://www.gstatic.com/generate_204"
    interval: 300

  # 对比：TUN 透明代理的默认出口使用 fallback
  # SOCKS5 场景下 consistent-hashing 更合适：
  # - 避免同一会话 IP 频繁变动触发风控
  # - 减少需要重新认证的次数
  # - 保持长连接稳定性
```

**consistent-hashing vs sticky-sessions**：

| 特性 | consistent-hashing | sticky-sessions |
|------|-------------------|-----------------|
| 哈希依据 | 目标地址（域名/IP） | 源+目标地址组合 |
| 节点变动影响 | 仅受影响域名的映射变化 | 仅受影响会话的映射变化 |
| 适合场景 | 按目标站点固定出口 | 按客户端会话固定出口 |
| SOCKS5 推荐 | **推荐** — 同一网站始终同出口 | 适用于多用户 SOCKS5 共享场景 |

### 8.3 精细化分流规则链（socks5-rules）

v7 实际部署的 `socks5-rules` 包含 26 条规则，按分层优先级排列：

```yaml
sub-rules:
  socks5-rules:
    # ── 第1层：私有地址直连 ──
    - RULE-SET,private_domain,DIRECT
    - RULE-SET,private_ip,DIRECT,no-resolve

    # ── 第2层：高级层 — AI 类（稳定优先，★ 版地区）──
    - RULE-SET,claude_domain,🤖 SOCKS5-AI
    - RULE-SET,openai_domain,🤖 SOCKS5-AI
    - RULE-SET,gemini_domain,🤖 SOCKS5-AI
    - RULE-SET,deepseek_domain,🤖 SOCKS5-AI
    - RULE-SET,ai_catchall,🤖 SOCKS5-AI

    # ── 第3层：中级层 — 搜索（域名+IP 配对）──
    - RULE-SET,google_domain,🔍 SOCKS5-搜索
    - RULE-SET,google_ip,🔍 SOCKS5-搜索,no-resolve
    - RULE-SET,bing_domain,🔍 SOCKS5-搜索

    # ── 第3层：中级层 — 电商 ──
    - RULE-SET,amazon_domain,🛒 SOCKS5-电商
    - RULE-SET,ebay_domain,🛒 SOCKS5-电商
    - RULE-SET,shopify_domain,🛒 SOCKS5-电商

    # ── 第4层：基础层 — 社交（域名+IP 配对）──
    - RULE-SET,twitter_domain,📱 SOCKS5-社交
    - RULE-SET,twitter_ip,📱 SOCKS5-社交,no-resolve
    - RULE-SET,instagram_domain,📱 SOCKS5-社交
    - RULE-SET,facebook_domain,📱 SOCKS5-社交
    - RULE-SET,facebook_ip,📱 SOCKS5-社交,no-resolve
    - RULE-SET,telegram_domain,📱 SOCKS5-社交
    - RULE-SET,telegram_ip,📱 SOCKS5-社交,no-resolve

    # ── 第4层：基础层 — 视频 ──
    - RULE-SET,youtube_domain,📱 SOCKS5-社交
    - RULE-SET,tiktok_domain,📱 SOCKS5-社交

    # ── 第5层：国内代理（非 DIRECT，走 🌏 SOCKS5-国内）──
    - RULE-SET,cn_domain,🌏 SOCKS5-国内
    - RULE-SET,cn_ip,🌏 SOCKS5-国内,no-resolve

    # ── 第6层：国外兜底（consistent-hashing 负载均衡）──
    - RULE-SET,geolocation_not_cn,🌍 SOCKS5-国外

    # ── 最终兜底 ──
    - MATCH,🌍 SOCKS5-国外
```

**与 32.x 主路由规则的关键差异**：

| 差异点 | 32.x 主路由 `rules` | SOCKS5 `socks5-rules` |
|--------|---------------------|----------------------|
| 国内流量 | `cn_domain/cn_ip → DIRECT` | `cn_domain/cn_ip → 🌏 SOCKS5-国内`（走代理，可自选节点） |
| 国外兜底 | `geolocation_not_cn → 🚀 默认出口 32.x`（fallback） | `geolocation_not_cn → 🌍 SOCKS5-国外`（load-balance + consistent-hashing） |
| 最终兜底 | `MATCH → 🐟 漏网之鱼` | `MATCH → 🌍 SOCKS5-国外`（同样走负载均衡） |
| 新增分类 | 无 | 搜索（Google/Bing）、电商（Amazon/eBay/Shopify）独立分层 |

**规则链设计原则**：

1. **private 优先**：内网地址直连，避免代理回环
2. **AI 最先匹配**：AI 域名规则在搜索引擎之前（AI 平台可能是搜索引擎超集）
3. **搜索先于电商**：Google Shopping 等搜索+电商混合场景，先按搜索匹配
4. **domain + IP 配对**：与 32.x 一致，域名规则匹配域名，IP 规则补漏 CDN/直连 IP
5. **国内走代理**：爬虫场景下国内网站也需经代理出口，避免暴露真实 IP
6. **国外兜底**：非中国流量走 consistent-hashing 负载均衡
7. **MATCH 走负载均衡**：未命中流量同样走 `🌍 SOCKS5-国外`，确保无直连泄漏

### 8.4 负载均衡决策：为什么 32.x 不加 load-balance

**结论**：`load-balance` 仅用于 SOCKS5 爬虫场景，32.x 日常使用不加。

| 场景 | 策略类型 | 理由 |
|------|---------|------|
| 32.x 日常使用 | `select` + `fallback` | 同一平台需保持同一 IP，频繁切换触发风控（Google 验证码、登录态丢失）；已有 fallback 故障转移足够 |
| SOCKS5 爬虫 | `load-balance` + `consistent-hashing` | 爬虫防封控才需要分散 IP；日常浏览不需要 |

**32.x 已有的故障转移机制**：

```
应用策略组 (select) → 地区组 (fallback) → 机场子组 (url-test)
  例: Claude 32.x      → 🇯🇵 日本★         → [A] 日本 / [C] 日本 / [B] 日本
                         fallback A→C→B       url-test 跨机场选最快
```

- **select**：用户手动选地区，同一时刻只走一个地区
- **fallback**：地区组内梯队故障转移，节点挂了自动切下一个梯队
- **url-test**：同梯队跨机场选最快节点

这套机制覆盖了"节点挂了自动切"的需求，不需要 load-balance。

**如果给 32.x 加 load-balance 的副作用**：
- 视频流媒体轮询不同地区节点 → CDN 调度混乱、缓冲频繁
- AI 平台换 IP → 丢会话、需重新认证
- 社交平台换 IP → 登录态丢失、触发安全验证

### 8.5 v6→v7 兼容性证明

v7 对 v6 的所有配置均为**纯新增、零修改**，不影响现有功能：

| 对比项 | v6 | v7 | 变化 |
|--------|----|----|------|
| 策略组 | 59 个 | 65 个 | +6 SOCKS5 组，原有 0 修改 |
| 路由规则 | 43 条 | 43 条 | 完全一致 |
| 子规则 | residential34 | +socks5-rules | residential34 未动 |
| 规则集 | 42 个 | 46 个 | +4 电商/搜索，原有 0 修改 |
| DNS | — | — | 完全一致 |
| 机场订阅 | 5 个 | 5 个 | 完全一致 |
| 静态节点 | 1 个 | 1 个 | 完全一致 |

**隔离机制**：SOCKS5 端口 7891 通过 `listeners.rule: socks5-rules` 绑定到独立子规则链，流量不经过主路由 `rules`，与 31.x/32.x/33.x/34.x 四个网段完全隔离。SOCKS5 策略组只引用已有的地区组（如 `🇯🇵 日本★`），不会反向影响地区组的节点选择。

### 8.6 防封控最佳实践

**IP 稳定性**：

- 使用 `consistent-hashing` 而非 `round-robin`，同一目标站点始终走同一出口 IP
- AI 策略组使用 `select` 手动控制或 `fallback` 优质优先，避免自动切换导致 IP 变动
- 长对话场景（Claude/ChatGPT）建议锁定到单一地区节点

**连接管理**：

```yaml
# SOCKS5 入站连接超时设置
listeners:
  - name: socks5-in
    type: socks
    port: 7891
    rule: socks5-rules

# 全局连接设置（与 SOCKS5 相关）
unified-delay: true          # 统一延迟计算，避免节点切换时的延迟抖动
keep-alive-interval: 30      # 保持长连接活跃，防止被中间设备断开
```

**流量特征防护**：

- SOCKS5 入站流量与 TUN 透明代理走不同出口组，IP 指纹互不关联
- 电商/搜索策略组使用与 AI 不同的地区出口，降低跨平台 IP 关联风险
- 避免在 SOCKS5 规则链中混入 TUN 网段的策略组，保持隔离性

**故障恢复**：

- 高级层 `fallback`：A→C→B，优质节点恢复后自动切回
- 基础层 `fallback`：C→B→A，主力恢复后自动切回
- `consistent-hashing` 在节点故障时自动 rehash 到存活节点，无需人工干预

## 9. 迭代历史

| 版本 | 日期 | 核心变更 |
|------|------|---------|
| v5 | 2026-03-xx | 5 机场阶梯式分层架构，双轨(★/省流)策略组 |
| v6 | 2026-04-12 | 住宅网段白名单（SUB-RULE + DIRECT-34 relays/domain 分文件），v6 住宅 IP 出口 |
| v7 | 2026-05-02 | SOCKS5 入站（listeners + rule 绑定），分层代理池（高级/中级/基础三层），consistent-hashing 负载均衡，socks5-rules 精细化分流规则链 |
