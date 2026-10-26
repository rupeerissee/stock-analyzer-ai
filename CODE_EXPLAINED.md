# Code explained: AI Stock Analyzer, section by section

## 0. The big picture

```
You type a name  ->  resolve_symbol()  ->  download 2 years of prices + company info (yfinance)
                                              |
        +-------------------------------------+------------------------------------+
        |                                     |                                    |
 analyze_fundamentals()            compute_indicators() +               news_sentiment()
 (company quality, 0-100)          analyze_technicals() (chart, 0-100)  (headline keywords)
        |                                     |                                    |
        +-------------------> combined score = 40% fundamental + 60% technical +/- 2 news
                                              |
                                          decide()  -> verdict
                                              |
                                      build_trade_plan() -> entry, stop, targets, quantity
                                              |
                          exit rules + chart + optional Claude summary
```

---

## 1. Imports and settings

- `yfinance` downloads prices and company data from Yahoo Finance. `pandas` / `numpy` do the maths.
- `warnings.filterwarnings("ignore")` hides noisy warnings.
- Settings you can change: `CAPITAL`, `RISK_PER_TRADE_PCT`, `ATR_STOP_MULT`, `WEIGHT_FUNDAMENTAL`, `WEIGHT_TECHNICAL`, `SAVE_CHART`.
- Note: `CAPITAL` is used in the **stock's own currency**. For a US stock it means dollars.

---

## 2. Small helper functions

| Function | What it does |
|---|---|
| `num(x)` | Safely turns a value into a float. Returns `None` if it is missing, NaN or infinite. Used everywhere so missing data never crashes the code. |
| `fmt(x)` | Formats a number with commas, or `N/A` if missing. |
| `pct(x)` | Turns 0.15 into `15.0%`. |
| `big_money(x, currency)` | For INR shows `Rs 12,345 Cr` (1 crore = 10,000,000). For others shows T / B / M. |
| `line()`, `title()` | Print the divider lines and section headings. |

---

## 3. `resolve_symbol()`: turn what you typed into a Yahoo symbol

1. Removes spaces and makes it upper case (`tata motors` becomes `TATAMOTORS`).
2. If it already has a dot or starts with `^` (like `RELIANCE.NS`, `^NSEI`), it uses it as it is.
3. Otherwise it tries `.NS` (NSE), `.BO` (BSE), then the plain symbol (US etc.). The first one that returns price data wins.
4. If none works, it uses Yahoo's name search and takes the **first** `.NS` or `.BO` result (or the first result of any kind).

Things to know:
- It does not show you the list. It silently picks the first match, so a wrong match is possible. The README has an optional patch that shows a numbered list.
- A company can have a new ticker after a demerger or rename. If a name fails, check the exact ticker on Yahoo Finance.

---

## 4. `score_block()`: how points become a 0-100 score

Each check gives points (`pts`) out of a maximum (`mx`). The score is:

```
score = (sum of pts + sum of mx) / (2 x sum of mx) x 100
```

- All checks at their best: 100. All at their worst: 0. Total points zero: 50.
- Example: points add up to +6 out of a maximum of 10, so (6 + 10) / 20 x 100 = **80**.
- A check with no data is skipped, so it does not count either way. If very little data exists, the score stays near 50.

---

## 5. `analyze_fundamentals()`: how healthy is the business?

`is_financial` is true if the sector contains "Financial". Banks and NBFCs get adjusted rules for P/B, ROA, debt and current ratio.

| Check | Rule (points) | Max |
|---|---|---|
| P/E (trailing) | under 15: +2, under 25: +1, under 40: 0, 40 or more: -1, loss making: -2 | 2 |
| Forward P/E | below 90% of trailing P/E: +1, above 110%: -1 | 1 |
| PEG ratio | under 1: +2, under 2: 0, 2 or more: -1 | 2 |
| Price / Book | normal companies under 3: +1, over 7: -1. Financials under 1.5: +1, over 3.5: -1 | 1 |
| ROE | over 20%: +2, over 15%: +1, over 8%: 0, else -2 | 2 |
| ROA (not financials) | over 10%: +1, under 3%: -1 | 1 |
| Net profit margin | over 15%: +2, over 8%: +1, over 3%: 0, else -1 | 2 |
| Operating margin | over 20%: +1, under 5%: -1 | 1 |
| Revenue growth (YoY) | over 15%: +2, over 8%: +1, 0 or more: 0, negative: -2 | 2 |
| Earnings growth | over 15%: +2, positive: +1, otherwise -2 | 2 |
| Debt / Equity (not financials) | under 0.5: +2, under 1: +1, under 2: -1, 2 or more: -2 | 2 |
| Current ratio (not financials) | 1.5 or more: +1, under 1: -1 | 1 |
| Free cash flow | positive: +1, negative: -1 | 1 |
| Dividend yield | over 2%: +1 (nothing otherwise) | 1 |
| Analyst target upside | over 15%: +1, below -5%: -1, else 0 | 1 |

Note: Yahoo gives `debtToEquity` as a percentage (45 means 0.45x), so the code divides by 100.

---

## 6. `compute_indicators()`: the chart maths

| Indicator | Meaning |
|---|---|
| SMA 20 / 50 / 200 | Average closing price over 20 / 50 / 200 days (short, medium, long trend). |
| EMA 20 | Like SMA 20, but newer days count more. Used as the "pullback" level. |
| RSI (14) | Momentum from 0 to 100. Under 30 is called oversold, over 70 overbought. Uses Wilder's smoothing. |
| MACD (12, 26, 9) | Difference of two EMAs and its signal line. MACD above signal is bullish. |
| ATR (14) | Average daily price range. A measure of how much the stock normally moves. Used for stop loss and sizing. |
| Bollinger Bands (20, 2) | SMA 20 plus or minus 2 standard deviations. Shows stretched prices. |
| ADX (14), +DI, -DI | ADX measures trend strength (over 25 is strong). +DI above -DI means the trend is up. |
| VOL5, VOL20 | 5-day and 20-day average volume. |

A stock with fewer than 200 trading days will not have SMA 200. The code skips that check instead of failing.

---

## 7. `analyze_technicals()`: score the chart

| Check | Rule (points) | Max |
|---|---|---|
| Price vs 200 DMA | above: +2, below: -2 | 2 |
| Price vs 50 DMA | above: +1, below: -1 | 1 |
| 50 DMA vs 200 DMA | above ("golden"): +1, below ("death"): -1 | 1 |
| Price vs 20 DMA | above: +1, below: -1 | 1 |
| RSI | under 30: +2, 30 to 45: 0, 45 to 65: +1, 65 to 75: 0, over 75: -2 | 2 |
| MACD | fresh bullish cross: +2, above signal: +1, fresh bearish cross: -2, below: -1 | 2 |
| Bollinger | below lower band: +1, above upper band: -1 | 1 |
| ADX | over 25 and +DI above -DI: +1, over 25 and below: -1, else 0 | 1 |
| Volume | 5-day volume over 1.2x the 20-day average: +1 if price rose, -1 if it fell | 1 |

It also finds levels: support and resistance (lowest low and highest high of the last 20 and 60 days) and the 52-week high and low.

---

## 8. `news_sentiment()` and `quarterly_trend()`

- **News:** takes up to 12 recent headlines, counts words from a positive list and a negative list, and labels the result Positive (score above 1), Negative (below -1) or Neutral.
- The words are matched as **parts of words**, so it can be fooled. For example "again" contains "gain", "refined" contains "fine", and "border" contains "order". Treat it as a weak hint. It only changes the final score by 2 points.
- **Quarterly trend:** prints the last 4 quarters of revenue and net profit.

---

## 9. The final score and `decide()`

```
combined = 0.40 x fundamental + 0.60 x technical
           then +2 if news Positive, -2 if Negative (kept between 0 and 100)
```

| Combined | Verdict |
|---|---|
| 70+ | STRONG BUY |
| 58 to 69.9 | BUY |
| 42 to 57.9 | HOLD / WAIT |
| 30 to 41.9 | SELL / AVOID |
| below 30 | STRONG SELL |

Overrides: a BUY with technical score under 40 becomes HOLD / WAIT. A BUY with RSI over 75 becomes WAIT FOR PULLBACK.

Example: fundamental 70, technical 55: 0.4 x 70 + 0.6 x 55 = 28 + 33 = 61, so BUY.

---

## 10. `build_trade_plan()`: entry, stop, targets, size

1. **Entry.** If RSI is above 72, or the price is more than 2.5 x ATR above the 20 EMA, the stock is "stretched". Then the entry is set to the **20 EMA** (wait for a pullback). Otherwise the entry is the current price.
2. **Buy zone:** 20 EMA plus or minus 0.5 x ATR. **Breakout buy:** a close above the 60-day high x 1.002.
3. **Stop loss:** entry minus `ATR_STOP_MULT` x ATR. If the 20-day swing low (minus 0.25 ATR) is above that but still at least 1 ATR below entry, that tighter level is used instead.
4. **Risk per share** = entry minus stop.
5. **Targets:** entry + 1.5 x risk, + 2.5 x risk, + 4 x risk (reward-to-risk of 1.5, 2.5 and 4).
6. **Quantity:** (capital x risk %) divided by risk per share, but never more than your capital can buy.
7. **Trailing stop:** highest close of the last 22 days minus 3 x ATR (a "chandelier exit"). It should only move up.

**Worked example (illustrative numbers):** capital 100,000, risk 1% = 1,000. Entry 500, ATR 10. Stop = 500 - 20 = 480, so risk per share = 20. Quantity = 1,000 / 20 = 50 shares (about 25,000 invested). T1 = 530, T2 = 550, T3 = 580. If the stop is hit you lose about 1,000, which is 1% of capital.

The plan is printed for every verdict except SELL / STRONG SELL, including HOLD / WAIT. For HOLD / WAIT read it as "only if the conditions appear".

---

## 11. Exit rules (printed at the end)

- Close below the stop loss: exit the full position.
- Targets T1, T2, T3: book profit in parts (40%, 30%, rest), and move the stop to entry after T1.
- Daily close below the trailing stop: exit.
- Close below the 50 DMA after a big rally: exit the swing trade.
- RSI above 75 and turning down, or a fresh bearish MACD cross: book profits.
- Bad news (fraud, regulatory action, big profit miss, promoter selling): reduce or exit.

---

## 12. `save_chart()` and `ai_summary()`

- **Chart:** last 180 days with SMA 50 / 200, Bollinger band, horizontal lines for entry, stop and the 3 targets, plus an RSI panel. Saved as `SYMBOL_analysis.png`. In Colab it is saved to the Files panel but not shown unless you add the display lines from the README.
- **AI summary:** runs only if the `ANTHROPIC_API_KEY` environment variable exists. It sends the collected text (fundamentals, technicals, scores, plan) to Claude (`claude-sonnet-5-5`) and prints a short balanced summary: business quality, risks, whether the plan looks sensible, and what would change the view. If the call fails, it prints "AI summary skipped" and carries on.

---

## 13. `main()`: how it runs in Colab

- Colab adds hidden arguments like `-f kernel.json`. The code removes arguments that start with `-` or end with `.json`.
- It detects a notebook through `"ipykernel" in sys.modules`. In a notebook it always uses the interactive loop with `input()`.
- On a normal computer, `python stock_analyzer.py RELIANCE` analyzes once and exits.
- Each analysis is wrapped in `try / except`, so one failure does not stop the loop. Type `q` (or press Enter on an empty line) to quit.

---

## 14. Honest review: things to know and easy improvements

**What works well**
- Safe handling of missing data (`num()` and skipped checks).
- Position sizing based on risk, not on guessing quantity.
- Clear separation: fundamentals, technicals, news, verdict, plan.

**Weak points**
1. **It is rule-based, not AI.** Only the optional summary uses AI. Name it that way on your page.
2. **Thresholds are fixed and not back-tested.** A P/E under 15 is called cheap for every sector, which can be wrong (for example a falling business can also look cheap).
3. **News check is crude.** Substring matching creates false positives and negatives (see section 8).
4. **Capital currency.** `CAPITAL` is in the stock's currency, so a US stock uses dollars. Convert if needed.
5. **Plan for HOLD / WAIT.** It prints entry, stop and targets even though the verdict is not a buy.
6. **Wrong-match risk.** The name search silently takes the first match (fixed by the optional patch in the README).
7. **Data risk.** Yahoo data through `yfinance` is unofficial and can be missing, delayed or rate-limited.
8. **Scores may be based on few metrics.** The tool does not show how many checks it could actually score. A useful addition is printing "metrics used: N".
9. **Short-term vs long-term mix.** Technicals (60%) are short-term signals, fundamentals (40%) are long-term. The blend is a design choice, not a proven method.
10. **No costs or gaps.** Brokerage, taxes, slippage, and overnight gaps over the stop are not modelled.

**Easy upgrades** (if you want them later)
- Show a numbered list of name matches (README patch).
- Display the chart inside Colab (README patch).
- Print "metrics used" next to each score.
- Add a market-direction check (for example Nifty above its 200 DMA) before showing BUY.
- Make thresholds sector-specific.

---

*Educational only. Not investment advice.*
