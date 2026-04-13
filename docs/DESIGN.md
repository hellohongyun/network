# 方案设计文档

> 项目：Mihomo 多网段精细分流配置  
> 版本：v6  
> 最后更新：2026-04-12

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
