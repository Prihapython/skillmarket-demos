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
- Do not log or print sensitive fields anywhere, including ad-hoc debug
  prints during a session — same rule as above, no exceptions for "just
  checking."

### Never hardcode secrets

- Credentials (GMGN key, Telegram token) belong only in the `.env` files
  above — never inline in `app.py` or committed alongside code, even
  temporarily "to test something."
- Don't change `.gitignore` to make either `.env` file trackable.

### Never modify without explicit instruction

- `requirements.txt` pins — version bumps must be deliberate and reviewed,
  not a side effect of "fixing" an unrelated bug.
- The `.env` files themselves (content, not just presence checks).

### Never run destructive operations without confirmation

- `git reset --hard`, `git push --force` to `fork`/`main`, or any command
  that rewrites shared history.
- Deleting or truncating anything under `outputs/` (`positions.json`,
  `trade_decisions.jsonl`, state files) — see the "reset ≠ delete" note
  under Workflow below.

### Other restrictions

- Do not disable or weaken the bind-to-`127.0.0.1`-only default, or any
  check that gates LIVE mode, without it being the explicit point of the
  task.

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
- **"Reset win-rate stats" ≠ delete the log.** The reset button only moves the
  stats-epoch pointer forward; `trade_decisions.jsonl` itself keeps every row.
  Don't reach for an "archive + truncate" flow to "clean up" the log — that
  destroys data needed for the threshold-tuning analysis above.
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

### Exceptions & error handling

- Never use bare `except:` or catch `BaseException` — catch `Exception` or a
  specific type, and either handle it or re-raise. A silently swallowed
  exception inside the position-monitoring loop can hide a failed sell or a
  stop-loss that never got re-armed, which is a real-money problem, not just
  a bug.
- Where a `try/except` already narrows to "log and continue" (e.g. a failed
  `gmgn-cli` call for one candidate shouldn't kill the whole scan), keep that
  pattern — the intent is isolating one bad token, not hiding real failures.

### Logging

- All trade/decision logging already goes through `log(action, symbol,
  reason, extra_dict)`, which writes one JSON line per event with whatever
  structured fields you pass in `extra_dict` — follow this existing
  convention for new fields rather than inventing a second logging path or
  free-form string concatenation.
- No bare `print()` in code that ends up committed — on the server, only
  what goes through `log()` (or an explicit `send_telegram`) is visible
  after the fact; a `print()` just vanishes into the systemd journal noise.

### General style

- Prefer guard clauses and early returns over deeply nested `if` — most of
  the gate/scoring functions in `app.py` already read this way.
- Absolute imports only; keep imports at the top of the module.
- State that needs to survive a restart lives in the global `ST` object
  (`AppState`) and gets persisted via the matching `load_*`/`save_*`
  functions in `outputs/*.json` — that's the existing pattern, not
  incidental global state to "clean up." A new piece of persistent state
  belongs on `ST` plus a save/load pair, not a fresh ad-hoc module-level
  variable that won't survive a deploy.
- Modern type hints (`X | Y`, built-in generics) where you're already
  touching a function — not a mandate to retrofit the whole file in one pass.
- Prefer keyword arguments once a function has more than ~2 positional
  params — this file has several gate/scoring functions where positional
  args have caused mixed-up-argument bugs before.
