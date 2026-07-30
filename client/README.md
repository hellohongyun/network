# 单设备配置

`client/` 维护桌面电脑和 Android 设备本机运行 Mihomo 时使用的配置。

- `v8.yaml`：当前单设备推荐模板。
- `rulesets/DIRECT.yaml`：当前设备的自定义直连目标。

配置直接接管运行客户端的设备，不包含家庭网段、住宅 IP 或 HTTP 7891 入站。使用前应复制 `v8.yaml` 为私有的 `v8.local.yaml`，填写 5 个机场订阅 URL，并在 Clash Verge Rev、Clash Party、FlClash 等较新的 Mihomo 客户端中启用系统代理或 TUN。

发布与使用边界：

- `v8.yaml` 中的 5 个 `https://example.com/...YOUR_TOKEN` 都必须替换为真实订阅。
- `rulesets/DIRECT.yaml` 通过 GitHub Raw URL 加载；仓库尚未推送该路径时会返回 404。
- 配置只监听本机，不向局域网共享代理，也不替代路由器版 `configs/v8.yaml`。
- iPhone/iPad 上的 Stash、Shadowrocket、Surge 等客户端并非完整 Mihomo 内核，本模板不承诺直接兼容。
- 当前模板包含 60 个策略组、40 个 rule-provider、12 个地区和 5 个机场 provider。
