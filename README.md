# vilanet-openwrt — public release & skill mirror

This repository is the **public distribution surface** for
[`vilanet-openwrt`](https://vilavpn.com), the VilaNet VPN client for
OpenWrt 24.10+ routers. It holds:

- **The AI Agent Skill** at
  [`ai/vilanet-openwrt/SKILL.md`](ai/vilanet-openwrt/SKILL.md), which
  teaches Claude Code, Codex CLI, Gemini CLI, Cursor, Cline, Aider,
  Copilot CLI, and any other modern AI assistant how to install,
  configure, and operate the router-side VPN on behalf of an end user.

The Go daemon source, the LuCI app, and the IPK build pipeline live
in a separate, private repository. This mirror is for distribution
of the skill (and, when released, the LICENSE) only.

The closed-source IPKs (`vilanet-core_*.ipk` and
`luci-app-vilanet_*.ipk`) are shipped through VilaVPN customer
channels — see the [user guide](https://vilavpn.com) for install
instructions for each supported architecture (`x86_64`,
`aarch64_cortex-a53`, `mipsel_24kc`). On a router, run
`opkg print-architecture` to identify which package to fetch.

---

## Use with AI assistants

`ai/vilanet-openwrt/SKILL.md` is a plain-markdown
[Anthropic Agent Skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
— any modern AI tool can consume it. The content is identical
everywhere; only the install path differs. Install on the **same
machine you SSH/LuCI into the router from**, not on the router
itself.

### Claude Code

```bash
mkdir -p ~/.claude/skills/vilanet-openwrt
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-openwrt@main/ai/vilanet-openwrt/SKILL.md \
  -o ~/.claude/skills/vilanet-openwrt/SKILL.md
```

### Gemini CLI ≥ 0.41

```bash
gemini skills install https://github.com/vilavpn/vilanet-openwrt.git --path ai/vilanet-openwrt
# Or from a local checkout:
gemini skills link ./ai/vilanet-openwrt
```

### Codex CLI · Cursor · Cline · Aider · Copilot CLI · anything else

```bash
mkdir -p .agents/skills/vilanet-openwrt
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-openwrt@main/ai/vilanet-openwrt/SKILL.md \
  -o .agents/skills/vilanet-openwrt/SKILL.md

# For tools that only read AGENTS.md, append (idempotent — re-runs are a no-op):
grep -qxF '## VilaNet OpenWrt Agent Skill' AGENTS.md 2>/dev/null || \
  { echo; echo '## VilaNet OpenWrt Agent Skill'; cat .agents/skills/vilanet-openwrt/SKILL.md; } >> AGENTS.md
```

Once installed, ask your AI to install the IPK, configure UCI keys,
toggle the kill switch, drive `ubus call vilanet ...` against the
router, or diagnose a `logread -f | grep vilanet` trace — it will
reach for the documented `vilanet` verbs and the UCI / ubus surface.

---

## Sibling clients

- **VilaNet CLI** — Linux/macOS/Windows headless client:
  <https://github.com/vilavpn/vilanet-cli>
- **VilaNet desktop & mobile apps** — iOS, Android, macOS, Windows
  GUIs: <https://vilavpn.com/download>

---

## Links

- VilaVPN: <https://vilavpn.com>
- Skill file (raw, CDN-cached):
  <https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-openwrt@main/ai/vilanet-openwrt/SKILL.md>
- VilaNet CLI sibling mirror: <https://github.com/vilavpn/vilanet-cli>
