# UC Davis VPN Split Tunnel Verification

Date: 2026-06-04

## Conclusion

This tool routes only IP traffic destined for UC Davis networks into the VPN. It creates the VPN virtual interface `utun6` and installs two split routes in the system routing table:

1. `128.120.0.0/16 -> interface utun6`
2. `169.237.0.0/16 -> interface utun6`

When a destination IP falls inside either network, the kernel selects the matching route by longest-prefix match and hands the packet to `utun6`. This VPN point-to-point interface does not require a separate gateway. Other traffic matches the system default route, leaves through the physical interface `en0`, and is forwarded to the next-hop gateway specified by that default route.

Therefore, this tool can run together with system proxy tools such as Clash or Shadowrocket: UC Davis internal traffic is handled by this VPN tool; YouTube traffic can be handled by Clash; Bilibili traffic that is not proxied by Clash follows the default route and leaves directly through `en0`.

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
VPN IP:       utun6 172.25.229.79
Health:      OK (tunnel)
Default route: 192.168.2.1 on en0 (guard active)
VPN gateway:  169.237.216.210 via 192.168.2.1 on en0
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
