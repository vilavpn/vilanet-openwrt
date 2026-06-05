# vilanet-openwrt

**VilaNet VPN client for OpenWrt 24.10+ routers.** One statically-linked Go
binary, a UCI configuration layer, and a full LuCI web interface — no cloud
agent, no root SSH required for day-to-day operation.

Current version: **v1.0.13**

- [Full user guide](https://openwrt.vilavpn.com)
- [VilaVPN](https://vilavpn.com)

---

## What it is

`vilanet-openwrt` turns a supported OpenWrt router into a whole-home VilaNet
gateway. All LAN devices get VPN coverage transparently — no per-device
configuration needed.

**Highlights:**

- **Transparent whole-LAN gateway** — `routing_mode=tun` routes all LAN
  traffic through the tunnel automatically; individual devices need no proxy
  settings.
- **Full LuCI web interface** — connect, switch servers, and change settings
  from your browser under **Network > VilaNet**. Three tabs: Overview,
  Settings, Servers.
- **Multilingual UI** — the LuCI app ships 9 languages built in: English,
  Japanese, Chinese (Simplified), Chinese (Traditional), Korean, Russian,
  Farsi, Arabic, and Vietnamese. The interface follows your router's language
  setting automatically — no extra download needed.
- **sing-box core** — embeds sing-box v1.13 as a library. Supports VMess,
  VLESS+Reality, Hysteria2, AnyTLS, Shadowsocks, and Trojan.
- **Encrypted credentials at rest** — passwords are stored in an AES-256-GCM
  envelope at `/etc/vilanet/.credentials`, keyed to the router's device secret;
  never written in plaintext.
- **Kill switch** — blocks LAN internet if the tunnel drops unexpectedly.
- **Auto-update** — the daemon checks for and applies updates automatically.
- **Chrome/Gemini compatibility** — QUIC no_drop fix (vilanet-core v0.1.8)
  prevents blank-page issues with Chrome and Google Gemini.
- **iStore support** — one-click install via iStore on compatible OpenWrt
  distributions (iStoreOS).
- **APK support** — compatible with OpenWrt 25.x package format (`apk`) as
  well as the classic `opkg` format.

---

## Supported architectures

| Architecture label        | Typical router hardware                           |
|---------------------------|---------------------------------------------------|
| `x86_64`                  | x86-based OpenWrt (PC routers, VMs, NanoPi R2S)  |
| `aarch64_cortex-a53`      | GL.iNet AX1800, NanoPi R4S, Raspberry Pi 4       |
| `aarch64_generic`         | Generic 64-bit ARM boards                        |
| `arm_cortex-a7`           | Older ARM routers (cortex-a7, neon-vfpv4)        |
| `mipsel_24kc`             | Xiaomi, Netgear, TP-Link MIPS routers             |

Run `opkg print-architecture` on your router to identify the right package.
The architecture label appears in the IPK/APK filename.

**Minimum requirements:** OpenWrt 24.10.x or newer, 128 MB RAM.

---

## Install

### iStoreOS / iStore (easiest)

Download `vilanet-istore-install_1.0.13.run` from the
[releases page](https://github.com/vilavpn/vilanet-openwrt/releases), then
open **iStore → Manual Install** and upload the `.run` file. iStore installs
everything automatically (core + LuCI + all 9 languages).

### OpenWrt 25.x (APK)

```sh
# Transfer APKs to the router (replace arch label as appropriate)
scp -O vilanet-core_1.0.13_x86_64.apk      root@<router-ip>:/tmp/
scp -O luci-app-vilanet_1.0.13_all.apk     root@<router-ip>:/tmp/

# Install
ssh root@<router-ip> \
    'apk add --allow-untrusted --force-non-repository \
     /tmp/vilanet-core_1.0.13_x86_64.apk \
     /tmp/luci-app-vilanet_1.0.13_all.apk'
```

### OpenWrt 24.10 (opkg)

The closed-source IPKs (`vilanet-core_*.ipk` and `luci-app-vilanet_*.ipk`)
are distributed through VilaVPN customer channels. See the
[full user guide](https://openwrt.vilavpn.com) for download links and the
interactive architecture selector.

```sh
# Transfer IPKs to the router (replace arch label and version as appropriate)
scp -O vilanet-core_1.0.13_x86_64.ipk     root@<router-ip>:/tmp/
scp -O luci-app-vilanet_1.0.13_all.ipk    root@<router-ip>:/tmp/

# Install on the router
ssh root@<router-ip> \
    'opkg install /tmp/vilanet-core_1.0.13_x86_64.ipk \
                  /tmp/luci-app-vilanet_1.0.13_all.ipk'
```

After install, hard-refresh your browser to clear the LuCI module cache, then
open **Network > VilaNet**.

### Upgrading

The daemon supports auto-update. To upgrade manually with opkg:

```sh
opkg install /tmp/vilanet-core_<new-version>_<arch>.ipk \
             /tmp/luci-app-vilanet_<new-version>_all.ipk
```

The v1.0.10+ packages resolve a file-clash that could block upgrades from
older split-package installs — upgrade cleanly without needing to remove
first.

---

## Quick start

```sh
# SSH into the router
ssh root@<router-ip>

# Log in with your VilaVPN account (password prompt — do not use --password flag)
vilanet login --email you@example.com

# Connect to the best Hong Kong node
vilanet connect --country HK

# Check status
vilanet status
```

The same flow is available in the browser via **Network > VilaNet > Overview**.

---

## Key settings

Change settings via the LuCI **Settings** tab or UCI directly:

```sh
# Enable transparent whole-LAN gateway (all LAN devices routed through tunnel)
uci set vilanet.global.routing_mode=tun
uci commit vilanet
/etc/init.d/vilanet restart

# Switch routing policy (rule = China bypass, global = route everything)
vilanet mode rule
vilanet mode global

# Enable kill switch
uci set vilanet.kill_switch.enabled=1
uci commit vilanet
/etc/init.d/vilanet restart

# Enable LAN sharing (SOCKS5/HTTP proxy for devices that prefer it)
uci set vilanet.proxy.enabled=1
uci set vilanet.proxy.port=10081
uci commit vilanet
/etc/init.d/vilanet restart

# Auto-connect on boot
uci set vilanet.global.auto_connect=1
uci commit vilanet
```

For the complete UCI key reference and all configuration options, see the
[full user guide](https://openwrt.vilavpn.com).

---

## Use with AI assistants

`ai/vilanet-openwrt/SKILL.md` is a plain-markdown
[Anthropic Agent Skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
that teaches Claude Code, Codex CLI, Gemini CLI, Cursor, Cline, Aider,
Copilot CLI, and any other modern AI assistant how to install, configure, and
operate the router-side VPN on your behalf.

Install the skill on the **machine you SSH from** (your laptop or desktop),
not on the router itself. The content is identical everywhere; only the
install path differs.

### Claude Code

```bash
mkdir -p ~/.claude/skills/vilanet-openwrt
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-openwrt@main/ai/vilanet-openwrt/SKILL.md \
  -o ~/.claude/skills/vilanet-openwrt/SKILL.md
```

### Gemini CLI >= 0.41

```bash
gemini skills install https://github.com/vilavpn/vilanet-openwrt.git --path ai/vilanet-openwrt
# Or from a local checkout:
gemini skills link ./ai/vilanet-openwrt
```

### Codex CLI

```bash
mkdir -p ~/.codex/skills/vilanet-openwrt
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-openwrt@main/ai/vilanet-openwrt/SKILL.md \
  -o ~/.codex/skills/vilanet-openwrt/SKILL.md
```

### Cursor

```bash
mkdir -p .cursor/rules
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-openwrt@main/ai/vilanet-openwrt/SKILL.md \
  -o .cursor/rules/vilanet-openwrt.mdc
```

### Cline, Aider, Copilot CLI, and anything else

```bash
mkdir -p .agents/skills/vilanet-openwrt
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-openwrt@main/ai/vilanet-openwrt/SKILL.md \
  -o .agents/skills/vilanet-openwrt/SKILL.md

# For tools that only read AGENTS.md, append (idempotent — re-runs are a no-op):
grep -qxF '## VilaNet OpenWrt Agent Skill' AGENTS.md 2>/dev/null || \
  { echo; echo '## VilaNet OpenWrt Agent Skill'; cat .agents/skills/vilanet-openwrt/SKILL.md; } >> AGENTS.md
```

Once installed, ask your AI to install the IPK, configure UCI keys, toggle
the kill switch, drive `ubus call vilanet ...` against the router, or
diagnose a `logread -f | grep vilanet` trace — it will reach for the correct
`vilanet` verbs and the UCI / ubus surface automatically.

---

## Sibling clients

- **VilaNet CLI** — Linux, macOS, Windows headless client:
  <https://github.com/vilavpn/vilanet-cli>
- **VilaNet desktop and mobile apps** — iOS, Android, macOS, Windows GUIs:
  <https://vilavpn.com/download>

---

## Links

- User guide: <https://openwrt.vilavpn.com>
- VilaVPN: <https://vilavpn.com>
- Skill file (raw, CDN-cached):
  <https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-openwrt@main/ai/vilanet-openwrt/SKILL.md>
- VilaNet CLI mirror: <https://github.com/vilavpn/vilanet-cli>
