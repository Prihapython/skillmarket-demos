# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## Read this first

Before doing anything else in a new session, read in this order:

1. **`CONTEXT.md`** — where things stand right now and why. Newest facts win over
   older sessions' assumptions; if something here conflicts with `CONTEXT.md`,
   trust `CONTEXT.md`.
2. **`NEXT.md`** — open theories, what's pending measurement, what not to
   conclude yet from small samples.
3. **`WALLET_PLAN.md`** — LIVE wallet mechanics, real-trade findings.
4. `STRATEGY.md` — full entry/exit filter logic. `TESTING_PLAN.md` — methodology
   for tuning thresholds empirically. `SPEC.md` — full technical spec (Chinese).
   `SUMMARY.md` — chronology of past sessions.

Don't re-derive conclusions these docs already reached. Don't re-litigate a
threshold that was already set from data — check `TESTING_PLAN.md`'s method
before proposing a new one on gut feeling.

## What this is

A pump.fun (Solana) memecoin screener + trading bot. FastAPI backend
(`app.py`, one file, no framework beyond FastAPI/uvicorn), static HTML/JS
frontend (`static/index.html`), no database — all state is JSON on disk under
`outputs/` (`positions.json`, `trade_decisions.jsonl`, `auto_trade_state.json`,
`trending_cmds.json`, …).

Real Solana market data and real trading go through `gmgn-cli` (npm global
package), invoked via `subprocess`. There is no ORM, no migrations, no test
suite — verification is empirical (SHADOW paper trades, then real trades,
then compare).

**This is a real-money trading bot in production**, not a demo. Treat config
changes that affect position sizing, stop/take-profit levels, or slippage as
production changes to financial logic, not just code edits.

## Security — never expose

- **GMGN API key / signing key** — lives in `~/.config/gmgn/.env` on both the
  dev machine and the server, `chmod 600`. Never `cat`, log, or echo this file
  or its contents. If you need to verify it's set, check presence/length, not
  the value. A previous leak (via a chat screenshot) required a full key
  rotation before LIVE trading could resume.
- **Telegram bot token / chat ID** — `~/.config/gmgn/telegram.env` on the
  server, separate file so it survives GMGN key re-saves from the UI (which
  overwrite the whole `.env`). Same rule: never print its contents.
- **Solana wallet private key** — never touches this server at all. The
  `.env` only holds an Ed25519 key that signs GMGN API requests; the actual
  wallet key stays in Phantom. Don't conflate the two when discussing "the
  key."

## Commands

### Run locally

```sh
pip install -r requirements.txt          # fastapi + uvicorn, that's it
npm install -g gmgn-cli                  # only needed for LIVE-mode data/trading
uvicorn app:app --host 127.0.0.1 --port 8000
# or: python app.py
```
Binds to `127.0.0.1` only by design. Defaults to a mock adapter + SHADOW
(paper) mode — no GMGN key needed to work on the frontend/pipeline logic.

### Deploy (real server, real money)

```sh
git push fork main                        # fork = Prihapython/skillmarket-demos
ssh aitrader@62.238.58.73                  # NOT root
cd /home/aitrader/skillmarket-demos && git pull && sudo systemctl restart aitrader
```

- Push to **`fork`**, never `origin` (`GMGNAI/skillmarket-demos` — no push
  rights, upstream demo repo).
- **Before restarting the service**, check `GET /api/status` → `live_positions`.
  A restart with an open real position is *usually* safe (mode forces back to
  LIVE on boot if positions exist) but confirm first rather than assume.
- `sudo` is scoped to exactly `systemctl {start,stop,restart,status,is-active}
  aitrader` for the `aitrader` user — no other sudo rights, and adding flags
  that change the command shape (e.g. piping through `head`) can fail to match
  the sudoers rule.
- After any deploy, re-check `/api/status` (`build` field shows
  `<commit>/<time>`) to confirm the running code matches what you pushed.

### Check current state without touching anything

```sh
curl -s http://127.0.0.1:8000/api/status         # mode, auto_trade, position size, build
cat outputs/positions.json                        # open positions (should be [] most of the time)
tail -n 5000 outputs/trade_decisions.jsonl | grep '"action": "\(BUY\|SELL\)"'
```
The log file grows fast (500MB+, millions of lines) — never read it from the
start; tail a bounded window or grep on it.

## Workflow

- **One machine = one mode.** The three modes (DATA / MANUAL-LIVE / AUTO-LIVE)
  are mutually exclusive per running process — never enable `auto_trade` on
  both local and server at once (duplicate/conflicting real trades).
- **Before flipping any stateful toggle** (mode, `auto_trade`) on an
  already-running process — local or server — check whether a browser tab is
  already open and polling `/api/run`/`/api/status`. Enabling something for a
  "quick test" affects every already-connected client immediately, not just
  your own call.
- **Never hand-mutate `outputs/positions.json` or call `save_positions()`** in
  an ad-hoc test script against the real file without restoring it afterward —
  this has broken real position tracking before.
- **Thresholds and gates come from data, not intuition.** See
  `TESTING_PLAN.md`. Don't tune a filter by eyeballing trades that already
  happened and calling it fixed — that's look-ahead bias on the same sample
  the threshold will be judged against. Change → accumulate fresh trades →
  then evaluate.
- **Don't conclude from 1–3 observations either direction.** This strategy is
  high-variance (a majority of small stop-outs offset by occasional large
  winners); a handful of trades proves nothing about the long-run edge.

## Architecture

Pipeline (screener, not an ML model — deterministic rules + one rule-based
"LLM-judge" stage, no actual LLM call in the hot path):

```
trending (cheap) → top-N prefilter → per-token due diligence (top-N only)
  → deterministic hard gates (rug/consensus, run first — cheapest rejects)
  → 0–99 composite score → rule-based "LLM judge" explains survivors
  → candidate surfaced (auto-buy only if every auto-entry gate also passes)
```

Exit strategy (see `STRATEGY.md` for the full rationale): hard stop -35%
before first take-profit → +20% sells 30% + stop moves to breakeven →
+50% sells another 30% → remainder managed by an uncapped trailing stop
(`stop = max(breakeven, peak_gain_pct − 25 percentage points)`). In LIVE,
the take-profit ladder is a native GMGN condition order; the
breakeven/trailing stop is a separate absolute-price order our own loop
keeps re-arming, because GMGN's `drawdown_rate` is a % of peak while ours is
flat percentage points off peak — different math, can't share one order.

Telegram trade notifications (LIVE only): yellow = entry, green = TP1/TP2
fired (deliberate profit-take), red = any stop-type exit (trailing-stop *or*
hard stop-loss) regardless of PnL sign — red means "the stop closed this,"
not "we lost money." Don't reintroduce a `(LIVE)` suffix or PnL-conditional
color on the trailing-stop message; both were deliberately removed/fixed.

## Coding practices

- Comments in this file are bilingual/mixed (Chinese from earlier sessions,
  Ukrainian from later ones) — match whichever convention surrounds the code
  you're editing rather than picking a third language.
- This is one large file by design so far (`app.py`) — don't split it into a
  package as a "cleanup" unless asked; several past sessions have found and
  fixed real bugs precisely by grepping this single file for whether a
  computed signal is actually read anywhere, which is easier in one file.
- **Recurring bug-finding pattern worth trying first**: for any "why did/didn't
  it buy/sell/flag X" question, grep whether the field in question is
  literally referenced anywhere besides where it's computed/displayed. Several
  real incidents were a signal that was computed and shown but never wired
  into a gate.
- No test suite exists. Verification is: change → deploy to SHADOW or a
  bounded LIVE test → read the resulting `trade_decisions.jsonl` entries
  and/or on-chain activity (`gmgn-cli portfolio activity`) → compare to
  expectation. Prefer reading real on-chain data (`portfolio activity`, `order
  strategy list --type history`) over trusting our own logged `pnl` when the
  two could disagree — the log has had real accounting bugs before that
  on-chain data would have caught immediately.
