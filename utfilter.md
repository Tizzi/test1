
 //@version=5
// UT Bot 2026 x SMC Filter v2
// Based on concepts from LuxAlgo SMC (CC BY-NC-SA 4.0)
// UT Bot logic attribution: QuantNomad / Jahigginsenterprises

strategy("UT Bot 2026 x SMC Filter v2",
     overlay=true,
     max_boxes_count=500,
     max_labels_count=500,
     initial_capital=10000,
     default_qty_type=strategy.percent_of_equity,
     default_qty_value=10,
     commission_type=strategy.commission.percent,
     commission_value=0.04,
     slippage=1,
     pyramiding=0,
     process_orders_on_close=true,
     calc_on_every_tick=false,
     calc_on_order_fills=false,
     use_bar_magnifier=true,
     backtest_fill_limits_assumption=1)

//====================================================
// CONSTANTS
//====================================================

BULLISH_LEG = 1
BEARISH_LEG = 0

BULLISH = 1
BEARISH = -1

GREEN = #089981
RED   = #F23645

//====================================================
// INPUTS
//====================================================

gU = "UT Bot 2026"

buy_key = input.float(
     2.0, "Buy Key Value",
     minval=0.1, step=0.1, group=gU)

buy_atr = input.int(
     300, "Buy ATR Period",
     minval=1, group=gU)

sell_key = input.float(
     2.0, "Sell Key Value",
     minval=0.1, step=0.1, group=gU)

sell_atr = input.int(
     1, "Sell ATR Period",
     minval=1, group=gU)

use_ha = input.bool(
     false, "Heikin Ashi source", group=gU)

gS = "SMC Filter"

swingsLengthInput = input.int(
     50, "Swing Length",
     minval=10, group=gS)

obMax = input.int(
     3, "Max active OB",
     minval=1, maxval=20, group=gS)

obAgeMax = input.int(
     200, "Max OB age",
     minval=1, group=gS)

drawOB = input.bool(
     true, "Draw order blocks", group=gS)

needInternal = input.bool(
     false, "Require internal bias", group=gS)

gR = "Risk Management"

rr = input.float(
     2.0, "Risk Reward",
     minval=0.1, step=0.1, group=gR)

slBuf = input.float(
     0.1, "SL buffer x ATR14",
     minval=0, step=0.05, group=gR)

gV = "Validation"

showSignals = input.bool(
     true, "Show filtered signals", group=gV)

showRawSignals = input.bool(
     false, "Show raw UT signals", group=gV)

//====================================================
// UT BOT ENGINE
//====================================================

// Calculate source and ATR from the same candle type.
// Prevent HA source / normal ATR mismatch.

tickerID = use_ha ?
     ticker.heikinashi(syminfo.tickerid) :
     syminfo.tickerid

[utSrc, atrBuy, atrSell] = request.security(
     tickerID,
     timeframe.period,
     [close, ta.atr(buy_atr), ta.atr(sell_atr)],
     gaps=barmerge.gaps_off,
     lookahead=barmerge.lookahead_off)

lossBuy = buy_key * atrBuy
lossSell = sell_key * atrSell

var float stopLvl = na
var int pos = 0

prevStop = stopLvl[1]
prevPos = nz(pos[1], 0)

if not na(lossBuy) and not na(lossSell)

    if na(prevStop) or prevPos == 0

        // Deterministic initial state.
        stopLvl := utSrc - lossSell
        pos := 1

    else if prevPos == 1

        if utSrc < prevStop
            stopLvl := utSrc + lossBuy
            pos := -1
        else
            stopLvl := math.max(
                 prevStop,
                 utSrc - lossSell)
            pos := 1

    else

        if utSrc > prevStop
            stopLvl := utSrc - lossSell
            pos := 1
        else
            stopLvl := math.min(
                 prevStop,
                 utSrc + lossBuy)
            pos := -1

else
    stopLvl := prevStop
    pos := prevPos

buySignal =
     pos == 1 and prevPos == -1

sellSignal =
     pos == -1 and prevPos == 1

//====================================================
// SMC DATA STRUCTURES
//====================================================

type pivot
    float currentLevel
    float lastLevel
    bool crossed
    int barTime
    int barIndex

type orderBlock
    float barHigh
    float barLow
    int barTime
    int barIndex
    int bias

var pivot swingHigh =
     pivot.new(na, na, false, na, na)

var pivot swingLow =
     pivot.new(na, na, false, na, na)

var pivot intHigh =
     pivot.new(na, na, false, na, na)

var pivot intLow =
     pivot.new(na, na, false, na, na)

var int swingBias = 0
var int intBias = 0

var array<orderBlock> swingOB =
     array.new<orderBlock>()

var array<box> swingOBBox =
     array.new<box>()

var array<float> parsedHighs =
     array.new<float>()

var array<float> parsedLows =
     array.new<float>()

//====================================================
// PARSED CANDLES
//====================================================

atrMeasure = ta.atr(200)

highVolatilityBar =
     not na(atrMeasure) and
     (high - low) >= 2.0 * atrMeasure

parsedHigh =
     highVolatilityBar ? low : high

parsedLow =
     highVolatilityBar ? high : low

parsedHighs.push(parsedHigh)
parsedLows.push(parsedLow)

//====================================================
// MARKET STRUCTURE
//====================================================

leg(int size) =>
    var int currentLeg = 0

    if high[size] > ta.highest(size)
        currentLeg := BEARISH_LEG
    else if low[size] < ta.lowest(size)
        currentLeg := BULLISH_LEG

    currentLeg

getStructure(int size, bool internal) =>

    currentLeg = leg(size)

    if ta.change(currentLeg) != 0

        if ta.change(currentLeg) == 1

            p = internal ? intLow : swingLow

            p.lastLevel := p.currentLevel
            p.currentLevel := low[size]
            p.crossed := false
            p.barTime := time[size]
            p.barIndex := bar_index[size]

        else

            p = internal ? intHigh : swingHigh

            p.lastLevel := p.currentLevel
            p.currentLevel := high[size]
            p.crossed := false
            p.barTime := time[size]
            p.barIndex := bar_index[size]

//====================================================
// ORDER BLOCK STORAGE
//====================================================

storeOB(pivot p, int bias) =>

    if not na(p.barIndex) and
         p.barIndex >= 0 and
         p.barIndex < bar_index

        int startIndex = p.barIndex
        int endIndex = bar_index

        array<float> a = na

        int pIdx = na

        if bias == BEARISH

            a := parsedHighs.slice(
                 startIndex, endIndex)

            if a.size() > 0
                pIdx := startIndex +
                     a.indexof(a.max())

        else

            a := parsedLows.slice(
                 startIndex, endIndex)

            if a.size() > 0
                pIdx := startIndex +
                     a.indexof(a.min())

        if not na(pIdx)

            int back = bar_index - pIdx

            if back >= 0

                ob = orderBlock.new(
                     parsedHighs.get(pIdx),
                     parsedLows.get(pIdx),
                     time[back],
                     pIdx,
                     bias)

                swingOB.unshift(ob)

                if drawOB

                    newBox = box.new(
                         time[back],
                         ob.barHigh,
                         time,
                         ob.barLow,
                         xloc=xloc.bar_time,
                         extend=extend.right,
                         border_color=color(na),
                         bgcolor=bias == BEARISH ?
                             color.new(RED, 80) :
                             color.new(GREEN, 80))

                    swingOBBox.unshift(newBox)

                if swingOB.size() > obMax

                    swingOB.pop()

                    if drawOB and swingOBBox.size() > 0
                        box.delete(swingOBBox.pop())

//====================================================
// DELETE INVALID ORDER BLOCKS
//====================================================

deleteOB() =>

    if swingOB.size() > 0

        for i = swingOB.size() - 1 to 0 by -1

            ob = swingOB.get(i)

            invalidBear =
                 ob.bias == BEARISH and
                 high > ob.barHigh

            invalidBull =
                 ob.bias == BULLISH and
                 low < ob.barLow

            if invalidBear or invalidBull

                swingOB.remove(i)

                if drawOB and swingOBBox.size() > i
                    box.delete(swingOBBox.remove(i))

//====================================================
// STRUCTURE BREAK DETECTION
// No global scalar modification inside function.
//====================================================

displayStructure(bool internal) =>

    pB = internal ? intHigh : swingHigh
    pS = internal ? intLow : swingLow

    bullBreak =
         not na(pB.currentLevel) and
         ta.crossover(close, pB.currentLevel) and
         not pB.crossed

    bearBreak =
         not na(pS.currentLevel) and
         ta.crossunder(close, pS.currentLevel) and
         not pS.crossed

    if bullBreak
        pB.crossed := true

    if bearBreak
        pS.crossed := true

    [bullBreak, bearBreak, pB, pS]

//====================================================
// UPDATE STRUCTURE
//====================================================

getStructure(swingsLengthInput, false)
getStructure(5, true)

deleteOB()

[iBull, iBear, iBullPivot, iBearPivot] =
     displayStructure(true)

[sBull, sBear, sBullPivot, sBearPivot] =
     displayStructure(false)

// Internal bias

if iBull
    intBias := BULLISH

if iBear
    intBias := BEARISH

// Swing bias and OB creation

if sBull
    swingBias := BULLISH
    storeOB(sBullPivot, BULLISH)

if sBear
    swingBias := BEARISH
    storeOB(sBearPivot, BEARISH)

//====================================================
// POINT OF INTEREST
//====================================================

bool inBull = false
bool inBear = false

float stBull = na
float stBear = na

if swingOB.size() > 0

    for i = 0 to swingOB.size() - 1

        ob = swingOB.get(i)

        validAge =
             bar_index - ob.barIndex <= obAgeMax

        if validAge

            overlap =
                 low <= ob.barHigh and
                 high >= ob.barLow

            if ob.bias == BULLISH and
                 overlap and
                 close > ob.barLow

                inBull := true

                // Choose closest valid bullish stop.
                stBull := na(stBull) ?
                     ob.barLow :
                     math.max(stBull, ob.barLow)

            if ob.bias == BEARISH and
                 overlap and
                 close < ob.barHigh

                inBear := true

                // Choose closest valid bearish stop.
                stBear := na(stBear) ?
                     ob.barHigh :
                     math.min(stBear, ob.barHigh)

//====================================================
// ENTRY FILTER
//====================================================

intBiasOK_L =
     not needInternal or intBias == BULLISH

intBiasOK_S =
     not needInternal or intBias == BEARISH

buf = slBuf * ta.atr(14)

// Require confirmed candle.

longCandidate =
     barstate.isconfirmed and
     buySignal and
     swingBias == BULLISH and
     inBull and
     intBiasOK_L

shortCandidate =
     barstate.isconfirmed and
     sellSignal and
     swingBias == BEARISH and
     inBear and
     intBiasOK_S

//====================================================
// RISK VALIDATION
//====================================================

longStop = stBull - buf
shortStop = stBear + buf

longRisk = close - longStop
shortRisk = shortStop - close

validLongRisk =
     not na(longRisk) and
     longRisk > syminfo.mintick and
     longStop < close

validShortRisk =
     not na(shortRisk) and
     shortRisk > syminfo.mintick and
     shortStop > close

longOK = longCandidate and validLongRisk
shortOK = shortCandidate and validShortRisk

//====================================================
// EXECUTION
//====================================================

// process_orders_on_close = true.
// The close is the assumed signal-bar execution price.
// Actual fills can still differ due to slippage.

if longOK and strategy.position_size <= 0

    longTP = close + longRisk * rr

    strategy.entry(
         "L",
         strategy.long)

    strategy.exit(
         "Lx",
         from_entry="L",
         stop=longStop,
         limit=longTP)

else if buySignal and
     barstate.isconfirmed and
     strategy.position_size < 0

    // Close short only if no valid reversal entry.
    strategy.close(
         "S",
         comment="UT flip")

if shortOK and strategy.position_size >= 0

    shortTP = close - shortRisk * rr

    strategy.entry(
         "S",
         strategy.short)

    strategy.exit(
         "Sx",
         from_entry="S",
         stop=shortStop,
         limit=shortTP)

else if sellSignal and
     barstate.isconfirmed and
     strategy.position_size > 0

    strategy.close(
         "L",
         comment="UT flip")

//====================================================
// VISUALIZATION
//====================================================

plot(
     stopLvl,
     "UT Trail",
     color=pos == 1 ? GREEN : RED,
     linewidth=2)

plotshape(
     showSignals and longOK,
     title="Filtered BUY",
     style=shape.labelup,
     location=location.belowbar,
     color=GREEN,
     text="BUY",
     textcolor=color.white,
     size=size.tiny)

plotshape(
     showSignals and shortOK,
     title="Filtered SELL",
     style=shape.labeldown,
     location=location.abovebar,
     color=RED,
     text="SELL",
     textcolor=color.white,
     size=size.tiny)

plotshape(
     showRawSignals and buySignal,
     title="Raw UT BUY",
     style=shape.circle,
     location=location.belowbar,
     color=color.new(GREEN, 30),
     size=size.tiny)

plotshape(
     showRawSignals and sellSignal,
     title="Raw UT SELL",
     style=shape.circle,
     location=location.abovebar,
     color=color.new(RED, 30),
     size=size.tiny)

//====================================================
// ALERTS
//====================================================

alertcondition(
     longOK,
     "Filtered Long",
     "UT Bot + Bullish SMC confirmed LONG")

alertcondition(
     shortOK,
     "Filtered Short",
     "UT Bot + Bearish SMC confirmed SHORT")
