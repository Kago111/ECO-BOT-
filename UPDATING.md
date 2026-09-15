# Updating the bot without losing anyone's money

## What went wrong last time

The economy lives in one file, `economy.db`. It runs in a SQLite mode called
WAL, which is the right mode for a bot on a host that can kill the container at
any moment — but it has one catch:

> A saved change goes into **`economy.db-wal`** first. It only moves into
> **`economy.db`** at a "checkpoint". Until that happens, `economy.db` on disk
> is genuinely out of date.

The default setting let that side file grow to about **4 MB** before folding
anything back — on a server this size, easily hours of play. So when you
downloaded `economy.db` from the panel, replaced the bot files and uploaded it
back, everything still sitting in the WAL was thrown away.

That is exactly the shape of the damage you saw. Not a wipe — just the most
recent writes. One or two players' balances snapped back to an older number,
and whoever had bought shares most recently lost the portfolio they had just
paid for.

Checked against the copy of the database you sent: **3 transactions and $1,306
were sitting in the WAL at that moment**, invisible to a straight copy of
`economy.db`. Mid-session, after a run of stock trades, it would have been far
more.

## What has changed

**The bot now keeps `economy.db` current at all times.**

- It folds the WAL back into the main file **every 30 seconds**, and after
  roughly every 256 KB of writes instead of 4 MB.
- So `economy.db` is never more than a few seconds behind. Copying it is safe
  at any moment, even with the bot running.

**The bot now notices if you restore the wrong file.**

- It keeps a running fingerprint of the economy — money supply, shares held,
  row counts — inside the database itself.
- On startup it compares that against what it has been handed. If the database
  is smaller than the one it was last running on, it posts a loud red warning
  to your audit log channel instead of quietly carrying on.
- That warning is your cue to restore a backup **before** players carry on and
  the loss becomes permanent.

**Two commands now bookend an update.**

- `/eco-admin db prepare-update` — run before. Flushes everything to disk,
  uploads a backup, and records the exact totals.
- `/eco-admin db verify-update` — run after. Compares the running economy
  against those totals and tells you in plain words whether anything is missing.
- `/eco-admin db status` — health check any time: where the file is, how
  current it is, when it was last backed up.

---

# The update routine

## Do this once, first: move the database out of the firing line

Right now `economy.db` sits in the same folder as the bot's code, so it is in
danger every single time you replace the files. Take it out of that folder and
most of this problem stops existing.

1. In your panel, create a folder called `data`.
2. Add an environment variable (panel → Startup / Variables):

   ```
   DATABASE_PATH=data/economy.db
   ```

   If your panel does not do variables, add the same line to `.env`.
3. Stop the bot, move `economy.db` into `data/`, start it again.
4. Check it worked: `/eco-admin db status` should show the path ending in
   `data/economy.db`, and `/balance` should show your normal amount.

From then on, **when you update, never touch the `data` folder.** Replace
everything else freely.

## Also do this once: turn on automatic backups

Make a private staff-only channel, copy its ID, and set:

```
BACKUP_CHANNEL_ID=<that channel ID>
```

The bot then uploads a complete copy of the economy there every 6 hours, and
`/backup-now` does it on demand. This is your safety net — if an update ever
does go wrong, you download the newest file from that channel instead of
losing a night of play.

## Every update, from now on

1. **`/eco-admin db prepare-update`**
   Note the numbers it shows you — that is what the economy should look like
   on the other side.

2. **Stop the bot** in the panel. Wait for it to actually stop.

3. **Replace the bot files** with the new version.
   - If `DATABASE_PATH` is set: leave the `data` folder completely alone.
     Nothing else to do.
   - If it is not set yet: download `economy.db` **and** `economy.db-wal` and
     `economy.db-shm` if they exist, then upload all of them back afterwards,
     into the same folder.

4. **Start the bot.** Watch the console for
   `Startup integrity check passed`.

5. **`/eco-admin db verify-update`**
   - ✅ green — everything survived, carry on.
   - 🚨 red — an old copy got restored. **Stop the bot now**, download the
     newest backup from your backup channel, rename it to `economy.db`, upload
     it in place of the current one, and start again.

## If it ever goes wrong anyway

1. Stop the bot immediately — every minute of play on a bad database makes the
   restore messier.
2. Go to your backup channel and download the most recent `economy-*.db`.
3. Rename it to `economy.db` and put it where `DATABASE_PATH` points.
4. Start the bot and run `/eco-admin db status` to confirm the numbers.

Players will lose whatever happened between that backup and now, which is why
`BACKUP_INTERVAL_HOURS` in `config.py` is worth lowering if your server is
busy.

---

## Quick reference

| Command | When | What it does |
|---|---|---|
| `/eco-admin db prepare-update` | Before an update | Flushes to disk, backs up, records the totals |
| `/eco-admin db verify-update` | After an update | Proves nothing was lost, or names exactly what is missing |
| `/eco-admin db status` | Any time | File location, how current it is, last backup |
| `/backup-now` | Any time | Uploads a copy to the backup channel |

| Setting | Where | Why |
|---|---|---|
| `DATABASE_PATH=data/economy.db` | Environment / `.env` | Keeps the economy out of the folder you replace |
| `BACKUP_CHANNEL_ID=...` | Environment / `.env` | Automatic backups to Discord |
| `CHECKPOINT_INTERVAL_SECONDS` | `config.py` | How often `economy.db` is brought up to date (default 30s) |
| `BACKUP_INTERVAL_HOURS` | `config.py` | How often a backup is uploaded (default 6) |
