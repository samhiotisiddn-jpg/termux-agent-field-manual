# AUTONOMOUS AGENT MEGA-PACK
*IronVision Nexus — distilled from months of daily Termux/ARM64 production failures and fixes.*

## 1. The quoted-heredoc rule (the #1 script killer)
Every bash deployment failure in a real Termux stack traces to unquoted heredocs.
`$(date)`, backticks and `$VARS` get eaten by bash before your file is written.

WRONG:
```
cat > agent.js <<EOF
const t = `tick ${Date.now()}`;
EOF
```

RIGHT — quote the delimiter, always:
```
cat > agent.js <<'JSEOF'
const t = `tick ${Date.now()}`;
JSEOF
```
Nothing inside expands. Pick a delimiter that never appears in the payload.

## 2. Termux ground rules
- Never use `/tmp`. It does not exist. Use `$HOME/tmp` or `$PREFIX/tmp`.
- Never depend on `jq`. Parse JSON in node/python instead.
- PM2 gets OOM-killed (signal 9) on 4GB devices. Use nohup + a watchdog loop.
- node 18+ has global `fetch`. You do not need axios or node-fetch.
- Never use eosjs@22+ constructors from old tutorials; raw HTTPS RPC calls survive upgrades.

## 3. The watchdog (replaces PM2)
```
while true; do
  pgrep -f "agents/server.js" >/dev/null || nohup node agents/server.js >> logs/server.log 2>&1 &
  sleep 30
done
```
Logs stay in `~/yourapp/logs/`. PM2 logs to its own store and dies with the daemon; nohup logs survive.

## 4. Zero-dependency static + API server pattern
Single `http.createServer` with route table. Serve a folder, expose `/health` and `/api/*`,
302-redirect tracking via an append-only NDJSON click log. No express. No npm install step.

## 5. GitHub Pages auto-deploy (free public hosting from a phone)
```
GET  /repos/:user/:repo            -> 404? then POST /user/repos {name, auto_init:true}
PUT  /repos/:user/:repo/contents/index.html   (base64 body, include sha on update)
POST /repos/:user/:repo/pages {source:{branch:main, path:/}}   (409? PUT instead)
```
Public URL: `https://:user.github.io/:repo/`. First publish takes 2-10 minutes.
This is the difference between localhost:7784 (zero traffic) and a real URL (real buyers).

## 6. dev.to programmatic publishing (free traffic)
```
POST https://dev.to/api/articles
Header: api-key: YOUR_KEY
Body: {"article":{"title":..., "body_markdown":..., "published":true, "tags":[...]}}
```
One genuine article per day beats ten spam posts. Link your store twice, naturally.

## 7. LLM failover chain
Free OpenRouter models rate-limit independently. Try them in order, move on non-200:
`z-ai/glm-5.2:free` -> `google/gemma-4-31b-it:free` -> `qwen/qwen3.8-27b:free` (re-pull the live list from /api/v1/models — free tiers rotate).
Log which model served. Never let one provider be a single point of failure.

## 8. Failure-mode field table
| Symptom | Real cause | Fix |
|---|---|---|
| syntax error near `(` | unquoted heredoc | quote delimiter |
| /tmp missing | Termux has no /tmp | $HOME/tmp |
| signal 9 kills | Android LMK / OOM | fewer procs, nohup+watchdog |
| EADDRINUSE | zombie node proc | pkill -f before start |
| 401 from every API | key rotated/dead | liveness-check before deploy |
| dashboard shows negative money | simulated revenue | real-only ledger or nothing |

## 9. Revenue honesty rule
Only count money from payment-processor events. A number you typed into a JSON file is not revenue.
Track checkout clicks (leading indicator), processor payouts (truth). Everything else is noise.

## 10. Deploy checklist
1. Liveness-check every key with a read-only curl before writing it into a script.
2. `bash -n` the script before running.
3. `node --check` every generated .js.
4. Start, then `curl localhost:PORT/health` from a second session.
5. Verify the PUBLIC url returns 200 — localhost is not a business.
