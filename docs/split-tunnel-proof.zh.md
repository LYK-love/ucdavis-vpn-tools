# UC Davis VPN Split Tunnel Verification 中文版

日期：2026-06-04

## 结论

本工具只让发往 UC Davis 网段的 IP 流量进入 VPN。它创建 VPN 虚拟接口 `utun6`，并在系统路由表中添加两条 split routes：

1. `128.120.0.0/16 -> interface utun6`
2. `169.237.0.0/16 -> interface utun6`

当目标 IP 落在这两个网段内时，内核通过最长前缀匹配命中对应路由，并把包交给 `utun6`。此类 VPN 点对点接口不需要额外指定网关。其他流量会命中系统默认路由，由内核从物理接口 `en0` 发出，并交给默认路由指定的下一跳网关继续转发。

因此，本工具可以和系统代理类工具同时使用，例如 Clash、Shadowrocket：UC Davis 内网流量由本工具走 VPN；YouTube 等流量可由 Clash 代理；Bilibili 等未被 Clash 规则代理的流量则按默认路由直接从 `en0` 出口访问。

实际使用时建议保持职责边界清晰：本工具只负责 UC Davis VPN tunnel 和 UC Davis split routes；Clash、Surge、Shadowrocket 等代理工具负责 system proxy、TUN mode 和普通公网代理规则。本工具也可以和本地 SOCKS 工具一起使用，例如 [proxyctl](https://github.com/LYK-love/proxyctl)。`proxyctl` 可以作为 Clash 的一个本地 SOCKS 节点，但 Clash 不限于这个节点；任何健康的公网订阅节点也应继续走物理默认路由，不受 UC Davis VPN 影响。

当 Clash 开启 system proxy 或 TUN mode 时，应用流量先进入 Clash；随后 Clash 再向当前选中的上游节点发起连接。在本工具默认 split-tunnel 配置下，只要该上游节点的 IP 不属于 UC Davis split routes，Clash 到节点的连接就会走系统默认路由 `en0`，而不是 VPN 接口。因此，上游节点可以是本地 SOCKS、Trojan、Vmess、Vless 或其他健康的公网代理节点。如果只有本地 SOCKS 节点可用、订阅节点不可用，应先更新订阅并检查节点健康；过期域名、失效端口和服务商侧故障不属于 VPN 路由问题。

开启 Clash/Mihomo fake-ip DNS 时，`198.18.x.x` 或 `198.19.x.x` 是代理 fake IP，不是真实 UC Davis VPN gateway。VPN 关闭且 Clash fake-ip 开启时，`ucdavis-vpnctl status` 不显示 `VPN gateway:` 行是正常现象；如果它显示 `VPN gateway: 198.18.x.x via 198.18.0.1 on utunX`，通常说明安装的 daemon 太旧或 fake-ip 过滤没有生效。

## 配置

```zsh
cfg="/Library/Application Support/ucdavis-vpn-daemon/config.env"
egrep "^(SERVER|PRESERVE_DEFAULT_ROUTE|VPN_SPLIT_ROUTES|VPN_ROUTE_PING_TARGET|HEALTH_CHECK_MODE)=" "$cfg"
```

```text
SERVER=vpn.engineering.ucdavis.edu
HEALTH_CHECK_MODE=tunnel
PRESERVE_DEFAULT_ROUTE=1
VPN_SPLIT_ROUTES="169.237.0.0/16 128.120.0.0/16"
VPN_ROUTE_PING_TARGET=1
```

判定：

- `PRESERVE_DEFAULT_ROUTE=1`：默认路由保留在物理网络。
- `VPN_SPLIT_ROUTES`：只有 `169.237.0.0/16` 和 `128.120.0.0/16` 被固定到 VPN。

## VPN 状态

```zsh
ucdavis-vpnctl status
```

关键输出：

```text
VPN IP:       utun6 172.25.229.79
Health:      OK (tunnel)
Default route: 192.168.2.1 on en0 (guard active)
VPN gateway:  169.237.216.210 via 192.168.2.1 on en0
```

判定：

- VPN 接口是 `utun6`，VPN 内侧 IP 是 `172.25.229.79`。
- 默认路由是 `gateway 192.168.2.1, interface en0`。
- VPN 服务器自身走 `en0`，避免递归进入 VPN。

## 路由表摘要

```zsh
netstat -rn -f inet | awk 'NR <= 4 || $1 == "default" || $1 == "128.120" || $1 == "169.237" || $1 == "169.237.216.210"'
```

```text
Routing tables

Internet:
Destination        Gateway            Flags               Netif Expire
default            192.168.2.1        UGScg                 en0
128.120            utun6              USc                 utun6
169.237            utun6              USc                 utun6
169.237.216.210    192.168.2.1        UGHS                  en0
```

判定：

- `default -> gateway 192.168.2.1, interface en0`
- `128.120.0.0/16 -> interface utun6`
- `169.237.0.0/16 -> interface utun6`
- `169.237.216.210 -> gateway 192.168.2.1, interface en0`

## 目标路由验证

### 默认路由

```zsh
route -n get default
```

```text
route to: default
destination: default
gateway: 192.168.2.1
interface: en0
```

判定：默认流量走 `gateway 192.168.2.1, interface en0`。

### 普通公网

```zsh
route -n get 8.8.8.8
```

```text
route to: 8.8.8.8
destination: default
gateway: 192.168.2.1
interface: en0
```

判定：普通公网目标未命中 UC Davis split routes，走默认路由。

### UC Davis: 128.120.0.0/16

```zsh
route -n get 128.120.1.1
```

```text
route to: 128.120.1.1
destination: 128.120.0.0
mask: 255.255.0.0
interface: utun6
```

判定：目标 `128.120.1.1` 命中路由 `128.120.0.0/16 -> interface utun6`，走 VPN。

### UC Davis: 169.237.0.0/16

```zsh
route -n get 169.237.1.1
```

```text
route to: 169.237.1.1
destination: 169.237.0.0
mask: 255.255.0.0
interface: utun6
```

判定：目标 `169.237.1.1` 命中路由 `169.237.0.0/16 -> interface utun6`，走 VPN。

### VPN 服务器

```zsh
route -n get vpn.engineering.ucdavis.edu
```

```text
route to: 169.237.216.210
destination: 169.237.216.210
gateway: 192.168.2.1
interface: en0
```

判定：VPN 服务器走 `gateway 192.168.2.1, interface en0`。

## 非 UC Davis 网站对照

### YouTube

绕过系统代理时，路由查询结果：

```zsh
route -n get www.youtube.com
```

```text
route to: 69.171.235.22
destination: default
gateway: 192.168.2.1
interface: en0
```

判定：YouTube 目标不命中 UC Davis split routes，内核路由为 `default -> gateway 192.168.2.1, interface en0`。

绕过系统代理直连：

```zsh
curl --noproxy '*' -L -I --connect-timeout 10 --max-time 20 \
  -w '\nremote_ip=%{remote_ip}\nlocal_ip=%{local_ip}\nhttp_code=%{http_code}\n' \
  https://www.youtube.com/
```

```text
curl: (28) Failed to connect to www.youtube.com port 443 after 10004 ms: Timeout was reached

remote_ip=
local_ip=
http_code=000
```

判定：直连超时，说明当前网络不能直接访问 YouTube；这不表示走了 VPN，路由查询已显示直连路径是 `en0`。

显式使用 Clash 本地代理：

```zsh
curl -L -I --proxy http://127.0.0.1:7897 --connect-timeout 10 --max-time 30 \
  -w '\nremote_ip=%{remote_ip}\nlocal_ip=%{local_ip}\nhttp_code=%{http_code}\n' \
  https://www.youtube.com/
```

```text
HTTP/1.1 200 Connection established

HTTP/2 200

remote_ip=127.0.0.1
local_ip=127.0.0.1
http_code=200
```

判定：通过 Clash 代理时访问成功；`remote_ip=127.0.0.1` 表示 `curl` 连接的是本地 Clash 代理端口，不是直接连接 YouTube，也不是进入 UC Davis VPN。

### Bilibili

```zsh
curl --noproxy '*' -L -I --connect-timeout 10 --max-time 20 \
  -w '\nremote_ip=%{remote_ip}\nlocal_ip=%{local_ip}\nhttp_code=%{http_code}\n' \
  https://www.bilibili.com/
```

```text
HTTP/2 200
date: Thu, 04 Jun 2026 08:45:43 GMT
content-type: text/html; charset=utf-8

remote_ip=183.131.147.48
local_ip=192.168.2.3
http_code=200
```

```zsh
route -n get 183.131.147.48
```

```text
route to: 183.131.147.48
destination: 183.131.147.48
gateway: 192.168.2.1
interface: en0
```

判定：Bilibili 访问成功，实际远端 IP `183.131.147.48` 走 `gateway 192.168.2.1, interface en0`。

## 实现位置

```zsh
rg -n "PRESERVE_DEFAULT_ROUTE|VPN_SPLIT_ROUTES|route -n add -net|restore_default_route|ensure_vpn_split_routes" \
  ucdavis-vpn-launchdaemon/bin/ucdavis-vpn-root-daemon \
  ucdavis-vpn-launchdaemon/config.env.example
```

```text
ucdavis-vpn-launchdaemon/config.env.example:40:PRESERVE_DEFAULT_ROUTE=1
ucdavis-vpn-launchdaemon/config.env.example:42:VPN_SPLIT_ROUTES="169.237.0.0/16 128.120.0.0/16"
ucdavis-vpn-launchdaemon/bin/ucdavis-vpn-root-daemon:71:PRESERVE_DEFAULT_ROUTE="${PRESERVE_DEFAULT_ROUTE:-1}"
ucdavis-vpn-launchdaemon/bin/ucdavis-vpn-root-daemon:74:VPN_SPLIT_ROUTES="${VPN_SPLIT_ROUTES:-169.237.0.0/16 128.120.0.0/16}"
ucdavis-vpn-launchdaemon/bin/ucdavis-vpn-root-daemon:497:restore_default_route() {
ucdavis-vpn-launchdaemon/bin/ucdavis-vpn-root-daemon:579:  /sbin/route -n add -net "$route" -interface "$iface" >/dev/null 2>&1 ||
ucdavis-vpn-launchdaemon/bin/ucdavis-vpn-root-daemon:625:ensure_vpn_split_routes() {
```

判定：

- `restore_default_route` 保持默认路由在物理网络。
- `ensure_vpn_split_routes` 只安装 `VPN_SPLIT_ROUTES` 中的 VPN 路由。
- `route -n add -net "$route" -interface "$iface"` 将 split route 写入 macOS 路由表。
