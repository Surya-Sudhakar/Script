# Script

//@version=6
indicator("Kronos NY Full 3H Body → Exact 45M Squeeze → Finite Fib v7.3", "Kronos Full-Body Fib v7.3", overlay = true, max_boxes_count = 500, max_lines_count = 500, max_labels_count = 500)

string TZ = "America/New_York"

// Detection
string detectionGroup = "1. Detection"
int lookback3H = input.int(8, "Recent 3H body lookback", minval = 3, maxval = 30, group = detectionGroup)
int lookback45M = input.int(8, "Recent 45M body lookback", minval = 3, maxval = 30, group = detectionGroup)
float decentDropRatio = input.float(100, "Decent 3H candle body ≥ median %", minval = 10, maxval = 500, step = 5, group = detectionGroup) / 100
float strongLimit = input.float(10, "Strong 45M squeeze ≤ 3H drop body %", minval = 0.1, maxval = 100, step = 0.1, group = detectionGroup) / 100
float moderateLimit = input.float(20, "Moderate 45M squeeze ≤ 3H drop body %", minval = 0.1, maxval = 100, step = 0.1, group = detectionGroup) / 100
float local45Limit = input.float(35, "Moderate squeeze ≤ recent 45M median %", minval = 1, maxval = 100, step = 0.5, group = detectionGroup) / 100
bool allowModerate = input.bool(true, "Allow moderate squeezes", group = detectionGroup)

// Fib
string fibGroup = "2. Fib"
bool showOnlyLatestFib = input.bool(false, "Display only latest confirmed Fib", group = fibGroup)
bool showFibLabels = input.bool(true, "Show Fib labels", group = fibGroup)
int squeezeOpacity = input.int(82, "Squeeze-zone transparency", minval = 0, maxval = 95, group = fibGroup)
color yellowColor = input.color(color.yellow, "0 / -1 and strong-squeeze colour", group = fibGroup)
color redColor = input.color(color.red, "1 / 1.27 / -2 / -2.27 colour", group = fibGroup)
color greenColor = input.color(color.lime, "3 / 3.3 / 3.5 colour", group = fibGroup)

// Chart
string chartGroup = "3. Chart"
bool show3H = input.bool(true, "Blue C1–C8 boundaries", group = chartGroup)
bool show45M = input.bool(true, "Grey A–D boundaries", group = chartGroup)
bool showDrops = input.bool(true, "Mark decent bullish/bearish 3H candles", group = chartGroup)
int dropOpacity = input.int(82, "3H drop-box transparency", minval = 0, maxval = 95, group = chartGroup)

bool isM15 = timeframe.isminutes and timeframe.multiplier == 15
if barstate.isfirst and not isM15
    runtime.error("Use this indicator on a standard 15-minute chart (15m). Current chart: " + timeframe.period)

medianRecent(array<float> source, int count) =>
    int available = math.min(array.size(source), count)
    float result = na
    if available > 0
        array<float> sample = array.new<float>()
        int first = array.size(source) - available
        for i = first to array.size(source) - 1
            array.push(sample, array.get(source, i))
        array.sort(sample, order.ascending)
        int middle = int(math.floor(available / 2))
        result := available % 2 == 1 ? array.get(sample, middle) : (array.get(sample, middle - 1) + array.get(sample, middle)) / 2
    result

partName(int slot45) =>
    int part = slot45 % 4
    part == 0 ? "A" : part == 1 ? "B" : part == 2 ? "C" : "D"

blockStartFor(int slot3H) =>
    timestamp(TZ, year(time, TZ), month(time, TZ), dayofmonth(time, TZ), slot3H * 3, 0)

nextBlockStartFor(int slot3H) =>
    slot3H < 7 ? timestamp(TZ, year(time, TZ), month(time, TZ), dayofmonth(time, TZ), (slot3H + 1) * 3, 0) : timestamp(TZ, year(time, TZ), month(time, TZ), dayofmonth(time, TZ) + 1, 0, 0)

drawFibLevel(array<line> lines, array<label> labels, int x1, int x2, float price, string title, color levelColor, string lineStyle, int width) =>
    // Finite setup line: never extend into later C blocks.
    line levelLine = line.new(x1, price, x2, price, xloc = xloc.bar_time, extend = extend.none, color = levelColor, style = lineStyle, width = width)
    array.push(lines, levelLine)
    if showFibLabels
        label levelLabel = label.new(x1, price, title + " (" + str.tostring(price, format.mintick) + ")", xloc = xloc.bar_time, style = label.style_none, textcolor = levelColor, size = size.small)
        array.push(labels, levelLabel)

clearFibObjects(array<line> lines, array<label> labels, array<label> points) =>
    while array.size(lines) > 0
        line.delete(array.pop(lines))
    while array.size(labels) > 0
        label.delete(array.pop(labels))
    while array.size(points) > 0
        label.delete(array.pop(points))

// New York hierarchy
int minuteOfDay = hour(time, TZ) * 60 + minute(time, TZ)
int slot45 = int(math.floor(minuteOfDay / 45))
int slot3H = int(math.floor(minuteOfDay / 180))
int currentBlockStart = blockStartFor(slot3H)
int nextBlockStart = nextBlockStartFor(slot3H)
bool begins45 = minuteOfDay % 45 == 0
bool completes45 = minuteOfDay % 45 == 30 and barstate.isconfirmed
bool begins3H = minuteOfDay % 180 == 0
bool completes3H = minuteOfDay % 180 == 165 and barstate.isconfirmed

if show3H and begins3H
    line.new(time, low, time, high, xloc = xloc.bar_time, extend = extend.both, color = color.rgb(20, 88, 255), width = 2)
    label.new(time, high, "C" + str.tostring(slot3H + 1), xloc = xloc.bar_time, style = label.style_label_down, color = color.new(color.blue, 75), textcolor = color.white, size = size.tiny)

if show45M and begins45 and not begins3H
    line.new(time, low, time, high, xloc = xloc.bar_time, extend = extend.both, color = color.new(color.gray, 42), width = 1)

if show45M and begins45
    label.new(time, low, partName(slot45), xloc = xloc.bar_time, style = label.style_label_up, color = color.new(color.gray, 84), textcolor = color.silver, size = size.tiny)

// Build completed 45M candles from three M15 candles.
var float open45 = na
var float close45 = na
var int start45 = na
var int count45 = 0

if begins45
    open45 := open
    close45 := close
    start45 := time
    count45 := 1
else if count45 > 0
    close45 := close
    count45 += 1

// Build completed 3H candles from twelve M15 candles.
var float open3H = na
var float close3H = na
var int start3H = na
var int count3H = 0

if begins3H
    open3H := open
    close3H := close
    start3H := time
    count3H := 1
else if count3H > 0
    close3H := close
    count3H += 1

var array<float> history45M = array.new<float>()
var array<float> history3H = array.new<float>()

// Completed bullish or bearish 3H source candles waiting only for their immediately following 3H block.
var array<float> dropOpen = array.new<float>()
var array<float> dropClose = array.new<float>()
var array<float> dropBody = array.new<float>()
var array<int> dropStart = array.new<int>()
var array<int> dropEnd = array.new<int>()
var array<int> dropTargetBlock = array.new<int>()

var array<line> fibLines = array.new<line>()
var array<label> fibLabels = array.new<label>()
var array<label> fibPoints = array.new<label>()

bool strongSignal = false
bool moderateSignal = false

// Step 1: inspect each completed 45M interval for a squeeze in the target 3H.
if completes45 and count45 == 3
    float intervalOpen = open45
    float intervalClose = close45
    float intervalTop = math.max(intervalOpen, intervalClose)
    float intervalBottom = math.min(intervalOpen, intervalClose)
    float intervalBody = math.abs(intervalClose - intervalOpen)
    bool bearish45 = intervalClose < intervalOpen
    int intervalStart = start45
    int intervalEnd = time_close
    float median45 = medianRecent(history45M, lookback45M)

    // Remove a 3H drop after its immediately following 3H search block has passed.
    int expired = array.size(dropTargetBlock) - 1
    while expired >= 0
        if array.get(dropTargetBlock, expired) < currentBlockStart
            array.remove(dropOpen, expired)
            array.remove(dropClose, expired)
            array.remove(dropBody, expired)
            array.remove(dropStart, expired)
            array.remove(dropEnd, expired)
            array.remove(dropTargetBlock, expired)
        expired -= 1

    // Search A–D of this 3H only for a bearish 45M squeeze.
    int matched = -1
    int grade = 0
    float squeezeRatio = na
    float localRatio = not na(median45) and median45 > 0 ? intervalBody / median45 : na

    if bearish45 and intervalBody > 0
        int candidate = array.size(dropBody) - 1
        while candidate >= 0 and matched == -1
            bool correctNext3H = array.get(dropTargetBlock, candidate) == currentBlockStart
            float reference3HBody = array.get(dropBody, candidate)
            float ratio = reference3HBody > 0 ? intervalBody / reference3HBody : na
            int candidateGrade = ratio <= strongLimit ? 2 : ratio <= moderateLimit and not na(localRatio) and localRatio <= local45Limit ? 1 : 0
            bool accepted = candidateGrade == 2 or candidateGrade == 1 and allowModerate
            if correctNext3H and accepted
                matched := candidate
                grade := candidateGrade
                squeezeRatio := ratio
            candidate -= 1

    // Step 2: after the 45M squeeze closes, draw Fib on the original 3H drop.
    if matched >= 0
        color squeezeColor = grade == 2 ? yellowColor : color.orange
        // Mark only the actual completed 45M source interval. The vertical prices
        // are its real-body open and close; wicks and the rest of the 3H are excluded.
        box.new(intervalStart, intervalTop, intervalEnd, intervalBottom, xloc = xloc.bar_time, bgcolor = color.new(squeezeColor, squeezeOpacity), border_color = squeezeColor, border_width = 3)
        string squeezeName = partName(slot45) + " · " + (grade == 2 ? "STRONG" : "MODERATE") + " SQUEEZE"
        label.new(intervalStart, intervalTop, squeezeName, xloc = xloc.bar_time, style = label.style_label_down, color = color.new(squeezeColor, 72), textcolor = squeezeColor, size = size.tiny)
        strongSignal := grade == 2
        moderateSignal := grade == 1

        float originalOpen = array.get(dropOpen, matched)
        float originalClose = array.get(dropClose, matched)
        float originalBody = array.get(dropBody, matched)
        int originalStart = array.get(dropStart, matched)
        int originalEnd = array.get(dropEnd, matched)

        // Fib uses 100% of the completed 3H real body, regardless of direction:
        // 0 = chronological 3H open (Point 1).
        // -1 = chronological 3H close (Point 2).
        // Wicks are always ignored. The signed move automatically flips the
        // projection for bullish versus bearish source candles.
        float f0 = originalOpen
        float signedMove = originalClose - originalOpen
        float fm1 = f0 + signedMove
        float fm2 = f0 + 2 * signedMove
        float fm227 = f0 + 2.27 * signedMove
        float fm3 = f0 + 3 * signedMove
        float fm33 = f0 + 3.3 * signedMove
        float fm35 = f0 + 3.5 * signedMove
        float fp1 = f0 - signedMove
        float fp127 = f0 - 1.27 * signedMove
        float fp3 = f0 - 3 * signedMove
        float fp33 = f0 - 3.3 * signedMove
        float fp35 = f0 - 3.5 * signedMove

        if showOnlyLatestFib
            clearFibObjects(fibLines, fibLabels, fibPoints)

        // Each Fib is isolated to exactly two C blocks:
        // original 3H drop start → end of the immediately following 3H block.
        int fibFinish = nextBlockStart
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fm35, "-3.5", greenColor, line.style_solid, 1)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fm33, "-3.3", greenColor, line.style_dotted, 1)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fm3, "-3", greenColor, line.style_solid, 1)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fm227, "-2.27", redColor, line.style_solid, 1)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fm2, "-2", redColor, line.style_solid, 1)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fm1, "-1", yellowColor, line.style_solid, 2)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, f0, "0", yellowColor, line.style_solid, 2)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fp1, "1", redColor, line.style_solid, 1)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fp127, "1.27", redColor, line.style_solid, 1)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fp3, "3", greenColor, line.style_solid, 1)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fp33, "3.3", greenColor, line.style_dotted, 1)
        drawFibLevel(fibLines, fibLabels, originalStart, fibFinish, fp35, "3.5", greenColor, line.style_solid, 1)

        line anchorLine = line.new(originalStart, f0, originalEnd, fm1, xloc = xloc.bar_time, color = color.black, width = 2)
        array.push(fibLines, anchorLine)
        label point0 = label.new(originalStart, f0, "", xloc = xloc.bar_time, style = label.style_circle, color = color.black, size = size.small)
        label pointMinus1 = label.new(originalEnd, fm1, "", xloc = xloc.bar_time, style = label.style_circle, color = color.black, size = size.small)
        array.push(fibPoints, point0)
        array.push(fibPoints, pointMinus1)

        // One Fib per matched 3H drop.
        array.remove(dropOpen, matched)
        array.remove(dropClose, matched)
        array.remove(dropBody, matched)
        array.remove(dropStart, matched)
        array.remove(dropEnd, matched)
        array.remove(dropTargetBlock, matched)

    array.push(history45M, intervalBody)
    if array.size(history45M) > 100
        array.shift(history45M)
    count45 := 0

// Step 3: register a completed directional 3H candle as a decent source candle.
// Registration happens after processing its final 45M interval, preventing self-matching.
if completes3H and count3H == 12
    float candleOpen = open3H
    float candleClose = close3H
    float candleTop = math.max(candleOpen, candleClose)
    float candleBottom = math.min(candleOpen, candleClose)
    float candleBody = math.abs(candleClose - candleOpen)
    bool bearish3H = candleClose < candleOpen
    bool bullish3H = candleClose > candleOpen
    float median3H = medianRecent(history3H, lookback3H)
    bool historyReady = array.size(history3H) >= lookback3H
    bool decent3HDrop = (bearish3H or bullish3H) and candleBody > 0 and historyReady and not na(median3H) and candleBody >= median3H * decentDropRatio

    if decent3HDrop
        array.push(dropOpen, candleOpen)
        array.push(dropClose, candleClose)
        array.push(dropBody, candleBody)
        array.push(dropStart, start3H)
        array.push(dropEnd, time_close)
        array.push(dropTargetBlock, nextBlockStart)
        if showDrops
            color candleColor = bearish3H ? color.red : color.lime
            string candleDirection = bearish3H ? "BEARISH" : "BULLISH"
            box.new(start3H, candleTop, time_close, candleBottom, xloc = xloc.bar_time, bgcolor = color.new(candleColor, dropOpacity), border_color = candleColor, border_width = 2)
            label.new(start3H, candleTop, "DECENT " + candleDirection + " 3H CANDLE\nFULL OPEN → CLOSE BODY\nC" + str.tostring(slot3H + 1), xloc = xloc.bar_time, style = label.style_label_down, color = color.new(candleColor, 70), textcolor = color.white, size = size.tiny)

    array.push(history3H, candleBody)
    if array.size(history3H) > 100
        array.shift(history3H)
    count3H := 0

alertcondition(strongSignal, "Strong 45M squeeze after decent 3H candle + Fib", "Strong bearish 45M squeeze confirmed in the next 3H after a decent bullish or bearish 3H candle. Full-body Fib displayed on {{ticker}} at {{time}}")
alertcondition(moderateSignal, "Moderate 45M squeeze after decent 3H candle + Fib", "Moderate bearish 45M squeeze confirmed in the next 3H after a decent bullish or bearish 3H candle. Full-body Fib displayed on {{ticker}} at {{time}}")

if strongSignal
    alert("Strong 45M squeeze after decent 3H candle; full-body Fib displayed: " + syminfo.ticker, alert.freq_once_per_bar_close)
if moderateSignal
    alert("Moderate 45M squeeze after decent 3H candle; full-body Fib displayed: " + syminfo.ticker, alert.freq_once_per_bar_close)
