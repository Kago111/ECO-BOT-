# Running this bot 24/7 on a free host

Free hosting is disposable by design. Three things about it can quietly ruin an
economy bot, and the bot now handles all three:

| Free-host behaviour | What it would break | What the bot does now |
|---|---|---|
| The filesystem is wiped on redeploy (and sometimes on restart) | `economy.db` disappears — everyone's money is gone | `DATABASE_PATH` puts the database on a persistent disk, and scheduled backups upload a copy to a Discord channel |
| The container is restarted constantly, and killed without warning | Half-written database, duplicated payouts, hammered rate limits | WAL storage + clean SIGTERM shutdown, commands re-synced only when they change, and every timer measured from the database instead of from the process clock |
| The service is only kept awake while it answers HTTP requests | The bot silently goes offline after ~15 minutes | A keep-alive web server binds `$PORT` automatically so an uptime pinger can keep it up |

---

## 1. Pick a host

Free tiers change often — check the current terms before committing. In rough
order of how well they suit this bot:

**A. A dedicated free Discord bot host** (bot-hosting.net and similar panels)
The easiest option. You upload the files, set a start command, and the panel
keeps the process running with a real, persistent filesystem. Nothing extra is
needed — no port, no pinger.
- Start command: `python -u main.py`
- Leave `DATABASE_PATH` empty; the panel's disk persists.

**B. Render (free web service)**
Free plans are web services, so the bot must listen on a port — it does, as
soon as Render sets `$PORT`. Two catches: free services sleep after about 15
minutes without traffic, and the free plan has no persistent disk.
- Build command: `pip install -r requirements.txt`
- Start command: `python -u main.py`
- Health check path: `/health`
- **Set `BACKUP_CHANNEL_ID`** — without a disk, the Discord backups are your
  only copy of the economy.
- Add a free uptime pinger (UptimeRobot, cron-job.org, BetterStack) hitting
  `https://your-service.onrender.com/health` every 5 minutes so it never sleeps.
- `render.yaml` in this folder sets all of that up if you deploy as a Blueprint.

**C. Replit**
Works, but the free plan sleeps aggressively; you need the same uptime pinger.
`.replit` is included and already sets `KEEPALIVE=1`. Put the token in Secrets,
never in a file.

**D. Oracle Cloud Always Free / any free VPS**
The most reliable of the lot — a real always-on machine with a real disk — but
you set it up yourself. Run it under systemd or `screen`, leave `PORT` unset,
and the keep-alive server stays off.

Whatever you choose: **hosts with a persistent disk are worth far more than
hosts with more RAM.** This bot idles happily in well under 512 MB.

---

## 2. Set the environment variables

Put these in the host's Environment Variables / Secrets panel. Do not upload
`.env` to a host that has a public file browser, and never commit it.

| Variable | Required | What it does |
|---|---|---|
| `DISCORD_TOKEN` | yes | Your bot token |
| `GUILD_ID` | yes | Your server's ID |
| `DATABASE_PATH` | if the host has a disk | Where the database lives, e.g. `/data/economy.db` |
| `BACKUP_CHANNEL_ID` | strongly recommended | Private channel that receives database backups |
| `LOAN_REQUEST_CHANNEL_ID` | optional | Staff-only channel where loan requests are posted for review, with Approve/Deny buttons |
| `PORT` | set by the host | The keep-alive server binds it automatically |
| `KEEPALIVE` | no | `1` forces the web server on, `0` off |
| `FORCE_SYNC` | no | `1` for a single start if slash commands look out of date |
| `UNB_API_TOKEN` | no | Only for `/migrate-unbelievaboat` |

In the Discord Developer Portal, under **Bot → Privileged Gateway Intents**,
**Server Members Intent must be on**. The bot will not start without it.

---

## 3. Back up the economy

Set `BACKUP_CHANNEL_ID` to a private, staff-only channel. Every
`BACKUP_INTERVAL_HOURS` (6 by default, in `config.py`) the bot uploads a
complete, consistent snapshot of the database there. Staff can also run
`/backup-now` before anything risky.

**To restore:** stop the bot, download the newest `economy-*.db` attachment,
rename it to `economy.db`, upload it to where `DATABASE_PATH` points (or next
to `main.py` if that variable is empty), and start the bot again.

If you ever copy a live database by hand, take `economy.db`, `economy.db-wal`
and `economy.db-shm` together, or use `/backup-now` instead — the `-wal` file
holds the most recent transactions.

---

## 4. Things that go wrong, and what they mean

**Commands don't appear after a deploy.** They're only re-synced when they
change. Set `FORCE_SYNC=1`, restart once, then remove it.

**"DISCORD_TOKEN is not set."** The host isn't passing the variable through —
check for a stray space or quotes around the value in the panel.

**The bot goes offline every 15 minutes.** The host is sleeping an idle web
service. Point an uptime pinger at `/health`.

**The economy reset itself after a deploy.** The filesystem was wiped. Restore
from the backup channel and set `DATABASE_PATH` to a persistent disk, or move
to a host that has one.

**The bot restarts in a loop.** Read the logs: a bad token or the missing
Members intent are the usual causes, and both are fatal by design — a bot that
kept retrying with a bad token would just get rate limited.

**Market prices jump around more than they should.** They shouldn't any more:
the market tick is measured from the database, so restarts can't trigger extra
ticks. If prices still move too fast, lower `STOCK_MAX_RANDOM_DRIFT` or raise
`STOCK_TICK_MINUTES` in `config.py`.

---

## 5. Files in this folder for hosts

| File | For |
|---|---|
| `Procfile` | Heroku-style hosts (`web:` and `worker:` both run the bot) |
| `render.yaml` | Render Blueprint deploys |
| `runtime.txt`, `.python-version` | Pin Python 3.11 |
| `start.sh` | Panels that ask for a shell start command |
| `.replit` | Replit |
| `requirements.txt` | Pinned to compatible major versions so a future release can't break a redeploy |
