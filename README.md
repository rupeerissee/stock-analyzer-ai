# AI Stock Analyzer (A to Z)

A simple Python tool you can run in **Google Colab**. Type a stock name or symbol, and it prints:

1. Company overview
2. Fundamental analysis (valuation, profitability, growth, debt, cash flow, analyst view)
3. Technical analysis (trend, RSI, MACD, Bollinger, ADX, ATR, volume, support / resistance)
4. News sentiment (simple keyword check)
5. A combined score and a **verdict** (STRONG BUY / BUY / HOLD / SELL / STRONG SELL)
6. A **trade plan**: buy zone, stop loss, 3 targets, trailing stop, how many shares to buy
7. **Exit rules**: when to sell
8. A chart saved as a PNG
9. Optional: a plain-English summary written by Claude (only if you add an API key)

> **Be clear about what it is:** the scoring is **rule-based** (fixed thresholds). Only the optional step 9 uses AI.
> **Educational tool only. Not investment advice. Not a recommendation to buy or sell any stock.**

---

## Quick start in Google Colab (3 steps)

### Step 1: Open Colab
Go to https://colab.research.google.com and click **New notebook**.

### Step 2: Install the libraries (first cell)
Copy this into the first cell and press the play button:

```python
!pip -q install -U yfinance matplotlib anthropic
```

### Step 3: Paste the code and run (second cell)
1. Click **+ Code** to add a new cell.
2. Copy the **entire** `stock_analyzer.py` code and paste it into this cell.
3. Press play.

It will ask:

```
Enter stock name / symbol:
```

Type for example `RELIANCE`, `TCS`, `tata motors`, `HDFCBANK` or `AAPL` and press Enter.
After the report, it asks again. Type `q` to quit.

**Which symbols work?**

| You type | What the tool tries |
|---|---|
| `RELIANCE` | `RELIANCE.NS` (NSE), then `RELIANCE.BO` (BSE), then `RELIANCE` as it is |
| `tata motors` | spaces are removed, same order as above |
| `AAPL` | not found on NSE or BSE, so it uses `AAPL` (US) |
| `RELIANCE.NS` or `^NSEI` | used exactly as typed |
| A company name that is not a ticker | falls back to a Yahoo Finance name search and takes the first Indian match |

If a name is not found, type the exact ticker as shown on Yahoo Finance.

---

## Run on your own computer

```bash
pip install -U yfinance matplotlib anthropic
python stock_analyzer.py RELIANCE
```

If you do not give a name, it asks for one.

---

## Settings (top of the file)

| Setting | Default | Meaning |
|---|---|---|
| `CAPITAL` | 100000 | Your total trading capital. **It is in the stock's own currency** (rupees for NSE/BSE, dollars for US stocks). |
| `RISK_PER_TRADE_PCT` | 1.0 | Maximum % of capital you risk on one trade. |
| `ATR_STOP_MULT` | 2.0 | Stop loss = entry minus 2 x ATR (average daily range). |
| `WEIGHT_FUNDAMENTAL` | 0.40 | Weight of fundamentals in the final score. |
| `WEIGHT_TECHNICAL` | 0.60 | Weight of technicals in the final score. |
| `SAVE_CHART` | True | Saves a PNG chart. |

---

## Reading the result

| Combined score | Verdict |
|---|---|
| 70 and above | STRONG BUY |
| 58 to 69.9 | BUY |
| 42 to 57.9 | HOLD / WAIT |
| 30 to 41.9 | SELL / AVOID |
| Below 30 | STRONG SELL |

Two safety overrides:
- A BUY becomes **HOLD / WAIT** if the technical score is below 40.
- A BUY becomes **WAIT FOR PULLBACK** if RSI is above 75.

Combined score = 40% fundamental + 60% technical, then +2 or -2 for positive or negative news.

**Important:** the trade plan is also printed for HOLD / WAIT. Read it as "only if the conditions are met", not as an instruction to buy.

---

## Optional: AI summary by Claude

1. In Colab, click the **key icon (Secrets)** on the left.
2. Add a secret named `ANTHROPIC_API_KEY` and paste your key. Turn on "Notebook access".
3. Run this cell **before** the analyzer cell:

```python
import os
from google.colab import userdata
os.environ["ANTHROPIC_API_KEY"] = userdata.get("ANTHROPIC_API_KEY")
```

Never paste your key into the code or share a notebook that contains it.

---

## Optional: show the chart inside Colab

By default the chart is only saved as a file (see the Files panel on the left). To show it directly, find this part inside `analyze()`:

```python
        if fname:
            print(f"\n  Chart saved: {fname}")
```

and change it to:

```python
        if fname:
            print(f"\n  Chart saved: {fname}")
            try:
                from IPython.display import display, Image
                display(Image(fname))
            except Exception:
                pass
```

---

## Optional: show name suggestions to choose from

Add this function above `analyze()`:

```python
def pick_symbol(query):
    try:
        quotes = yf.Search(query, max_results=8).quotes
    except Exception:
        quotes = []
    quotes = [q for q in quotes if q.get("symbol")][:6]
    if not quotes:
        return None
    print("\n  Matches found:")
    for i, q in enumerate(quotes, 1):
        name = q.get("shortname") or q.get("longname") or ""
        print(f"   {i}. {q['symbol']:<14} {name}  ({q.get('exchDisp', '')})")
    choice = input("  Pick a number (press Enter for 1): ").strip()
    idx = int(choice) - 1 if choice.isdigit() and 1 <= int(choice) <= len(quotes) else 0
    return quotes[idx]["symbol"]
```

Then in `analyze()`, change:

```python
    symbol = resolve_symbol(user_input)
```

to:

```python
    symbol = resolve_symbol(user_input) or pick_symbol(user_input)
```

Now, if the name is not an exact ticker, it shows a numbered list and lets you choose.

---

## Troubleshooting

| Problem | What to do |
|---|---|
| `Could not find a stock for ...` | Use the exact Yahoo ticker, for example `RELIANCE.NS`. |
| `Too Many Requests` or empty data | Wait a few minutes and try again. Run `!pip -q install -U yfinance` to get the latest version. |
| `yf.Search` error | Your yfinance is old. Run `!pip -q install -U yfinance` and restart the runtime. |
| Many fundamentals show `N/A` | Yahoo has no data for that stock, or it is an index or ETF. The fundamental score then stays near 50. |
| No chart visible | Open the Files panel in Colab, or use the "show the chart" change above. |
| The prompt does not appear | Run the cell again. The prompt appears under the cell. |

---

## Limits you should know

- Data comes from Yahoo Finance through `yfinance` (unofficial). It can be delayed, missing or wrong.
- The scores use fixed thresholds that have **not been back-tested**. A high score does not mean a stock will go up.
- The news score is a crude keyword count and has only a small effect (plus or minus 2 points).
- Stop loss and sizing ignore brokerage, taxes, and price gaps (for example after results).
- Results are not tuned for every sector. Banks and finance companies get a few adjusted rules, other sectors use the same thresholds.

---

## Disclaimer

This project is for **education only**. It is not investment advice, not a research report, and not a recommendation to buy or sell any security. Markets are risky and you can lose money. Do your own research before you invest.
