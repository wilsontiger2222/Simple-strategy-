//@version=5
strategy("SMC: BOS + Liquidity Sweeps + FVG (60–70% WR target)",
     overlay=true, initial_capital=10000, commission_type=strategy.commission.percent,
     commission_value=0.01, slippage=1, calc_on_bar_close=true, pyramiding=0,
     default_qty_type=strategy.percent_of_equity, default_qty_value=10)

//--------------------------- Inputs
rr                = input.float(1.3, "Take Profit RR", minval=0.5, step=0.1)
emaLen            = input.int(200, "EMA Trend Filter Length", minval=20)
useEmaFilter      = input.bool(true, "Use EMA Trend Filter")
pivotLen          = input.int(2, "Swing Pivot Length (Structure)", minval=1, maxval=10)
bosLookback       = input.int(50, "BOS Lookback (bars)", minval=5, maxval=300)
sweepLookback     = input.int(20, "Liquidity Sweep Lookback (bars)", minval=5, maxval=500)
atrLen            = input.int(14, "ATR Length", minval=5)
atrMultFloor      = input.float(1.0, "ATR Floor (if pivot SL invalid)", minval=0.1, step=0.1)
fvgLife           = input.int(60, "FVG Life (bars)", minval=5, maxval=500)
requireFvgMidReclaim = input.bool(true, "Require FVG Midline Reclaim on Tag")
onlySession       = input.bool(false, "Use Session Filter?")
sess              = input.session("0000-2359", "Session")

//--------------------------- Helpers
inSession = not onlySession or not na(time(timeframe.period, sess))
ema200    = ta.ema(close, emaLen)
atr       = ta.atr(atrLen)

// Swings (structure)
ph = ta.pivothigh(high, pivotLen, pivotLen)   // returns float when confirmed, else na
pl = ta.pivotlow(low, pivotLen, pivotLen)
lastSwingHigh = ta.valuewhen(not na(ph), ph, 0)
lastSwingLow  = ta.valuewhen(not na(pl), pl, 0)

// BOS detection (close breaks last swing)
bullBOS = not na(lastSwingHigh) and ta.crossover(close, lastSwingHigh)
bearBOS = not na(lastSwingLow)  and ta.crossunder(close, lastSwingLow)
bullBOSrecent = ta.barssince(bullBOS) <= bosLookback
bearBOSrecent = ta.barssince(bearBOS) <= bosLookback

// Liquidity sweep (sweep and reject)
sweptDown = low < ta.lowest(low[1], sweepLookback - 1) and close > open
sweptUp   = high > ta.highest(high[1], sweepLookback - 1) and close < open

// FVG detection (3-candle imbalance)
// Bullish FVG when current low > high[2]
bullFVGnow = low > high[2]
// Bearish FVG when current high < low[2]
bearFVGnow = high < low[2]

// Track last FVG zones
var float bullFvgLow  = na
var float bullFvgHigh = na
var int   bullFvgBar  = na

var float bearFvgLow  = na
var float bearFvgHigh = na
var int   bearFvgBar  = na

if bullFVGnow
    bullFvgLow  := high[2]
    bullFvgHigh := low
    bullFvgBar  := bar_index

if bearFVGnow
    bearFvgHigh := low[2]
    bearFvgLow  := high
    bearFvgBar  := bar_index

bullFvgActive = not na(bullFvgBar) and (bar_index - bullFvgBar) <= fvgLife
bearFvgActive = not na(bearFvgBar) and (bar_index - bearFvgBar) <= fvgLife

// Tag/reclaim conditions
bullFvgTagged = bullFvgActive and not na(bullFvgLow) and not na(bullFvgHigh) and low <= bullFvgHigh and high >= bullFvgLow
bearFvgTagged = bearFvgActive and not na(bearFvgLow) and not na(bearFvgHigh) and high >= bearFvgLow and low <= bearFvgHigh

bullFvgMid = (bullFvgLow + bullFvgHigh) / 2.0
bearFvgMid = (bearFvgLow + bearFvgHigh) / 2.0

bullReclaimOk = not requireFvgMidReclaim or (bullFvgTagged and close > bullFvgMid)
bearReclaimOk = not requireFvgMidReclaim or (bearFvgTagged and close < bearFvgMid)

// Trend filter
trendUp   = not useEmaFilter or close > ema200
trendDown = not useEmaFilter or close < ema200

// Entry conditions (SMC confluence)
// Long: trend up, recent BOS up or down-sweep, tag/reclaim bullish FVG
longCond = inSession and trendUp and bullFvgTagged and bullReclaimOk and (bullBOSrecent or sweptDown)

// Short: trend down, recent BOS down or up-sweep, tag/reclaim bearish FVG
shortCond = inSession and trendDown and bearFvgTagged and bearReclaimOk and (bearBOSrecent or sweptUp)

//--------------------------- Risk management: SL at pivot with ATR floor
getLongSL() =>
    float sl = na
    sl := not na(lastSwingLow) and lastSwingLow < close ? lastSwingLow : na
    sl := na(sl) ? close - atr * atrMultFloor : sl
    sl

getShortSL() =>
    float sl = na
    sl := not na(lastSwingHigh) and lastSwingHigh > close ? lastSwingHigh : na
    sl := na(sl) ? close + atr * atrMultFloor : sl
    sl

//--------------------------- Entries and exits
var string longId  = "Long"
var string shortId = "Short"

if longCond and strategy.position_size <= 0
    longSL = getLongSL()
    longRisk = close - longSL
    if longRisk > 0
        longTP = close + rr * longRisk
        strategy.entry(longId, strategy.long)
        strategy.exit("Long TP/SL", from_entry=longId, stop=longSL, limit=longTP)

if shortCond and strategy.position_size >= 0
    shortSL = getShortSL()
    shortRisk = shortSL - close
    if shortRisk > 0
        shortTP = close - rr * shortRisk
        strategy.entry(shortId, strategy.short)
        strategy.exit("Short TP/SL", from_entry=shortId, stop=shortSL, limit=shortTP)

//--------------------------- Plots & visuals
plot(ema200, "EMA", color=color.new(color.orange, 0), linewidth=2)

fvgBullColor = color.new(color.lime, 85)
fvgBearColor = color.new(color.red, 85)

plot(bullFvgActive ? bullFvgLow  : na, "Bull FVG Low", color=fvgBullColor, style=plot.style_linebr)
plot(bullFvgActive ? bullFvgHigh : na, "Bull FVG High", color=fvgBullColor, style=plot.style_linebr)
plot(bearFvgActive ? bearFvgLow  : na, "Bear FVG Low", color=fvgBearColor, style=plot.style_linebr)
plot(bearFvgActive ? bearFvgHigh : na, "Bear FVG High", color=fvgBearColor, style=plot.style_linebr)

plotshape(bullBOS, title="Bull BOS", style=shape.triangleup, location=location.belowbar, color=color.new(color.lime, 0), size=size.tiny, text="BOS↑")
plotshape(bearBOS, title="Bear BOS", style=shape.triangledown, location=location.abovebar, color=color.new(color.red, 0), size=size.tiny, text="BOS↓")

plotshape(longCond,  title="Long Setup",  style=shape.circle, location=location.belowbar, color=color.new(color.lime, 0), size=size.tiny, text="L")
plotshape(shortCond, title="Short Setup", style=shape.circle, location=location.abovebar, color=color.new(color.red, 0), size=size.tiny, text="S")

// Alerts
alertcondition(longCond,  title="Long Setup",  message="SMC Long setup (BOS/sweep + FVG tag)")
alertcondition(shortCond, title="Short Setup", message="SMC Short setup (BOS/sweep + FVG tag)")
