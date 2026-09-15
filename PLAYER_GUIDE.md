# BeamState — Player Guide

Welcome to the server economy. Everything runs on slash commands: type `/` in
any channel and Discord will list them. You don't need to register — the moment
you run your first command, an account is opened for you with **$17,000 in the
bank**.

This guide covers everything a normal player can do. Staff-only commands are
listed at the very end.

---

## The one shortcut worth learning

Anywhere the bot asks for an **amount**, you can type it the short way:

| You type | The bot reads |
|---|---|
| `10k` | 10,000 |
| `1.5k` | 1,500 |
| `2m` | 2,000,000 |
| `$1,500` | 1,500 |
| `all` | everything you have |
| `half` | half of it |

So `/pay user:@Mike amount:25k` and `/pay user:@Mike amount:25000` do exactly
the same thing. Amounts always round **down**, never up.

---

## 1. Your money

**`/balance`** (or just **`/bal`**) — your full financial profile: cash on hand,
bank, savings, outstanding loan and your credit score. Add `user:` to look at
someone else's.

You have three places to keep money, and the difference matters:

| Where | What it's for |
|---|---|
| **Cash** | What you spend. `/pay`, fines and most purchases come out of here. |
| **Bank** | Safe storage. Your loan limit is based on it. |
| **Savings** | Earns **1% interest every 24 hours**, automatically. |

- **`/deposit amount:`** — cash → bank (short version: **`/dep`**)
- **`/withdraw amount:`** — bank → cash (short version: **`/with`**)
- **`/savings deposit amount:`** / **`/savings withdraw amount:`**
- **`/savings info`** — your rate and when the next interest lands

**`/pay user: amount:`** — send cash to another player. Straight from your cash
on hand, so `/withdraw` first if your money's in the bank.

**`/transactions`** — your recent history, in case you're wondering where it all
went.

**`/leaderboard`** — the richest players on the server (cash + bank + savings).

---

## 2. Making money

**`/daily`** — your best regular income. Pays **$4,000–$10,000**, once every
**24 hours**.

Claim it on consecutive days and you build a **streak**: +$500 per day in a row,
growing up to 7 days (so a maxed streak adds $3,500 on top). Miss a day — leave
more than 24 hours past your cooldown — and the streak drops back to 1. Setting
a reminder is genuinely worth it.

**`/work`** — a quick odd job for **$1,000–$3,000**, once an hour. Free money
while you're online; there's no reason not to run it.

**Roleplay jobs** — Police, EMS and DOT are role-based jobs with in-character
duties and staff-paid wages, not a command you clock into.
- **`/joblist`** — every job on the server and what it pays
- **`/jobinfo`** — the job you currently hold

**Other income:** working for a business (see §5), selling shares at a profit
(§6), and whatever staff pay out for roleplay events.

---

## 3. Loans and your credit score

Everyone starts at a credit score of **650** (the range is 300–850).

- Fully pay off a loan **on time**: **+15**
- Pay a loan off **late**: **−25**
- Get fined by police: **−5**

Your score decides whether you can borrow and how much.

**`/loanrequest amount: days:`** — asks staff for a loan. Your limit is based on
your bank balance (up to 5×) scaled by your credit score, minimum $200. Below a
score of **400**, you're refused outright.

**You choose how long you get to pay it back — and that's what it costs you.**
The maximum is **three weeks**, and the bank charges accordingly:

| You want | Interest | Borrow $10,000, repay |
|---|---|---|
| 1 day | 10% | $11,000 |
| 3 days | 10.5% | $10,500 |
| 1 week | 14.5% | $11,450 |
| 2 weeks | 31% | $13,113 |
| **3 weeks** | **60%** | **$16,000** |

A couple of extra days is almost free. Three weeks costs you **six thousand
dollars** on a ten thousand dollar loan. Borrow for the shortest term you can
actually meet — the dropdown shows you the price of each option before you
commit.

Interest is added up front, and the deadline starts when staff approve it.

**`/loanstatus`** — what you still owe and when it's due.
**`/loanpay amount:`** — pay it down from cash. Clearing it **before the
deadline** is the easiest credit boost in the game; clearing it late costs you
25 points and staff get a reminder that you're overdue.

A staff member has to approve the request, so it isn't instant. They see your
balances, your credit score and how you've repaid before, so the way to get a
yes is to ask for a sensible amount over a sensible term. Ask once and wait
rather than spamming it.

---

## 4. Vehicles and the law

**`/garage`** — every vehicle registered to you, with its value and plate. Add
`user:` to view someone else's. Vehicles are registered by staff when you buy
one in character.

**Police** (`/fine`, `/wanted`, `/arrest`) are staff-side, but they land on you:

- **`/fines`** — your unpaid fines. Check after any traffic stop.
- **`/payfine`** — settle one. Unpaid fines don't quietly disappear.
- **`/wantedstatus`** — a player's wanted level, 0–5.
- **`/bounty user: amount:`** — put cash on someone's head. The money leaves
  your account immediately and sits in the server treasury until staff pay it
  out, so don't post one you can't afford.

Every fine costs you 5 credit score points as well as the cash.

---

## 5. Businesses

Registered businesses have their own bank account, staff and tax bill. Staff
create them; after that the owner runs it.

**Anyone can look, in the business info channel:**
- **`/company list`** — every business on the server, ranked by all-time
  revenue, with owner, staff count and balance
- **`/company info business:`** — the full profile: owner, staff roster,
  all-time revenue, balance, tax paid and outstanding, recent activity, and its
  share price if it's listed on the market
- **`/company staff business:`** — who works there and in what role

**Paying a business:**
- **`/paycompany business: amount:`** — pay for goods or services in character.
  This counts as the company's revenue, and is taxed.

**If you own or work at one:**

| Command | What it does |
|---|---|
| `/company balance` | Your business account, revenue and tax owed |
| `/company deposit amount:` | Put your own money in — **never taxed** |
| `/company withdraw amount:` | Take money out to your cash |
| `/company hire user: position:` | Hire someone, or change their position |
| `/company fire user:` | Remove an employee |
| `/company transfer user:` | Hand the whole business to someone else |
| `/tax pay amount:` | Pay the outstanding tax bill |

**Positions**, most senior first: **COO → CFO → Senior Manager → Manager →
Supervisor → Employee**. You can never hire, promote or fire at or above your
own rank, so a COO can appoint a Manager but not another COO.

By default everyone on staff can deposit; only **COO and CFO** can withdraw or
pay tax; **COO and Senior Manager** can manage staff.

**Tax:** business income is taxed at **20%**. Tax owed is held back inside the
account, so you can't withdraw your way out of a tax bill — but money *you*
deposit yourself is untaxed and fully withdrawable. Funding your own company is
free; earning revenue is not.

---

## 6. The stock market

Businesses listed on the market have shares you can trade. All stock commands
only work in the **stock market channel** — the bot will point you to it if you
try elsewhere, and only you will see that message.

- **`/market`** — every listed business and its current price
- **`/stockinfo business:`** — detail and recent price history for one
- **`/topmovers`** — today's biggest gainers and losers
- **`/portfolio`** — what you hold, what you paid, and what it's worth now
- **`/stock buy business: shares:`**
- **`/stock sell business: shares:`**

**Stocks follow the business, not the weather.** If a listed business is also a
real registered business, its share price is driven by **how much money that
business actually makes**. Every 30 minutes the bot checks what it earned and
compares it to what a business that size is expected to earn (about 2% of its
market value per tick): earn more and the stock climbs, earn less and it slides,
earn nothing and it falls. Paying a business with `/paycompany` even nudges its
stock immediately.

Two things follow from that, and they're the whole game:

- A stock that has already run up needs **more** revenue just to hold its price,
  so buying at the top only pays if the business keeps growing.
- A quiet business always drifts back down, no matter how high it got.

`/stockinfo` tells you which businesses back which stocks, roughly how much
revenue each one needs to hold its price, and why it last moved. Read it before
you buy.

**Five more things that will save you money:**

1. **Prices also move on their own** every 30 minutes, drifting up to 4% either
   way, plus whatever staff do to reflect in-character events.
2. **Your own trades move the price.** Buying pushes it up, selling pushes it
   down, and you settle in the middle of the move you cause. Buying and instantly
   selling loses money every time — this is not an arbitrage machine.
3. **Selling has a 30-minute cooldown**, shared across all stocks. Choose your
   moment. If an order is rejected, you get the cooldown back.
4. **You can't trade more than 10% of a company's shares in one order.** Split
   large positions.
5. **Prices can't fall below $0.50**, but that's a long way down from the top.

Shares come out of your **cash**, not your bank.

---

## 7. Getting started — a five-minute routine

1. `/balance` — meet your $17,000.
2. `/daily` — grab it, then once every day from now on.
3. `/work` — and again every hour you're around.
4. `/savings deposit amount:10k` — park what you won't spend and let it earn.
5. `/market` and `/company list` — see what's worth owning before you buy.

Good habits: claim the streak, keep fines paid, clear loans in full, and never
buy a stock purely because it went up this morning.

---

## Common questions

**"This command can only be used in #channel."** Stock and business info
commands are limited to their own channels to keep chat readable. Use the one
the bot links.

**My command failed and I'm on cooldown anyway.** You aren't — rejected `/work`,
`/daily` and stock sells hand the cooldown straight back.

**I typed an amount and it complained.** Use a plain number or shorthand:
`5000`, `10k`, `1.5m`, `all`.

**Someone owes me money.** There's no enforcement command — settle it in
character, and let staff know if someone's scamming.

**The bot's offline.** Nothing is lost; balances are stored and backed up.

---

## Staff-only commands

Listed so you know what exists, not for general use. Most are hidden unless you
have the role.

**Police:** `/fine`, `/wanted`, `/arrest`
**Loans:** `/loanrequests`, `/approveloan`, `/denyloan`
**Economy staff:** `/eco-admin give`, `take`, `reset`, `setjob`,
`credit-adjust`, `givevehicle`, `removevehicle`, `treasury`, `market-channel`,
`log-channel`, `channels add|remove|clear|show`, `/backup-now`
**Market:** `/eco-admin business create|adjust|link|unlink|setowner|delist`
**Businesses:** `/eco-admin company create|delete|addrevenue|setowner|setstaff|info|ledger`
**Owner only:** `/eco-admin reset-economy`, `/migrate-unbelievaboat`
