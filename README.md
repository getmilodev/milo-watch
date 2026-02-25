# 🔒 Milo Watch

**Daily automated security & health monitoring for OpenClaw deployments.**

Your OpenClaw agent runs 24/7 — but do you know if it's secure? Milo Watch checks your configuration, flags vulnerabilities, and catches problems before they cause damage.

## What It Checks

| Check | What It Detects |
|-------|----------------|
| 🛡️ Gateway Security | Exposed gateway (bound to 0.0.0.0 instead of localhost) |
| 🔑 Authentication | Missing API key, no allowed origins |
| ⚡ Exec Security | Unrestricted shell command execution |
| 📦 Version | Outdated OpenClaw with known vulnerabilities |
| 💾 Disk & Sessions | Bloated session files, low disk space |
| 📊 Workspace Overhead | Excessive token injection from workspace files |

## Quick Start

### Install
```bash
# Copy the skill to your OpenClaw skills directory
git clone https://github.com/getmilodev/milo-watch.git
cp -r milo-watch ~/.openclaw/skills/milo-watch
```

### Run Once
Ask your agent: *"Run a Milo Watch security scan"*

### Daily Monitoring (Recommended)
```bash
openclaw cron add --name milo-watch \
  --schedule "0 8 * * *" \
  --task "Run the milo-watch skill and report results"
```

## Sample Report

```
🔒 Milo Watch — Security & Health Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🛡️ Gateway Security: OK
   Bound to 127.0.0.1:3000

🔑 Authentication: WARNING
   No API key set — anyone with network access can connect

⚡ Exec Security: WARNING
   Mode is "full" — agent can run any command

📦 Version: OK
   Running 2026.2.24 (latest)

💾 Disk & Sessions: OK
   42 session files, 128MB total, 67% disk used

📊 Workspace Overhead: WARNING
   ~12,400 tokens injected per message — consider trimming

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Score: 3/6 checks passed
Recommendation: Set an API key and switch exec to allowlist mode
```

## Pro Features (Coming Soon)

- 📈 Historical trend tracking
- 💰 Token cost analysis & optimization
- 📱 WhatsApp/Telegram connection monitoring  
- 🔧 Automated fix suggestions
- 📧 Weekly summary email reports

## Links

- [Full blog: OpenClaw Security Guide](https://github.com/getmilodev/milo-shield)
- [Free Security Audit](https://getmilo.dev)
- [Milo Shield — Automated Hardening ($29)](https://getmilo.dev/products)

## License

MIT — use it, modify it, share it.

---

Built by [Milo](https://getmilo.dev) — autonomous AI security for OpenClaw.
