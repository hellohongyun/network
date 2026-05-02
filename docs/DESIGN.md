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

爬虫/批量抓取走独立 SOCKS5 端口（7891），每个平台有独立策略组，第一选项引用对应 32.x 组，实现 IP 信誉共享：

```
                  ┌──────────────────────────────────────────┐
                  │          OpenWrt 路由器 Mihomo 内核         │
                  └───────────────┬──────────────────────────┘
                                  │
          ┌───────────────────────┼──────────────────────────┐
          │ TUN 透明代理           │  SOCKS5 入站 :7891         │
          │ (32.x → 应用策略组)   │  users: crawler1/2/3     │
          └───────────┬───────────┴─────────────┬────────────┘
                      │                         │
              主路由 rules 链           socks5-rules 子链
                      │                         │
          ┌───────────▼───────────┐   ┌─────────▼────────────────────┐
          │  🤖 Claude 32.x        │◄──│  🤖 !Claude SOCKS5            │
          │  （当前选: 日本★）     │   │  第1选项 = Claude 32.x        │
          │  出口 IP: 1.2.3.4     │   │  → 跟随节点，共享出口 IP 信誉  │
          └───────────────────────┘   │  第2+选项 = ★稳定/省流（手选）│
                                      └──────────────────────────────┘
```

**IP 信誉共享**：浏览器（32.x）访问并完成人机验证 → 出口 IP 被目标站点标记为可信 → 爬虫走相同 SOCKS5 节点（第一选项引用同一 32.x 组）→ 共享出口 IP → 继承信誉，大幅降低 403 概率。

**与 TUN 透明代理的关系**：`listeners.rule: socks5-rules` 绑定独立子链，SOCKS5 流量不经过主 `rules`，不匹配任何 `SRC-IP-CIDR` 网段规则，与 TUN 透明代理零耦合。

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
# listeners 配置
listeners:
  - name: socks5-in
    type: socks
    port: 7891
    rule: socks5-rules      # 绑定专用 sub-rules
    users:
      - username: crawler1
        password: ★填写密码★

# sub-rules：26 个平台策略组与 32.x 一一对应
sub-rules:
  socks5-rules:
    - RULE-SET,private_domain,DIRECT
    - RULE-SET,private_ip,DIRECT,no-resolve
    - RULE-SET,claude_domain,🤖 !Claude SOCKS5
    - RULE-SET,openai_domain,🤖 !ChatGPT SOCKS5
    # ...（每个平台独立规则，详见 §8.3）
    - RULE-SET,cn_domain,🌏 SOCKS5-国内
    - RULE-SET,geolocation_not_cn,🌍 SOCKS5-国外
    - MATCH,🌍 SOCKS5-国外
```

**设计要点**：

- **完全隔离**：SOCKS5 流量不经过主 `rules`，不匹配任何 `SRC-IP-CIDR` 网段规则
- **32.x 镜像**：26 个 SOCKS5 组与 32.x 平台组一一对应，每组第一选项引用对应 32.x 组
- **IP 信誉共享**：浏览器和爬虫走同一出口 IP，浏览器验证人机后 IP 获得信誉，爬虫继承
- **端口 7891**：仅接受 SOCKS5 协议连接，支持用户认证

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
├─ 第2-B区：SOCKS5 精细化策略组（26个，与 32.x 一一对应）──┤
│  24个平台组：首选项引用对应 32.x 组（共享 IP 信誉）     │
│  2个兜底组：SOCKS5-国内 / SOCKS5-国外                   │
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

**新增 SOCKS5 平台组**（与新增 32.x 平台组同步操作）：

```yaml
# 1. 在 proxy-groups 第2区添加 32.x 策略组
- name: "🎯 NewApp 32.x"
  type: select
  proxies: ["🇺🇸 美国★", "🇯🇵 日本★", ...]

# 2. 在 proxy-groups 第2-B区添加对应 SOCKS5 策略组
- name: "🎯 NewApp SOCKS5"
  type: select
  proxies:
    - "🎯 NewApp 32.x"   # 第1选项：跟随浏览器节点，共享 IP 信誉
    - "🇺🇸 美国★"
    - "🇯🇵 日本★"
    # ... 与其他 SOCKS5 组相同节点列表

# 3. 在 sub-rules.socks5-rules 对应分类位置插入规则
- RULE-SET,newapp_domain,🎯 NewApp SOCKS5
```

**新增 SOCKS5 监听端口**：

```yaml
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

## 8. SOCKS5 精细化代理池设计（v7 新增）

### 8.1 32.x 镜像架构

v7 最终设计：SOCKS5 不使用分类分层，而是与 32.x **逐平台一一对应**。26 个 SOCKS5 组（24 个应用组 + 2 个兜底组），每个组的第一选项引用对应 32.x 策略组，实现节点自动跟随和 IP 信誉共享。

```
┌────────────────────────────────────────────────────────────────────┐
│ 浏览器（32.x）                    爬虫（SOCKS5）                   │
│                                                                    │
│  🤖 Claude 32.x                   🤖 !Claude SOCKS5               │
│  当前节点: 日本★                  第1选项: Claude 32.x ← 引用     │
│  出口 IP: 1.2.3.4                 → 同样走日本★ → 出口 1.2.3.4    │
│                                                                    │
│  用户浏览器访问 claude.ai          爬虫 curl_cffi 访问 claude.ai   │
│  → 完成人机验证 → IP 获得信誉     → 共享 IP → 继承信誉 → 通过    │
└────────────────────────────────────────────────────────────────────┘
```

**为什么不使用分类分层**：
- 旧设计的 `SOCKS5-AI`（4 个 AI 平台合并）导致无法独立控制每个平台的节点
- 32.x 已经有 24 个独立平台组，SOCKS5 镜像后每个平台可以单独切换节点
- 爬虫被封时需要按平台粒度排查和切换节点，合并组无法做到

### 8.2 IP 信誉共享机制

这是 v7 SOCKS5 设计的核心价值：

```
┌─ 步骤 1：浏览器预热 IP 信誉 ──────────────────────────────────┐
│  设备(32.x) → Claude 32.x(日本★) → 出口 IP: 1.2.3.4          │
│  浏览器访问 claude.ai → 完成 Cloudflare 人机验证               │
│  → Cloudflare 标记 IP 1.2.3.4 为可信                           │
├─ 步骤 2：爬虫继承信誉 ───────────────────────────────────────┤
│  爬虫 → SOCKS5:7891 → !Claude SOCKS5(第1选项=Claude 32.x)     │
│  → 同样走日本★ → 出口 IP: 1.2.3.4（与浏览器相同）             │
│  → Cloudflare 识别为可信 IP → 请求通过                         │
└─ 注意：TLS 指纹仍需 curl_cffi 处理，仅共享 IP 不够（见 §8.5）┘
```

**cf_clearance cookie**：Cloudflare 验证人机后下发此 cookie，绑定到 IP + TLS 指纹。浏览器验证后，同 IP 的 curl_cffi 请求可复用该 cookie（但 TLS 指纹不同时仍可能失效）。实践中，共享 IP 的信誉预热效果是主要的，cookie 复用是附加的。

### 8.3 策略组清单（26 个）

| 分类 | SOCKS5 策略组 | 32.x 默认选项 | 说明 |
|------|-------------|-------------|------|
| AI | 🤖 !Claude SOCKS5 | 🤖 Claude 32.x | 严格 IP 检测平台 |
| AI | 🤖 !ChatGPT SOCKS5 | 🤖 ChatGPT 32.x | 严格 IP 检测平台 |
| AI | 🔮 Gemini SOCKS5 | 🔮 Gemini 32.x | |
| AI | 🐋 DeepSeek SOCKS5 | 🐋 DeepSeek 32.x | |
| 视频 | 📹 YouTube SOCKS5 | 📹 YouTube 32.x | |
| 视频 | 🎥 Netflix SOCKS5 | 🎥 Netflix 32.x | |
| 视频 | 🎵 TikTok SOCKS5 | 🎵 TikTok 32.x | |
| 视频 | 🎧 Spotify SOCKS5 | 🎧 Spotify 32.x | |
| 视频 | 🏰 Disney+ SOCKS5 | 🏰 Disney+ 32.x | |
| 社交 | 📲 Telegram SOCKS5 | 📲 Telegram 32.x | |
| 社交 | 🐦 !Twitter SOCKS5 | 🐦 X 32.x | 严格 IP 检测平台 |
| 社交 | 📸 !Instagram SOCKS5 | 📸 Instagram 32.x | 严格 IP 检测平台 |
| 社交 | 👤 !Facebook SOCKS5 | 👤 Facebook 32.x | 严格 IP 检测平台 |
| 社交 | 🎮 Discord SOCKS5 | 🎮 Discord 32.x | |
| 开发 | 🍀 Google SOCKS5 | 🍀 Google 32.x | |
| 开发 | 👨‍💻 GitHub SOCKS5 | 👨‍💻 GitHub 32.x | |
| 开发 | ☁️ Cloudflare SOCKS5 | ☁️ Cloudflare 32.x | |
| 开发 | 🖌️ Figma SOCKS5 | 🖌️ Figma 32.x | |
| 开发 | 📝 Notion SOCKS5 | 📝 Notion 32.x | |
| 系统 | 🪟 Microsoft SOCKS5 | 🪟 Microsoft 32.x | |
| 系统 | 🍎 Apple SOCKS5 | 🍎 Apple 32.x | |
| 金融 | 🪙 加密货币 SOCKS5 | 🪙 加密货币 32.x | |
| 金融 | 💰 PayPal SOCKS5 | 💰 PayPal 32.x | |
| 游戏 | 🎮 Steam SOCKS5 | 🎮 Steam 32.x | |
| 兜底 | 🌏 SOCKS5-国内 | — | 国内域名/IP，手选节点 |
| 兜底 | 🌍 SOCKS5-国外 | 🚀 默认出口 32.x | 国外流量兜底 |

**`!` 前缀**：表示该平台有严格 IP 封控（Claude/ChatGPT/Twitter/Instagram/Facebook），切换节点需谨慎。

### 8.4 精细化分流规则链（socks5-rules）

```yaml
sub-rules:
  socks5-rules:
    # ── 私有地址直连 ──
    - RULE-SET,private_domain,DIRECT
    - RULE-SET,private_ip,DIRECT,no-resolve

    # ── AI ──
    - RULE-SET,claude_domain,🤖 !Claude SOCKS5
    - RULE-SET,openai_domain,🤖 !ChatGPT SOCKS5
    - RULE-SET,gemini_domain,🔮 Gemini SOCKS5
    - RULE-SET,deepseek_domain,🐋 DeepSeek SOCKS5
    - RULE-SET,ai_catchall,🤖 !Claude SOCKS5

    # ── 视频 ──
    - RULE-SET,youtube_domain,📹 YouTube SOCKS5
    - RULE-SET,netflix_domain,🎥 Netflix SOCKS5
    - RULE-SET,netflix_ip,🎥 Netflix SOCKS5,no-resolve
    - RULE-SET,tiktok_domain,🎵 TikTok SOCKS5
    - RULE-SET,spotify_domain,🎧 Spotify SOCKS5
    - RULE-SET,disney_domain,🏰 Disney+ SOCKS5

    # ── 社交 ──
    - RULE-SET,telegram_domain,📲 Telegram SOCKS5
    - RULE-SET,telegram_ip,📲 Telegram SOCKS5,no-resolve
    - RULE-SET,twitter_domain,🐦 !Twitter SOCKS5
    - RULE-SET,twitter_ip,🐦 !Twitter SOCKS5,no-resolve
    - RULE-SET,instagram_domain,📸 !Instagram SOCKS5
    - RULE-SET,facebook_domain,👤 !Facebook SOCKS5
    - RULE-SET,facebook_ip,👤 !Facebook SOCKS5,no-resolve
    - RULE-SET,discord_domain,🎮 Discord SOCKS5

    # ── 开发/生产力 ──
    - RULE-SET,google_domain,🍀 Google SOCKS5
    - RULE-SET,google_ip,🍀 Google SOCKS5,no-resolve
    - RULE-SET,github_domain,👨‍💻 GitHub SOCKS5
    - RULE-SET,cloudflare_domain,☁️ Cloudflare SOCKS5
    - RULE-SET,cloudflare_ip,☁️ Cloudflare SOCKS5,no-resolve
    - RULE-SET,figma_domain,🖌️ Figma SOCKS5
    - RULE-SET,notion_domain,📝 Notion SOCKS5

    # ── 系统/办公 ──
    - RULE-SET,microsoft_domain,🪟 Microsoft SOCKS5
    - RULE-SET,apple_domain,🍎 Apple SOCKS5
    - RULE-SET,apple_ip,🍎 Apple SOCKS5,no-resolve

    # ── 金融/游戏 ──
    - RULE-SET,crypto_domain,🪙 加密货币 SOCKS5
    - RULE-SET,paypal_domain,💰 PayPal SOCKS5
    - RULE-SET,steam_domain,🎮 Steam SOCKS5

    # ── 兜底 ──
    - RULE-SET,cn_domain,🌏 SOCKS5-国内
    - RULE-SET,cn_ip,🌏 SOCKS5-国内,no-resolve
    - RULE-SET,geolocation_not_cn,🌍 SOCKS5-国外
    - MATCH,🌍 SOCKS5-国外
```

**与 32.x 主路由规则的关键差异**：

| 差异点 | 32.x 主路由 `rules` | SOCKS5 `socks5-rules` |
|--------|---------------------|----------------------|
| 国内流量 | `cn_domain/cn_ip → DIRECT` | `cn_domain/cn_ip → 🌏 SOCKS5-国内`（走代理，隐藏爬虫真实 IP） |
| 国外兜底 | `geolocation_not_cn → 🚀 默认出口 32.x` | `geolocation_not_cn → 🌍 SOCKS5-国外`（引用默认出口 32.x） |
| 最终兜底 | `MATCH → 🐟 漏网之鱼` | `MATCH → 🌍 SOCKS5-国外`（确保无直连泄漏） |
| 平台覆盖 | 完全相同 | 完全相同，一一对应 |

### 8.5 Cloudflare TLS 指纹检测与 curl_cffi

Cloudflare 不仅检测 IP，还检测 **TLS Client Hello 指纹**（JA3/JA4）：

| 客户端 | TLS 指纹特征 | Cloudflare 判定 |
|--------|-------------|----------------|
| Chrome 浏览器 | Chrome JA3 hash | ✅ 合法浏览器 |
| curl | 非浏览器 JA3 hash | ❌ 机器人 → 403 |
| Python requests | 非浏览器 JA3 hash | ❌ 机器人 → 403 |
| **curl_cffi** | **Chrome JA3 hash**（`impersonate="chrome124"`） | ✅ **模拟浏览器指纹，通过** |

这就是为什么**仅共享 IP 不够**：浏览器访问 Claude 通过，但用标准 curl 走相同节点仍被 403，因为 TLS 指纹暴露了爬虫身份。

**生产环境必须使用 curl_cffi**：

```python
from curl_cffi import requests

response = requests.get(
    "https://claude.ai/api/...",
    impersonate="chrome124",   # 模拟 Chrome TLS 指纹
    proxies={"https": "socks5://crawler1:password@192.168.31.1:7891"}
)
```

### 8.6 IP 信誉预热流程

新节点或切换节点后，IP 信誉为空，爬虫请求大概率被拦截。需预热：

```
1. 在 32.x 策略组中选好节点（如 日本★）
2. 用浏览器访问目标站点（如 claude.ai）
3. 如遇 Cloudflare 人机验证，手动完成
4. 此时该节点 IP 已被目标站点标记为可信
5. 爬虫通过 SOCKS5 走同一节点 → 继承信誉 → 请求通过
```

**何时需要预热**：
- 切换节点后（新 IP 无信誉）
- 爬虫突然大量 403（IP 信誉衰减或被封）
- 首次配置 SOCKS5 时

### 8.7 调试流程

```
爬虫返回 403
├─ 用浏览器(32.x)访问同一网站
│  ├─ 浏览器正常 → SOCKS5 403 → TLS 指纹问题
│  │  → 确认爬虫使用 curl_cffi + impersonate="chrome124"
│  │
│  └─ 浏览器也 403 → IP 被封
│     → 在 32.x 面板切换节点
│     → 浏览器验证新节点是否可用
│     → 浏览器能开后，爬虫走同一节点
```

### 8.8 v6→v7 兼容性证明

v7 对 v6 的路由规则和子规则**零修改**，仅新增 SOCKS5 相关配置：

| 对比项 | v6 | v7 | 变化 |
|--------|----|----|------|
| 策略组 | 59 个 | 85+ | +26 SOCKS5 组，原有 0 修改 |
| 路由规则 | 43 条 | 43 条 | 完全一致 |
| 子规则 | residential34 | +socks5-rules | residential34 未动 |
| 规则集 | 42 个 | 42+ | SOCKS5 复用现有规则集，无新增 |
| DNS | — | — | 完全一致 |
| 机场订阅 | 5 个 | 5 个 | 完全一致 |
| 静态节点 | 1 个 | 1 个 | 完全一致 |

**隔离机制**：SOCKS5 端口 7891 通过 `listeners.rule: socks5-rules` 绑定到独立子规则链，流量不经过主路由 `rules`。SOCKS5 策略组引用 32.x 组和地区组，但 32.x 组不会反向引用 SOCKS5 组——引用关系是单向的，SOCKS5 跟随 32.x 的节点选择，不会干扰 32.x。

### 8.9 防封控最佳实践

**节点选择**：

- 默认使用第 1 选项（引用 32.x 组），自动跟随浏览器节点，共享 IP 信誉
- 如爬虫需用不同节点（如浏览器用日本★、爬虫用美国★），在 SOCKS5 组中手动切换到其他选项

**TLS 指纹**：

- 生产环境必须使用 curl_cffi + `impersonate="chrome124"`
- 标准 curl 和 Python requests 会暴露非浏览器 TLS 指纹，被 Cloudflare 识别为机器人

**IP 信誉预热**：

- 新节点或切换节点后，先用浏览器访问目标站点完成人机验证
- 预热后的 IP 信誉可显著降低爬虫 403 概率

**故障排查**：

- 大量 403 → 检查 IP 是否被封（浏览器验证）→ 换节点 + 预热
- 偶发 403 → 可能是 Cloudflare 临时策略，等待或预热即可
- 持续 403 + 浏览器正常 → 检查 TLS 指纹（确认使用 curl_cffi）

## 9. 迭代历史

| 版本 | 日期 | 核心变更 |
|------|------|---------|
| v5 | 2026-03-xx | 5 机场阶梯式分层架构，双轨(★/省流)策略组 |
| v6 | 2026-04-12 | 住宅网段白名单（SUB-RULE + DIRECT-34 relays/domain 分文件），v6 住宅 IP 出口 |
| v7 | 2026-05-02 | SOCKS5 入站（listeners + rule 绑定 + 多用户认证），26 个平台镜像策略组（与 32.x 一一对应），第 1 选项引用 32.x 组共享节点/IP 信誉，TLS 指纹检测（curl_cffi），IP 信誉预热流程 |
