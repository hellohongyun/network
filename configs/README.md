# 路由器配置

`configs/` 维护 OpenWrt / ImmortalWrt + OpenClash 使用的多网段配置。

- `v8.yaml`：当前路由器推荐模板。
- `v7.yaml` 至 `v3.yaml`：历史版本。
- `rulesets/`：路由器网段专用的自维护直连和住宅白名单规则（仓库内置为占位示例，发布/使用前替换为你自己的域名与 IP）。

使用前应复制 `v8.yaml` 为私有的 `v8.local.yaml`，填写机场订阅、控制器密钥、HTTP 入站密码及按需使用的住宅代理凭据。该配置依赖 OpenWrt 的接口、DHCP、Wi-Fi 和防火墙设置，不适合直接导入单设备客户端。

`configs/*.yaml` 与 `configs/rulesets/*` 已被部署配置和 GitHub Raw URL 引用，属于稳定公共路径。后续不得为了目录整理而移动或改名；单设备配置统一维护在仓库顶层 `client/`。
