# UC Davis VPN Split Tunnel Verification

This document verifies two claims:

1. This tool routes only UC Davis internal network traffic through the VPN.
2. This tool can be used together with system proxy tools such as Clash or Shadowrocket.

Operational notes for proxy-tool coexistence:

- This VPN tool should own only UC Davis split routes.
- Proxy tools should own system proxy, TUN mode, and ordinary internet proxy rules.
- Local SOCKS helpers such as
  [proxyctl](https://github.com/LYK-love/proxyctl) can be used as one Clash
  node, but Clash is not limited to that local node. Any healthy public
  subscription node should also keep using the physical default route.
- In Clash/Mihomo fake-ip mode, `198.18.x.x` and `198.19.x.x` route results are
  proxy fake IPs, not real UC Davis VPN gateways.
- `ucdavis-vpnctl status` intentionally hides fake-ip `VPN gateway` routes while
  the VPN is off. Seeing no `VPN gateway:` line in that state is normal.

When Clash is enabled, application traffic first enters Clash through system
proxy or TUN mode. Clash then opens its own outbound connection to the selected
node. With this VPN tool's default split-tunnel configuration, that outbound
connection is routed by the normal system default route unless the selected
node's IP is inside a UC Davis split route. Therefore the selected node can be a
local SOCKS node, a Trojan/Vmess/Vless subscription node, or another healthy
public proxy node; the VPN should not change that path.

If only the local SOCKS node works and subscription nodes fail, update the
subscription and check node health before changing VPN settings. Stale domains,
expired provider records, and dead ports look similar in Clash but are not VPN
route failures.

Full versions:

- [中文版本](split-tunnel-proof.zh.md)
- [English version](split-tunnel-proof.en.md)
