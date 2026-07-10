# UC Davis VPN Split Tunnel Verification

Date: 2026-06-04

## Conclusion

This tool routes only IP traffic destined for UC Davis networks into the VPN. It creates the VPN virtual interface `utun6` and installs two split routes in the system routing table:

1. `128.120.0.0/16 -> interface utun6`
2. `169.237.0.0/16 -> interface utun6`

When a destination IP falls inside either network, the kernel selects the matching route by longest-prefix match and hands the packet to `utun6`. This VPN point-to-point interface does not require a separate gateway. Other traffic matches the system default route, leaves through the physical interface `en0`, and is forwarded to the next-hop gateway specified by that default route.

Therefore, this tool can run together with system proxy tools such as Clash or Shadowrocket: UC Davis internal traffic is handled by this VPN tool; YouTube traffic can be handled by Clash; Bilibili traffic that is not proxied by Clash follows the default route and leaves directly through `en0`.

In day-to-day use, keep the ownership boundary clear: this tool owns only the UC Davis VPN tunnel and UC Davis split routes; Clash, Surge, Shadowrocket, or another proxy tool owns system proxy, TUN mode, and ordinary public-internet proxy rules. This tool can also run alongside local SOCKS helpers such as [proxyctl](https://github.com/LYK-love/proxyctl). `proxyctl` can be one local SOCKS node in Clash, but Clash is not limited to that node; any healthy public subscription node should also keep using the physical default route and should not be affected by the UC Davis VPN.

When Clash system proxy or TUN mode is enabled, application traffic first enters Clash. Clash then opens its own outbound connection to the currently selected upstream node. With this tool's default split-tunnel configuration, that outbound connection uses the system default route on `en0` unless the selected node's IP is inside a UC Davis split route. The upstream node can therefore be a local SOCKS node, Trojan, Vmess, Vless, or another healthy public proxy node. If only the local SOCKS node works and subscription nodes fail, update the subscription and check node health first; expired domains, dead ports, and provider-side failures are not VPN route failures.

When Clash/Mihomo fake-ip DNS is enabled, `198.18.x.x` or `198.19.x.x` is a proxy fake IP, not a real UC Davis VPN gateway. If the VPN is off and Clash fake-ip is on, it is normal for `ucdavis-vpnctl status` to omit the `VPN gateway:` line. If it shows `VPN gateway: 198.18.x.x via 198.18.0.1 on utunX`, the installed daemon is likely too old or fake-ip filtering is not active.

## Configuration

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

Verdict:

- `PRESERVE_DEFAULT_ROUTE=1`: keep the default route on the physical network.
- `VPN_SPLIT_ROUTES`: only `169.237.0.0/16` and `128.120.0.0/16` are pinned to the VPN.

## VPN State

```zsh
ucdavis-vpnctl status
```

Relevant output:

```text
UC Davis VPN: connected
Tunnel:       utun6 172.25.229.79
Check:        OK (tunnel)
Campus routes: OK (169.237.0.0/16 128.120.0.0/16 -> utun6)
Internet route: OK 192.168.2.1 on en0 (not VPN)
VPN server:     169.237.216.210 via 192.168.2.1 on en0
```

Verdict:

- VPN interface: `utun6`; VPN internal IP: `172.25.229.79`.
- Default route: `gateway 192.168.2.1, interface en0`.
- The VPN server itself uses `en0`, avoiding recursive VPN routing.

## Routing Table Summary

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

Verdict:

- `default -> gateway 192.168.2.1, interface en0`
- `128.120.0.0/16 -> interface utun6`
- `169.237.0.0/16 -> interface utun6`
- `169.237.216.210 -> gateway 192.168.2.1, interface en0`

## Destination Route Checks

### Default Route

```zsh
route -n get default
```

```text
route to: default
destination: default
gateway: 192.168.2.1
interface: en0
```

Verdict: default traffic uses `gateway 192.168.2.1, interface en0`.

### Public Internet

```zsh
route -n get 8.8.8.8
```

```text
route to: 8.8.8.8
destination: default
gateway: 192.168.2.1
interface: en0
```

Verdict: ordinary public internet traffic does not match UC Davis split routes and uses the default route.

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

Verdict: destination `128.120.1.1` matches `128.120.0.0/16 -> interface utun6` and uses the VPN.

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

Verdict: destination `169.237.1.1` matches `169.237.0.0/16 -> interface utun6` and uses the VPN.

### VPN Server

```zsh
route -n get vpn.engineering.ucdavis.edu
```

```text
route to: 169.237.216.210
destination: 169.237.216.210
gateway: 192.168.2.1
interface: en0
```

Verdict: the VPN server uses `gateway 192.168.2.1, interface en0`.

## Non-UC Davis Website Checks

### YouTube

Route lookup while bypassing the system proxy:

```zsh
route -n get www.youtube.com
```

```text
route to: 69.171.235.22
destination: default
gateway: 192.168.2.1
interface: en0
```

Verdict: the YouTube target does not match UC Davis split routes. Its kernel route is `default -> gateway 192.168.2.1, interface en0`.

Direct connection while bypassing the system proxy:

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

Verdict: direct YouTube access timed out on this network. This does not indicate VPN routing; the route lookup above shows the direct path is `en0`.

Explicitly using the local Clash proxy:

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

Verdict: YouTube succeeds through Clash. `remote_ip=127.0.0.1` means `curl` connected to the local Clash proxy port, not directly to YouTube and not through the UC Davis VPN.

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

Verdict: Bilibili is reachable. Its actual remote IP `183.131.147.48` uses `gateway 192.168.2.1, interface en0`.

## Implementation Points

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

Verdict:

- `restore_default_route` keeps the default route on the physical network.
- `ensure_vpn_split_routes` installs VPN routes only for `VPN_SPLIT_ROUTES`.
- `route -n add -net "$route" -interface "$iface"` writes split routes into the macOS routing table.
