# BeamNG RP Economy Bot

A Discord economy bot built for BeamNG.drive roleplay servers. Cash, bank and
interest-earning savings, role-based jobs, a garage, credit-scored bank loans, a police/legal system for fines, wanted
levels and bounties, and a player-tradeable stock market. Everything is backed
by SQLite so nothing resets when the bot restarts.

## Project layout

```
main.py          entry point, loads every cog in cogs/
config.py        all balancing knobs and server-specific settings
amounts.py       shared shorthand amount parsing (10k, 1.5m, all) used by every command
database.py      async SQLite layer (all SQL lives here)
cogs/
  economy.py     balance, daily, work, deposit/withdraw, savings, pay, loans, leaderboard, transactions
  jobs.py        joblist, jobinfo
  vehicles.py    garage
  legal.py       fine, fines, payfine, wanted, wantedstatus, arrest, bounty
  stocks.py      market, stockinfo, portfolio, stock buy/sell + background price ticks
  companies.py   paycompany, company balance/deposit/withdraw/staff/hire/fire/transfer, tax pay
  admin.py       /eco-admin ... management commands
  migration.py   /migrate-unbelievaboat importer
  backup.py      scheduled database backups to a Discord channel + /backup-now
  persistence.py keeps economy.db on disk current, and detects data lost in an update
  webauth.py     /weblogin, /weblogout, /dashboard — signing in to the website
  marketdata.py  samples prices for the website's charts
stockhistory.py  per-player trading analysis behind /stock history
keepalive.py     tiny web server so free hosts keep the bot awake
webapi.py        the JSON API the web dashboard trades through
webdb.py         website sessions, order idempotency and price history tables
HOSTING.md       how to run this 24/7 on a free host
WEBSITE.md       how to turn on the web dashboard and what its API does
BASE44_PROMPT.md the text to paste into Base44 to build the dashboard
UPDATING.md      how to update the bot without losing anyone's money — READ THIS FIRST
```

## Typing amounts

Every command that takes an amount — money *or* a share count — understands the
same shorthand, so `/pay 10k` and `/stock buy 10k` both do what you'd expect:

| You type | The bot reads |
| --- | --- |
| `500` | 500 |
| `1,500` or `$1500` | 1,500 |
| `10k` / `20K` | 10,000 / 20,000 |
| `1.5k` | 1,500 |
| `2m` | 2,000,000 |
| `0.5b` | 500,000,000 |
| `all` / `max` | your whole available balance (or holding) |
| `half` | half of it |

Anything else gets a friendly error instead of a failed command, and fractions
of a dollar are always rounded down. The parsing lives in `amounts.py`, so a new
command gets the same behaviour with a single call.

## Commands

**Economy**

| Command | Description |
| --- | --- |
| `/balance [user]` (`/bal`) | Full financial profile: cash, bank, savings, garage value, loan debt, credit score, net worth |
| `/work` | Quick odd job for cash on a short cooldown |
| `/deposit <amount>` (`/dep`) / `/withdraw <amount>` (`/with`) | Move money between cash and bank. Accepts `500`, `1,500`, `1.5k`, `2m` or `all` |
| `/savings deposit` / `/savings withdraw` / `/savings info` | Savings account that earns interest every `SAVINGS_TICK_HOURS` |
| `/pay <user> <amount>` | Send cash to another player (atomic, can't overdraw) |
| `/loanrequest <amount> [days]` | Request a bank loan and choose how long to repay it in (1–21 days). The cap scales with bank balance and credit score, but a staff member must approve it before funds are issued (see Loan approval below) |
| `/loanstatus` / `/loanpay <amount>` | Check or pay down your active loan, with its deadline. Paying it off on time raises your credit; paying late lowers it |
| `/leaderboard` | Richest players by cash + bank + savings |
| `/transactions [user] [limit]` | Your recent transaction history. Economy staff can pass `user` to review any player's — see Staff tools below |

**Loan approval** (requires Administrator, Manage Server, or Moderate Members
permission, or a role listed in `LOAN_APPROVER_ROLE_NAMES`)

| Command | Description |
| --- | --- |
| `/loanrequests` | View pending loan requests awaiting a decision |
| `/approveloan <request_id>` | Approve a request and issue the loan |
| `/denyloan <request_id> [reason]` | Deny a request |

Players can no longer approve their own loans — `/loanrequest` only files a
request, and nothing is disbursed until staff approve it.

### The loan review channel

Set **`LOAN_REQUEST_CHANNEL_ID`** in `.env` to a staff-only channel and every
new request is posted there as a complete credit file:

- **Who** is asking (display name, mention and user ID), for how much, over what
  term, at what interest, and what the total repayment would be.
- **What they're worth** — cash, bank, savings and credit score.
- **A straight recommendation**: ✅ RECOMMENDED, 🟡 BORDERLINE or
  ❌ NOT RECOMMENDED, with the reasoning listed line by line (credit score,
  whether they can already cover the repayment, how much of their limit they're
  asking for, how they repaid previous loans, and whether they've asked for the
  maximum term).
- **Approve / Deny buttons** — Deny opens a box for a reason the borrower sees.
  Once decided, the post rewrites itself with the outcome and who decided it, so
  the channel is never a list of stale requests.

The buttons survive a restart, so a request sitting in the channel overnight is
still actionable in the morning. `/approveloan` and `/denyloan` still work
exactly as before, and if the channel isn't set nothing breaks — staff just use
`/loanrequests`.

Overdue loans are reported to the same channel once every
`LOAN_OVERDUE_ALERT_HOURS`. Nothing is ever seized automatically; the reminder
is there so staff can chase it in character.

### Loan terms and interest

Borrowers pick their own repayment window, up to a hard ceiling of **three
weeks**, and the bank prices the risk: interest runs from
`LOAN_INTEREST_RATE` (10%) at the shortest term up to
`LOAN_MAX_TERM_INTEREST_RATE` (60%) at the full 21 days, along a curve set by
`LOAN_TERM_RATE_CURVE`.

| Term | Interest | $10,000 becomes |
| --- | --- | --- |
| 1 day | 10% | $11,000 |
| 3 days | 10.5% | $10,500 |
| 1 week | 14.5% | $11,450 |
| 2 weeks | 31.1% | $13,113 |
| **3 weeks** | **60%** | **$16,000** |

A few extra days costs almost nothing; stretching a loan to the limit is a
serious decision. The term dropdown shows the cost of each option before the
player commits, and the deadline is shown in `/loanstatus` and `/balance`.
Settling on time earns `CREDIT_SCORE_LOAN_PAYOFF_BONUS` credit points; settling
late costs `LOAN_LATE_PAYOFF_PENALTY`.

**Jobs**

Jobs are restricted to members holding a Police, EMS, or DOT Discord role —
there's no self-service sign-up. `/jobapply` has been removed; a player's job
is detected automatically from the Discord roles listed in `config.JOB_ROLE_IDS`.

| Command | Description |
| --- | --- |
| `/joblist` | Jobs and hourly pay |
| `/jobinfo` | Your job (based on your current roles) |

`/clockin` and `/clockout` have been removed. Shift payroll was exploitable
(simultaneous clock-outs paid for the same shift several times over), so jobs
are now roleplay roles only and hourly pay is informational.

To configure which roles count for which job, see "Setting up job roles"
below.

**Vehicles**

| Command | Description |
| --- | --- |
| `/garage [user]` | Owned vehicles with condition and value |

Vehicle insurance (`/insure` and `/claim`) has been removed. Claims paid out
against a vehicle's full purchase price for a flat $25 premium, which let any
player mint unlimited cash.

Vehicles are registered to a player's garage by staff with
`/eco-admin givevehicle`, so acquisitions can be roleplayed however your
server prefers. Players can no longer sell their own vehicles — `/sellvehicle`
has been removed; use `/eco-admin removevehicle` for staff-run buybacks if
your server wants that flow.

**Legal / Police** (requires a role listed in `config.POLICE_ROLE_IDS`, or Administrator)

| Command | Description |
| --- | --- |
| `/fine <user> <amount> <reason>` | Issue a fine (also lowers the target's credit score) |
| `/fines [user]` / `/payfine <id>` | View and pay fines. Paid fines go to the server treasury |
| `/wanted <user> <0-5>` / `/wantedstatus [user]` / `/arrest <user>` | Wanted level management |
| `/bounty <user> <amount>` | Anyone can post a cash bounty; funds are held in the treasury |

**Daily reward**

| Command | Description |
| --- | --- |
| `/daily` | Claim the daily reward. Pays a random amount between `DAILY_REWARD_MIN` and `DAILY_REWARD_MAX`, plus `DAILY_STREAK_BONUS` per consecutive day (capped at `DAILY_STREAK_MAX_DAYS`). Claim again within `DAILY_STREAK_GRACE_HOURS` to keep the streak; miss it and the streak resets to 1. The cooldown is claimed atomically, so spamming the command can't pay twice |

**Stock market**

| Command | Description |
| --- | --- |
| `/market` | Browse listed businesses, sorted by today's biggest movers, with price, market cap and shares available |
| `/stockinfo <business>` | Full detail on one business: price, market cap, owner, recent history |
| `/topmovers` | Today's top 5 gainers and top 5 losers at a glance |
| `/stock buy` / `/stock sell` | Trade shares. Prices move with trading volume and a background drift tick. Single orders are capped at `STOCK_MAX_TRADE_PCT_OF_SHARES` of total shares to stop one player from pumping/dumping a stock in one order. Selling has a per-player cooldown (`STOCK_SELL_COOLDOWN_MINUTES`, default 30 minutes); a rejected order does not consume it |
| `/portfolio [user]` | Holdings with profit/loss |

**Every command in this table is restricted to your stock channel(s)** — see
"Restricting commands to a channel" below. Until you set one, they work
everywhere.

When staff list a business with `/eco-admin business create` and attach an
owner, that owner is automatically given a free "founder" stake
(`BUSINESS_FOUNDER_SHARE_PERCENT` of total shares, overridable per-business
with the `founder_percent` option) — the rest of the shares go straight to
the open market for other players to buy into.

### Stock prices follow real revenue

A listing can be tied to a real registered business, and then its price stops
being a coin flip: **the stock moves on how much that business actually earns.**

A listing is matched to a business either by an explicit
`/eco-admin business link`, or — if you never run it — automatically whenever a
listing and a registered business share a name. A listing with no business
behind it keeps the old random-drift behaviour, so nothing you already have
changes until you link it.

How a linked stock is priced, every `STOCK_TICK_MINUTES`:

1. The bot works out what the business earned since the last tick (revenue only
   — `/paycompany` and `/eco-admin company addrevenue`; owner deposits never
   count).
2. It compares that to what a business of its size is *expected* to earn:
   `STOCK_REVENUE_EXPECTED_YIELD` of its market cap (default 2%).
3. Beat expectations and the price rises, miss them and it falls, by
   `STOCK_REVENUE_IMPACT` x how far off it was — capped at
   `STOCK_REVENUE_MAX_GAIN` / `STOCK_REVENUE_MAX_DROP` per tick.
4. The usual small random drift is applied on top, so the market still breathes.

Because expectation is a share of the *market cap*, the model corrects itself: a
stock that has run up needs more revenue just to stand still, so a price can
only stay high while the business keeps trading, and a business that goes quiet
always bleeds back down. There is no way to park a stock at a high price.

Being paid also nudges the stock straight away, so players see the market react
to their custom (`STOCK_REVENUE_INSTANT_*`, capped at +2% per payment). Every
revenue-driven move is written to the stock history, so `/stockinfo` shows
*why* a price moved instead of leaving players to guess.

`/eco-admin business unlink` removes a stock's link for good: that listing goes
back to drifting on trading alone, and the automatic name match won't quietly
re-link it. `/eco-admin business link` with no company puts it back to automatic
matching.

Set `STOCK_REVENUE_LINK_ENABLED = False` in `config.py` to turn the whole thing
off and go back to a purely random market.

**Businesses (companies)**

Companies are real in-character businesses with their own bank account,
separate from the stock market. Staff register one with
`/eco-admin company create`.

| Command | Description |
| --- | --- |
| `/paycompany <business> <amount>` | Pay a business as a customer. Counts as revenue, and is therefore taxed |
| `/company balance <business>` | Account balance, withdrawable amount, revenue, deposits, tax bill and your role |
| `/company deposit <business> <amount> [cash\|bank]` | Put **your own** money into the business account. Never taxed, never counted as revenue, never moves the company's stock price |
| `/company withdraw <business> <amount>` | Take money out. Outstanding tax always stays reserved in the account |
| `/tax pay <business> [amount]` | Settle the business tax bill; the money goes to the server treasury |
| `/company staff <business>` | The org chart: owner plus every employee and their title |
| `/company hire <business> <user> <position>` | Hire someone, or move an existing employee to a new title |
| `/company fire <business> <user>` | Remove an employee |
| `/company transfer <business> <new_owner>` | Hand the whole business — account balance and tax bill included — to another player |
| `/company list` | **Public.** Every business on the server ranked by all-time revenue, with owner, staff count, account balance and outstanding tax |
| `/company info <business>` | **Public.** Full profile of any business: owner, the whole staff roster, all-time revenue, account balance, untaxed owner deposits, tax paid and outstanding, recent activity, and its share price if it is also listed on the stock market |

`/company list` and `/company info` are open to every player, not just owners
and staff — but only in the business channel(s) you set (see below). Set
`COMPANY_PUBLIC_INFO_SHOW_BALANCE = False` in `config.py` to keep account
balances and owner deposits private to the business; revenue, owner, staff and
tax status stay public either way. `COMPANY_PUBLIC_INFO_SHOW_LEDGER = False`
hides the recent-activity list.

**Owner deposits vs revenue.** Money the business *earns* is revenue and is
taxed at `BUSINESS_TAX_RATE`. Money the owner *puts in* with `/company deposit`
is their own already-taxed cash: it raises the account balance only. It never
increases the tax bill and never touches a share price, so an owner can cover
payroll or a tax bill out of pocket without being taxed twice.

**Staff and ranks.** Job titles come from `config.COMPANY_POSITIONS`, listed
most senior first (`COO`, `CFO`, `Senior Manager`, `Manager`, `Supervisor`,
`Employee` by default). Seniority is what gives a title its power:

- Nobody can hire, promote, demote or fire into a position at or above their
  own rank — a COO can appoint a Manager, but not another COO.
- The owner (and economy staff) outrank everyone.
- Which titles may deposit, withdraw, pay tax and manage staff is set per
  position in `config.py` (`COMPANY_DEPOSIT_POSITIONS`,
  `COMPANY_WITHDRAW_POSITIONS`, `COMPANY_TAX_POSITIONS`,
  `COMPANY_MANAGE_STAFF_POSITIONS`).
- Anyone who can withdraw from a company is blocked from `/paycompany`-ing it,
  because paying money into an account you can empty is a laundering loop.

**Admin** (`/eco-admin ...`, economy staff roles only)

`give`, `take`, `reset`, `setjob`, `credit-adjust`, `givevehicle`,
`removevehicle`, `treasury`, `market-channel`, `log-channel`,
`channels show|add|remove|clear`,
`business create|adjust|link|unlink|setowner|delist` for the stock market, and
`company create|addrevenue|info|ledger|setowner|setstaff|delete` for real
businesses.

**Deleting a business.** `/eco-admin company delete <business>` permanently
removes one registered business — the company record, its staff roster and its
whole ledger — behind a confirmation button only the admin who ran it can press.
Whatever is left in the business account is handled by the `money` option:
settle the outstanding tax and refund the rest to the owner (the default), send
the whole balance to the treasury, or destroy it with the business.

This **never touches the stock market**. If the business had a listing, that
stock keeps trading and every shareholder keeps their shares — it simply stops
being priced off revenue and goes back to drifting. Removing a stock and paying
its shareholders out is a separate, deliberate action:
`/eco-admin business delist`.
`/migrate-unbelievaboat` imports balances from UnbelievaBoat (see below).
`/eco-admin reset-economy` wipes the whole economy and is owner-only (see
below).

## Restricting commands to a channel

Two groups of commands can be pinned to specific channels so they don't flood
general chat:

| Area | Commands |
| --- | --- |
| Stock market | `/market`, `/topmovers`, `/stockinfo`, `/portfolio`, `/stock buy`, `/stock sell` |
| Business info | `/company list`, `/company info` |

Set them up in Discord — no redeploy needed, and the setting survives one:

```
/eco-admin channels add area:Stock market commands   channel:#stock-market
/eco-admin channels add area:Business info commands  channel:#businesses
/eco-admin channels show
```

- Add more than one channel per area by running `add` again.
- `remove` takes a channel back off the list; `clear` unrestricts the area.
- Threads inside an allowed channel count as that channel.
- Used in the wrong channel, the command replies **only to that player**,
  pointing them at the right one — no public telling-off.
- Economy staff can use these commands anywhere, so they can help someone
  without moving. Set `CHANNEL_LOCK_STAFF_BYPASS = False` in `config.py` to
  apply the restriction to everyone.
- `STOCK_CHANNEL_IDS` and `BUSINESS_INFO_CHANNEL_IDS` in `config.py` are a
  fallback if you'd rather hard-code the channels. Anything set in Discord
  takes priority. Both empty = no restriction.

## Resetting the whole economy

`/eco-admin reset-economy` starts the season over: it deletes **every**
balance, bank, savings pot, credit score, vehicle, fine, loan, transaction,
the treasury, every stock listing and shareholding, and every registered
business with its staff and ledger. Players are recreated at
`STARTING_CASH` / `STARTING_BANK` the next time they run a command.

It cannot be undone, so it has four locks on it:

1. **Owner and co-owners only.** The server owner always qualifies. Anyone
   else has to be listed by user ID in `ECONOMY_RESET_USER_IDS` in
   `config.py` — an economy staff role or even the Administrator permission is
   not enough on its own.
2. **A typed phrase.** You must type `RESET EVERYTHING` into the command's
   `confirm` option (change it with `ECONOMY_RESET_CONFIRM_PHRASE`).
3. **A confirmation button**, shown with an exact count of what is about to be
   destroyed. It expires after 60 seconds, and only the person who ran the
   command can press it.
4. **An automatic backup first.** If `BACKUP_CHANNEL_ID` is set, a full
   database snapshot is uploaded to that channel before anything is deleted,
   so a mistake can still be undone by restoring it. Turn this off with
   `ECONOMY_RESET_BACKUP_FIRST = False`.

The wipe runs as a single database transaction: either all of it goes or none
of it does, so an interrupted reset can never leave half an economy behind.
Server settings (log channel, market channel, channel restrictions) are kept
unless you tick `also_reset_settings`. Other servers using the same bot are
untouched, and the result is announced in the channel where it was run.

## Setting up job roles

Since `/jobapply` was removed, Police/EMS/DOT jobs are only granted through
Discord roles:

1. Enable Developer Mode: User Settings -> Advanced -> Developer Mode.
2. In Server Settings -> Roles, right-click each role that should count as a
   job (you can have more than one role per job, e.g. `Police Cadet` and
   `Police Officer` both counting as `Police`) and choose **Copy Role ID**.
3. Paste the IDs into `config.JOB_ROLE_IDS`, e.g.:
   ```python
   JOB_ROLE_IDS = {
       "Police": [111111111111111111, 222222222222222222],
       "EMS": [333333333333333333],
       "DOT": [444444444444444444],
   }
   ```
4. Restart the bot. `/jobinfo` will now pick up a member's job
   automatically the moment they hold a listed role — no command needed.

## Setup

1. Create a bot application at https://discord.com/developers/applications,
   add a Bot user, and enable the **Server Members Intent** under
   Bot > Privileged Gateway Intents.
2. Invite the bot with the `applications.commands` and `bot` scopes and at
   least: Send Messages, Embed Links, Use Slash Commands.
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
4. Copy `.env.example` to `.env` and paste in your bot token:
   ```
   DISCORD_TOKEN=your_bot_token_here
   ```
   `.env` is git-ignored. Never commit it. If a token has ever been
   committed or shared, regenerate it in the Developer Portal.
5. Run the bot:
   ```bash
   python main.py
   ```

Slash commands sync on startup, but only when a command has actually changed
since the last run — that keeps a host that restarts often from hitting
Discord's sync rate limit. Set `FORCE_SYNC=1` for one start if they ever look
out of date.

## Trading on a website instead of in Discord

Players can trade in a browser rather than typing `/market` and `/stock buy`.
Turn on the API in `WEBSITE.md` (one environment variable and a restart), then
build the site itself with the prompts in `BASE44_PROMPT.md`.

A website order runs through the **same code** a slash command does, so every
cooldown, order cap, volume cap, circuit breaker and insider rule applies
identically — there is no advantage to using one over the other. Players sign
in by running `/weblogin` and pasting the code the bot gives them; no OAuth app
and no passwords.

Both can run at once. When everyone has moved over, set
`STOCK_DISCORD_COMMANDS_ENABLED = False` in `config.py` and the Discord trading
commands reply with a link to the site instead. `/stock history` keeps working
for staff either way.

## Hosting it 24/7 for free

See **HOSTING.md**. Short version: set `DATABASE_PATH` to a persistent disk if
your host has one, set `BACKUP_CHANNEL_ID` so the database is backed up to a
private Discord channel either way, and if the host gives the app a `$PORT`,
point a free uptime pinger at `/health` to stop it sleeping.

## Customizing for your server

Almost everything lives in `config.py`:

- `JOBS` - job names, hourly pay and descriptions.
- `JOB_ROLE_IDS` - which Discord role IDs count as which job (see "Setting up
  job roles" above). This replaces the old name-matching behavior.
- `VEHICLE_CATEGORIES` - choices offered by `/eco-admin givevehicle`.
- `POLICE_ROLE_IDS` - which Discord role IDs may use the police commands.
  Matched by ID, never by name. **Fill this in** — while it is empty, only
  Administrators can use `/fine`, `/wanted` and `/arrest`.
- `ECONOMY_BRAND_NAME` - embed branding.
- `LOAN_APPROVER_ROLE_NAMES` - extra role names (beyond Administrator/Manage
  Server/Moderate Members) allowed to approve or deny loan requests.
- `WORK_*`, `SAVINGS_*`, `LOAN_*`, `CREDIT_SCORE_*` - economy
  balancing knobs.
- `STOCK_*` - market volatility, tick frequency, per-order trade cap,
  `STOCK_SELL_COOLDOWN_MINUTES` (minutes between a player's sell orders; set
  to `0` to disable), and the default founder share percentage for new
  businesses.
- `COMPANY_POSITIONS` - the job titles a business can hand out, in seniority
  order, plus `COMPANY_MAX_EMPLOYEES` and the per-position permission lists
  (`COMPANY_DEPOSIT_POSITIONS`, `COMPANY_WITHDRAW_POSITIONS`,
  `COMPANY_TAX_POSITIONS`, `COMPANY_MANAGE_STAFF_POSITIONS`).
- `COMPANY_DEPOSITS_ENABLED`, `COMPANY_DEPOSIT_MIN/MAX` - owner deposits into a
  business account, and `COMPANY_OWNER_CAN_TRANSFER` to decide whether owners
  may hand over a business themselves or only staff can.
- `BUSINESS_TAX_RATE` - the share of all-time company revenue owed as tax.
- `BACKUP_ENABLED`, `BACKUP_INTERVAL_HOURS` - automatic database backups to the
  channel set by `BACKUP_CHANNEL_ID` in `.env`.
- `DAILY_*` - the `/daily` reward: `DAILY_ENABLED`, payout range
  (`DAILY_REWARD_MIN`/`DAILY_REWARD_MAX`), `DAILY_COOLDOWN_HOURS`, streak
  bonus size and cap, the streak grace window, and `DAILY_PAY_TO_BANK`.

## Staff tools for investigating players

| Command | Description |
| --- | --- |
| `/transactions <user>` | Every recent money movement for one player, with totals in and out and their balance now. Staff only when `user` is given; logged to the audit channel |
| `/stock history <player>` | Full trading review for one player: profit taken, what they still hold, and any exploit patterns — buying just before a price jump, instant flips, repeated maximum-size orders, selling their own business's stock. Leads with a plain verdict so you can stop reading after one line |
| `/eco-admin audit` | Server-wide sweep: money supply reconciliation and abuse patterns across everyone |

Tuning for `/stock history` lives under `HISTORY_*` in `config.py`.

## Updating the bot

**Read UPDATING.md before you replace any files.** In short: set
`DATABASE_PATH` to a folder you never delete, run `/eco-admin db prepare-update`
before, and `/eco-admin db verify-update` after.

| Command | Description |
| --- | --- |
| `/eco-admin db prepare-update` | Flushes everything to `economy.db`, uploads a backup and records the economy's exact totals |
| `/eco-admin db verify-update` | Compares the running economy against those totals and says whether anything was lost |
| `/eco-admin db status` | Where the database lives, how current the file is, when it was last backed up |

## Data

Everything is stored in `economy.db` (SQLite) in the working directory, or at
`DATABASE_PATH` if you set it. Backups upload themselves to the channel in
`BACKUP_CHANNEL_ID` every `BACKUP_INTERVAL_HOURS`, and staff can run
`/backup-now` on demand — see HOSTING.md for restoring one. All tables are keyed by `guild_id`, so one bot instance
can serve multiple servers. Schema changes are applied automatically on
startup for databases created by older versions.

## Migrating balances from UnbelievaBoat

1. Generate an API token at https://unbelievaboat.com/api/docs for an
   application authorized on your server with the **Economy** permission.
2. Add `UNB_API_TOKEN=...` to `.env` and restart the bot.
3. Run `/migrate-unbelievaboat mode:Replace dry_run:True` to preview.
4. Re-run with `dry_run:False` to apply. `mode:Add` stacks UnbelievaBoat's
   numbers on top of existing balances instead of replacing them.
5. Revoke the UnbelievaBoat token afterwards.

---

## Whitelist — who may use the bot

Only members with a whitelisted role can run any command, and only they ever
get an economy account. Everyone else gets a short private note and stays
completely outside the economy (no balance, no leaderboard entry, nobody can
pay them).

Set it up in Discord — no redeploy needed:

```
/eco-admin whitelist add role:@Whitelisted
/eco-admin whitelist show
/eco-admin whitelist remove role:@Whitelisted
```

Config knobs in `config.py`:

| Setting | Default | What it does |
|---|---|---|
| `WHITELIST_ENABLED` | `True` | Turns the whitelist off entirely |
| `WHITELIST_ROLE_IDS` | `[]` | Fallback list for hard-coding roles in the file |
| `WHITELIST_STAFF_BYPASS` | `True` | Economy staff and Administrators skip the whitelist |
| `WHITELIST_MESSAGE` | — | The private note a non-whitelisted member receives |

The server owner always bypasses the whitelist, so a misconfigured list can
never lock the server out of its own bot.

---

## Troubleshooting: "the commands don't show up for normal members"

Slash commands appearing for admins but not for regular members is a **Discord
permission setting**, not a bot setting — the bot registers every player
command visible to everyone by default. Check these, in order:

1. **Server Settings → Integrations → (your bot) → Manage.** This page can
   restrict commands to specific roles or channels. If anything is limited
   here, set the top-level entry back to `@everyone` and allow `All channels`,
   then restrict individual commands only if you want to.
2. **Role permission: "Use Application Commands".** If it is denied on the role
   (Server Settings → Roles) or on the channel (channel → Edit Channel →
   Permissions), members with that role cannot see any slash commands there.
   Administrators bypass it, which is exactly why it looks like an admin-only
   problem.
3. **Channel overwrite for the bot.** The bot also needs *View Channel* and
   *Send Messages* in the channel.
4. **Was the bot invited with the `applications.commands` scope?** If not,
   re-invite it from the Developer Portal with both `bot` and
   `applications.commands` ticked. No data is lost by re-inviting.

The `/eco-admin channels add` restriction does **not** hide commands — a player
who runs a stock command in the wrong channel still sees the command and gets a
private note pointing at the right channel.
