# Agent Note: Host 配置的直接客户端地址栅栏

Status: implemented

[English](2026-08-28-configuration-client-address-fence.md) | 中文

## 问题

Web GUI 可以绑定所有网卡供 LAN 浏览器访问，而 Host settings 服务会读写部署配置。Host 与 Origin 信任可以授权 LAN authority，却不能识别浏览器的直接 TCP 对端。因此 Client UI 需要一个明确的配置能力；从页面 hostname 推导该能力会把服务器地址误当成浏览器地址，使 LAN settings 请求仍处于仅内存模式。

## 决策

`dsh-client-connection` 接受 `configurationClientAddresses`，将 IPv4 或 IPv6 字面量与 Node 的直接 `socket.remoteAddress` 匹配。loopback 对端始终允许。IPv4 映射的 IPv6 对端会归一化为 IPv4 字面量，IPv6 字面量按不区分大小写的方式比较；畸形或带空白的条目会让插件加载失败。绝不读取转发的地址 header。

Connection carrier 在浏览器认证之后，对 `/api/settings` 及其子路径执行此检查；不允许的对端返回 403。已认证的 frontend index 通过结构化 global 注入接收布尔能力。Client settings owner 只在该能力为 true 时选择 Host mirror，其他浏览器继续使用 memory mirror。该能力是访问提示而不是身份：所有 Host API 路由仍要求既有 Host/Origin 栅栏和签名浏览器会话。

本地 LAN overlay 将 GUI 绑定到 `0.0.0.0`，信任服务器的 LAN authority，并明确允许 `192.168.1.249` 作为配置客户端地址。不再保留旧的宽泛配置开关。

## 曾考虑的替代方案

**从浏览器页面 hostname 推导权限。** 否决：hostname 标识的是服务器 socket（报告部署中为 `192.168.1.100`），而不是 `192.168.1.249` 上的客户端。

**信任转发的客户端地址 header。** 否决：除非建立独立的代理约定，否则 header 可由调用方控制，不能用于授权 Host settings。

**把对端地址用作 API 身份。** 否决：socket 对端只提供狭窄的配置能力。浏览器会话认证仍是每个 Host API 方法与 stream 的身份。

**保留宽泛的 trusted-host 配置开关。** 否决：Host/Origin authority 授权不应悄悄向所有能声明该 authority 的客户端授予 settings 写权限。显式地址列表让额外权限在组合配置中可见。

## 后果

LAN 浏览器只有在配置了其直接地址时才能使用 Host settings；loopback 开发行为不变。网络变化或客户端地址变化需要修改 overlay 并重启 Web 进程。代理与端口转发部署需要单独审查对端地址策略；本决策不解释转发 header，也不增加代理身份支持。静态资源与非 settings API 路由继续使用既有可达性和浏览器会话规则。
