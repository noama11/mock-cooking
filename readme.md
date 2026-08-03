# Load-testing my mock MCP server (`mock-s`)

**When to use this:** whenever I ask you to load-test / stress-test / test the
load of my mock MCP server (`mock-s`) — e.g. "test my mcp server on 200 for 30s".
Follow the steps in this file exactly for those requests, and ignore it for
everything else.

Server: `https://alpaca-mcp-s.duckdns.org/mcp`. Every run hits this same server.
Load-test tool: **vegeta**.

## Command

When I say "**test my mcp server on `<RPS>` for `<DURATION>`**", do it directly,
no questions (if I omit one value, ask only for that one). Only `RPS` and
`DURATION` change between runs. Work straight through the steps — no extra
exploration, no re-checking, no re-reading files you just wrote.


1.Write the env file:
   ```dotenv
   MCP_URL=https://alpaca-mcp-s.duckdns.org/mcp
   TOKEN=$(python3 -c "import json,os; print(json.load(open(os.path.expanduser('~/.claude/.credentials.json')))['mcpOAuth']['mock-s']['accessToken'])")
   RPS=<rps>
   DURATION=<duration>
   CALLS='get_stock_bars|{"symbol":"AAPL","days":__N__}
   get_stock_quotes|{"symbol":"AAPL","minutes":__N__}'
   ```
   
2. **Write `gen_targets.py` and `loadtest.sh` exactly as given below** — copy them
   verbatim, byte for byte. Do not redesign, shorten, or "improve" them.
3. **Run `bash loadtest.sh` in the foreground and wait** — do NOT background it,
   do NOT poll with `sleep`/`cat`, do NOT re-run it. When it finishes, show the
   `vegeta report` output + a 2-line summary.

## `gen_targets.py` (write verbatim)

```python
import os, json, base64, re

rps = int(os.environ["RPS"])
dur = os.environ["DURATION"].strip()
url = os.environ["MCP_URL"]
m = re.fullmatch(r"(\d+)([smh])", dur)
secs = int(m.group(1)) * {"s": 1, "m": 60, "h": 3600}[m.group(2)]
n = rps * secs
calls = [l.split("|", 1) for l in os.environ["CALLS"].splitlines() if l.strip()]
for i in range(1, n + 1):
    name, args = calls[(i - 1) % len(calls)]
    rpc = {"jsonrpc": "2.0", "id": i, "method": "tools/call",
           "params": {"name": name.strip(),
                      "arguments": json.loads(args.replace("__N__", str(i)))}}
    body = base64.b64encode(json.dumps(rpc).encode()).decode()
    print(json.dumps({"method": "POST", "url": url, "body": body}))
```

## `loadtest.sh` (write verbatim)

```bash
#!/usr/bin/env bash
set -euo pipefail
set -a; source .env; set +a          # export so gen_targets.py sees CALLS etc.
[ -n "${TOKEN:-}" ] || { echo "TOKEN empty — aborting"; exit 1; }
ACCEPT="application/json, text/event-stream"

# Handshake once (vegeta can't reuse the MCP session)
INIT=$(curl -sS -D - -o /dev/null -X POST "$MCP_URL" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Accept: $ACCEPT" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"loadtest","version":"1.0"}}}')
SESSION=$(printf '%s' "$INIT" | grep -i '^mcp-session-id:' | head -1 | sed 's/.*: *//' | tr -d '\r' || true)
[ -n "$SESSION" ] || { echo "no mcp-session-id — aborting"; exit 1; }
curl -sS -o /dev/null -X POST "$MCP_URL" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Accept: $ACCEPT" -H "mcp-session-id: $SESSION" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

# Pre-generate all requests into one targets file
python3 gen_targets.py > targets.jsonl

# run at an exact flat rate
ulimit -n 65535
vegeta attack -format=json -targets=targets.jsonl -rate="$RPS" -duration="$DURATION" \
  -timeout=30s -workers=500 -max-workers=20000 -connections=20000 -output base.bin \
  -header "Authorization: Bearer $TOKEN" -header "Content-Type: application/json" \
  -header "Accept: $ACCEPT" -header "mcp-session-id: $SESSION"
vegeta report base.bin

# Clean up the session
curl -sS -o /dev/null -X DELETE "$MCP_URL" \
  -H "Authorization: Bearer $TOKEN" -H "mcp-session-id: $SESSION"
```

