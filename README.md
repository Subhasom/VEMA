# VEMA

**A volatility-window indicator for TradingView (Pine Script v6)**

| | |
|---|---|
| **Author** | Subhasom Mandal ([github.com/Subhasom](https://github.com/Subhasom)) |
| **Current release** | 2.1.24092026.0124 |
| **Platform** | TradingView, Pine Script v6, pane indicator that also draws on the price chart |
| **Files** | Indicator: `VEMA.2.1.24092026.0124.pine` |

> ⚠️ **Disclaimer:** VEMA is an analysis tool, not financial advice. It describes volatility that has already happened and projects a statistical range; it does not predict direction. Always test on your own instruments and timeframes, and use proper risk management.

---

## Table of contents

1. [What VEMA does](#1-what-vema-does)
2. [Installation](#2-installation)
3. [Reading the chart](#3-reading-the-chart)
4. [The indicator pane](#4-the-indicator-pane)
5. [How it works: full logic](#5-how-it-works-full-logic)
6. [Parameter reference](#6-parameter-reference)
7. [How to use it: practical guide](#7-how-to-use-it-practical-guide)
8. [Alerts](#8-alerts)
9. [Limitations and honest notes](#9-limitations-and-honest-notes)
10. [Release versioning](#10-release-versioning)
11. [Credits and licensing](#11-credits-and-licensing)
12. [Implementation notes](#12-implementation-notes)

---

## 1. What VEMA does

VEMA finds the stretches of a chart where volatility suddenly expands — the bursts where most of a session's movement happens — and measures them. It combines four ideas:

- **Volatility windows from ATR.** Average True Range is compared with an EMA of itself. When ATR rises well above that baseline, a **high-volatility window** opens; when it falls back, the window closes.
- **Triple confirmation.** A window only opens when ATR expansion is backed by a **MACD histogram impulse** (momentum) and a **Historical Volatility (HV) expansion** (the spread of close-to-close returns). All three measure volatility differently, so agreement filters out noise such as a single wide bar.
- **Window extremes and change.** Every window is shaded on the price chart, its **highest high** and **lowest low** are labelled, and the **% change from start to end** is printed on it. The last window's High and Low stay on the chart as dashed levels.
- **HV expected range to end of day.** From the latest bar, two grey dashed lines project how far price would typically travel by the session close if volatility stays at its recent level.

---

## 2. Installation

1. Open [TradingView](https://www.tradingview.com) and any chart.
2. Open the **Pine Editor** tab at the bottom of the screen.
3. Delete the template code, then paste the full contents of `VEMA.<master>.<release>.DDMMYYYY.HHMM.pine` (currently `VEMA.2.1.24092026.0124.pine`).
4. Click **Save**, then **Add to chart**.

VEMA opens in its own pane below the price chart. The windows, labels, levels and projection lines are drawn on the **price chart** itself from that pane, so there is nothing extra to add.

> 💡 **When updating to a new release**, remove the old indicator from the chart and add the new one. TradingView remembers old settings, so new defaults only apply to a freshly added indicator.

---

## 3. Reading the chart

| Element | Look | Meaning |
|---|---|---|
| **Window box** | Light orange shaded box with a dotted border | A high-volatility window. It spans from the bar where the window opened to its last bar, and from its lowest low to its highest high. |
| **% change** | Text at the top of the box | Change from the **open of the first bar** to the **close of the last bar**, with both prices underneath, e.g. `+42.35%` / `330.50 → 470.60`. Green when positive, red when negative. Shows `(live)` while the window is still open. |
| **H label** | Red label above a candle | The highest high inside the window. |
| **L label** | Green label below a candle | The lowest low inside the window. |
| **Label tooltip** | Hover over H or L | Window length in bars, start → end prices and %, High, Low, and the full Low → High swing in %. |
| **Window levels** | Red and green dashed lines extending right | The High and Low of the **most recent** completed window, extended into the future as reference levels. They move to the newest window when it closes. |
| **HV projection** | Two grey dashed lines from the last bar to the session close | The expected upper and lower price by end of day at the chosen width (default ±1σ). A grey label at the right end gives the price and % distance, e.g. `HV +1σ EOD 512.40 (+4.35%)`. |

> **A window is not a signal.** It marks where volatility expanded and how far price travelled inside it. The direction of the move is only known once it has happened.

---

## 4. The indicator pane

| Element | Look | Meaning |
|---|---|---|
| **ATR / baseline** | Thick line, light grey; red while a window is open | ATR divided by its EMA baseline. 1.0 means normal volatility; 1.5 means ATR is 50% above normal. |
| **HV / baseline** | Thin blue line | HV divided by its EMA baseline. Reacts faster than ATR. |
| **Baseline** | Grey dotted line at 1.0 | Normal volatility. |
| **ATR open level** | Red dashed line (default 1.30) | ATR / baseline must cross above this for a window to open. |
| **ATR close level** | Orange dashed line (default 1.05) | The window closes when ATR / baseline drops below this. |
| **HV confirm level** | Blue dotted line (default 1.30) | HV / baseline must have been above this within the last few bars. |
| **MACD impulse** | Small orange dots along the bottom | Bars where the MACD histogram is unusually large. |
| **Window highlight** | Faint red background | The bars inside an open window. |

Reading the pane tells you *why* a window did or did not open: if the red ATR line is above its open level but no window appears, look for a missing blue HV move or missing orange dots.

---

## 5. How it works: full logic

The pipeline runs on every bar:

```
Price data
   │
   ├─► 1. ATR ratio (ATR ÷ EMA of ATR)
   │
   ├─► 2. MACD impulse (|histogram| vs its average)
   │
   ├─► 3. HV ratio (HV ÷ EMA of HV)
   │
   ├─► 4. Window state machine ──► open / extend / close
   │
   ├─► 5. Window extremes & % change ──► box, H/L labels, levels
   │
   └─► 6. HV projection (last bar only) ──► grey lines to session close
```

### 5.1 ATR ratio

True Range is calculated as in TradingView's standard ATR, smoothed with the chosen method (RMA by default, length 14):

```
ATR   = RMA( TrueRange, 14 )
ratio = ATR ÷ EMA( ATR, 50 )
```

Dividing by its own baseline makes the measure independent of price: the same settings work on a ₹300 option premium and a ₹25,000 index. With **Count session-open gaps in True Range** switched off, the first bar of each session uses High − Low only, so an overnight gap cannot open a window on its own.

### 5.2 MACD impulse

MACD is calculated as in TradingView's standard MACD (12, 26, 9, EMA). The indicator then compares the size of the histogram with its own recent average:

```
impulse = |histogram| > 1.5 × EMA( |histogram|, 50 )
```

Using the absolute value makes it direction-agnostic: a sharp rise and a sharp fall both count. Because ATR lags MACD, the impulse may occur up to **3 bars** before the ATR crosses its open level and still confirm the window.

### 5.3 Historical Volatility

HV is calculated as in TradingView's standard HV script:

```
σ_bar = stdev( ln(close ÷ close[1]), 10 )
HV    = 100 × σ_bar × √(365 ÷ per)          per = 1 intraday/daily, 7 otherwise
HV ratio = HV ÷ EMA( HV, 50 )
```

HV measures the spread of close-to-close returns, while ATR measures bar ranges including gaps; the two disagree on wide-wicked bars that close where they opened. A window needs HV ratio above **1.30** within the last **3 bars**. Because the ratio is used, the annualisation factor has no effect on the signal.

### 5.4 Window state machine

```
Open   when  ATR ratio > 1.30   AND  MACD impulse (last 3 bars)  AND  HV expansion (last 3 bars)
Stay   while ATR ratio ≥ 1.05
Close  on the first bar where ATR ratio < 1.05
```

The open and close levels are deliberately different (hysteresis), so a ratio hovering around one level does not open and close windows on every bar. Only one window can be open at a time. The window starts on the bar where it opened and ends on the last bar that kept it open; the closing bar itself is not included.

### 5.5 Window extremes and % change

While a window is open, every new bar is checked for a higher high or lower low, and the H / L labels move to the new extreme. The start price is the **open of the first bar** and the end price the **close of the latest bar**:

```
% change = ( end − start ) ÷ start × 100
```

When the window closes it is checked against **Minimum window length** (default 3 bars). Shorter windows are deleted along with their labels. Qualified windows keep their drawings, get the full tooltip, and their High and Low replace the previous window's dashed levels.

### 5.6 HV projection to end of day

On the latest bar only, on intraday charts, VEMA counts the bars left in the session and projects the recent per-bar volatility forward:

```
n     = bars remaining until the daily session close
move  = k × σ_bar × √n                       k = Projection width (default 1)
upper = close × e^(+move)
lower = close × e^(−move)
```

This is the standard square-root-of-time scaling for a random walk in log prices. At 1σ, roughly 68% of outcomes would fall inside the range *if* volatility stays at its recent level and returns are roughly normal; at 2σ, roughly 95%. The lines are redrawn on every new bar, so the range narrows as the close approaches.

---

## 6. Parameter reference

### ATR – volatility regime

| Parameter | Default | Description |
|---|---|---|
| ATR length | 14 | Length of the ATR smoothing. |
| ATR smoothing | RMA | `RMA`, `SMA`, `EMA` or `WMA`, as in the standard ATR. |
| Count session-open gaps in True Range | ✅ On | Off = the first bar of each session uses High − Low only. |
| Baseline EMA length (of ATR) | 50 | Length of the EMA that defines "normal" ATR. |
| Open window when ATR > baseline × | 1.30 | Higher = fewer, stronger windows. |
| Close window when ATR < baseline × | 1.05 | Keep below the open level. Lower = longer windows. |

### MACD – momentum confirmation

| Parameter | Default | Description |
|---|---|---|
| Require MACD impulse to open a window | ✅ On | Off = MACD is ignored. |
| Source | close | Price used for MACD. |
| Fast / Slow / Signal length | 12 / 26 / 9 | Standard MACD lengths. |
| Oscillator / Signal MA type | EMA / EMA | `EMA` or `SMA`. |
| Impulse baseline length | 50 | Length of the average \|histogram\| is compared with. |
| Impulse: \|histogram\| > its average × | 1.5 | Higher = only stronger momentum bursts count. |
| Impulse allowed within last N bars | 3 | How early the impulse may come before ATR confirms. |

### Historical Volatility

| Parameter | Default | Description |
|---|---|---|
| Require HV expansion to open a window | ✅ On | Off = HV is ignored for window opening (the projection still works). |
| HV length | 10 | Bars used for the standard deviation of log returns. Also sets σ for the projection. |
| HV baseline EMA length | 50 | Length of the EMA that defines "normal" HV. |
| HV expansion: HV > baseline × | 1.30 | Higher = stricter confirmation. |
| HV expansion allowed within last N bars | 3 | How early the HV expansion may come before ATR confirms. |
| Project HV expected range to end of day | ✅ On | Draw the grey projection lines. |
| Projection width (σ) | 1.0 | 1 ≈ 68% range, 2 ≈ 95% range. |
| Projection line colour | Grey `#9598a1` | Colour of the projection lines and their labels. |

### Window

| Parameter | Default | Description |
|---|---|---|
| Minimum window length (bars) | 3 | Windows shorter than this are removed once they close. |

### Display

| Parameter | Default | Description |
|---|---|---|
| Shade window range on price chart | ✅ On | Draw the window box. |
| Show % change (start → end) on window | ✅ On | Print the % change on the box. Needs the box to be on. |
| Label window High / Low | ✅ On | Draw the H and L labels. |
| Extend last window's High / Low to the right | ✅ On | Keep the latest window's extremes as dashed levels. |
| Highlight window in indicator pane | ✅ On | Faint red background in the pane while a window is open. |
| High marker / negative change | Red `#ff5252` | Colour of the H label, High level and negative % text. |
| Low marker / positive change | Green `#26a69a` | Colour of the L label, Low level and positive % text. |
| Window fill | Orange, 88% transparent | Fill colour of the window box. |

---

## 7. How to use it: practical guide

**Getting started**

1. Add VEMA to an intraday chart with default settings.
2. Scroll back through a few sessions. Each box shows where volatility expanded, how far price travelled inside it, and where its extremes were.
3. Compare the windows with what you would have called "the move" by eye. If windows are too frequent or too short, raise the open level; if obvious bursts are missed, lower it.

**Suggested starting points**

| Situation | Try |
|---|---|
| Too many small windows | Raise **Open window** to 1.4–1.5, or **Minimum window length** to 5. |
| Obvious bursts are missed | Lower **Open window** to 1.2, or switch off one confirmation to see which one is blocking. |
| Windows end too early | Lower **Close window** to 1.0 or 0.95. |
| Every session opens with a window | Turn off **Count session-open gaps in True Range**. |
| Projection range too narrow for your risk | Set **Projection width** to 1.5 or 2.0. |
| Chart looks cluttered | Turn off **Extend last window's High / Low** or the pane highlight. |

**Ways to use the output**

- **Window levels as reference.** The latest window's High and Low mark where the last burst peaked and bottomed; price often reacts around them later in the session.
- **Sizing stops and targets.** The % changes on past windows show how large a typical burst is on this instrument and timeframe.
- **End-of-day expectations.** The grey HV lines give a statistical sense of how much room is left in the session. A target outside the 2σ range needs an unusually volatile finish to be reached.
- **Options premiums.** Premium moves combine the underlying's move with changes in implied volatility and time decay, so windows on an option chart can appear without a proportional move in the future.

---

## 8. Alerts

Create alerts from TradingView's **Alert** menu → condition **VEMA**:

| Alert | Message | Fires when |
|---|---|---|
| HV window opened | `VEMA: high-volatility window opened on {{ticker}}` | ATR, MACD and HV all confirm and a new window opens. |
| HV window closed | `VEMA: high-volatility window closed on {{ticker}} — H/L and % change marked` | A window closes **and** meets the minimum length, so its drawings are kept. |

Short windows that are discarded do not fire the close alert. Note that `alertcondition()` uses the settings saved when the alert was created, so changing an input later does not affect a running alert.

Use **"Once Per Bar Close"** so that a window opened by an unfinished bar, and then cancelled when the bar closes, does not trigger an alert.

---

## 9. Limitations and honest notes

- **ATR lags.** It is a smoothed average, so a window opens a few bars after the burst begins. The first part of the move, and sometimes its true low or high, lies before the window box.
- **Windows describe, they do not predict.** A window says volatility is high, not which way price will go next. The % change is only final when the window closes.
- **Live bars change.** While the latest bar is still forming, a window can open and disappear again, and the H/L labels and % text update on every tick. A window shorter than the minimum length is deleted when it closes. Completed windows on closed bars do not change.
- **The HV projection is a statistical range, not a forecast.** It assumes volatility stays at its recent level and returns are roughly normal. Real intraday returns have fat tails, so moves beyond 2σ happen more often than 5% of the time, especially around news.
- **Projection only during the session.** It is drawn on intraday charts only, and only while bars remain in the session. After the close (for example, late at night after MCX closes) no lines are shown until the next session starts.
- **Session end comes from the exchange.** The number of bars left is taken from the symbol's daily session close, so extended or special sessions follow whatever TradingView reports for that symbol.
- **Drawing limits.** TradingView keeps up to 500 boxes and 500 labels per script; on very long histories the oldest windows are removed.
- **Illiquid contracts.** Deep out-of-the-money options with sparse trades produce flat bars and sudden jumps; ATR and HV ratios can spike on a single print.

---

## 10. Release versioning

Releases follow the format:

```
<master release>.<release version>.<DDMMYYYY>.<HHMM>
```

- **Master release** marks a major generation of the script. It changes only when explicitly bumped, and the release version resets to 1 when it does. Currently **2**.
- **Release version** increases by 1 with every code change.
- Date and time are in IST (UTC+5:30).
- The indicator file is named `VEMA.<master>.<release>.<DDMMYYYY>.<HHMM>.pine`, e.g. `VEMA.2.1.24092026.0124.pine`.

| Release | Date | Summary |
|---|---|---|
| 2.1.24092026.0124 | 24 Sep 2026 | Master release 2, first versioned release. ATR windows confirmed by MACD impulse and HV expansion; window box with H/L labels and start → end % change; last window's High/Low extended as levels; grey HV expected-range projection to the session close. The Bollinger Band early-start and the lowest-low / volume back-extension from development builds are removed; windows start on their confirmation bar. |
| Development (unversioned) | 23–24 Sep 2026 | ATR + MACD window detection; HV confirmation; % change text; back-extension of the window start by lowest low and volume, later by Bollinger Band squeeze/breakout (both since removed). |

---

## 11. Credits and licensing

The ATR, MACD and Historical Volatility calculations follow TradingView's standard built-in scripts.

The **volatility-window detection** with ATR, MACD and HV confirmation, the **window extremes and % change** marking, the **HV end-of-day projection**, the alert set and the Pine v6 implementation are the work of **Subhasom Mandal**.

---

## 12. Implementation notes

The notes below follow the order in which the code runs.

**Inputs.** Grouped as ATR – volatility regime, MACD – momentum confirmation, Historical Volatility, Window and Display.

**Pane plus price chart.** The indicator is declared with `overlay = false`, so its ratios get their own pane. Every drawing meant for the price chart — boxes, labels, lines and projection — is created with `force_overlay = true`, which places it on the main chart from a pane script.

**ATR, MACD and HV.** Each is calculated exactly as in the corresponding TradingView built-in, then normalised by an EMA of itself. `ta.barssince()` gives the "within the last N bars" test for the MACD and HV confirmations. Divisions by a zero baseline return `na`, which fails every comparison, so flat or empty history cannot open a window.

**Window state machine.** A set of `var` variables holds the open window: its start bar, start and end price, High and Low with their bar indices, and handles to its box and labels. Only one window exists at a time. The open, stay and close conditions are evaluated once per bar in a single `if / else if` block.

**Live drawing.** While a window is open, its box, labels and % text are updated on every bar. On the closing bar, windows below the minimum length are deleted; qualified ones get their final text, tooltips and extended levels, and the previous window's levels are deleted.

**HV projection.** Runs only on `barstate.islast` and on intraday timeframes. Bars remaining come from `time_close("D")` minus the current bar's close time, divided by the chart's bar length. The lines and labels use `xloc.bar_time` so they can be placed at the future session-close timestamp, beyond the last bar. They are created once and then moved with `line.set_xy1/xy2` and `label.set_xy`, so they never accumulate.

**Alerts.** `opened` and `qualified` are per-bar booleans set inside the state machine and exposed through `alertcondition()`.

**Header.** The release line in the file header must be updated with every change, following section 10.
