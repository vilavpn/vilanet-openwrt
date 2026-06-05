---
name: vilanet-openwrt
description: Drive vilanet on OpenWRT 24.10+ end-to-end — install IPKs via opkg, configure via UCI (uci set vilanet....), drive via ubus calls (ubus call vilanet status), use the LuCI app under Network → VPN → VilaNet, manage credentials in the AES-GCM envelope at /etc/vilanet/storage/, troubleshoot the rpcd shim. Use whenever the user mentions VilaNet OpenWrt, the router-side VPN, LuCI VilaNet app, opkg vilanet-core / luci-app-vilanet, UCI keys (network.dns_mode, hy2.speed_mode, routing_mode, block_porn, block_stun, block_dot, block_quic), or a ubus method (vilanet.status / list_servers / connect / etc).
---

# vilanet-openwrt — operator skill for AI agents

Homepage: <https://github.com/vilavpn/vilanet-openwrt>

`vilanet-openwrt` is the router-resident VilaNet VPN client for OpenWRT
24.10 and newer. One Go binary at `/usr/bin/vilanet` (statically linked,
sing-box compiled in as a library), one UCI config at `/etc/config/vilanet`,
one rpcd shim at `/usr/libexec/rpcd/vilanet`, and a LuCI JS app mounted at
`Network → VPN → VilaNet`. This skill teaches you (the AI) how to drive
the router-side client for an end user.

## When to use this skill

- The user mentions VilaNet on a router, OpenWRT, LuCI, opkg, or names
  the IPK packages `vilanet-core` / `luci-app-vilanet`.
- The user is SSHed into a router (e.g. `root@192.168.x.x`) and wants
  to connect, disconnect, switch servers, or check VPN status.
- The user reports a problem with the **Network → VPN → VilaNet**
  LuCI screens (Overview / Settings / Servers tabs).
- The user wants to inspect or change a UCI key (`uci set
  vilanet....`), call a ubus method (`ubus call vilanet ...`), or
  troubleshoot the rpcd shim.
- The user mentions the firewall-side LAN-sharing redirect, the
  AES-GCM credentials envelope, or the kill-switch.

## When NOT to use this skill

- VilaNet on Linux/macOS/Windows desktops — that's the **vilanet-cli**
  sibling client (different binary, different transport, different
  install surface).
- VilaNet on iOS, macOS GUI, tvOS, Android, or the Flutter desktop
  app — those are separate clients with their own GUIs and skills.
- Generic OpenWRT, sing-box, or VPN questions unrelated to VilaNet.
- Server-side VilaNet operations (panel admin, node provisioning,
  billing). Out of scope for this router client.

## Mental model

```
SSH root@router
 ├── /usr/bin/vilanet                      ← Go daemon + CLI in one binary
 │     ├── -daemon                         ← procd-managed long-running process
 │     └── subcommands: login, status,
 │           connect, disconnect, servers,
 │           packages, mode, config, diagnostics
 ├── /etc/init.d/vilanet                   ← procd init script; "/etc/init.d/vilanet
 │                                           start|stop|restart|enable|disable"
 ├── /etc/config/vilanet                   ← UCI config (uci show vilanet)
 ├── /etc/vilanet/                         ← runtime state directory (mode 0700)
 │     ├── device.secret                   ← HKDF root key — NEVER back up, NEVER copy
 │     ├── storage/                        ← AES-256-GCM creds envelope (see A4 below)
 │     └── .credentials                    ← legacy path; same envelope format
 ├── /usr/libexec/rpcd/vilanet             ← rpcd shim: ubus ↔ vilanet CLI bridge
 ├── /www/luci-static/resources/view/vilanet/  ← LuCI JS app (overview/settings/servers)
 ├── /usr/share/luci/menu.d/luci-app-vilanet.json   ← menu mount
 ├── /usr/share/rpcd/acl.d/luci-app-vilanet.json    ← rpcd ACL
 ├── /var/log/vilanet.log                  ← daemon log (mode 0600)
 ├── /var/run/vilanet.pid                  ← procd PID file
 └── /tmp/vilanet/                         ← scratch (cleared on reboot)
```

There is exactly **one** daemon instance, managed by procd. The
LuCI app and the rpcd shim DO NOT talk to the daemon directly — they
invoke `/usr/bin/vilanet <subcommand>` and parse the result, OR they
mutate `/etc/config/vilanet` via UCI and ask the init script to
`restart`. Three things to remember:

1. **UCI is the source of truth.** Every persistent setting lives in
   `/etc/config/vilanet`. The daemon reads it on start and reload.
2. **Credentials are encrypted at rest.** Plaintext passwords have
   not been on disk since the A4 hardening. The shell side **cannot**
   parse the envelope — it goes through the daemon's `--store-credentials`
   / `--read-credential-password` shell ABI verbs.
3. **The rpcd shim runs as root.** ubus methods are gated by an ACL
   (`/usr/share/rpcd/acl.d/luci-app-vilanet.json`); only authenticated
   LuCI sessions can call them.

## Install

`vilanet-openwrt` ships as **two IPK packages**:

- `vilanet-core` — the daemon binary, UCI defaults, init script, the
  rpcd shim. Architecture-specific.
- `luci-app-vilanet` — the LuCI JS views, menu mount, ACL. `all` arch
  (no native code). Depends on `vilanet-core`.

### Identify your router's opkg architecture

```sh
opkg print-architecture
```

Look for the most specific non-`all` arch (e.g. `mipsel_24kc`, `x86_64`,
`aarch64_cortex-a53`). The IPK filename embeds the same string.

### Officially built targets

| Arch label                 | Router examples                          |
|----------------------------|------------------------------------------|
| `x86_64`                   | x86-based OpenWRT (PC routers, VMs)      |
| `aarch64_cortex-a53`       | GL.iNet AX1800, NanoPi R4S, RPi 4        |
| `mipsel_24kc`              | Xiaomi/Netgear/TP-Link MIPS routers      |

For arches outside this table, the operator must build from source
with `make release` (see *Build from source* below). **vilanet-openwrt
is closed-source** — end users without source access install the
prebuilt IPKs only.

### Install on the router

```sh
# Copy IPKs to the router. CHOOSE THE ARCH THAT MATCHES YOUR OUTPUT
# FROM `opkg print-architecture` — installing a mismatched arch yields
# "exec format error" at install time.
scp -O bin/vilanet-core_1.0.0_x86_64.ipk     root@198.51.100.42:/tmp/
scp -O bin/luci-app-vilanet_1.0.0_all.ipk    root@198.51.100.42:/tmp/

ssh root@198.51.100.42 \
    'opkg install /tmp/vilanet-core_1.0.0_x86_64.ipk \
                  /tmp/luci-app-vilanet_1.0.0_all.ipk'
```

The `vilanet-core` postinst:

1. Creates `/var/lib/vilanet`, `/var/cache/vilanet`, `/var/run/vilanet`,
   `/tmp/vilanet`.
2. Tightens `/etc/vilanet` to mode 0700 and `/etc/vilanet/.credentials`
   to mode 0600.
3. Runs a one-shot UCI migration: drops retired sections from
   pre-v0.2 installs and clamps `global.log_level` to `warn` if it
   was set to a verbose level.
4. `/etc/init.d/vilanet enable` — the service is wired to start at
   boot but only actually starts when `global.enabled=1` AND
   `global.auto_connect=1`.

The `luci-app-vilanet` postinst clears `/tmp/luci-indexcache.*` and
reloads `rpcd` so the new ubus methods + menu entry appear without a
reboot. **Hard refresh the browser** to drop the LuCI module cache.

### Uninstall

```sh
opkg remove luci-app-vilanet vilanet-core
```

The `vilanet-core` postrm wipes `/var/lib/vilanet`, `/var/cache/vilanet`,
`/var/run/vilanet`, and `/tmp/vilanet`. It deliberately leaves
`/etc/vilanet/` (with `device.secret` and the encrypted credentials
envelope) in place so a reinstall picks the same encryption key back
up. To force a fresh-install state, also `rm -rf /etc/vilanet/`.

## Quick start (most common task)

```sh
# 1. SSH in
ssh root@198.51.100.42

# 2. Log in (interactive password prompt; never pass --password
#    literally in an AI session — let the user type it)
vilanet login --email you@example.com

# 3. Connect via country auto-select (recommended)
vilanet connect --country HK

# 4. Check what's running
vilanet status
```

The same flow via LuCI: open `http://198.51.100.42/cgi-bin/luci/admin/network/vilanet`,
land on the **Overview** tab, click **Sign in**, then on the **Servers**
tab click a server. The LuCI screens are just an alternate driver
that ends up writing the same UCI keys.

## Global flags

The router-side binary has no global flags — every subcommand owns its
own flag set. The cross-cutting environment variables are:

| Env var            | Meaning                                                                       |
|--------------------|-------------------------------------------------------------------------------|
| `VILANET_EMAIL`    | Account email. Used by the rpcd shim and the `-daemon` mode list ops so creds don't appear in `ps aux`. |
| `VILANET_PASSWORD` | Account password. Same rationale as above.                                    |
| `VILANET_DEBUG=1`  | Unlocks the internal `-generate-config` daemon debug flag. Off by default.    |

If both `--email`/`--password` flags AND environment variables are
provided, the flags win (and you get a warning on stderr that the
password is visible in `ps aux`).

## Command reference — the `vilanet` binary on the router

> Every command below was verified against `src/main.go` and
> `src/cli.go`. Flag names match exactly. Do not invent new flags.

### `vilanet help` / `vilanet -h` / `vilanet --help`

Prints the subcommand list. Use this first if you are unsure what is
installed.

### `vilanet version`

Prints `VilaNet OpenWRT Client v<version>` and exits 0.

### `vilanet login`

```
vilanet login [--email <addr>] [--password <pw>]
              [--package <id-or-uuid>] [--connect]
```

Stores credentials in `/etc/config/vilanet` (email only) and in the
AES-GCM envelope under `/etc/vilanet/storage/` (password). With
`--connect=true` (the default) it also restarts the daemon so the new
credentials take effect immediately.

Omit `--password` to be prompted (preferred). Never pass `--password`
literally in an AI session — quote the flag to the user and let them
type it. The CLI prints a warning if `--password` is used because the
plaintext is visible in `ps aux` for the lifetime of the call.

### `vilanet logout`

Clears the UCI email/password fields, removes
`/var/etc/vilanet.json`, removes `/etc/vilanet/.credentials`, removes
`/etc/vilanet/storage/`, and stops the daemon. Idempotent — exits 0
even when no daemon is running.

### `vilanet status`

```
vilanet status [--json]
```

Reads UCI and `pgrep`s for the daemon. Prints:

```
Service: running        (or "stopped")
Connection: connected   (or "disconnected" / "connecting")
Enabled: yes            (UCI global.enabled)
Server: <selected_server>
Package: <selected_package>
Mode: <connection_mode>
```

With `--json`, prints the same fields as a flat object — use this when
chaining commands. **`connection_state` derivation**: `running &&
enabled → "connected"`; `!running && enabled → "connecting"`;
otherwise `"disconnected"`. The status check does NOT query sing-box;
it only checks `pgrep -f /usr/bin/vilanet.*-daemon`.

### `vilanet connect`

```
vilanet connect [<server>]
vilanet connect --server <id-or-name>
vilanet connect --country <ISO2>          # e.g. HK, JP, US
```

`--country` synthesises `auto_<ISO2>` as the selected_server, which the
core generator interprets as "auto-select the best node in that
country". `--country` and a positional/`--server` argument are
**mutually exclusive** — passing both is a hard error.

Connect requires that login has run first (UCI auth.email is
populated). If you call connect without credentials the CLI prints
`not logged in. Run 'vilanet login' first`.

Internally it: (1) writes `global.enabled=1` and `selected_server` to
UCI, (2) runs `/etc/init.d/vilanet stop`, (3) sleeps 500 ms,
(4) runs `/etc/init.d/vilanet start`. Then it prints
`VPN connection initiated.` and tails into `status` to surface the new
state.

### `vilanet disconnect`

Sets `global.enabled=0` in UCI and runs `/etc/init.d/vilanet stop`.
Idempotent — exits 0 even if the daemon was already down. Always
safe.

### `vilanet servers`

```
vilanet servers [--package <id-or-uuid>]
```

Authenticates against the API using cached credentials and prints a
JSON array of `PublicNode` DTOs:

```json
[
  { "id": "<derived-hex>", "name": "hk-01", "type": "hysteria2",
    "country": "HK", "package_uuid": "<pkg-uuid>" }
]
```

Address / port / metadata are NEVER emitted — the entry-IP concealment
pipeline rewrites every node through `auth.ToPublicNode` so raw IPs
stay off stdout. The legacy `--raw` flag is accepted for
backwards-compatibility but silently ignored; output is always
sanitized.

`--package` filters by the package ID *or* UUID returned by `vilanet
packages`. Without it, all packages' nodes are returned, deduplicated
by `PublicNode.ID`.

### `vilanet packages`

Prints a JSON array of `PublicPackage` DTOs (one entry per active
package). Same redaction rules as `servers`.

### `vilanet mode`

```
vilanet mode                  # print current mode
vilanet mode global           # route everything through the tunnel
vilanet mode rule             # China bypass via sing-box ruleset
vilanet mode direct           # no tunnel (effectively pause)
```

Setting a new mode rewrites `global.connection_mode` and, if the
daemon is currently running, calls `/etc/init.d/vilanet restart`.

### `vilanet config`

```
vilanet config show
vilanet config get <key>
vilanet config set <key> <value>
vilanet config show-singbox
```

`show` prints a redacted JSON dump of the current UCI as a
`PublicConfig` (no password — not even as `[redacted]`; the field
simply doesn't exist on the DTO).

`get` / `set` operate on the dotted-key surface listed in
*Config key reference* below. `set` rejects mutating `auth.email` /
`auth.password` — those go through `vilanet login` so the email and
the encrypted-envelope password update atomically.

`show-singbox` is the operator's window into what the daemon would
actually feed sing-box. It tries three sources in order:

1. **live regenerate** — invokes the same `APIProvider` path the
   daemon uses, with current UCI. Requires cached creds + network.
2. **`/var/etc/vilanet.json`** — what the running daemon last wrote.
3. **encrypted obfuscator cache** — the most recent successful build.

The source name is printed to stderr (`# source: live-regenerate
(current UCI)`); the JSON itself goes to stdout. Redaction strips URLs
and IP literals but keeps UUIDs/passwords intact so the harness can
hash the output (config_sha256 invariant).

### `vilanet diagnostics` (alias: `vilanet diag`)

Writes a per-invocation directory under `/var/lib/vilanet/diag/`
(mode 0700, files mode 0600) containing the recent log tail, a
redacted config dump (`config-redacted.txt`), and the list of
currently-known PublicNode IDs. Passwords/tokens are masked; the
account email is partially obscured. Older bundles auto-remove after
24 h. Use this when asking the user to send something to support.

## Internal shell ABI (NOT for end users)

These verbs are dispatched by the rpcd shim and the init script and
NEVER advertised in `vilanet help`. AI agents should know they exist
so they recognize the calls in logs / process listings, but should not
recommend them to users — they are subject to internal rework.

```
vilanet --has-credentials <email>          # exit 0 iff envelope decrypts and email matches
vilanet --read-credential-email            # print stored email (no trailing newline)
vilanet --read-credential-password         # print stored password (no trailing newline)
vilanet --store-credentials                # read 'email:password\n' from stdin, write envelope
vilanet --clear-credentials                # truncate envelope to zero bytes
```

`--store-credentials` reads stdin (NEVER argv) so the password never
appears in `ps aux`. `--read-credential-password` is privileged: only
`/usr/libexec/rpcd/vilanet` and `/etc/init.d/vilanet` should call it,
and they propagate the value to the daemon via `VILANET_PASSWORD` env,
again never argv.

## Daemon flags (`vilanet -daemon` mode)

The daemon is normally launched by procd via `/etc/init.d/vilanet`,
NOT directly by hand. Flags are listed here for diagnostic invocations
only.

| Flag                | Default                  | Meaning                                                                |
|---------------------|--------------------------|------------------------------------------------------------------------|
| `-daemon`           | `false`                  | Run as the long-running service. procd sets this.                      |
| `-config <path>`    | `/var/etc/vilanet.json`  | Legacy file-based config. Deprecated; default is now in-memory.        |
| `-log <path>`       | `/var/log/vilanet.log`   | Log file. Forced to mode 0600 on every start.                          |
| `-version`          | —                        | Print version and exit.                                                |
| `-list-servers`     | —                        | Print PublicNode JSON and exit.                                        |
| `-list-packages`    | —                        | Print PublicPackage JSON and exit.                                     |
| `-package <id>`     | —                        | Filter for `-list-servers`.                                            |
| `-email <addr>`     | env `VILANET_EMAIL`      | Account email (prefer env to avoid `ps aux` leak).                     |
| `-password <pw>`    | env `VILANET_PASSWORD`   | Account password (prefer env).                                         |
| `-raw`              | `false`                  | Deprecated; output is always sanitized via PublicNode/PublicPackage.   |

The `-generate-config` flag exists but is gated on `VILANET_DEBUG=1`
and intentionally NOT in the help listing. Do not surface it to users.

## Config key reference — `vilanet config get / set`

These are the dotted keys understood by the CLI's `config get` / `config
set`. The list is **authoritative** (read from `src/cli.go`'s
`configGetValue` / `configSetValue`).

| Key                         | Type                          | Notes                                                                                  |
|-----------------------------|-------------------------------|----------------------------------------------------------------------------------------|
| `auth.email`                | string (read-only via set)    | Set via `vilanet login` — never `config set`.                                          |
| `auth.password`             | string (read-only via set)    | `get` returns `********` placeholder if any password is stored; otherwise empty.       |
| `global.enabled`            | bool (`1`/`0`)                | Whether procd should keep the daemon up.                                               |
| `global.selected_server`    | string                        | Node id, name, or `auto_<ISO2>` for country-scoped auto.                               |
| `global.selected_package`   | string                        | Package id or UUID.                                                                    |
| `global.connection_mode`    | `global` \| `rule` \| `direct`| `rule` = China-bypass via sing-box ruleset.                                            |
| `network.domain_strategy`   | `ipv4_only` \| `prefer_ipv4` \| `prefer_ipv6` | DNS lookup preference. Default: `ipv4_only`.                            |
| `network.dns_remote`        | string                        | Tunnel-side DNS (DoH URL or plain IP). Default: `https://1.0.0.1/dns-query`.           |
| `network.dns_local`         | string                        | Local-side DNS. Default: `223.5.5.5`.                                                  |
| `network.bypass_china`      | bool                          | GeoIP/GeoSite-based China bypass.                                                      |
| `network.bypass_lan`        | bool                          | Skip RFC 1918 destinations.                                                            |
| `network.sniff_enabled`     | bool                          | sing-box inbound sniff.                                                                |
| `network.mux_enabled`       | bool                          | Connection multiplexing.                                                               |
| `network.tun_enabled`       | bool                          | Bring up the TUN inbound.                                                              |
| `network.dns_mode`          | `fakeip` \| `real`            | Fake-IP vs. real DNS resolution.                                                       |
| `network.block_ads`         | bool                          | Apply ad-block ruleset.                                                                |
| `network.block_porn`        | bool                          | Apply adult-content ruleset.                                                           |
| `network.block_dot`         | tri-state                     | `true` / `false` / `default` (or `auto` / empty). Block DNS-over-TLS.                  |
| `network.block_quic`        | tri-state                     | Block UDP-443 / HTTP/3.                                                                |
| `network.block_stun`        | tri-state                     | Block WebRTC STUN.                                                                    |
| `proxy.enabled`             | bool                          | Mixed LAN-sharing inbound.                                                             |
| `proxy.port`                | string (port number)          | Mixed inbound port. Default: `10081`.                                                  |

## UCI surface

The exact same data lives at `/etc/config/vilanet`. Use `uci show
vilanet` to dump the whole tree. The sections shipped by the package
defaults are:

```
config vilanet 'global'                # enabled, auto_connect, auto_reconnect,
                                        # selected_server, selected_package,
                                        # connection_mode, log_level
config credentials 'auth'              # email (password lives in encrypted envelope)
config network 'settings'              # domain_strategy, dns_remote, dns_local,
                                        # bypass_china, bypass_lan, enable_ipv6, mtu
config proxy 'lan_sharing'             # enabled, port, auth_enabled,
                                        # auth_user, auth_pass, allowed_ips
config kill_switch 'kill_switch'       # enabled (G6)
config hy2 'hy2'                       # speed_mode (off|server_default|custom),
                                        # custom_mbps
config domains 'domains'               # list entry '<domain> bypass|proxy|block'
config clash_api 'clash_api'           # enabled, port, secret
```

**Notes**:

- `proxy.lan_sharing.allowed_ips` is **informational** — the firewall
  rules in `setup_firewall()` do not enforce a CIDR allowlist yet. Do
  not promise this works.
- `clash_api.enabled` is `0` by default (the router product is
  headless). Turning it on exposes the clash-mode switch + traffic
  API on `127.0.0.1:<port>`.
- `hy2.speed_mode` defaults to `off` (B7 alignment) — sing-box uses
  Hysteria2's BBR rather than the package's advertised cap. Flip to
  `server_default` to respect the package limit, or `custom` + set
  `hy2.custom_mbps` to force a specific rate in both directions.

## Ubus method reference

LuCI calls these via `/usr/libexec/rpcd/vilanet`. Use them directly
from the SSH shell when LuCI is unavailable:

```sh
ubus call vilanet status
ubus call vilanet list_packages
ubus call vilanet list_servers '{"package":"<pkg-id>"}'
ubus call vilanet get_config
ubus call vilanet get_credentials
ubus call vilanet get_mode
ubus call vilanet connect '{"server":"hk-01","package":"<pkg-id>"}'
ubus call vilanet disconnect
ubus call vilanet set_mode '{"mode":"rule"}'
ubus call vilanet update_config '{"section":"settings","option":"dns_remote","value":"https://1.0.0.1/dns-query"}'
ubus call vilanet set_credentials '{"email":"you@example.com","password":"<pw>"}'
ubus call vilanet clear_credentials
```

### Response shapes (verified against the rpcd shim)

`vilanet.status`:
```json
{ "service": "running", "connection": "connected",
  "selected_server": "hk-01", "selected_package": "<pkg-id>",
  "mode": "rule", "vpn_enabled": true }
```

`vilanet.get_credentials`:
```json
{ "email": "you@example.com", "logged_in": true }
```

`vilanet.list_packages`:
```json
{ "packages": "[ ... PublicPackage[] as JSON string ... ]" }
```

`vilanet.list_servers`:
```json
{ "servers": "[ ... PublicNode[] as JSON string ... ]" }
```

> ubus returns objects, not arrays; the shim wraps the binary's
> `PublicNode[]` / `PublicPackage[]` JSON in a one-key object. LuCI
> parses the inner string via `JSON.parse`.

`vilanet.connect` (success):
```json
{ "success": true, "status": "connected" }
```

The shim polls `/usr/bin/vilanet status --json` up to 20 times at
1-second intervals; if `connection_state` never reaches `"connected"`,
it rolls back `global.enabled=0` and returns
`{ "error": "Connection attempt did not succeed. ...", "kind": "unknown" }`.

### update_config allowlist

`vilanet.update_config` writes through a **static allowlist** of
(section, option) pairs (kept in lockstep with the LuCI widgets):

```
global.{auto_connect, auto_reconnect, log_level}
auth.email
kill_switch.enabled
settings.{domain_strategy, dns_remote, dns_local}
settings.{bypass_china, bypass_lan, enable_ipv6, mtu}
lan_sharing.{enabled, port, auth_enabled, auth_user, auth_pass}
domains.entry
hy2.{speed_mode, custom_mbps}
```

Anything else returns
`{ "error": "writes_not_allowlisted", "section": "...", "option": "..." }`.
If you need to write a key outside this list (e.g.
`global.selected_server`), use `uci set vilanet.<section>.<option>=...`
+ `uci commit` over SSH, then `/etc/init.d/vilanet restart`.

`global.log_level` is further constrained to `error|warn` —
`info`/`debug`/`trace` are rejected so config details never reach the
log file.

## On-disk file reference

| Path                                 | Purpose                                                                                          | Permission |
|--------------------------------------|--------------------------------------------------------------------------------------------------|------------|
| `/usr/bin/vilanet`                   | Daemon + CLI (one binary, single source).                                                        | 0755       |
| `/etc/config/vilanet`                | UCI config (source of truth for persistent settings).                                            | 0644       |
| `/etc/init.d/vilanet`                | procd init script.                                                                               | 0755       |
| `/etc/hotplug.d/iface/30-vilanet`    | Network-change hotplug hook.                                                                     | 0755       |
| `/etc/vilanet/`                      | Runtime state directory.                                                                         | 0700       |
| `/etc/vilanet/device.secret`         | **CRITICAL**: HKDF root key for the credentials envelope. Never back up. Never copy off-router. | 0600       |
| `/etc/vilanet/.credentials`          | Legacy path for the AES-GCM v1 envelope.                                                         | 0600       |
| `/etc/vilanet/storage/`              | AES-256-GCM credentials envelope (current path).                                                 | 0600       |
| `/usr/libexec/rpcd/vilanet`          | rpcd shim. Bridges ubus → vilanet CLI.                                                           | 0755       |
| `/www/luci-static/resources/view/vilanet/` | LuCI JS views: `overview.js`, `settings.js`, `servers.js`.                                 | 0644       |
| `/usr/share/luci/menu.d/luci-app-vilanet.json` | LuCI menu mount.                                                                       | 0644       |
| `/usr/share/rpcd/acl.d/luci-app-vilanet.json`  | rpcd ACL grants for the `luci-app-vilanet` role.                                       | 0644       |
| `/var/log/vilanet.log`               | Daemon log. Forced to 0600 on every start (chmod on existing).                                   | 0600       |
| `/var/run/vilanet.pid`               | procd PID file.                                                                                  | 0644       |
| `/var/etc/vilanet.json`              | Live sing-box config the daemon last wrote (legacy fallback path).                               | 0600       |
| `/tmp/vilanet/`                      | Scratch (cleared on reboot).                                                                     | 0700       |

> **device.secret is the keystone.** It is the HKDF root from which
> the AES key for the credentials envelope is derived. If a user
> restores `/etc/vilanet/.credentials` from one router to another,
> decryption will fail because `device.secret` doesn't match. There
> is no recovery path — recommend `vilanet logout` + `vilanet login`
> instead of envelope migration.

## Exit codes

The router-side `vilanet` is less exit-code-rich than `vilanet-cli`.

| Code | Meaning                                                                                                        |
|------|----------------------------------------------------------------------------------------------------------------|
| 0    | Success.                                                                                                       |
| 1    | Generic error (auth failure, network failure, UCI failure, bad flag, missing dependency, etc.). Surface stderr.|
| 2    | Internal credstore verb usage error (e.g. `--has-credentials` called without an email arg).                    |

Always surface `stderr` verbatim — the daemon and CLI emit a one-line
human-readable message that classifies most failures (e.g.
`not logged in. Run 'vilanet login' first`, `failed to load
configuration: ...`).

## Troubleshooting playbook

Diagnose top-down. **Always** start with `vilanet status --json` and
`logread -e vilanet` — they're cheap and tell you whether the daemon
is alive and what its last log line was.

### LuCI Overview tab always shows the sign-in form even after `vilanet login`

**Root cause** (W1 hardening regression class): the rpcd shim's
`handle_get_credentials` returns `logged_in: false`, so the LuCI
Overview re-renders the sign-in form. Two specific surfaces caused
this historically:

1. The shim calls `vilanet --has-credentials "$email"` and expects
   exit 0 + clean stdout. Any daemon stderr that gets mixed in via
   `2>&1` corrupts the response. **Fix**: the production shim uses
   `2>/dev/null` for the check; if you see `2>&1` instead, restore
   `2>/dev/null` (see the rpcd shim, around `handle_get_credentials`
   and `get_secure_password_for_email`).
2. Daemon-vs-plaintext-file mismatch: pre-A4 routers had an awk-parsed
   plaintext `.credentials`. After A4, the file is AES-GCM and only
   `vilanet --has-credentials` / `vilanet --read-credential-password`
   can decrypt it. Mismatched shim and daemon versions yield this
   exact symptom.

**Quick diagnosis**:
```sh
EMAIL=$(uci -q get vilanet.auth.email)
/usr/bin/vilanet --has-credentials "$EMAIL"  # expect exit 0
echo "exit=$?"
ubus call vilanet get_credentials             # expect logged_in:true
```

If `--has-credentials` exits 0 but ubus says `logged_in: false`,
the shim is wrong (re-deploy the matching `luci-app-vilanet` IPK or
fix the `2>&1` → `2>/dev/null` regression). If `--has-credentials`
itself exits 1, the envelope is missing — run `vilanet login` again.

### LuCI Servers tab is empty / "Failed to list servers"

**Root cause**: same family as the credentials problem above —
daemon stderr leaking into the JSON capture via `2>&1`, OR the
daemon itself failed to authenticate. The shim now extracts a
classified error and returns one of:

```
{ "error": "...", "kind": "auth_failed",     "rc": 1 }   # bad creds
{ "error": "...", "kind": "network_error",   "rc": 1 }   # no upstream
{ "error": "...", "kind": "no_packages",     "rc": 1 }   # account has none
{ "error": "...", "kind": "package_not_found", "rc": 1 } # bad -package filter
{ "error": "...", "kind": "unknown",         "rc": 1 }
```

Surface `kind` to the user; the human-readable `error` is already
URL/IP-redacted at the shim boundary.

### `opkg install` fails with `pkg_init_from_file: Malformed package file` / "incompatible archive format"

**Root cause** (A3 fix): opkg shipped with OpenWRT 24.10.2 expects
**IPK 1.0** layout — a gzipped tar containing `debian-binary`,
`control.tar.gz`, `data.tar.gz`. Older builds produced a `.deb`-style
ar archive that the new opkg rejects.

**Fix**: rebuild with `./build.sh` from the official repo (the
inner tars use `--format=ustar` and the outer wrapper uses
`tar -czf`, not `ar`). If you're given an older `.ipk`, ask for a
fresh build instead of trying to repackage by hand.

### `vilanet connect` exits with `not logged in. Run 'vilanet login' first`

`uci -q get vilanet.auth.email` is empty. Run `vilanet login` to
populate it AND write the encrypted-envelope password.

### `vilanet connect` returns but `status` says `disconnected`

Inspect `/var/log/vilanet.log`:

```sh
tail -n 200 /var/log/vilanet.log
```

Common patterns:

- `Failed to start service: provider load failed: authentication
  failed: ...` — bad creds; re-run `vilanet login`.
- `Failed to start service: provider load failed: ...no such host...`
  — network unreachable; verify the router has WAN connectivity
  (`ping 1.1.1.1` for reachability, then a name lookup for DNS).
- `failed to start VPN: ...permission denied...` — `kmod-tun` not
  loaded; `opkg install kmod-tun ip-full` and reboot.

### Service won't start at boot

The init script's `boot()` only starts the service when **both**
`vilanet.global.enabled='1'` AND `vilanet.global.auto_connect='1'`.
By default `auto_connect` is `0`. To enable autoconnect:

```sh
uci set vilanet.global.auto_connect=1
uci commit vilanet
```

### Kill switch behavior

`vilanet.kill_switch.enabled='1'` opts the daemon into the G6 kill
switch — when the tunnel goes down, LAN traffic is restricted to the
daemon's internally-maintained maintenance allow-list. Enabling the
kill switch **without TUN mode active** will interrupt LAN internet
during service restarts — warn the user before flipping it on.

### LAN sharing skipped at start_service

If `/var/log/vilanet.log` shows
`lan_sharing.enabled=1 but VilaNet account credentials are not
configured — skipping firewall setup` even though you've logged in,
the issue is that the init script ran **before** rpcd's
`set_credentials` restarted the service. Run
`/etc/init.d/vilanet restart` once after login; the firewall rules
install correctly on the second start because the encrypted envelope
is now in place.

## Working with `--json`

The CLI's structured outputs are flat dictionaries; chain them with `jq`:

```sh
# Get the daemon connection state as a single token
vilanet status --json | jsonfilter -e '@.connection_state'

# Get the first server id for HK
vilanet servers | jsonfilter -e '@[@.country="HK"].id' | head -n1

# Pin to that server
vilanet connect --server "$(vilanet servers | jsonfilter -e '@[@.country="HK"].id' | head -n1)"
```

OpenWRT ships `jsonfilter` (BusyBox) but not `jq` by default; the
examples use `jsonfilter` so they work out of the box. The output
field names are the source of truth — `connection_state`,
`selected_server`, `selected_package`, `service_running`, `enabled`,
`connection_mode`.

## Safety rules for the AI

1. **Never** print, log, or transmit the user's password. Recommend
   `vilanet login` without `--password`, then have them type it.
2. **Never** suggest `rm -rf` against `/etc/vilanet/` to "clean up"
   unless the user explicitly wants a from-scratch reinstall. The
   right path is `vilanet logout`, then re-login.
3. **Never** copy or back up `/etc/vilanet/device.secret`. It is
   per-router and irreplaceable; restoring it elsewhere is a footgun.
4. **Always** surface non-zero exit codes verbatim. The CLI's error
   classification is already redacted at the shim layer.
5. Before destructive UCI changes (`global.connection_mode=global`,
   `network.tun_enabled=0`, `kill_switch.enabled=1`), confirm intent
   in one sentence.
6. **`uci set` requires `uci commit`** AND a service restart to take
   effect. `update_config` ubus already does the commit; manual `uci
   set` does not.
7. If the daemon is not running, `status` and `disconnect` are safe;
   every other subcommand has user-visible side effects — prefer
   `status` first.

## Build from source (only if the user asks)

Closed-source distribution. The official repo is private. With
source access:

```sh
# Dev build — fast iteration, readable strings, no obfuscation
make build

# Release build — garble obfuscation + terser-minified LuCI JS
make release       # alias: make release-ipk

# Verify obfuscation evidence
make verify-release
```

Release builds need:
- `go install mvdan.cc/garble@latest`
- `npm install -g terser`

The garble seed for each release is printed at the bottom of the
build log — record it alongside the IPK so the artifact is
reproducible. `build.sh` honors `CORE_ARCH=<opkg-arch>` to cross-build;
the mapping table is in `build.sh` near the top. Unknown arches need
explicit `CORE_GOARCH` + optional `CORE_GOMIPS` / `CORE_GOARM`.

## Cross-references

Sibling clients (separate skills, do NOT use this skill for them):

- **vilanet-cli** — the Linux / macOS / Windows desktop CLI. Different
  binary, different transport, different install. If the user's
  machine is a laptop or desktop, use that skill.
- **Flutter desktop / mobile app** — GUI client for iOS, Android,
  macOS, Windows.
- **tvOS app** — Apple TV.

If the user installed the wrong package — e.g. they ran
`vilanet-cli` on the router because they didn't realize it was
desktop-only, OR they installed `vilanet-core` on a laptop — point
them at the correct distribution. The two binaries are NOT
interchangeable.

## Reference

- Repo (public release + skill mirror): <https://github.com/vilavpn/vilanet-openwrt>
- README: in the repo root.
- User guide: `docs/vilanet-openwrt-user-guide.html` in the repo (HTML, customer-facing).
- Companion clients: <https://github.com/vilavpn/vilanet-cli>
