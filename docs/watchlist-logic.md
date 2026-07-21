# Auto-Watchlist Logic
## IRIS EDGE — Intraday Trade Card System

The auto-watchlist is a separate, rule-based ranking engine from the trade-card
[signal evaluator](signal-rules.md). It scans the full stock universe once per
generation and ranks candidates by a weighted score, rather than gating them
pass/fail. No ML or predictive models are involved.

Source: [`apps/api/app/services/watchlist_scorer.py`](../apps/api/app/services/watchlist_scorer.py)

---

## Purpose

The signal evaluator (see `signal-rules.md`) decides whether a **specific**
symbol currently qualifies for a trade card. The watchlist scorer instead asks,
across the **entire universe**, "which symbols look most interesting right
now?" — producing a ranked shortlist a trader can scan or feed into card
generation.

---

## Pipeline

`generate_auto_watchlist(db, universe)`:

1. Defaults `universe` to `COMMON_STOCKS` (the full tracked symbol list) if none is passed.
2. Deletes any existing watchlist rows for today's date (idempotent re-run).
3. For each symbol: fetch a live quote, run it through `_score()`, skip if score is `0`.
4. Sort all scored symbols descending by score.
5. Keep the top `TOP_N = 30`.
6. Persist each as a `DailyWatchlistModel` row with its rank, score, category, action, reason tags, and an indicator snapshot.

---

## Scoring — weighted additive model

Unlike the signal evaluator's hard-gate/point tiers, every factor here is an
additive weight (can be positive or negative). Starting score is `0`.

| Signal | Condition | Weight |
|---|---|---|
| Gap Up | gap % > 1.5% | **+15** |
| Gap Down | gap % < −1.5% | **+10** |
| Volume Spike | volume ratio > 2.0× | **+20** |
| High Volume | volume ratio > 1.5× (and not a spike) | **+12** |
| RSI Bullish | RSI between 55–73 | **+10** |
| RSI Reversal Zone | RSI between 27–45 | **+8** |
| Bullish EMA Stack | price > EMA21 > EMA50 | **+10** |
| Bearish EMA Stack | price < EMA21 < EMA50 | **+8** |
| MACD Positive | MACD histogram > 0 | **+5** |
| Strong Sector | sector direction = "up" | **+10** |
| Weak Sector | sector direction = "down" | **−5** |
| High ATR | ATR ÷ price > 1.5% | **+10** |
| OR Breakout ↑ | price broke above opening range high | **+15** |
| OR Breakdown ↓ | price broke below opening range low | **+8** |
| Nifty Aligned | Nifty trend agrees with price vs EMA21 | **+5** |
| Low Liquidity | volume < 100,000 shares | **−25** |
| Near Circuit Limit | quote flagged `near_circuit` | **−10** |

Notes:
- Gap, RSI, and EMA checks are mutually exclusive pairs (bullish *or* bearish branch fires, not both).
- Final score is floored at `0` (`score = max(0, score)`).
- A symbol scoring `0` is dropped entirely — it never reaches the ranked list.

---

## Action bias (buy vs. sell)

Tags are split into a bullish set and a bearish set:

- **Bullish tags**: Gap Up, RSI Bullish, Bullish EMA Stack, OR Breakout ↑, MACD Positive
- **Bearish tags**: Gap Down, RSI Reversal Zone, Bearish EMA Stack, OR Breakdown ↓

`action = "buy"` if the count of bullish tags hit ≥ count of bearish tags hit, else `"sell"`. Ties default to `"buy"`.

---

## Category assignment

Applied in priority order — first match wins:

1. **Gap Up Momentum** — has "Gap Up" *and* ("Volume Spike" or "High Volume")
2. **Gap Down Reversal** — has "Gap Down" *and* ("RSI Reversal Zone", "High Volume", or "Volume Spike")
3. **Breakout Candidate** — has "OR Breakout ↑", *or* ("Bullish EMA Stack" *and* volume tag)
4. **High Volume Mover** — has "Volume Spike" or "High Volume" (and none of the above matched)
5. **Strong Momentum** — fallback, nothing else matched

The `/today` endpoint groups the persisted list by this category for display.

---

## Quote fields used

| Field | Used for |
|---|---|
| `price` | base for gap %, ATR %, EMA comparisons |
| `gap_pct` | gap up/down |
| `volume_ratio` | volume spike / high volume |
| `volume` | low liquidity penalty |
| `rsi` | RSI bullish / reversal zone |
| `ema21`, `ema50` | EMA stack direction |
| `macd_histogram` | MACD positive |
| `sector_direction` | sector strength/weakness |
| `atr` | high ATR (as % of price) |
| `or_breakout` | opening range breakout up/down |
| `nifty_direction` | index alignment |
| `near_circuit` | circuit-limit penalty |

Same quote shape and data source (live Upstox / deterministic mock) as the signal evaluator — see `signal-rules.md` → **Quote Data Source**.

---

## Persistence & lifecycle

- One row per symbol per day in `DailyWatchlistModel`, keyed by `date`.
- Re-running `/generate` on the same day wipes and replaces that day's rows.
- Each entry can be individually `dismissed` (hidden from `/today`) or marked `traded`, independent of re-generation.
- `indicator_snapshot` freezes the exact indicator values used for scoring, for later audit/debugging.

---

## Relationship to trade cards

The watchlist and the signal evaluator run independently:

- Watchlist = broad, ranked discovery across the universe (scored, not gated).
- Signal evaluator ([`signal-rules.md`](signal-rules.md)) = strict pass/fail gating for a single symbol to actually generate a trade card.

A symbol appearing high on the watchlist is **not** guaranteed to pass the signal evaluator's hard gates (e.g. it may fail the ATR minimum or land near a circuit breaker) — the two systems use overlapping but differently-weighted logic on purpose.
