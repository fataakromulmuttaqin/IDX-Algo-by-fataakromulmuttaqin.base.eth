# IDX-Algo-by-fataakromulmuttaqin.base.eth
A TradingView Pine v5 strategy that combines momentum (RSI), intraday trend anchor (VWAP), higher-timeframe trend (1-hour EMA), and volume confirmation (Relative Volume + Capital Flow) — tailored for the Indonesian Stock Exchange (IDX), but adaptable to other markets.
Features

RSI (14) + VWAP daily confirmation

1-hour EMA trend filter to avoid counter-trend trades

Volume confirmation using Relative Volume (RVOL) and a cumulative money-flow trend (inflow/outflow)

Intraday session filter (09:00–15:00 WIB) for IDX

Alerts for Long/Short signals (can be used with webhooks)

On-chart visible volume panel (RVOL + inflow/outflow histogram) and a live dashboard with strategy metrics

Plots strategy equity on the chart

Files

idx-rsi-vwap-volume-strategy.pine — main Pine v5 strategy file (paste into TradingView Pine editor)

README.md — documentation (this file)

LICENSE — recommended MIT license

Quick start

Open TradingView → Pine Editor → New script.

Paste idx-rsi-vwap-volume-strategy.pine.

Save and Add to Chart on an IDX ticker (e.g., BBRI.JK or BBCA.JK).

Recommended timeframe: 15-minute (default settings tuned for IDX).

Open Strategy Tester to view metrics. Add Alerts from the script alertcondition names if you want notifications.

Recommended settings for IDX

Timeframe: 15m (30m for slower stocks)

RSI: 14; long level = 58; short level = 42

RVOL threshold: 1.1–1.4 (1.2 default)

EMA(1H): 50

Limitations

Strategy uses daily VWAP — works best on intraday charts.

On-chart volume panel is rendered inside overlay (not a separate pane). If you prefer a true lower pane, split into two scripts (strategy + indicator).

TradingView’s strategy backtest fills and real order execution differ — paper-test in live conditions.

Script uses request.security() for the 1-hour EMA — be mindful of lookahead and historical loading.

License
MIT — feel free to fork, improve, and contribute!

Explanation: section by section

Below is the conceptual map + annotated explanation so you (and contributors) can understand and evolve the code.

File header and strategy declaration
//@version=5
strategy("IDX RSI+VWAP+VOL Strategy — Visible Volume Panel + Dashboard (Fixed)", overlay=true,
     default_qty_type = strategy.percent_of_equity, default_qty_value = 2,
     initial_capital  = 10000000, pyramiding = 1)


Declares this as a strategy script (so TradingView's Strategy Tester can run it).

overlay=true draws plots on the price pane.

Default position sizing = 2% of equity per trade.

initial_capital set to 10,000,000 (IDR assumed — user can change).

pyramiding=1 allows 1 position of the same direction (no multiple stacked entries).

Inputs (user-tunable parameters)
rsiLen = input.int(14, "RSI Length")
rsiLongLevel = input.int(58, "RSI Long Level (Buy >)")
rsiShortLevel = input.int(42, "RSI Short Level (Sell <)")
rrMultiplier = input.float(1.8, "R:R Target", step=0.1)
emaLenHTF = input.int(50, "EMA(1H) Trend Filter Length")
sessionFilter = input.bool(true, "Trade only 09:00–15:00 WIB")
sessionStr = "0900-1500:1234567"
...
rvolPeriod = input.int(20, "Relative Volume Period")
rvolThreshold = input.float(1.2, "Relative Volume Min (x avg)")
flowFast = input.int(5, "Capital Inflow Short SMA")
flowSlow = input.int(20, "Capital Inflow Long SMA")


Exposes the important parameters for tuning. Keep these adjustable so users can optimize per-symbol.

Core indicators / signals calculation
rsiVal  = ta.rsi(close, rsiLen)
vwapVal = ta.vwap(hlc3)
emaHTF  = request.security(syminfo.tickerid, "60", ta.ema(close, emaLenHTF))
trendUp   = close > emaHTF
trendDown = close < emaHTF


RSI (momentum), VWAP (daily intraday anchor), and a 1-hour EMA fetched via request.security provide the three price-based confirmations.

trendUp and trendDown are boolean flags used to allow/disallow trades based on the 1-hour trend.

Volume analytics — RVOL and Capital Flow
v_sma = ta.sma(volume, rvolPeriod)
rvol = v_sma == 0 ? 0.0 : volume / v_sma

moneyFlow = ((close - low) - (high - close)) / (high - low + 1e-9) * volume
cumFlow   = ta.cum(moneyFlow)
flowShort = ta.sma(cumFlow, flowFast)
flowLong  = ta.sma(cumFlow, flowSlow)
flowDiff  = flowShort - flowLong
flowTrendUp = flowShort > flowLong
flowTrendDn = flowShort < flowLong
volInflow  = (rvol > rvolThreshold) and flowTrendUp
volOutflow = (rvol > rvolThreshold) and flowTrendDn


Relative Volume (RVOL) = current bar volume / average volume over rvolPeriod. RVOL > threshold indicates unusually large participation.

Money Flow approximates buy vs sell pressure weighted by volume (not the official MFI indicator, but similar spirit).

cumFlow accumulates moneyFlow so we can smooth and check trend via short/long SMAs (flowShort and flowLong).

flowTrendUp means cumulative money flow is in a rising regime (buying pressure).

volInflow only true when both volume spike and inflow trend align.

Session filter
inSession = not sessionFilter or (time(timeframe.period, sessionStr) != na)
bgcolor(inSession ? color.new(color.green,95) : color.new(color.red,97))


Ensures trades occur only during IDX trading hours (09:00–15:00 WIB) when sessionFilter is true.

time(...) != na is the correct boolean check for being inside session.

Entry logic — combining the 4 confirmations
longSignal  = ta.crossover(rsiVal, rsiLongLevel) and close > vwapVal and trendUp and inSession and (rvol > rvolThreshold) and flowTrendUp
shortSignal = ta.crossunder(rsiVal, rsiShortLevel) and close < vwapVal and trendDown and inSession and (rvol > rvolThreshold) and flowTrendDn


Long entry requires: RSI crossover, price above VWAP, HTF trend Up, in session, RVOL filter, and money flow rising.

Short entry is the mirror.

Strategy execution (entries, stops, TP)
signalBarLow  = low[1]
signalBarHigh = high[1]
eps = syminfo.mintick

if (longSignal)
    strategy.entry(longId, strategy.long)
    stopP = math.min(signalBarLow, open - eps)
    tp = open + (open - stopP) * rrMultiplier
    strategy.exit(longId + "_exit", from_entry=longId, stop=stopP, limit=tp)

if (shortSignal)
    strategy.entry(shortId, strategy.short)
    stopS = math.max(signalBarHigh, open + eps)
    tpS = open - (stopS - open) * rrMultiplier
    strategy.exit(shortId + "_exit", from_entry=shortId, stop=stopS, limit=tpS)


Entries are placed at market when signal is true (it detects on the close of the signal candle and places entry on the same bar — you can change to next-bar entry if you want).

Stop loss uses the signal candle low (long) or high (short) — clipped slightly with eps.

Take profit is R:R * risk.

Note: in Pine Script strategy.entry on the same bar may behave differently in backtest live; many prefer enter next bar logic — consider moving to barstate.isconfirmed or using strategy.order() with limits.

Visuals (plots and shapes)
plot(vwapVal, "VWAP", color=color.orange, linewidth=2)
plot(emaHTF,  "1H EMA", color=color.new(color.teal, 60))
plotshape(longSignal, title="Long", style=shape.labelup, color=color.new(color.green,0), text="BUY", location=location.belowbar)
plotshape(shortSignal, title="Short", style=shape.labeldown, color=color.new(color.red,0), text="SELL", location=location.abovebar)


Plots VWAP and 1H EMA on price pane and prints signal labels.

Visible volume panel (overlaid)
scaleFactor = 100000.0
flowPlot = flowDiff / scaleFactor
volColor = volInflow ? color.new(color.lime, 0) : volOutflow ? color.new(color.red, 0) : color.new(color.gray, 70)
plot(flowPlot, title="FlowDiff (scaled)", style=plot.style_columns, color=volColor, transp=40)
plot(rvol, title="Relative Volume", color=color.new(color.blue, 0), linewidth=1)
hline(rvolThreshold, "RVOL Threshold", color=color.new(color.gray, 80), linestyle=hline.style_dotted)


Because a single script can't create a separate lower pane when overlay=true, this draws a scaled histogram and RVOL line at the bottom of the price pane.

scaleFactor scales the money flow so it fits visually — you can adjust or compute dynamically (e.g., scale by recent ATR or max flow).

Equity line & Dashboard
plot(strategy.equity, title="Equity Curve", color=color.new(color.yellow, 0), linewidth=2)
var table dash = table.new(position.top_right, 2, 6, ...)
table.cell(...)  // shows Trades, Win%, Profit Factor, Net Profit, Equity


Plots strategy.equity for a live equity curve on the chart.

The table shows built-in metrics using strategy.* variables (closed trades, netprofit, grossprofit, grossloss, etc.).

Info label
if barstate.islast
    infoText = "RSI+VWAP+VOL IDX | RVOL>" + ...
    label.new(bar_index, high, infoText, ...)


Displays a short text on the last bar summarizing settings (handy for screenshots and docs).

Usage & Publishing Checklist for GitHub

Create a repository (e.g., idx-rsi-vwap-volume-strategy).

Add files:

README.md (use the suggested content above)

idx-rsi-vwap-volume-strategy.pine — the exact Pine script code.

LICENSE — use MIT (text below).

CONTRIBUTING.md (optional): explain PR process and coding style (keep inputs, clean variable names, keep comments).

Suggested initial commit message:

Initial commit — add Pine v5 IDX RSI+VWAP+Volume strategy, README, MIT license


Create useful tags: pine, tradingview, idx, strategy, volume, vwap, rsi.

Add sample screenshots (chart with signals + volume panel) to docs/screenshots/.

Suggested LICENSE (MIT)
MIT License

Copyright (c) 2025 <Your Name>

Permission is hereby granted, free of charge, to any person obtaining a copy...


(Use full MIT text.)

Troubleshooting & FAQ

Q: Script throws errors about session/time or table.
A: Ensure you’re using Pine v5. The session check uses time(...) != na. If TradingView points to a line number, paste that line in an issue.

Q: Signals appear on the same bar — how to enter next open?
A: To enter at next bar open, wrap entry logic: detect signal on close, then set signalFlag := true and if signalFlag and barstate.isconfirmed then strategy.entry() on the next bar. Or use indexing like longSignalPrev = longSignal[1] when executing.

Q: Plots overlapping or scale bad for some tickers.
A: Adjust scaleFactor for flowDiff or compute a dynamic scale: e.g. divide by ta.atr(14) * sma(volume,20).

Q: Want separate lower pane for volume?
A: Split into two files: strategy.pine (overlay=true) and volume_indicator.pine (overlay=false) — keep identical RVOL/flow logic; use both on the chart.

Next steps / ideas for contributors

Add input for enter_on_next_bar vs immediate entry to control signal execution timing.

Add position sizing as percent-of-equity risk per trade (currently default percent_of_equity is used).

Improve money flow: implement Chaikin Money Flow or on-balance volume (OBV) as options.

Add backtest meta-outputs to CSV export (via label.new or array dumps) for offline analysis.

Add unit tests / small backtest harness and example output images.

Suggested README snippet for usage example (paste into README)
## Example: quick start with BBRI.JK
1. Open TradingView and load `BBRI.JK`.
2. Set timeframe to `15` (15-minute).
3. Open Pine Editor → New script → paste `idx-rsi-vwap-volume-strategy.pine` → Save → Add to Chart.
4. Open Strategy Tester and set "From" date range (e.g., last 6 months).
5. Inspect dashboard (top-right), equity plot, and lower volume panel.
6. Tweak `rvolThreshold` (1.0–1.4) and `rrMultiplier` (1.5–2.0) to improve PF.

Quick annotated header comment you can include in the Pine file

At the top of the Pine script include a short block comment:

/*
IDX RSI + VWAP + Volume-Smart Strategy (Pine v5)
Author: <Your Name> — MIT License
Description: Strategy combining RSI momentum (14), VWAP daily confirmation, 1H EMA trend filter,
             Relative Volume (RVOL) & Capital Inflow trend confirmation. Includes on-chart RVOL / inflow histogram and a live dashboard.
Recommended TF: 15m (IDX), RVOL threshold 1.2, EMA1H=50, R:R 1.8.
Usage: Paste into TradingView Pine Editor, Save, Add to Chart, open Strategy Tester.
*/
