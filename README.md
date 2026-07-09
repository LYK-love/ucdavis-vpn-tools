# UC Davis OpenConnect VPN Tools

Small macOS tools for keeping a UC Davis Engineering VPN connection alive with
OpenConnect.

## Projects

- `ucdavis-openconnect-vpn/`: user-level OpenConnect wrapper plus a Chrome/CDP
  helper that obtains the VPN web login cookie.
- `ucdavis-vpn-launchdaemon/`: root LaunchDaemon that monitors reachability and
  starts OpenConnect automatically.

The tools were tested against a VPN gateway that accepts Juniper Network Connect
style cookies via:

```zsh
openconnect --protocol=nc
```

Your school or department may use a different realm, URL, or policy. Treat the
defaults as examples and review the generated config before enabling any daemon.

## Clash / Proxy Tool Compatibility

This tool is designed as a split-tunnel VPN: only UC Davis internal routes are
sent through the VPN, while ordinary internet traffic stays on the physical
network route.

This means it is intended to run alongside Clash, Mihomo, Surge, Shadowrocket,
and local SOCKS helpers such as
[proxyctl](https://github.com/LYK-love/proxyctl). Proxy tools can continue to
own system proxy, TUN mode, DNS fake-ip behavior, and ordinary public-internet
proxy rules; this VPN tool should own only UC Davis split routes.

In the normal setup, Clash can use any healthy upstream node, including a
`proxyctl` local SOCKS node or ordinary subscription nodes. Those node
connections stay on the physical default route and should not be captured by the
UC Davis VPN.

See the technical verification:

- [Split tunnel verification](docs/split-tunnel-proof.md)
- [中文版本](docs/split-tunnel-proof.zh.md)
- [English version](docs/split-tunnel-proof.en.md)

Related notes:

- [Clash Proxy Tools](https://lyk-love.cn/2026/05/06/clash-proxy-tools/)
- [如何使用远程服务器作为本机的代理](https://lyk-love.cn/2026/07/08/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8%E8%BF%9C%E7%A8%8B%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%BD%9C%E4%B8%BA%E6%9C%AC%E6%9C%BA%E7%9A%84%E4%BB%A3%E7%90%86/)

## What Is Not Committed

This repository should not contain:

- VPN passwords
- Keychain exports
- Browser profiles
- VPN cookies
- Logs, pid files, or local config files

Passwords are read from macOS Keychain at runtime.

## Quick Start

For a first-time install, use the guided setup from the repository root:

```zsh
./setup.sh
```

The setup script asks for:

- UC Davis email
- VPN password, stored in macOS Keychain
- a simple health-check choice
- whether to start the LaunchDaemon now

It also installs missing Homebrew dependencies if you approve, creates the user
config, installs the root LaunchDaemon, and writes the same basic settings to
both places.
The installer copies the browser cookie helper into `/usr/local/libexec`, so
the daemon does not depend on the repository staying in your Documents folder.

After setup:

```zsh
ucdavis-vpnctl status
ucdavis-vpnctl connect
ucdavis-vpnctl disconnect
ucdavis-vpnctl on
ucdavis-vpnctl off
ucdavis-vpnctl set-password
```

## First Use

If `./setup.sh` starts the daemon, Chrome may open to the UC Davis/Microsoft
login flow. Complete the login and Duo approval in that browser window. After
the VPN connects, check:

```zsh
ucdavis-vpnctl status
```

If you skipped starting the daemon during setup, start it later with:

```zsh
sudo launchctl bootstrap system /Library/LaunchDaemons/local.ucdavis-openconnect-daemon.plist
ucdavis-vpnctl on
```

## Basic Commands

```zsh
ucdavis-vpnctl status        # show daemon, tunnel, health check, and cookie state
ucdavis-vpnctl connect       # connect or reconnect now
ucdavis-vpnctl disconnect    # drop the current tunnel; auto reconnect stays enabled
ucdavis-vpnctl off           # pause auto reconnect and log out the tunnel
ucdavis-vpnctl on            # resume auto reconnect and connect now
ucdavis-vpnctl set-password  # update the Keychain password
```

Use `off` when you intentionally do not want the VPN to reconnect, for example
on a network where VPN access is broken. Use `on` when you want the daemon to
resume normal monitoring.

## Config Files

There are two config files because the tools run in different security contexts:

- `~/.config/ucdavis-openconnect-vpn/config.env` is for the user-level helper
  and manual debugging commands.
- `/Library/Application Support/ucdavis-vpn-daemon/config.env` is for the root
  LaunchDaemon that keeps the VPN connected.

Most users do not need to edit either file during first install. `./setup.sh`
keeps the important values in sync. Edit them only for advanced settings such as
custom split routes, retry timing, browser profile paths, or multiple health
targets.

## Manual Install

Use the manual path only if you do not want the guided setup:

```zsh
brew install openconnect node
cd ucdavis-vpn-launchdaemon
sudo ./install.sh
ucdavis-vpnctl set-password
${EDITOR:-nano} "/Library/Application Support/ucdavis-vpn-daemon/config.env"
sudo launchctl bootstrap system /Library/LaunchDaemons/local.ucdavis-openconnect-daemon.plist
```

## Safety

These tools automate authentication and network routing. Before sharing or
publishing your fork, verify that no personal config, cookies, logs, or tokens
are staged:

```zsh
git status --short
rg -n 'your[_]email|g[h]p_|D[S]ID|D[S]SIGNIN|/User[s]/[^ ]+' .
```
