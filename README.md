# Network：让家庭网络自动选择能用的出口

这是一套运行在 OpenWrt / ImmortalWrt 路由器上的 Mihomo（Clash Meta）配置方案。

它要解决的不是“让代理跑起来”，而是代理跑起来以后更常见的麻烦：

- 一个机场的美国节点坏了，需要手动换另一个机场；
- 所有美国节点都不可用，手机上又不方便切到日本；
- ChatGPT、Claude、YouTube 等服务适合的地区和线路不同；
- 全局代理浪费流量，国内网站变慢，完全直连又无法访问海外服务；
- 家里设备很多，不想每台设备都安装和维护代理客户端；
- 已经购买多个机场，但它们彼此独立，无法形成真正的故障接力。

本项目把机场、地区、平台和家庭网段组织成一套自动调度系统。平时你只需要选择“这个平台走哪个地区”，剩下的机场测速、同国切换和跨国兜底由 Mihomo 完成。

它能减少人工切换，但不能创造不存在的线路：如果所有机场都不可用，网络仍然无法通过代理恢复。

路由器推荐配置：[configs/v8.yaml](configs/v8.yaml)

桌面电脑或 Android 设备本机运行 Mihomo 时，使用 [client/v8.yaml](client/v8.yaml)。该版本适配 Clash Verge Rev、Clash Party（原 Mihomo Party）、FlClash 等较新的 Mihomo 客户端，直接接管当前设备流量，提供应用分流、地区故障转移、机场梯队和健康检查，不包含任何家庭网段、住宅 IP 与 HTTP 7891 入站。iPhone/iPad 客户端不保证兼容。

## 先看它是否适合你

适合：

- 路由器运行 OpenWrt / ImmortalWrt，并安装了 OpenClash；
- 希望手机、电脑、电视等设备无需安装代理客户端；
- 有多个机场，想让它们互相接管；
- 想按平台选择国家，而不是频繁选择具体节点；
- 愿意为不同用途规划多个 Wi-Fi 或网段；
- 希望把个人域名规则放在独立文件中长期维护。

暂时不适合：

- 只有一个普通订阅，只需要最简单的全局代理；
- 无法控制路由器的 Wi-Fi、DHCP 或防火墙；
- 希望直接上传模板，但不填写订阅、密码和网络信息；
- 不准备阅读 OpenClash 日志，也不愿在首次部署后做基本验证。

> v8 按 5 个机场和 4 个网段设计。它可以继续扩展，但减少机场、删除网段或更换策略结构需要同步调整相关引用。

### 桌面和 Android 客户端

客户端版本的使用顺序：

1. 将 `client/v8.yaml` 复制为私有文件；
2. 填写 5 个 `proxy-providers` 订阅 URL，并按需设置本机控制器 `secret`；
3. 在 Clash Verge Rev、Clash Party（原 Mihomo Party）、FlClash 或其他较新的 Mihomo 客户端中导入完整 YAML；
4. 在客户端中启用系统代理或 TUN。

该文件仅接管运行客户端的设备，不识别或依赖 `192.168.32.0/24` 等源网段。自定义直连目标维护在 `client/rulesets/DIRECT.yaml`。传统 Clash Premium、Clash for Windows，以及不使用 Mihomo 内核的软件不保证兼容。

## 它是怎么工作的

一次请求会经过四层选择：

```text
设备所在网段
    ↓
目标平台（ChatGPT / YouTube / GitHub / 其他网站）
    ↓
选择的地区（日本 / 美国 / 新加坡等）
    ↓
机场梯队与具体节点
```

你负责表达意图，例如“ChatGPT 走美国★”；配置负责找到当前可用的美国线路。如果美国所有机场都失效，则继续尝试其他国家。

### 同国切换，再跨国兜底

以美国稳定版为例：

```text
美国优质机场 A
    ↓ 不可用
美国主力机场 C
    ↓ 不可用
美国保底机场 B
    ↓ 美国全部不可用
日本 → 新加坡 → 台湾 → 马来西亚 → 韩国
    → 荷兰 → 英国 → 德国 → 法国 → 越南
```

跨国兜底不包含香港。香港仍然可以手动选择，但不会在其他国家失败后被自动选中。

v8 将跨国候选直接展开到主要国家的顶层策略组，不再增加独立的 `fallback → fallback` 层级。面板中因此没有单独的“跨国最终兜底”卡片。

### 机场分成三个梯队

| 梯队 | 默认组成 | 定位 |
| --- | --- | --- |
| A 优质 | YunTu | 价格较高、稳定优先，适合 AI 和重要业务 |
| C 主力 | CrossWall、DuoBaoYiYuan | 日常流量主力，兼顾速度和成本 |
| B 保底 | YiYuan、Kitty | 低成本备用线路 |

每个梯队使用 `url-test` 在所包含的节点中测速：C/B 梯队可以跨两个机场选择，A 梯队在单个优质机场的节点内选择。主要地区使用 `fallback` 在梯队之间接力。

- 省流版地区组：`C → B → A`
- 稳定版地区组（带 `★`）：`A → C → B`

机场名称只是模板中的逻辑标识。你可以换成自己的机场，但必须同步修改 `proxy-providers` 和各锚点的 `use` 列表。

## 主要能力

### 四种家庭网络

| 默认网段 | 示例 Wi-Fi | 行为 | 适合场景 |
| --- | --- | --- | --- |
| `192.168.31.0/24` | `ap-direct` | 全部直连 | 国内应用、游戏、故障排查 |
| `192.168.32.0/24` | `ap-rule` | 国内直连、海外按平台分流 | 日常主要网络 |
| `192.168.33.0/24` | `ap-Global` | 全部经过代理 | 临时全局代理 |
| `192.168.34.0/24` | `ap-34x` | 默认住宅代理，白名单直连 | 住宅出口、远程连接 |

这些 Wi-Fi 名称不是 Mihomo 自动创建的。你需要在 OpenWrt 中建立对应接口、DHCP 网段和无线网络。

如果你只使用 32.x，其他网段不会影响日常分流；但 34.x 的住宅出口在填写真实住宅代理前不可用。

需要特别注意：v8 对 31.x、33.x、34.x 设置了显式源网段入口，其余没有提前命中的来源会继续进入默认的平台分流链。32.x 是这条默认路径的主要使用者，但新建访客 VLAN 后不能假设它会自动隔离；必须通过 OpenWrt 防火墙限制访客网络，或为新网段增加明确规则。

### 按平台独立选择地区

32.x 为常用服务提供独立策略组：

| 类别 | 平台 |
| --- | --- |
| AI | Claude、ChatGPT、Gemini、DeepSeek |
| 视频与音乐 | YouTube、Netflix、TikTok、Spotify、Disney+ |
| 社交 | Telegram、X、Instagram、Facebook、Discord |
| 开发与生产力 | Google、GitHub、Cloudflare、Figma、Notion |
| 系统与办公 | Microsoft、Apple |
| 金融与游戏 | 加密货币、PayPal、Steam |

未单独列出的海外网站由海外兜底规则处理，国内域名和 IP 默认直连。

### 12 个可选地区

- 主要地区：香港、日本、新加坡、美国、台湾
- 次要地区：英国、德国、法国、韩国、马来西亚、荷兰、越南

主要地区拥有 A/C/B 三梯队；次要地区从全部机场中联合测速。

### DNS 分流与减少泄漏

- `fake-ip` 模式；
- 国内域名使用阿里、腾讯 DoH；
- 海外域名使用 Google、Cloudflare DoH；
- DNS 请求遵循代理规则；
- ARC DNS 缓存；
- 关闭 IPv6，避免未规划的 IPv6 流量绕过规则。

DNS 安全不只取决于这份 YAML。OpenWrt 的 DNS 劫持和防火墙必须正确配置；客户端自带 DoH、访客网络或额外启用的 IPv6 都可能绕过路由器规则。

### 可选的独立 HTTP 代理入口

端口 `7891` 提供带账号密码的 HTTP 代理入口，使用独立 `http-rules` 规则链，适合爬虫或自动化任务。它与 TUN/网段规则隔离，并为不同平台提供独立策略组。

不使用该功能也必须修改模板密码，或自行关闭 `listeners`。不要把带默认占位密码的 7891 端口暴露到公网。

### 自定义域名规则

个人直连域名维护在：

- [configs/rulesets/DIRECT-32.yaml](configs/rulesets/DIRECT-32.yaml)：32.x 自定义直连域名；
- [configs/rulesets/DIRECT-34-relays.yaml](configs/rulesets/DIRECT-34-relays.yaml)：34.x 端口/IP 直连白名单；
- [configs/rulesets/DIRECT-34.yaml](configs/rulesets/DIRECT-34.yaml)：34.x 域名直连白名单。

修改规则文件并推送到自己的 GitHub 仓库后，OpenClash 更新对应 rule-provider 即可，不必每次重传主配置。

## 已有四网段后的配置部署指南

下面的步骤从“路由器已经创建 31.x/32.x/33.x/34.x 接口、DHCP、Wi-Fi 和防火墙区域”开始。本仓库目前不提供 LuCI 四网段创建向导；完全没有配置过 OpenWrt 网络的用户，应先完成网段与无线网络规划，再部署 Mihomo 配置。

### 1. 准备环境

你需要：

- OpenWrt 或 ImmortalWrt 路由器；
- OpenClash `Master` 分支；
- 较新的 Mihomo Meta 内核；
- 5 个可用机场订阅，或者具备按技术文档调整 provider 与锚点引用的能力；
- 能够通过 LuCI 或 SSH 管理路由器。

v8 已使用 Mihomo v1.19.28 完成原生配置测试，也已在 OpenClash 提供的新版 Meta Alpha 内核上实际运行。

### 2. 下载并创建私有副本

下载本仓库后，将 `configs/v8.yaml` 复制为不会被 Git 跟踪的本地文件，例如：

```text
configs/v8.local.yaml
```

仓库的 `.gitignore` 已忽略 `*.local.yaml` 和 `*.secret.yaml`。不要直接在准备公开推送的配置中填写真实密钥。

### 3. 填写必须项

打开私有副本，至少检查以下内容：

1. `proxy-providers` 中的 5 个订阅 URL；
2. `secret`：Mihomo `external-controller` API 和 Zashboard/YACD 使用的访问密钥，它不是 LuCI/OpenClash 登录密码；
3. HTTP 7891 的三个用户密码；
4. 如果使用 34.x，填写住宅代理地址、端口、用户名和密码；
5. 如果 fork 了仓库，把自维护 rule-provider URL 改成你的 GitHub Raw 地址。

模板中的 `YOUR_TOKEN`、`YOUR_RESIDENTIAL_*` 和 `★填写...★` 都是占位符，不能当作真实配置使用。

### 4. 上传到 OpenClash

通过 OpenClash 的配置管理页面上传私有配置，通常会保存到：

```text
/etc/openclash/config/
```

不要覆盖当前能正常工作的配置。先保留旧配置，方便出现问题时快速切回。

### 5. 启用前检查语法

SSH 登录路由器，执行：

```sh
/etc/openclash/core/clash_meta -t -f /etc/openclash/config/v8.local.yaml
```

看到以下内容才继续：

```text
configuration file ... test is successful
```

实际文件名不同就替换命令中的路径。

### 6. 启用并完成首次验证

在 OpenClash 中选择新配置并启动，然后依次检查：

1. 运行日志没有配置解析或 rule-provider 下载错误；
2. 5 个 proxy-provider 能显示节点；
3. 日本、美国、新加坡、台湾等地区组不是空组；
4. 浏览器打开 `http://路由器IP:9090/ui`，使用配置中的 `secret` 连接控制器；
5. 32.x 设备能打开国内网站和海外网站；
6. 面板切换 ChatGPT、YouTube 等策略后，新连接使用所选地区；
7. 31.x、32.x、33.x 能按预期访问局域网服务；尤其确认 33.x 的全局代理没有影响 NAS、打印机等本地访问；
8. DNS 没有明显泄漏或解析失败；
9. 有条件时做一次可控的同国故障演练，验证跨国接管。

## 日常怎么用

- 日常设备连接 32.x 的规则分流 Wi-Fi；
- 在面板中只调整 ChatGPT、Claude、YouTube 等应用策略组；
- 优先稳定时选择带 `★` 的地区；
- 视频或大流量业务优先选择无 `★` 的省流地区；
- 机场子组通常不需要手动操作；
- 临时需要全部代理时连接 33.x；
- 排查问题时先连接 31.x，确认是否与代理有关。

故障转移会改变出口 IP，原有长连接可能断开。应用重新连接后会使用新的出口；部分严格绑定 IP 的网站可能要求重新登录或验证。

## 常见问题

### 配置测试成功，但 OpenClash 无法启动

先检查 OpenClash 运行日志。常见原因是订阅拉取失败、规则集 URL 不可访问、端口冲突，或者 TUN 路由与插件设置冲突。

### 某个机场没有节点

在路由器上测试订阅 URL 是否可访问，并检查 Token 是否过期。若订阅域名无法直连，可让该 provider 通过已经成功加载的机场拉取。

### 某个国家组是空的

机场节点名称可能不符合配置中的地区正则。检查真实节点名称，再调整对应 `filter`。

### 节点显示 `0 ms`

这可能只是 `lazy` 健康检查尚未执行，不等于节点一定死亡。结合 OpenClash 日志和实际连接判断。

### 为什么健康检查用 gstatic

它用于判断节点是否具备基础联网能力。它不承诺某个平台一定接受这个出口 IP，但能够处理本项目重点解决的“节点完全断网”场景。

### 为什么跨国兜底没有香港

这是项目的默认安全边界。香港可以手动使用，但不会在美国、日本等地区全部失效后被自动选中。

### OpenClash 为什么显示 Alpha 内核

OpenClash 的 Meta 内核更新通常提供 Mihomo Alpha，这是正常情况。日常建议使用 OpenClash `Master` 分支和界面提供的最新 Meta 内核；遇到内核崩溃、CPU 异常或兼容问题时再考虑回退。

## 安全提醒

- 不要向公开仓库提交真实订阅 URL、Token、住宅代理凭据或 HTTP 密码；
- 为 `external-controller` 设置强 `secret`；
- `7890` 混合代理、`9090` 控制器、`1053` DNS 和 `7891` HTTP 代理都只允许可信内网访问，并按需与访客网隔离；
- 不要把 LuCI、SSH、控制面板或代理入口直接暴露到公网；
- 上传新配置前保留旧配置和 OpenClash 备份；
- 发布配置前搜索 `token`、`password`、公网 IP 和域名，避免泄密。

## 项目结构

```text
.
├── configs/
│   ├── v8.yaml                    # 当前路由器推荐模板
│   ├── v7.yaml ... v3.yaml        # 路由器历史版本
│   └── rulesets/                  # 路由器网段规则
├── client/                        # 电脑/手机单设备配置
│   ├── v8.yaml                    # 当前单设备推荐模板
│   └── rulesets/                  # 单设备自定义规则
├── docs/
│   ├── REQUIREMENTS.md            # 需求与验收标准
│   ├── DESIGN.md                  # 架构和设计决策
│   ├── TECHNICAL.md               # 字段、规则与维护规范
│   └── archive/                   # 历史讨论资料
├── README.md
└── LICENSE
```

需要理解设计或继续扩展时：

- [需求与验收标准](docs/REQUIREMENTS.md)
- [方案设计](docs/DESIGN.md)
- [技术规范](docs/TECHNICAL.md)

## 当前版本

v8 的重点变化：

- 主要国家全部失效后自动跨国接管；
- 跨国兜底不包含香港；
- 新增马来西亚、荷兰、越南；
- 所有 gstatic 健康检查严格要求 HTTP 204；
- 地区 fallback 检查周期调整为 60 秒；
- 修复微博规则源；
- 适配新版 Mihomo 的 provider 客户端指纹配置；
- HTTP 规则链补齐 OneDrive。

路由器模板包含 93 个策略组、51 个 rule-provider、12 个地区和 5 个机场 provider。单设备模板保留 60 个策略组、40 个 rule-provider、12 个地区和相同的 5 个机场 provider。

## 开源与贡献

本项目使用 [MIT License](LICENSE)。

欢迎通过 Issue 或 Pull Request 提交：

- 新平台规则；
- 节点命名正则改进；
- OpenClash 兼容性经验；
- 故障转移测试结果；
- 面向新手的部署说明。

提交前请删除所有私人凭据。配置能够通过语法检查不代表适合所有家庭网络，涉及网段、DNS、TUN 和防火墙的变更应先理解再部署。
