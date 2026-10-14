// This Pine Script® code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/ MPL-2.0
//@version=6
indicator("FVG (Nephew sam remake) - MTF", overlay = true, max_boxes_count = 500, max_labels_count = 500)

// --- Constants ---
const int DEFAULT_BOX_LENGTH = 20
const int FOUR_HOUR_BOX_LENGTH = 30
const int MAX_TRACK_BARS = 300
const int CHECK_FORWARD_BARS = 300
const int MAX_STORED_FVGS = 450
const int MINUTE_MS = 60 * 1000

// --- Inputs ---
string DISPLAY_GROUP = "Timeframes"
string SETTINGS_GROUP = "FVG Settings"
string STYLE_GROUP = "Style"

bool show1mInput = input.bool(true, "1m", inline = "tf1", group = DISPLAY_GROUP, tooltip = "Show one-minute FVGs.")
bool show3mInput = input.bool(true, "3m", inline = "tf1", group = DISPLAY_GROUP, tooltip = "Show three-minute FVGs.")
bool show5mInput = input.bool(true, "5m", inline = "tf1", group = DISPLAY_GROUP, tooltip = "Show five-minute FVGs.")
bool show15mInput = input.bool(true, "15m", inline = "tf2", group = DISPLAY_GROUP, tooltip = "Show fifteen-minute FVGs.")
bool show45mInput = input.bool(true, "45m", inline = "tf2", group = DISPLAY_GROUP, tooltip = "Show forty-five-minute FVGs.")
bool show1hInput = input.bool(true, "1H", inline = "tf2", group = DISPLAY_GROUP, tooltip = "Show one-hour FVGs.")
bool show4hInput = input.bool(true, "4H", inline = "tf2", group = DISPLAY_GROUP, tooltip = "Show four-hour FVGs.")

bool waitForCloseInput = input.bool(true, "Wait for source candle close", group = SETTINGS_GROUP, tooltip = "When enabled, identify an FVG only after its source-timeframe candle closes. When disabled, detect it while that candle is forming; it may disappear before the candle closes.")
int boxLengthInput = input.int(DEFAULT_BOX_LENGTH, "Lower timeframe box length", minval = 1, maxval = 500, group = SETTINGS_GROUP, tooltip = "Number of source-timeframe bars used to extend FVG boxes, excluding 4H boxes.")
int boxLength4HInput = input.int(FOUR_HOUR_BOX_LENGTH, "4H box length", minval = 1, maxval = 500, group = SETTINGS_GROUP, tooltip = "Number of four-hour bars used to extend 4H FVG boxes.")
int maxTrackBarsInput = input.int(MAX_TRACK_BARS, "Maximum tracking bars", minval = 1, maxval = 5000, group = SETTINGS_GROUP, tooltip = "Delete FVGs after this many chart bars from detection.")
int checkForwardBarsInput = input.int(CHECK_FORWARD_BARS, "Disrespect window", minval = 1, maxval = 5000, group = SETTINGS_GROUP, tooltip = "Only remove an FVG for disrespect during this many chart bars after detection.")

color bullishColorInput = input.color(color.green, "Bullish FVG", group = STYLE_GROUP, tooltip = "Color used for bullish FVGs.")
color bearishColorInput = input.color(color.red, "Bearish FVG", group = STYLE_GROUP, tooltip = "Color used for bearish FVGs.")

// --- Storage ---
var box[] fvgBoxes = array.new_box()
var label[] fvgLabels = array.new_label()
var int[] fvgCreatedBars = array.new_int()
var bool[] fvgIsBullish = array.new_bool()
var float[] fvgCompareLevels = array.new_float()
var int[] fvgOriginTimes = array.new_int()
var string[] fvgTimeframes = array.new_string()

// --- FVG Data ---
getFvgData(bool waitForClose) =>
    int offset = waitForClose ? 1 : 0
    bool bullish = low[offset] > high[offset + 2]
    bool bearish = high[offset] < low[offset + 2]
    [bullish ? 1 : 0, bearish ? 1 : 0, time[offset + 2], high[offset + 2], low[offset + 2], high[offset], low[offset]]

// --- Object Helpers ---
fvgExists(string timeframeName, int originTime) =>
    bool exists = false
    int storedCount = array.size(fvgBoxes)
    if storedCount > 0
        for i = 0 to storedCount - 1
            if array.get(fvgTimeframes, i) == timeframeName and array.get(fvgOriginTimes, i) == originTime
                exists := true
                break
    exists

removeFvg(int index) =>
    box.delete(array.get(fvgBoxes, index))
    label.delete(array.get(fvgLabels, index))
    array.remove(fvgBoxes, index)
    array.remove(fvgLabels, index)
    array.remove(fvgCreatedBars, index)
    array.remove(fvgIsBullish, index)
    array.remove(fvgCompareLevels, index)
    array.remove(fvgOriginTimes, index)
    array.remove(fvgTimeframes, index)

drawFvg(int originTime, float top, float bottom, string timeframeName, color zoneColor, float compareLevel, bool bullish, int boxLength, int timeframeMinutes) =>
    bool validData = not na(originTime) and not na(top) and not na(bottom) and not na(compareLevel)
    if validData and not fvgExists(timeframeName, originTime)
        while array.size(fvgBoxes) >= MAX_STORED_FVGS
            removeFvg(0)

        float realTop = math.max(top, bottom)
        float realBottom = math.min(top, bottom)
        float middle = (realTop + realBottom) / 2.0
        int rightTime = originTime + timeframeMinutes * MINUTE_MS * boxLength
        int labelTime = rightTime - timeframeMinutes * MINUTE_MS
        int transparency = timeframeName == "4H" ? 60 : timeframeName == "1H" ? 72 : 85
        color borderColor = timeframeName == "4H" ? color.gray : na
        color backgroundColor = color.new(zoneColor, transparency)

        box fvgBox = box.new(left = originTime, right = rightTime, top = realTop, bottom = realBottom, xloc = xloc.bar_time, border_color = borderColor, bgcolor = backgroundColor)
        label fvgLabel = label.new(x = labelTime, y = bullish ? middle : middle - (realTop - realBottom) * 0.3, xloc = xloc.bar_time, text = timeframeName, style = label.style_none, size = size.small, textcolor = zoneColor)

        array.push(fvgBoxes, fvgBox)
        array.push(fvgLabels, fvgLabel)
        array.push(fvgCreatedBars, bar_index)
        array.push(fvgIsBullish, bullish)
        array.push(fvgCompareLevels, compareLevel)
        array.push(fvgOriginTimes, originTime)
        array.push(fvgTimeframes, timeframeName)

// --- Multi-timeframe FVG Processing ---
processTimeframe(string timeframeName, string labelText, int timeframeMinutes, color zoneColorBullish, color zoneColorBearish, int boxLength, bool enabled) =>
    if enabled
        int chartSeconds = timeframe.in_seconds()
        int requestedSeconds = timeframe.in_seconds(timeframeName)

        if requestedSeconds < chartSeconds
            [bullishArray, bearishArray, originArray, highThreeArray, lowThreeArray, highOneArray, lowOneArray] = request.security_lower_tf(syminfo.tickerid, timeframeName, getFvgData(waitForCloseInput))
            int intrabarCount = array.size(bullishArray)

            if intrabarCount > 0
                for i = 0 to intrabarCount - 1
                    int bullish = array.get(bullishArray, i)
                    int bearish = array.get(bearishArray, i)
                    int originTime = array.get(originArray, i)
                    float highThree = array.get(highThreeArray, i)
                    float lowThree = array.get(lowThreeArray, i)
                    float highOne = array.get(highOneArray, i)
                    float lowOne = array.get(lowOneArray, i)

                    if bullish == 1
                        drawFvg(originTime, lowOne, highThree, labelText, zoneColorBullish, highThree, true, boxLength, timeframeMinutes)
                    if bearish == 1
                        drawFvg(originTime, lowThree, highOne, labelText, zoneColorBearish, lowThree, false, boxLength, timeframeMinutes)
        else
            [bullish, bearish, originTime, highThree, lowThree, highOne, lowOne] = request.security(syminfo.tickerid, timeframeName, getFvgData(waitForCloseInput), gaps = barmerge.gaps_on, lookahead = barmerge.lookahead_off)

            if bullish == 1
                drawFvg(originTime, lowOne, highThree, labelText, zoneColorBullish, highThree, true, boxLength, timeframeMinutes)
            if bearish == 1
                drawFvg(originTime, lowThree, highOne, labelText, zoneColorBearish, lowThree, false, boxLength, timeframeMinutes)

// --- FVGs from 1m, 3m, 5m, 15m, 45m, 1H, and 4H ---
processTimeframe("1", "1m", 1, bullishColorInput, bearishColorInput, boxLengthInput, show1mInput)
processTimeframe("3", "3m", 3, bullishColorInput, bearishColorInput, boxLengthInput, show3mInput)
processTimeframe("5", "5m", 5, bullishColorInput, bearishColorInput, boxLengthInput, show5mInput)
processTimeframe("15", "15m", 15, bullishColorInput, bearishColorInput, boxLengthInput, show15mInput)
processTimeframe("45", "45m", 45, bullishColorInput, bearishColorInput, boxLengthInput, show45mInput)
processTimeframe("60", "1H", 60, bullishColorInput, bearishColorInput, boxLength4HInput, show1hInput)
processTimeframe("240", "4H", 240, bullishColorInput, bearishColorInput, boxLength4HInput, show4hInput)

// --- Disrespect and Cleanup ---
if array.size(fvgBoxes) > 0
    for i = array.size(fvgBoxes) - 1 to 0
        bool bullish = array.get(fvgIsBullish, i)
        float compareLevel = array.get(fvgCompareLevels, i)
        int createdBar = array.get(fvgCreatedBars, i)
        int age = bar_index - createdBar

        bool tooOld = age > maxTrackBarsInput
        bool disrespectWindow = bar_index > createdBar and age <= checkForwardBarsInput
        bool disrespected = (bullish and close < compareLevel) or (not bullish and close > compareLevel)

        if tooOld or (disrespectWindow and disrespected)
            removeFvg(i)
