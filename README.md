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
- 可选 HTTP 7891 入站：带账号密码，给爬虫/自动化任务用独立分流链
- 自定义直连域名放在独立规则文件，改完推送 GitHub、更新 rule-provider 即生效

> `configs/rulesets/` 和 `client/rulesets/DIRECT.yaml` 里是**作者家庭网络的示例条目**，使用前请替换为你自己的域名、中继端口与 IP。

## 怎么配置使用

### 路由器版

1. 把 `configs/v8.yaml` 复制为 `configs/v8.local.yaml`（已被 .gitignore 忽略，不会上传）；
2. 在副本里填写：
   - `proxy-providers`：5 个通用槽位（AirportA / AirportB1 / AirportB2 / AirportC1 / AirportC2），把你的订阅 URL 填进 `YOUR_TOKEN` 位置；机场不足 5 个就删掉多余槽位，并同步移除锚点 `use:` 里的引用；
   - `secret`：控制面板（9090）访问密钥；
   - `listeners`：HTTP 7891 三个用户密码（`★填写crawler密码★` 位置）；
   - 使用 34.x 才需要：住宅代理的地址、端口、用户名、密码（`YOUR_RESIDENTIAL_*` 位置）；
3. 把 `rulesets/` 里的示例域名、IP 换成你自己的。

### 电脑 / Android 单机版

1. 把 `client/v8.yaml` 复制为私有文件；
2. 把订阅 URL 填进 5 个通用槽位（按需设置 `secret`）；
3. 导入 Clash Verge Rev、Clash Party、FlClash 等 Mihomo 客户端，开启系统代理或 TUN。

## 怎么部署（OpenClash）

前提：路由器已装 OpenClash（`Master` 分支）+ 较新 Mihomo Meta 内核，并已创建 31/32/33/34 四个网段。

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
- `external-controller` 设置强 `secret`；7890/9090/1053/7891 端口只允许可信内网访问；
- 不要把 LuCI、SSH、控制面板或代理入口暴露到公网；
- 升级前保留旧配置和 OpenClash 备份。

## 项目结构

```text
configs/            # 路由器配置：v8.yaml（当前）+ v7..v3（历史）+ rulesets/
client/             # 电脑/手机单机配置：v8.yaml + rulesets/
docs/               # REQUIREMENTS / DESIGN / TECHNICAL 详细文档
```

需要理解设计或继续扩展时，看 [需求与验收标准](docs/REQUIREMENTS.md)、[方案设计](docs/DESIGN.md)、[技术规范](docs/TECHNICAL.md)。

v8 路由器版含 93 个策略组、51 个 rule-provider、12 个地区、5 个机场 provider，已在 Mihomo v1.19.28 上测试通过。

## 开源与贡献

MIT License。欢迎通过 Issue / PR 提交新平台规则、节点正则改进、OpenClash 兼容经验和部署说明。提交前请删除所有私人凭据。
