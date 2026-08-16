# Quality Guard sidecar

This is the **live host sidecar** that backs the Grok2API admin page `/quality-guard`.

The official grok2api tree already contains a quality-guard implementation (see `patches/` in this repo). Production still runs this standalone Python sidecar next to the container: it reads bootstrap/runtime-config, audits egress TPS, quarantines bad nodes, and can rotate sticky sessions.

## Files

| file | role |
|------|------|
| `quality_guard.py` | sidecar process (no baked-in host URLs or secrets) |
| `QUALITY_GUARD.md` | operator / agent reference (desensitized) |

## Run

```text
GROK2API_BASE_URL=http://127.0.0.1:8000 \
GROK2API_ADMIN_PASSWORD_FILE=/path/to/admin-password.txt \
python3 quality_guard.py
```

Do **not** commit `quality-guard.env`, `bootstrap.json`, admin passwords, or node proxy URLs.

Config is loaded from grok2api runtime-config / bootstrap (internal token, node list, thresholds). See `QUALITY_GUARD.md`.
