# 方案设计文档

> 项目：Mihomo 多网段精细分流配置  
> 版本：v8
> 最后更新：2026-10-02

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

### 1.1 普通代理入口架构

程序通过 OpenClash 普通代理端口进入 Mihomo，在规则模式下与透明代理共用主 `rules` 和既有策略组：

```text
程序（建议 192.168.32.x）
    → HTTP :7890 / SOCKS5 :7891 / 混合 :7893
    → OpenClash 页面设置的账号认证
    → 主 rules 按实际来源 IP 和目标匹配
    → 平台策略组 → 地区组 → 机场节点
```

普通入口不单独绑定机场，也不维护 HTTP 平台镜像组。程序实际来源为 32.x 时，平台请求使用与其他 32.x 流量相同的策略组；地区内测速与故障切换仍可能改变最终节点。目标站点是否接受请求，需要另外验证。

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

**住宅类网段且需白名单（推荐与 v8 一致）**：

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

### 2.5 普通代理入口路由设计

全局普通端口配置为：

```yaml
port: 7890        # HTTP 代理
socks-port: 7891  # SOCKS5 代理
mixed-port: 7893  # 同时接受 HTTP 和 SOCKS5
```

账号密码在 OpenClash 的 SOCKS5/HTTP(S) 认证页面设置并应用，模板不维护 crawler 用户或真实认证凭据。控制面板 `secret` 是独立的 API 密钥，不作为代理入口密码。

在规则模式下，普通入口进入 §2.1 的主规则，按内核实际看到的来源分流：

| 实际来源 IP | 行为 |
|------------|------|
| 192.168.31.x | DIRECT |
| 192.168.32.x | 继续匹配平台及国内/国外兜底规则 |
| 192.168.33.x | `🌐 全局代理 33.x` 当前选择 |
| 192.168.34.x | `residential34` 白名单直连，其余住宅代理 |
| 其他来源 | 不匹配上述专用源规则，继续后续主规则 |

需要按平台代理的程序应位于 32.x；容器、路由器本机任务、中继或 NAT 可能改变来源，部署后须用 Zashboard 连接详情核验 sourceIP。连接到哪个路由器地址、使用哪个认证账号，都不能替代来源 IP 匹配。

OpenClash 页面可能覆写模板字段。保存并应用后需检查内核实际加载的端口、认证和规则模式，避免旧运行配置保留历史 HTTP listener。

### 2.6 单设备衍生配置

单设备配置维护在 `client/v8.yaml`，直接接管运行 Mihomo 的桌面电脑或 Android 设备本机流量。它复用路由器版的应用分流、地区选择、机场梯队和健康检查，但不再通过源网段表达入口角色。

```text
本机应用流量
    → TUN / 系统代理
    → 私有地址与自定义目标直连
    → 平台规则
    → 地区组
    → A/C/B 机场梯队与跨国兜底
```

设计边界：

- 删除所有 `SRC-IP-CIDR` 和 `SUB-RULE` 网段入口。
- 不包含 33.x 全局出口、34.x 住宅代理或路由器专用入口。
- 保留 24 个应用组、19 个地区组、15 个机场子组、默认出口和漏网之鱼，共 60 个策略组。
- `allow-lan: false`，混合代理、控制器和 DNS 只监听本机回环地址。
- 客户端自定义直连规则独立维护在 `client/rulesets/DIRECT.yaml`。
- 路由器的 `configs/*.yaml` 和 `configs/rulesets/*` 是外部部署依赖的稳定路径，不得随目录整理迁移。

## 3. 机场阶梯式分层体系设计

### 3.1 三梯队多机场架构

每个梯队包含 1~2 个机场，梯队内 url-test 跨机场选最快节点，梯队间 fallback 故障转移。

```
┌─────────────────────────────────────────────────────────┐
│ 第1层：梯队子组（url-test，跨该梯队所有机场选最快）      │
│                                                          │
│  [C] 香港 = AirportC1+AirportC2 的香港节点选最快      │
│  [B] 香港 = AirportB1+AirportB2 的香港节点选最快                │
│  [A] 香港 = AirportA 的香港节点选最快                       │
├─────────────────────────────────────────────────────────┤
│ 第2层：地区组（fallback，同国优先，随后展开跨国叶子组）  │
│  🇭🇰 香港   = [C]→[B]→[A]→日→美→新→台→…（省流版）     │
│  🇭🇰 香港★  = [A]→[C]→[B]→日→美→新→台→…（稳定版）    │
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
| 主力 C | AirportC1 + AirportC2 | 速度适中价格合理 | 视频/社交/日常大流量 |
| 保底 B | AirportB1 + AirportB2 | 最便宜，兜底用 | 主力挂了时备用 |
| 优质 A | AirportA | 最贵最稳 | AI/默认出口/全局代理 |

### 3.3 省流版 vs 稳定版

| 属性 | 省流版（无★） | 稳定版（带★） |
|------|-------------|-------------|
| 故障转移顺序 | C→B→A | A→C→B |
| 设计意图 | 主力先上，保底兜底 | 优质先上，主力次之 |
| 适用场景 | 视频/社交/日常 | AI/默认出口/全局代理 |
| 为什么这样设计 | 看视频流量大，优先省钱 | AI 长对话不能断，优先稳定 |

两种地区组在同国三个梯队全部失效后，都会继续尝试直接展开的跨国叶子组，公共顺序为日本→美国→新加坡→台湾→马来西亚→韩国→荷兰→英国→德国→法国→越南，不含香港。这里刻意不建立独立的跨国 `fallback`，避免在 v7 已长期使用的 `fallback → url-test` 结构上再增加一层 `fallback → fallback`。Mihomo 对嵌套代理组存在已知运行时风险，因此每个主要国家顶层组直接引用叶子组，顶层国家组之间也不得互相引用。

### 3.4 调整梯队归属

机场的梯队归属只由锚点的 `use:` 列表控制。调整时只需移动机场名：

```yaml
# 例：把 AirportB2 从保底移到主力
sub_ut_c: use: [AirportC1, AirportC2, AirportB2]  # 加入主力
sub_ut_b: use: [AirportB1]                           # 保底只剩一个
```

无需改动策略组、规则、规则集等任何其他部分。

### 3.5 次要地区处理

英国/德国/法国/韩国/马来西亚/荷兰/越南节点数量少，不值得拆成 3 个梯队子组。直接用 `url-test + include-all-providers + filter` 从全部 5 个机场选最快节点。马来西亚、荷兰、越南在 v8 中同时作为跨国兜底尾部；香港不进入跨国兜底。

## 4. 策略组分区设计

### 4.1 面板布局（从上到下）

```
┌─ 第1区：出口组（4个）──────────────────────────┐
│  🚀 默认出口 32.x    → 未命中规则的海外流量（可选 ★稳定 / 无★省流）│
│  🌐 全局代理 33.x    → 33.x 网段全部流量        │
│  🏠 住宅IP 34.x      → 34.x 非白名单流量        │
│  🐟 漏网之鱼         → MATCH 兜底               │
├─ 第2区：应用策略组（24个，多数含 ★稳定+省流双轨）───────┤
│  AI: Claude, ChatGPT, Gemini, DeepSeek          │
│  视频: YouTube, Netflix, TikTok, Spotify, Disney│
│  社交: Telegram, X, Instagram, Facebook, Discord│
│  开发: Google, GitHub, Cloudflare, Figma, Notion│
│  系统: Microsoft, Apple                          │
│  金融: 加密货币, PayPal                          │
│  游戏: Steam                                     │
├─ 第3区：地区组（19个）─────────────────────────┤
│  5主要地区 × 2版本 + 7次要地区 + 全部 + 故转；跨国兜底展开 │
├─ 第4区：机场子组（15个）───────────────────────┤
│  5主要地区 × 3机场 = 15个 url-test 子组          │
└────────────────────────────────────────────────┘
```

当前路由器 v8 共 62 个策略组：4 个出口组、24 个应用组、19 个地区组和 15 个机场子组；另有 42 个 rule-provider、5 个机场 provider。

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

### 5.2 完整请求链路示例

以 32.x 网段设备访问 `claude.ai` 为例，展示请求从进入到出站的调用链：

```
设备(32.x) 请求 claude.ai
  │
  ▼
【第1层：路由规则】rules 链逐条匹配
  ├─ 31.x? ❌  33.x? ❌  34.x? ❌    → 32.x 不拦截，继续
  ├─ private? ❌  direct_32? ❌      → 继续
  └─ RULE-SET,claude_domain 命中 ✅  → 转交 🤖 Claude 32.x
       │
       ▼
【第2层：应用策略组】🤖 Claude 32.x（select，用户面板手选）
  默认选中：🇯🇵 日本★
       │
       ▼
【第3层：地区组】🇯🇵 日本★（fallback，自动故障转移）
  优先顺序：[A] 日本 → [C] 日本 → [B] 日本
            → 直接展开的跨国叶子组（日→美→新→台→马→韩→荷→英→德→法→越）
  [A] 日本 健康检查通过? ✅ → 选它
       │
       ▼
【第4层：机场子组】[A] 日本（url-test，自动选最快节点）
  use: [AirportA] → filter 出日本节点 → url-test 选延迟最低
       │
       ▼
【出口】AirportA 日本节点（如 jp1.yuntu.com:443）
  → 加密隧道 → 出口 IP 1.2.3.4 → 访问 claude.ai ✅
```

**各层职责**：

| 层级 | 类型 | 谁决定 | 作用 |
|------|------|--------|------|
| 路由规则 | `rules` | 自动匹配 | 按域名/IP 分流到对应策略组 |
| 应用策略组 | `select` | 用户面板手选 | 选择出口地区（日本★/美国/新加坡…） |
| 地区组 | `fallback` | 自动故障转移 | 梯队间自动切换，保证可用性 |
| 机场子组 | `url-test` | 自动选最快 | 梯队内跨机场选延迟最低的节点 |

**故障转移场景**：若 [A] 日本失败则依次尝试 [C]/[B] 日本；同国全部失败后，当前顶层组继续遍历直接展开的跨国叶子组，按公共顺序选择首个健康地区。既有连接会中断，应用重连后的新连接使用新出口。

### 5.3 特殊策略组默认值

以下策略组默认选择 `DIRECT`（国内服务优先直连，不消耗机场流量）：

| 策略组 | 默认选项 | 原因 |
|--------|---------|------|
| 🐋 DeepSeek 32.x | DIRECT | 国内服务，直连更快 |
| 🪟 Microsoft 32.x | DIRECT | OneDrive/Office 国内 CDN 可用 |
| 🍎 Apple 32.x | DIRECT | iCloud/App Store 国内有服务器 |
| 🎮 Steam 32.x | DIRECT | 游戏下载量大，直连省流量 |

以下策略组兜底走 `🚀 默认出口 32.x`（非独立平台出口）：

| 规则集 | 所属策略组 | 说明 |
|--------|-----------|------|
| ai_catchall（category-ai-!cn） | 🚀 默认出口 32.x | 未被 Claude/ChatGPT/Gemini/DeepSeek 覆盖的 AI 服务 |

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

# 3b. 若与 34.x 相同需求（白名单直连 + 默认住宅）：复制 v8 的
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
  use: [AirportC1, AirportC2, NewAirport]  # 加入主力梯队
```

无需修改策略组、规则或规则集。

### 7.4 调整机场梯队归属

```yaml
# 把 AirportB2 从保底梯队移到主力梯队
sub_ut_c: use: [AirportC1, AirportC2, AirportB2]
sub_ut_b: use: [AirportB1]
```

### 7.5 住宅网段白名单与文档索引（v6）

- **需求与验收**：`REQUIREMENTS.md`（NET-04、RES-*、NFR-05）
- **落地配置**：`configs/v8.yaml`、`configs/rulesets/DIRECT-34-relays.yaml`、`configs/rulesets/DIRECT-34.yaml`
- **内核说明**：[Route Rules](https://wiki.metacubex.one/en/config/rules/)（`SUB-RULE`）、[sub-rule](https://wiki.metacubex.one/en/config/sub-rule/)

### 7.6 程序接入与部署

1. 在私有配置副本填写机场订阅、控制器 `secret` 和住宅代理凭据（使用 34.x 时）；保留旧配置用于切回。
2. 上传并选择当前 v8，使用实际路由器内核执行语法检查；本地语法通过不替代路由器启动检查。
3. 在 OpenClash 页面统一设置 HTTP `7890`、SOCKS5 `7891`、混合 `7893` 及代理认证，保存并应用。
4. 将需要按平台分流的程序放在 32.x，填写可从该设备访问的路由器 IP、正确协议/端口和页面中的账号密码。
5. 确认运行在规则模式；从程序发起新连接，在 Zashboard 核验来源 IP、匹配规则、平台策略链和最终节点。
6. 检查运行日志无重复监听、认证或规则加载错误，再验证目标平台实际访问；切换平台组后以新连接核对出口。

新增平台只需维护 §7.1 的主策略组、规则和 provider，普通代理程序自动复用；无需新增 HTTP 镜像组。

## 8. 普通代理出口与排错

### 8.1 机场流量归属

入口端口不绑定机场。流量由实际出站节点所属的机场统计；最终 DIRECT 不消耗机场代理流量，住宅节点使用住宅服务的流量额度。

例如 32.x 程序访问 ChatGPT，面板将 `🤖 ChatGPT 32.x` 选为日本★，且 `[A] 日本` 可用时，连接可能为：

```text
HTTP :7890 → 主 rules → ChatGPT 32.x → 日本★ → [A] 日本 → AirportA 的日本节点
```

此时使用 AirportA 流量。若 `[A] 日本` 失效，下一条连接可能进入 `[C] 日本`，使用 AirportC1 或 AirportC2 中最终节点的流量。地区名或 A/C/B 梯队只能说明选择路径，不能单独确认具体机场。

### 8.2 观察方法

在 Zashboard 的连接列表找到程序请求，打开详情查看来源 IP、规则、完整代理链和最终节点，再与机场 provider 中的节点对应。若不同机场节点重名，需要结合 provider 归属确认，不能仅凭显示名判断。机场面板流量统计作为计费核对依据，连接流量用于排查单次请求。

### 8.3 故障定位

| 现象 | 检查方向 |
|------|----------|
| 连接拒绝 | OpenClash 是否启动、实际监听端口、路由器地址与防火墙可达性、端口占用日志 |
| 连接超时 | 先确认客户端到入口是否可达，再检查规则集、机场订阅、当前节点与 DNS |
| HTTP 407 / SOCKS5 认证失败 | 核对 OpenClash 页面启用的账号与实际运行认证，确认客户端发送认证信息 |
| 协议握手错误 | HTTP 使用 7890，SOCKS5 使用 7891，混合端口 7893 接受两种协议；清除旧 7891 HTTP 配置 |
| 入口可用但出口不符 | 核验规则模式、实际 sourceIP、匹配规则、平台当前选择及新连接代理链 |
| 目标站点 403/429 | 入口连通与站点访问分别排查；检查目标站点授权、请求频率、出口和客户端行为，不根据状态码断定唯一原因 |

### 8.4 当前方案边界

当前 v8 已移除 v7 的 crawler listener、31 个 HTTP 专用组、`http-rules` 及 9 个 HTTP 专用 rule-provider。主 `rules`、住宅白名单子链、机场梯队、DNS 和地区故障转移沿用原设计。代理接入不承诺目标平台返回 200，也不承诺浏览器认证状态能由其他程序继承。

## 9. 迭代历史

以下 v7 独立 HTTP 方案仅描述历史配置，当前 v8 使用 §1.1 和 §2.5 的普通入口。

| 版本 | 日期 | 核心变更 |
|------|------|---------|
| v5 | 2026-03-xx | 5 机场阶梯式分层架构，双轨(★/省流)策略组 |
| v6 | 2026-04-12 | 住宅网段白名单（SUB-RULE + DIRECT-34 relays/domain 分文件），v6 住宅 IP 出口 |
| v7 | 2026-05-02 | HTTP 入站（listeners + rule 绑定 + 多用户认证），24 个平台镜像组、5 个国内平台组和 2 个兜底组，镜像组引用 32.x 组跟随节点（历史功能，当前已移除） |
| v8 | 2026-10-02 | 不含香港的跨国兜底直接展开到主要国家顶层组，避免新增 `fallback → fallback` 层级；新增马来西亚/荷兰/越南；健康检查严格要求 204；地区 fallback 周期调整为 60 秒；移除独立 HTTP 池，程序复用 OpenClash 普通入口及主规则 |
