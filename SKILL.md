# Milo Watch — OpenClaw Security & Health Monitor

Daily automated security and health monitoring for your OpenClaw deployment. Detects misconfigurations, exposed services, token waste, and reliability issues before they cause problems.

## When to Use
Run this as a daily cron job or manually when you want a health check of your OpenClaw setup.

## How to Run

### One-Time Check
Ask your agent: "Run a Milo Watch security scan"

### Daily Cron (Recommended)
```
openclaw cron add --name milo-watch \
  --schedule "0 8 * * *" \
  --task "Run the milo-watch skill and report results"
```

## Scan Protocol

Run these checks in order. Report results using the format at the bottom.

### 1. Gateway Security

```bash
# Check gateway binding
cat ~/.openclaw/openclaw.json 2>/dev/null | python3 -c "
import json, sys
try:
    config = json.load(sys.stdin)
    host = config.get('gateway', {}).get('host', '127.0.0.1')
    port = config.get('gateway', {}).get('port', 3000)
    print(f'HOST={host}')
    print(f'PORT={port}')
    if host in ['0.0.0.0', '']:
        print('STATUS=CRITICAL: Gateway bound to 0.0.0.0 — exposed to all network interfaces')
    else:
        print('STATUS=OK: Gateway bound to ' + host)
except Exception as e:
    print(f'STATUS=ERROR: Could not parse config: {e}')
"
```

**CRITICAL** if host is `0.0.0.0` — the gateway is accessible from any network interface. Fix: change to `127.0.0.1` or use a reverse proxy.

### 2. Authentication

```bash
# Check if API key is set
cat ~/.openclaw/openclaw.json 2>/dev/null | python3 -c "
import json, sys
try:
    config = json.load(sys.stdin)
    api_key = config.get('gateway', {}).get('apiKey', '')
    allowed_origins = config.get('gateway', {}).get('allowedOrigins', [])
    if not api_key:
        print('STATUS=WARNING: No API key set on gateway — anyone with network access can connect')
    else:
        print('STATUS=OK: API key configured')
    if not allowed_origins:
        print('ORIGINS=WARNING: No allowed origins configured')
    else:
        print(f'ORIGINS=OK: {len(allowed_origins)} origin(s) configured')
except Exception as e:
    print(f'STATUS=ERROR: {e}')
"
```

### 3. Exec Security

```bash
# Check execution security mode
cat ~/.openclaw/openclaw.json 2>/dev/null | python3 -c "
import json, sys
try:
    config = json.load(sys.stdin)
    exec_security = config.get('tools', {}).get('exec', {}).get('security', 'full')
    if exec_security == 'full':
        print('STATUS=WARNING: Exec security is \"full\" — agent can run any shell command without restrictions')
    elif exec_security == 'deny':
        print('STATUS=OK: Exec is denied — most secure')
    elif exec_security == 'allowlist':
        print('STATUS=OK: Exec uses allowlist — restricted command set')
    else:
        print(f'STATUS=INFO: Exec security mode: {exec_security}')
except Exception as e:
    print(f'STATUS=ERROR: {e}')
"
```

### 4. Version Check

```bash
# Check installed version vs latest
INSTALLED=$(openclaw --version 2>/dev/null || echo "unknown")
echo "INSTALLED=$INSTALLED"

# Check latest available
LATEST=$(npm view openclaw version 2>/dev/null || echo "unknown")
echo "LATEST=$LATEST"

if [ "$INSTALLED" != "$LATEST" ] && [ "$INSTALLED" != "unknown" ] && [ "$LATEST" != "unknown" ]; then
    echo "STATUS=WARNING: Running $INSTALLED, latest is $LATEST — update recommended"
else
    echo "STATUS=OK: Running latest version ($INSTALLED)"
fi
```

### 5. Disk & Session Health

```bash
# Check session file sizes
SESSIONS_DIR="$HOME/.openclaw/sessions"
if [ -d "$SESSIONS_DIR" ]; then
    TOTAL_SIZE=$(du -sh "$SESSIONS_DIR" 2>/dev/null | cut -f1)
    FILE_COUNT=$(find "$SESSIONS_DIR" -type f 2>/dev/null | wc -l)
    LARGE_FILES=$(find "$SESSIONS_DIR" -type f -size +10M 2>/dev/null | wc -l)
    echo "SESSIONS_SIZE=$TOTAL_SIZE"
    echo "SESSIONS_COUNT=$FILE_COUNT files"
    if [ "$LARGE_FILES" -gt 0 ]; then
        echo "STATUS=WARNING: $LARGE_FILES session file(s) over 10MB — consider cleanup"
    else
        echo "STATUS=OK: Session files healthy"
    fi
fi

# Check disk space
DISK_FREE=$(df -h ~ 2>/dev/null | tail -1 | awk '{print $4}')
DISK_PCT=$(df ~ 2>/dev/null | tail -1 | awk '{print $5}' | tr -d '%')
echo "DISK_FREE=$DISK_FREE"
if [ "$DISK_PCT" -gt 90 ] 2>/dev/null; then
    echo "DISK_STATUS=CRITICAL: Disk ${DISK_PCT}% full"
elif [ "$DISK_PCT" -gt 80 ] 2>/dev/null; then
    echo "DISK_STATUS=WARNING: Disk ${DISK_PCT}% full"
else
    echo "DISK_STATUS=OK: Disk ${DISK_PCT}% used"
fi
```

### 6. Workspace File Size Audit

```bash
# Check workspace file injection overhead
WORKSPACE_DIR="$HOME/.openclaw/workspace"
if [ -d "$WORKSPACE_DIR" ]; then
    echo "=== Workspace Files (injected every prompt) ==="
    for f in "$WORKSPACE_DIR"/*.md; do
        if [ -f "$f" ]; then
            SIZE=$(wc -c < "$f")
            TOKENS=$((SIZE / 4))  # rough estimate
            BASENAME=$(basename "$f")
            echo "$BASENAME: ${SIZE} bytes (~${TOKENS} tokens/message)"
        fi
    done
    TOTAL=$(cat "$WORKSPACE_DIR"/*.md 2>/dev/null | wc -c)
    TOTAL_TOKENS=$((TOTAL / 4))
    echo "TOTAL: ~${TOTAL_TOKENS} tokens injected per message"
    if [ "$TOTAL_TOKENS" -gt 10000 ]; then
        echo "STATUS=WARNING: Workspace injection over 10K tokens — consider trimming"
    else
        echo "STATUS=OK: Workspace injection size reasonable"
    fi
fi
```

## Report Format

After running all checks, compile results into this format:

```
🔒 Milo Watch — Security & Health Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🛡️ Gateway Security: [OK/WARNING/CRITICAL]
   [details]

🔑 Authentication: [OK/WARNING]
   [details]

⚡ Exec Security: [OK/WARNING]
   [details]

📦 Version: [OK/WARNING]
   [details]

💾 Disk & Sessions: [OK/WARNING/CRITICAL]
   [details]

📊 Workspace Overhead: [OK/WARNING]
   [details]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Score: [X/6 checks passed]
[Summary recommendation if any issues found]

Powered by Milo Watch — getmilo.dev
```

## Severity Levels
- **CRITICAL** — Immediate security risk. Fix now.
- **WARNING** — Suboptimal configuration. Should fix.
- **OK** — Configured correctly.
- **INFO** — Informational, no action needed.

## Pro Features (Coming Soon)
- Historical trend tracking
- Token cost analysis and optimization recommendations
- WhatsApp/Telegram connection monitoring
- Automated fix suggestions with one-click apply
- Weekly summary email reports

Learn more: https://getmilo.dev
