# Network：让家庭网络自动选择能用的出口

一套运行在 OpenWrt / ImmortalWrt 路由器上的 Mihomo（Clash Meta）多机场分流配置。

你只需要决定「某个平台走哪个国家」，剩下的机场测速、同国切换、跨国兜底全部自动完成。

- 路由器版：[configs/v8.yaml](configs/v8.yaml)
- 电脑 / Android 单机版：[client/v8.yaml](client/v8.yaml)（适配 Clash Verge Rev、Clash Party、FlClash）

## 使用场景

适合：

- 家里有 OpenWrt 路由器 + OpenClash，想让手机、电视等设备**免装代理客户端**直接上网；
- 手上有**多个机场**，想让它们互相备份、自动接力；
- 想按**平台**（ChatGPT、YouTube、GitHub…）选择国家，而不是天天手动挑节点；
- 愿意为不同用途划分多个 Wi-Fi / 网段（直连、分流、全局、住宅）。

不适合：

- 只有一个普通订阅、只需要全局代理；
- 不能管理路由器的 Wi-Fi、DHCP 和防火墙；
- 想直接上传模板却不填订阅和密码。

## 功能

一次请求经过四层选择：

```text
设备所在网段 → 目标平台 → 选择的国家 → 机场梯队与具体节点
```

**1. 四个网段，各管一摊**

| 网段 | 示例 Wi-Fi | 行为 |
| --- | --- | --- |
| 192.168.31.x | ap-direct | 全部直连（国内应用、游戏、排障） |
| 192.168.32.x | ap-rule | 国内直连、海外按平台分流（日常主力） |
| 192.168.33.x | ap-Global | 全部走代理（临时全局） |
| 192.168.34.x | ap-34x | 默认走住宅代理，白名单直连 |

Wi-Fi 和网段需要在 OpenWrt 里自己创建，配置只负责分流。

**2. 按平台独立选国家**

ChatGPT、Claude、Gemini、YouTube、Netflix、Telegram、GitHub、Google、Steam、PayPal 等都有独立策略组，在面板里各自选择地区。

**3. 机场三梯队 + 故障接力**

| 梯队 | 槽位 | 定位 |
| --- | --- | --- |
| A 优质 | AirportA（1 个） | 稳定优先，AI 和重要业务 |
| C 主力 | AirportC1、AirportC2 | 日常流量主力 |
| B 保底 | AirportB1、AirportB2 | 低成本兜底 |

- 省流地区组：`C → B → A`；稳定地区组（带 ★）：`A → C → B`
- 同国机场全挂后自动跨国兜底（美国→日本→新加坡→台湾→…，不含香港）
- 槽位名与你的机场无关：把订阅 URL 填进对应 provider 即可；调整梯队归属只需移动锚点 `use:` 列表里的名字

**4. 其他**

- 12 个可选地区：香港、日本、新加坡、美国、台湾 + 英国、德国、法国、韩国、马来西亚、荷兰、越南
- DNS 防泄漏：fake-ip、国内走阿里/腾讯 DoH、海外走 Google/Cloudflare DoH、关闭 IPv6
- 普通代理入口：HTTP 7890、SOCKS5 7891、混合代理 7893；认证在 OpenClash 页面管理，程序与同网段设备共用分流规则
- 自定义直连域名放在独立规则文件，改完推送 GitHub、更新 rule-provider 即生效

> `configs/rulesets/` 和 `client/rulesets/DIRECT.yaml` 里是**作者家庭网络的示例条目**，使用前请替换为你自己的域名、中继端口与 IP。

## 怎么配置使用

### 路由器版

1. 把 `configs/v8.yaml` 复制为 `configs/v8.local.yaml`（已被 .gitignore 忽略，不会上传）；
2. 在副本里填写：
   - `proxy-providers`：5 个通用槽位（AirportA / AirportB1 / AirportB2 / AirportC1 / AirportC2），把你的订阅 URL 填进 `YOUR_TOKEN` 位置；机场不足 5 个就删掉多余槽位，并同步移除锚点 `use:` 里的引用；
   - `secret`：控制面板（9090）访问密钥；
   - 使用 34.x 才需要：住宅代理的地址、端口、用户名、密码（`YOUR_RESIDENTIAL_*` 位置）；
3. 把 `rulesets/` 里的示例域名、IP 换成你自己的。
4. 在 OpenClash 页面设置 HTTP 7890、SOCKS5 7891、混合代理 7893，并启用账号密码认证。模板不包含真实代理凭据；直接运行 Mihomo 时，需要在私有副本中自行配置 `authentication`。

### 电脑 / Android 单机版

1. 把 `client/v8.yaml` 复制为私有文件；
2. 把订阅 URL 填进 5 个通用槽位（按需设置 `secret`）；
3. 导入 Clash Verge Rev、Clash Party、FlClash 等 Mihomo 客户端，开启系统代理或 TUN。

## 怎么部署（OpenClash）

**使用前提（先完成路由器侧的网络建设，本配置才能生效）：**

1. 在 OpenWrt 创建 4 个接口 + DHCP 网段：`192.168.31.x` / `32.x` / `33.x` / `34.x`；
2. 创建并绑定对应 Wi-Fi（如 `ap-direct` / `ap-rule` / `ap-Global` / `ap-34x`）；
3. 配置防火墙区域，确认各网段能正常上网；
4. 安装 OpenClash（`Master` 分支）+ 较新 Mihomo Meta 内核。

本配置只负责分流，不负责创建接口、Wi-Fi 和 DHCP。

1. **上传**：OpenClash 配置管理页面上传 `v8.local.yaml`（保存到 `/etc/openclash/config/`）。保留旧配置，出问题能一键切回；
2. **检查语法**：SSH 执行
   ```sh
   /etc/openclash/core/clash_meta -t -f /etc/openclash/config/v8.local.yaml
   ```
   看到 `configuration file ... test is successful` 才继续；
3. **启用**：在 OpenClash 中选择新配置启动；
4. **验证**：
   - 日志无规则下载错误，5 个机场 provider 都能显示节点，地区组不是空组；
   - 浏览器打开 `http://路由器IP:9090/ui`，用 `secret` 连接面板；
   - 32.x 设备国内外网站都正常；33.x 全局代理不影响 NAS、打印机等局域网设备；
   - 有条件时做一次断线演练，确认跨国接管生效。
   - 核对 OpenClash 实际加载配置中的端口与认证；从 32.x 设备分别测试 HTTP 7890、SOCKS5 7891、混合代理 7893，确认正确账号可连接、错误账号被拒绝。

### 程序如何使用普通代理入口

需要按平台分流的程序部署在 `192.168.32.x` 网段，代理地址填写路由器在该网段的 IP，HTTP 使用 7890，SOCKS5 使用 7891，或通过混合端口 7893 使用对应协议。账号密码使用 OpenClash 页面中启用的认证信息。

OpenClash 使用规则模式时，普通入口按连接的源 IP 分流：31.x 直连、32.x 按平台分流、33.x 使用全局出口、34.x 按住宅白名单分流。路由器地址或代理端口不会把 31.x 设备变成 32.x 设备；需要在面板连接详情中核对实际源 IP。

32.x 程序访问 ChatGPT、YouTube 等目标时使用现有平台策略组。最终节点属于哪个机场，就消耗哪个机场的流量；地区故障转移后实际机场可能变化。打开 Zashboard 的连接详情，查看命中规则、代理链和最终节点。连接成功只代表代理入口可用，目标网站是否可访问仍需实际验证。

## 日常使用

- 日常连 32.x 分流 Wi-Fi，只在面板里调整 ChatGPT、YouTube 等应用策略组；
- 稳定优先选带 ★ 的地区，视频大流量选不带 ★ 的省流地区；
- 临时全局代理连 33.x；排障先连 31.x 直连网确认问题是否与代理有关。

## 常见问题

- **OpenClash 启动失败**：先看运行日志，常见原因是订阅拉取失败、规则集 URL 不可达、端口冲突；
- **某机场没节点**：路由器上 `curl` 订阅 URL 确认可达、Token 未过期；订阅域名被墙时可让该 provider 借道已成功的机场拉取；
- **某国家组是空的**：节点名不匹配地区正则，检查真实节点名后调整 `filter`；
- **节点显示 0 ms**：多为 `lazy` 健康检查还没跑，不等于节点死了，看日志和实际连接判断。

## 安全提醒

- 真实订阅、Token、住宅代理凭据只写进 `*.local.yaml` 等被忽略的私有文件，不进公开仓库；
- `external-controller` 设置强 `secret`；7890/7891/7893/9090/1053 端口只允许可信内网访问；
- 不要把 LuCI、SSH、控制面板或代理入口暴露到公网；
- 升级前保留旧配置和 OpenClash 备份。

## 项目结构

```text
configs/            # 路由器配置：v8.yaml（当前）+ v7..v3（历史）+ rulesets/
client/             # 电脑/手机单机配置：v8.yaml + rulesets/
docs/               # REQUIREMENTS / DESIGN / TECHNICAL 详细文档
```

需要理解设计或继续扩展时，看 [需求与验收标准](docs/REQUIREMENTS.md)、[方案设计](docs/DESIGN.md)、[技术规范](docs/TECHNICAL.md)。

v8 路由器版含 62 个策略组、42 个 rule-provider、12 个地区、5 个机场 provider。部署前需完成内核语法检查及路由器上的端口、认证与分流验证。

## 开源与贡献

MIT License。欢迎通过 Issue / PR 提交新平台规则、节点正则改进、OpenClash 兼容经验和部署说明。提交前请删除所有私人凭据。
