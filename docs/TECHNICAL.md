# 技术规范文档

> 项目：Mihomo 多网段精细分流配置  
> 版本：v4  
> 最后更新：2026-04-10

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
| 路由规则 | https://wiki.metacubex.one/config/rules/ | RULE-SET/DOMAIN/GEOSITE/SRC-IP-CIDR 等 |
| 规则集 | https://wiki.metacubex.one/config/rule-providers/ | behavior/format/interval 等 |
| 代理集 | https://wiki.metacubex.one/config/proxy-providers/ | health-check/exclude-filter/override 等 |
| 入站配置 | https://wiki.metacubex.one/config/inbound/ | TUN/sniffer 等 |
| 完整示例 | https://github.com/MetaCubeX/mihomo/blob/Meta/docs/config.yaml | 官方示例配置 |

### 2.2 规则集仓库

| 仓库 | URL | 说明 |
|------|-----|------|
| MetaCubeX meta-rules-dat | https://github.com/MetaCubeX/meta-rules-dat/tree/meta | 主要规则源，提供 .mrs/.yaml/.list 三种格式 |
| blackmatrix7 ios_rule_script | https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Clash | 最全面的 Clash 规则库，数百个应用分类 |
| ACL4SSR | https://github.com/ACL4SSR/ACL4SSR/tree/master/Clash | 经典 Clash 规则模板 |
| wwqgtxx clash-rules | https://github.com/wwqgtxx/clash-rules | fakeip-filter 等特殊规则集 |

### 2.3 参考配置

| 名称 | URL | 说明 |
|------|-----|------|
| qichiyuhub DNS 防泄露配置 | https://raw.githubusercontent.com/qichiyuhub/rule/refs/heads/main/config/mihomo/configdns.yaml | DNS 配置最佳实践参考 |
| usbog232 精细分流模板 | https://raw.githubusercontent.com/usbog232/clashmetadingyue/main/new-meta.ini | subconverter INI 模板参考 |
| ACL4SSR 全量模板 | https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/config/ACL4SSR_Online_Full.ini | 经典全量分流 INI 模板 |

## 3. 规则集 URL 清单

### 3.1 MetaCubeX geosite（behavior: domain, format: mrs）

URL 模式：`https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/{name}.mrs`

| 配置名 | name | 验证状态 |
|--------|------|---------|
| openai_domain | openai | ✅ 200 |
| deepseek_domain | deepseek | ✅ 200 |
| ai_catchall | category-ai-!cn | ✅ 200 |
| youtube_domain | youtube | ✅ 200 |
| netflix_domain | netflix | ✅ 200 |
| tiktok_domain | tiktok | ✅ 200 |
| spotify_domain | spotify | ✅ 200 |
| disney_domain | disney | ✅ 200 |
| telegram_domain | telegram | ✅ 200 |
| twitter_domain | twitter | ✅ 200 |
| instagram_domain | instagram | ✅ 200 |
| facebook_domain | facebook | ✅ 200 |
| discord_domain | discord | ✅ 200 |
| google_domain | google | ✅ 200 |
| github_domain | github | ✅ 200 |
| cloudflare_domain | cloudflare | ✅ 200 |
| figma_domain | figma | ✅ 200 |
| notion_domain | notion | ✅ 200 |
| onedrive_domain | onedrive | ✅ 200 |
| microsoft_domain | microsoft | ✅ 200 |
| apple_domain | apple | ✅ 200 |
| steam_domain | steam | ✅ 200 |
| paypal_domain | paypal | ✅ 200 |
| private_domain | private | ✅ 200 |
| geolocation_not_cn | geolocation-!cn | ✅ 200 |
| cn_domain | cn | ✅ 200 |

### 3.2 MetaCubeX geoip（behavior: ipcidr, format: mrs）

URL 模式：`https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/{name}.mrs`

| 配置名 | name | 验证状态 |
|--------|------|---------|
| netflix_ip | netflix | ✅ 200 |
| telegram_ip | telegram | ✅ 200 |
| twitter_ip | twitter | ✅ 200 |
| facebook_ip | facebook | ✅ 200 |
| google_ip | google | ✅ 200 |
| cloudflare_ip | cloudflare | ✅ 200 |
| private_ip | private | ✅ 200 |
| cn_ip | cn | ✅ 200 |

特殊：`apple_ip` 使用 **geo-lite** 路径（完整版无此文件）：
`https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo-lite/geoip/apple.mrs`

### 3.3 blackmatrix7（behavior: classical）

| 配置名 | URL | 格式 | 验证状态 |
|--------|-----|------|---------|
| claude_domain | `.../Clash/Claude/Claude.yaml` | yaml | ✅ 200 |
| gemini_domain | `.../Clash/Gemini/Gemini.yaml` | yaml | ✅ 200 |
| crypto_domain | `.../Clash/Cryptocurrency/Cryptocurrency.list` | text | ✅ 200 |

### 3.4 已弃用/404 的 URL（避免使用）

| URL | 状态 | 替代方案 |
|-----|------|---------|
| `blackmatrix7/.../DeepSeek/DeepSeek.yaml` | ❌ 404 | MetaCubeX `geosite/deepseek.mrs` |
| `blackmatrix7/.../DeepSeek/DeepSeek.list` | ❌ 404 | 同上 |
| `MetaCubeX/geo/geoip/apple.mrs` | ❌ 404 | 使用 `geo-lite/geoip/apple.mrs` |

### 3.5 其他

| 配置名 | URL | 验证状态 |
|--------|-----|---------|
| fakeipfilter | `wwqgtxx/clash-rules/release/fakeip-filter.mrs` | ✅ 200 |

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

### 4.3 锚点使用规范

```yaml
# 定义锚点（放在第八节"代理组锚点"）
sub_ut_c: &sub_ut_c
  type: url-test
  use: [AirportC]
  url: https://www.gstatic.com/generate_204
  interval: 300

# 使用锚点（在 proxy-groups 中通过 <<: * 合并）
- name: "[C] 香港"
  <<: *sub_ut_c           # 继承所有字段
  filter: "(?=.*(港|HK))" # 添加/覆盖特定字段
```

### 4.4 rule-providers 锚点

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

### 4.5 正则表达式规范

节点过滤使用正则表达式，格式为正向+负向双重匹配：

```
(?=.*(港|HK|(?i)Hong))           ← 正向：必须包含这些关键字
^((?!(台|日|韩|新|美)).)*$       ← 负向：不能包含这些关键字
```

组合后：
```
(?=.*(港|HK|(?i)Hong))^((?!(台|日|韩|新|深|美|英|法|德|澳|韓)).)*$
```

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
proxy: AirportA   # 替代原来的 proxy: DIRECT
```

### 5.3 某地区子组为空

原因：该机场没有对应地区的节点，或 filter 正则过滤了所有节点。

验证：在面板中检查 `[C] 日本` 等子组是否有节点。如果为空，fallback 会自动跳过该子组。

### 5.4 健康检查通过但网站不通

原因：健康检查只测试 `gstatic.com/generate_204`，不代表所有网站都能访问。
解决：将默认出口的地区组从省流版（C→B→A）改为稳定版★（A→B→C），已在 v4 修复。

## 6. 版本变更日志

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
