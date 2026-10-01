# 技术规范文档

> 项目：Mihomo 多网段精细分流配置  
> 版本：v8
> 最后更新：2026-10-02

---

## 1. 技术栈

| 组件 | 版本/说明 |
|------|----------|
| 代理内核 | [Mihomo](https://github.com/MetaCubeX/mihomo)（Clash Meta 继承者） |
| 运行平台 | OpenWrt + Mihomo 插件 |
| 配置格式 | YAML（支持锚点 `&` / 合并 `<<: *`） |
| 控制面板 | [Zashboard](https://github.com/Zephyruso/zashboard)（通过 API 9090 端口访问） |
| 规则集格式 | `.mrs`（二进制，推荐）/ `.yaml` / `.list`（文本） |

## 2. 参考资源

### 2.1 官方文档

| 名称 | URL | 说明 |
|------|-----|------|
| Mihomo Wiki | https://wiki.metacubex.one/config/ | 配置字段完整参考 |
| 全局配置 | https://wiki.metacubex.one/config/general/ | mixed-port/allow-lan/ipv6 等 |
| DNS 配置 | https://wiki.metacubex.one/config/dns/ | fake-ip/respect-rules/nameserver 等 |
| 代理组 | https://wiki.metacubex.one/config/proxy-groups/ | select/url-test/fallback/filter 等 |
| 路由规则 | https://wiki.metacubex.one/config/rules/ | RULE-SET/DOMAIN/GEOSITE/SRC-IP-CIDR、`SUB-RULE`、`AND`/`OR` 等 |
| 子规则 | https://wiki.metacubex.one/config/sub-rule/ | 住宅网段白名单子链（v6） |
| 规则集 | https://wiki.metacubex.one/config/rule-providers/ | behavior/format/interval 等 |
| 代理集 | https://wiki.metacubex.one/config/proxy-providers/ | health-check/exclude-filter/override 等 |
| 入站配置 | https://wiki.metacubex.one/config/inbound/ | TUN/sniffer 等 |
| 普通入站端口 | https://wiki.metacubex.one/config/inbound/port/ | HTTP/SOCKS5/mixed 端口；认证见全局配置 |
| 完整示例 | https://github.com/MetaCubeX/mihomo/blob/Meta/docs/config.yaml | 官方示例配置 |

### 2.2 客户端衍生配置

`client/v8.yaml` 面向 Clash Verge Rev、Clash Party（原 Mihomo Party）、FlClash 及其他较新的 Mihomo 客户端。它直接处理桌面电脑或 Android 设备的本机流量，不包含源网段匹配，也不依赖任何 `192.168.x.x` 网段；iPhone/iPad 客户端不在直接兼容范围内。

- 保留 24 个应用组、19 个地区组、15 个机场子组、默认出口和漏网之鱼，共 60 个策略组。
- 保留 5 个 proxy-provider、日常分流所需的 40 个 rule-provider，以及地区 fallback 和机场子组的原健康检查参数。
- 自定义直连目标由 `direct_client` 引用 `client/rulesets/DIRECT.yaml`，与 `configs/rulesets/` 下的路由器规则分开维护。
- 不包含家庭源网段入口、住宅代理、住宅子规则和路由器共享端口；仅处理本机流量。
- TUN 保留 `auto-route`、`auto-detect-interface` 和 DNS 劫持，删除 Linux/OpenWrt 导向的 `auto-redirect`。
- 混合代理、控制器和 DNS 只监听本机；内置外部面板下载配置不进入客户端模板。
- 需要较新的 Mihomo 内核支持 `.mrs`、provider 指纹覆盖、`expected-status` 和 `include-all-providers`；传统 Clash Premium 或非 Mihomo 客户端不在兼容范围内。
- `configs/*.yaml` 与 `configs/rulesets/*` 是路由器部署使用的稳定公共路径，不得迁移；单设备配置只能在顶层 `client/` 内演进。

### 2.3 规则集仓库

| 仓库 | URL | 说明 |
|------|-----|------|
| MetaCubeX meta-rules-dat | https://github.com/MetaCubeX/meta-rules-dat/tree/meta | 主要规则源，提供 .mrs/.yaml/.list 三种格式 |
| blackmatrix7 ios_rule_script | https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Clash | 最全面的 Clash 规则库，数百个应用分类 |
| ACL4SSR | https://github.com/ACL4SSR/ACL4SSR/tree/master/Clash | 经典 Clash 规则模板 |
| wwqgtxx clash-rules | https://github.com/wwqgtxx/clash-rules | fakeip-filter 等特殊规则集 |

### 2.4 参考配置

| 名称 | URL | 说明 |
|------|-----|------|
| qichiyuhub DNS 防泄露配置 | https://raw.githubusercontent.com/qichiyuhub/rule/refs/heads/main/config/mihomo/configdns.yaml | DNS 配置最佳实践参考 |
| usbog232 精细分流模板 | https://raw.githubusercontent.com/usbog232/clashmetadingyue/main/new-meta.ini | subconverter INI 模板参考 |
| ACL4SSR 全量模板 | https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/config/ACL4SSR_Online_Full.ini | 经典全量分流 INI 模板 |

## 3. 规则集 URL 清单

以下为当前配置采用的路径；本轮没有逐个重新请求 URL，部署时需验证下载成功且内容可被内核加载。旧文档的 HTTP 200 记录不能作为当前可用性保证。当前路由器模板共 42 个 rule-provider。

### 3.1 MetaCubeX geosite（behavior: domain, format: mrs）

URL 模式：`https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/{name}.mrs`

| 配置名 | name | 验证状态 |
|--------|------|---------|
| openai_domain | openai | 部署时验证 |
| deepseek_domain | deepseek | 部署时验证 |
| ai_catchall | category-ai-!cn | 部署时验证 |
| youtube_domain | youtube | 部署时验证 |
| netflix_domain | netflix | 部署时验证 |
| tiktok_domain | tiktok | 部署时验证 |
| spotify_domain | spotify | 部署时验证 |
| disney_domain | disney | 部署时验证 |
| telegram_domain | telegram | 部署时验证 |
| twitter_domain | twitter | 部署时验证 |
| instagram_domain | instagram | 部署时验证 |
| facebook_domain | facebook | 部署时验证 |
| discord_domain | discord | 部署时验证 |
| google_domain | google | 部署时验证 |
| github_domain | github | 部署时验证 |
| cloudflare_domain | cloudflare | 部署时验证 |
| figma_domain | figma | 部署时验证 |
| notion_domain | notion | 部署时验证 |
| onedrive_domain | onedrive | 部署时验证 |
| microsoft_domain | microsoft | 部署时验证 |
| apple_domain | apple | 部署时验证 |
| steam_domain | steam | 部署时验证 |
| paypal_domain | paypal | 部署时验证 |
| private_domain | private | 部署时验证 |
| geolocation_not_cn | geolocation-!cn | 部署时验证 |
| cn_domain | cn | 部署时验证 |

### 3.2 MetaCubeX geoip（behavior: ipcidr, format: mrs）

URL 模式：`https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/{name}.mrs`

| 配置名 | name | 验证状态 |
|--------|------|---------|
| netflix_ip | netflix | 部署时验证 |
| telegram_ip | telegram | 部署时验证 |
| twitter_ip | twitter | 部署时验证 |
| facebook_ip | facebook | 部署时验证 |
| google_ip | google | 部署时验证 |
| cloudflare_ip | cloudflare | 部署时验证 |
| private_ip | private | 部署时验证 |
| cn_ip | cn | 部署时验证 |

特殊：`apple_ip` 使用 **geo-lite** 路径（完整版无此文件）：
`https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo-lite/geoip/apple.mrs`

### 3.3 blackmatrix7（behavior: classical）

| 配置名 | URL | 格式 | 验证状态 |
|--------|-----|------|---------|
| claude_domain | `.../Clash/Claude/Claude.yaml` | yaml | 部署时验证 |
| gemini_domain | `.../Clash/Gemini/Gemini.yaml` | yaml | 部署时验证 |
| crypto_domain | `.../Clash/Cryptocurrency/Cryptocurrency.list` | text | 部署时验证 |

### 3.4 历史失效路径（当前不采用）

| URL | 历史记录 | 当前处理 |
|-----|------|---------|
| `blackmatrix7/.../DeepSeek/DeepSeek.yaml` | 曾返回 404 | MetaCubeX `geosite/deepseek.mrs` |
| `blackmatrix7/.../DeepSeek/DeepSeek.list` | 曾返回 404 | 同上 |
| `MetaCubeX/geo/geoip/apple.mrs` | 曾返回 404 | 使用 `geo-lite/geoip/apple.mrs` |
| `MetaCubeX/meta/geo/geosite/weibo.mrs` | 曾返回 404 | 当前 v8 已移除 HTTP 专用微博 provider，不需替代 |

### 3.5 其他

| 配置名 | URL | 验证状态 |
|--------|-----|---------|
| fakeipfilter | `wwqgtxx/clash-rules/release/fakeip-filter.mrs` | 部署时验证 |

### 3.6 本仓 `configs/rulesets`（v6）

与上游 Meta/blackmatrix7 并列维护的 **classical + yaml** 规则集，raw 路径需与 `rule-providers` 中 URL 一致（推送本仓库后生效）。

| rule-provider 键 | 文件 | 说明 |
|------------------|------|------|
| `direct_32` | `DIRECT-32.yaml` | 32.x 分流场景下个人维护的直连域名等（`RULE-SET,direct_32,DIRECT`） |
| `direct_34_relays` | `DIRECT-34-relays.yaml` | **仅**在 `sub-rules.residential34` 内、**先于** `direct_34`；Tun/UDP 下端口与落地 IP（无 Sniff 时与域名拆文件更稳） |
| `direct_34` | `DIRECT-34.yaml` | 同上子链内第二步；**仅域名** |

**命名约定**：文件名 `策略简写-网段.yaml`；住宅网段需拆 **`-relays`**（端口/IP）与主文件（域名）时，用 `DIRECT-34-relays.yaml` + `DIRECT-34.yaml`，键名 `direct_34_relays`、`direct_34`。

## 4. YAML 编码规范

### 4.1 文件结构

配置文件按以下章节顺序排列，每节用 `━━━` 分割线隔开：

```
一、全局配置
二、Web 控制面板
三、流量嗅探
四、TUN 入站
五、DNS 配置
六、机场订阅（proxy-providers）
七、静态住宅 IP（proxies）
八、代理组锚点
九、代理组（proxy-groups）
（v6）子规则 sub-rules：住宅网段白名单子链，紧挨在「十、路由规则」之前书写即可
十、路由规则（rules）
十一、规则集（rule-providers）
```

### 4.2 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 策略组名 | `emoji + 中文名 + 网段` | `🤖 Claude 32.x` |
| 地区组-省流版 | `国旗 + 国家名` | `🇯🇵 日本` |
| 地区组-稳定版 | `国旗 + 国家名 + ★` | `🇯🇵 日本★` |
| 机场子组 | `[机场等级] + 地区名` | `[C] 香港` |
| 规则集名 | `服务名_类型` | `telegram_domain`, `telegram_ip` |
| 锚点名 | `小写_缩写` | `sub_ut_c`, `region_eco` |
| 本仓 rulesets 文件名 | `策略简写-网段.yaml`；住宅白名单可拆 `-relays` | `DIRECT-32.yaml`, `DIRECT-34-relays.yaml`, `DIRECT-34.yaml` |
| 本仓 rulesets provider 键 | `小写_网段数字`；relays 加后缀 | `direct_32`, `direct_34_relays`, `direct_34` |

### 4.3 锚点使用规范

```yaml
# 定义锚点（放在第八节"代理组锚点"）
# 每个梯队的 use: 包含该梯队所有机场，url-test 跨机场选最快
sub_ut_c: &sub_ut_c
  type: url-test
  use: [AirportC1, AirportC2]  # 主力梯队：多机场
  url: https://www.gstatic.com/generate_204
  expected-status: 204
  interval: 300

# 使用锚点（在 proxy-groups 中通过 <<: * 合并）
- name: "[C] 香港"
  <<: *sub_ut_c           # 继承所有字段
  filter: "(?=.*(港|HK))" # 添加/覆盖特定字段
```

### 4.4 proxy-providers 客户端指纹

v8 不使用全局 `global-client-fingerprint`，在每个订阅提供器中通过 `override` 下发 Chrome 指纹：

```yaml
AirportC1:
  type: http
  url: "..."
  override:
    client-fingerprint: chrome
```

新增订阅提供器时同步添加该 `override`。它只设置 Mihomo 代理节点的客户端指纹，不改变应用自身的请求或认证行为。

### 4.5 rule-providers 锚点

```yaml
# 三种类型的模板
ip: &ip           # behavior: ipcidr, format: mrs
domain: &domain   # behavior: domain, format: mrs
classical: &classical  # behavior: classical, format: text

# 使用时如需覆盖 format，在继承后添加
claude_domain:
  <<: *classical     # 继承 behavior: classical, format: text
  url: "..."
  format: yaml       # 覆盖为 yaml（blackmatrix7 的 .yaml 文件）
```

### 4.6 正则表达式规范

节点过滤使用正则表达式，格式为正向+负向双重匹配：

```
(?=.*(港|HK|(?i)Hong))           ← 正向：必须包含这些关键字
^((?!(台|日|韩|新|美)).)*$       ← 负向：不能包含这些关键字
```

组合后：
```
(?=.*(港|HK|(?i)Hong))^((?!(台|日|韩|新|深|美|英|法|德|澳|韓)).)*$
```

### 4.7 普通代理入口配置规范

路由器 v8 使用全局普通端口：

```yaml
port: 7890
socks-port: 7891
mixed-port: 7893
allow-lan: true
mode: rule
```

HTTP 客户端使用 `http://路由器IP:7890`；SOCKS5 客户端使用 `socks5://路由器IP:7891`，支持由代理解析域名的客户端可使用 `socks5h://`；混合 `7893` 接受 HTTP 和 SOCKS5。`http://` 入口可通过 CONNECT 代理 HTTPS 目标，不代表入口自身使用 TLS。

代理用户名/密码在 OpenClash 的 SOCKS5/HTTP(S) 认证页面统一设置并启用，保存后应用配置。模板不写入真实凭据，不再定义 crawler 用户、`listeners.rule` 或 `http-rules`。代理认证与 9090 控制器 `secret` 分开管理。

普通入口在规则模式下进入主 `rules`，实际来源 IP 决定网段规则：31.x 直连、32.x 继续按平台匹配、33.x 全局代理、34.x 住宅白名单。需要按平台分流的程序放在 32.x，部署后核验内核连接详情中的 sourceIP；路由器本机、容器、中继及 NAT 请求不保证保留原始设备来源。

只允许可信内网访问共享入口。OpenClash 可能覆写上传模板，实际运行端口、认证和规则模式须在保存/应用后核验；模板语法测试不验证页面覆写后的状态。

## 5. 常见问题排查

### 5.1 规则集 URL 404

```bash
# 在路由器终端验证 URL
curl -sL -o /dev/null -w "%{http_code}" "URL地址"
```

如果返回 404，说明上游仓库删除了该文件，需要寻找替代源。
优先顺序：MetaCubeX .mrs > MetaCubeX .yaml > blackmatrix7 .yaml > blackmatrix7 .list

### 5.2 机场订阅拉取失败

```bash
# 测试订阅 URL 是否可达
curl -I "订阅URL"

# 如果超时，域名可能被墙，改为借道已有机场拉取：
proxy: AirportC1   # 替代原来的 proxy: DIRECT
```

### 5.3 某地区子组为空

原因：该机场没有对应地区的节点，或 filter 正则过滤了所有节点。

验证：在面板中检查 `[C] 日本` 等子组是否有节点。如果为空，fallback 会自动跳过该子组。

v8 的顶层主要地区组在同国 A/C/B 全部不可用后，继续遍历直接展开的跨国叶子组。公共顺序为日本→美国→新加坡→台湾→马来西亚→韩国→荷兰→英国→德国→法国→越南，不包含香港。配置不创建独立 `🛟 跨国最终兜底` 组，避免在 v7 既有的 `fallback → url-test` 结构上再增加一层 `fallback → fallback`；顶层国家组也不得互相引用。

### 5.4 普通代理入口调试

从 32.x 客户端执行测试，替换路由器地址与已启用的账号名。以下 curl 命令会提示输入密码，示例不把密码写入 URL：

```sh
# HTTP 7890
curl --connect-timeout 10 --max-time 30 --proxy http://192.168.32.1:7890 --proxy-user 'OPENCLASH_USERNAME' https://www.gstatic.com/generate_204

# SOCKS5 7891（由代理解析目标域名）
curl --connect-timeout 10 --max-time 30 --proxy socks5h://192.168.32.1:7891 --proxy-user 'OPENCLASH_USERNAME' https://www.gstatic.com/generate_204

# 混合 7893 的 HTTP 接入
curl --connect-timeout 10 --max-time 30 --proxy http://192.168.32.1:7893 --proxy-user 'OPENCLASH_USERNAME' https://www.gstatic.com/generate_204
```

Windows PowerShell 中使用 `curl.exe`，避免旧版本 PowerShell 的 `curl` 别名。示例地址假设路由器 32.x 接口为 `192.168.32.1`，以实际可达地址为准。检测目标成功时返回 204、无正文；可加 `--include` 观察状态。它验证当前请求连通，不证明其他目标网站可访问。

排查顺序：

1. **入口**：连接拒绝或超时时，检查 OpenClash 是否运行、客户端到路由器可达性、实际监听端口与端口占用日志。SOCKS5 7891 不接受历史 HTTP 7891 的协议配置。
2. **认证**：HTTP 407 或 SOCKS5 认证失败时，检查页面中的账号是否启用、密码是否正确，以及保存/应用后实际运行配置是否包含认证；分别测试有效、错误与缺失凭据。
3. **规则**：在 Zashboard 连接详情核验 sourceIP、匹配规则与代理链。32.x 才按预期复用平台组；31.x 会优先直连，33.x/34.x 会优先走自己的来源规则。
4. **节点**：追踪到最终节点与机场 provider，确认使用哪家机场流量。切换策略或发生故障转移后，重建连接再观察；入口端口不绑定机场。
5. **目标**：入口与节点连通后再测试业务目标。403/429 可能涉及目标站点授权、访问频率、出口或客户端行为，不能仅凭状态码断定 TLS 指纹或 IP 原因。

同一平台组可服务浏览器和程序，但实际节点受测速与故障切换影响。代理配置不保证浏览器人机验证或认证状态可被程序复用，也不承诺目标平台返回 200。

### 5.5 健康检查通过但网站不通

原因：健康检查只测试 `gstatic.com/generate_204`，不代表所有网站都能访问。
解决：v4 起默认出口以 ★稳定版（A→C→B）为首选；v5 在 `🚀 默认出口 32.x` 中同时列出省流版五地，便于在面板切到 C→B→A 省梯队费用。

v8 所有 gstatic 检测增加 `expected-status: 204`，避免劫持页或错误响应被误判健康；主要地区顶层 fallback 每 60 秒检查一次。面板的 `0 ms` 也可能表示 `lazy` 检测尚未执行，不应单独作为节点死亡证据。

## 6. 版本变更日志

当前公共模板在 2026-10-02 用官方 Mihomo v1.19.32 Windows compatible 内核执行配置语法检查，结果为 `test is successful`。该结果仅覆盖本地模板加载；真实 OpenClash 端口、认证、源网段分流和目标网络连通仍需部署后验证。

### v8（2026-10-02）

| 变更项 | 说明 |
|--------|------|
| 跨国最终兜底 | 跨国叶子组直接展开到各主要国家顶层 fallback，同国 A/C/B 全失效后跨国家恢复；不含香港，也不显示独立卡片 |
| fallback 兼容性 | 不新增 `fallback → fallback` 层级，降低 Mihomo 嵌套代理组已知运行时风险；保留已长期使用的顶层 `fallback → url-test` 结构；proxy-groups 总数 62、地区部分 19 |
| 新增地区 | 马来西亚、荷兰、越南使用全机场 `url-test` |
| 健康判定 | 所有 gstatic 检测增加 `expected-status: 204`；主要地区顶层 fallback 为 60s、`max-failed-times: 2` |
| Mihomo 指纹兼容 | 移除已废弃的 `global-client-fingerprint`，改由 5 个 proxy-provider 的 `override.client-fingerprint: chrome` 下发 |
| 配置主文件 | `configs/v8.yaml`；`configs/v7.yaml` 保留为历史版本 |
| 普通入口 | HTTP 7890、SOCKS5 7891、混合 7893；账号统一在 OpenClash 页面管理 |
| 移除独立 HTTP 池 | 删除 crawler listener、31 个 HTTP 专用组、`http-rules` 及 9 个专用 rule-provider；当前保留 42 个 rule-provider |
| 程序分流 | 规则模式按实际源网段匹配，32.x 程序复用平台组；最终节点决定机场流量归属 |
| 单设备衍生配置 | 新增 `client/v8.yaml` 与 `client/rulesets/DIRECT.yaml`；保留日常应用分流和故障转移，删除所有路由器入口、住宅与 HTTP 入站 |

### v7（2026-05-02，历史）

v7 曾提供独立 HTTP 7891 listener、多用户认证、31 个 HTTP 专用策略组与 `http-rules` 子链；24 个平台镜像组通过引用 32.x 组跟随选择。历史配置保存在 `configs/v7.yaml`。

此方案不适用于当前 v8。当前 7891 为 SOCKS5，请使用 §4.7 的端口和 OpenClash 页面认证；历史浏览器信誉继承、TLS 模拟与站点返回 200 的假设不作为当前设计或验收标准。

### v6（2026-04-12）

| 变更项 | 说明 |
|--------|------|
| 34.x 住宅网段 | `SUB-RULE` + `sub-rules`：`direct_34_relays` → `direct_34` → `MATCH,🏠 住宅IP 34.x` |
| 白名单维护 | `DIRECT-34-relays.yaml`（端口/IP）+ `DIRECT-34.yaml`（域名），避免 Tun/UDP 单 RULE-SET 不命中；`DIRECT-32.yaml` 承接原 `prdiy` |
| 仓库 rulesets 命名 | `策略-网段` 文件名 + `direct_*` provider 键；删除未使用的 `proxy-32` 举例文件 |
| 配置主文件 | `configs/v6.yaml`；v3/v4/v5 中个人直连 provider 更名为 `direct_32` 并指向 `configs/rulesets/DIRECT-32.yaml` |

### v5（2026-04-11）

| 变更项 | 原值 | 新值 | 原因 |
|--------|------|------|------|
| 机场数量 | 3 个（AirportA/B/C） | 5 个（AirportC1/AirportC2/AirportB1/AirportB2/AirportA） | 阶梯式多机场分层 |
| 梯队设计 | 每层 1 个机场 | 每层 1~2 个机场，层内 url-test 跨机场选最快 | 层内多机场冗余 |
| 稳定版★ fallback | A→B→C | A→C→B | C(主力)质量优于B(保底)，挂了先落主力 |
| 优质梯队 tolerance | 50ms | 100ms | AI 长对话减少不必要的节点切换 |
| 故障转移 filter | 6 个排除词 | 9 个排除词（+地址/客户端/紧急备用） | 适配新机场的信息节点 |
| 德国 filter | 无 Deutschland | 增加 Deutschland 匹配 | AirportB2 机场使用德语命名 |
| v4 typo 修复 | `英国住宅-35"d` | `英国住宅-35"` | 去掉多余字符 |
| v4 typo 修复 | 注释 `A→B→A` | 注释 `A→C→B` | 修正注释笔误 |
| 默认出口 32.x 可选 | 仅 ★稳定五地 | ★稳定 + 省流五地 + 次要地区等 | 面板可切省流，自定义范围更大 |
| 全局 33.x / 应用组 | 仅单轨地区 | 全局与各类应用组均补全 ★/省流双轨（有则加） | 与默认出口一致的自选能力 |

### v4（2026-04-10）

| 变更项 | 原值 | 新值 | 原因 |
|--------|------|------|------|
| 默认出口地区引用 | 省流版 (C→B→A) | ★稳定版 (A→B→C) | 修复选 HK 后部分网站不通 |
| deepseek_domain URL | blackmatrix7 (404) | MetaCubeX deepseek.mrs | 原 URL 已失效 |
| instagram_domain | blackmatrix7 .list | MetaCubeX .mrs | 统一格式，提升加载速度 |
| figma_domain | blackmatrix7 .yaml | MetaCubeX .mrs | 同上 |
| notion_domain | blackmatrix7 .yaml | MetaCubeX .mrs | 同上 |
| 新增 twitter_ip | — | MetaCubeX geoip/twitter.mrs | 补全 IP 规则 |
| 新增 facebook_ip | — | MetaCubeX geoip/facebook.mrs | 补全 IP 规则 |
| 新增 cloudflare_ip | — | MetaCubeX geoip/cloudflare.mrs | 补全 IP 规则 |
| 新增 ai_catchall | — | category-ai-!cn.mrs | AI 服务兜底 |
| sniffer | force-dns-mapping + parse-pure-ip | 移除 | 弃用字段 |
| DeepSeek 策略组 | 无 DIRECT | DIRECT 置顶 | 国内服务优先直连 |
