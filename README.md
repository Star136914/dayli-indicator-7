//@version=6
// @description Daily Close Predictor - Gold (XAUUSD) with Enhanced Analysis
indicator("Daily Close Predictor - Gold (XAUUSD)", overlay=true, max_bars_back=500)

// ═══════════════════════════════════════════════════════════════════════════════
// User Information & Time
// ═══════════════════════════════════════════════════════════════════════════════
var USER_LOGIN = "Star136914"
var CREATION_TIME = "2025-05-20 00:38:05" // Note: This time is hardcoded in the script

// ═══════════════════════════════════════════════════════════════════════════════
// Input Parameters
// ═══════════════════════════════════════════════════════════════════════════════
// Time Settings
startHour = input.int(1, "Analysis Start Hour (UTC)", minval=0, maxval=23, group="Time Settings")
historicalDays = input.int(30, "Historical Reference Days", minval=5, maxval=60, group="Time Settings")
predictionLookaheadMinutes = input.int(60, "Prediction Lookahead (Minutes)", minval=1, maxval=240, group="Time Settings", tooltip="How far into the future (minutes) the prediction considers for end-of-day momentum.")

// S/R Detection Settings
pivotLength = input.int(10, "Pivot Length", minval=3, group="S/R Detection")
minDistance = input.float(1.0, "Minimum Distance Between Levels (%)", minval=0.1, group="S/R Detection")
touchSensitivity = input.float(0.005, "Touch Sensitivity", minval=0.001, group="S/R Detection")
levelStrength = input.int(3, "Required Touches for Strong Level", minval=2, group="S/R Detection")
showMajorLevels = input.bool(true, "Show S/R Levels", group="S/R Detection")
maxVisibleSRLevels = input.int(10, "Max. Visible S/R Levels", minval=1, maxval=50, group="S/R Detection", tooltip="Maximum number of Support/Resistance levels to display on the chart. Oldest levels are removed.")
showStrength = input.bool(true, "Show Level Strength Text", group="S/R Detection")
lineStyle = input.string("Solid", "S/R Line Style", options=["Solid", "Dashed", "Dotted"], group="S/R Detection")

// Set line style based on input (For S/R lines)
var lineType = lineStyle == "Solid" ? line.style_solid :
             lineStyle == "Dashed" ? line.style_dashed :
             line.style_dotted


// Technical Analysis Settings
useMacd = input.bool(true, "Use MACD Analysis", group="Technical Analysis")
useRsi = input.bool(true, "Use RSI Analysis", group="Technical Analysis")
showEma200 = input.bool(true, "Show EMA200", group="Technical Analysis")
rsiLength = input.int(14, "RSI Length", group="Technical Analysis")
macdFast = input.int(12, "MACD Fast Length", group="Technical Analysis")
macdSlow = input.int(26, "MACD Slow Length", group="Technical Analysis")
macdSignal = input.int(9, "MACD Signal Length", group="Technical Analysis")

// Price Zone Analysis Settings
zoneWidth = input.float(15, "Zone Width", minval=1, group="Zone Analysis")
historyDaysForZoneAnalysis = input.int(30, "History Days for Zone Analysis", minval=1, maxval=60, group="Zone Analysis", tooltip="Number of past days to consider for historical zone behavior.")
minCandleCount = input.int(3, "Minimum Candles per Zone", minval=1, group="Zone Analysis")
zoneTimeWeight = input.float(0.5, "Zone Time Spent Weight", minval=0.0, maxval=1.0, group="Zone Analysis", tooltip="Weight given to time spent in a zone vs. candle count.")
zoneStrengthFactor = input.float(2.0, "Zone Strength Factor", minval=1.0, group="Zone Analysis", tooltip="Multiplier for strong zones in prediction.")

// +++ Historical Zone Analysis (Plus Feature) +++
useHistoricalZones = input.bool(false, "Enable Historical Zone Analysis", group="Historical Zones Plus", tooltip="If true, zone analysis will consider historical data from previous days. If false, it will only consider today's data.")
historicalZoneDecayFactor = input.float(0.9, "Historical Zone Decay Decay (per day)", minval=0.0, maxval=1.0, step=0.01, group="Historical Zones Plus", tooltip="Factor by which older historical zone data decays each day. 1.0 means no decay, 0.0 means full decay.")

// === NEW: S/R Influence on Zones (The "Plus" Feature Inputs) ===
// These inputs control the new feature for S/R influence on zone weighting.
// They are grouped separately for clarity and modularity.
group_sr_influence = "S/R Influence on Zones (Plus Module)"
useSRInfluenceOnZones = input.bool(false, "Enable S/R Influence on Zones", group=group_sr_influence, tooltip="If true, proximity to Strong/Moderate Support/Resistance levels will influence zone strength calculation. If false, this feature is completely disabled.")
srProximityDistancePercent = input.float(0.5, "S/R Proximity Distance (%)", minval=0.01, maxval=5.0, step=0.01, group=group_sr_influence, tooltip="Maximum percentage distance from zone edges for S/R influence detection. Closer means stronger influence.")
srModerateStrengthMultiplier = input.float(1.5, "S/R Moderate Strength Multiplier", minval=0.1, maxval=10.0, step=0.1, group=group_sr_influence, tooltip="Multiplier for the influence of 'Moderate' S/R levels compared to 'New' levels.")
srStrongStrengthMultiplier = input.float(3.0, "S/R Strong Strength Multiplier", minval=0.1, maxval=10.0, step=0.1, group=group_sr_influence, tooltip="Multiplier for the influence of 'Strong' S/R levels compared to 'New' levels.")
srOverallAdditiveWeight = input.float(0.5, "S/R Overall Additive Weight", minval=0.0, maxval=5.0, step=0.1, group=group_sr_influence, tooltip="Overall weight added to zone score based on S/R proximity. Set to 0 to effectively disable the additive effect even if enabled.")

// === NEW: Daily Anchors (Plus Module) ===
// These inputs control the new feature for Daily Pivot Points and Daily SMA influence.
group_daily_anchors = "Daily Anchors (Plus Module)"
useDailyPivots = input.bool(false, "Enable Daily Pivot Points", group=group_daily_anchors, tooltip="If true, Daily Pivot Points (from previous day) will be used as a daily anchor.")
useDailySMA = input.bool(false, "Enable Daily Simple Moving Average", group=group_daily_anchors, tooltip="If true, a daily SMA will be used as a dynamic anchor.")
pivotInfluenceStrength = input.float(0.5, "Pivot Influence Strength", minval=0.0, maxval=1.0, step=0.1, group=group_daily_anchors, tooltip="Weight of Pivot Points in the combined daily anchor price. (0.0 = no influence, 1.0 = full influence)")
dailySMAInfluenceStrength = input.float(0.5, "Daily SMA Influence Strength", minval=0.0, maxval=1.0, step=0.1, group=group_daily_anchors, tooltip="Weight of Daily SMA in the combined daily anchor price. (0.0 = no influence, 1.0 = full influence)")
dailySMALength = input.int(20, "Daily SMA Length", minval=5, group=group_daily_anchors, tooltip="Length for the Daily Simple Moving Average calculation.")
zonePivotProximityBoost = input.float(0.8, "Zone Pivot Proximity Boost", minval=0.0, maxval=5.0, step=0.1, group=group_daily_anchors, tooltip="Additive score to zone weight if it's close to a Pivot Point. Set to 0 to disable this specific effect.")
pivotProximityDistancePercentZone = input.float(0.2, "Pivot Proximity Zone Distance (%)", minval=0.01, maxval=1.0, step=0.01, group=group_daily_anchors, tooltip="Max. percentage distance from zone center to a Pivot for proximity boost.")


// Display Settings
showZones = input.bool(true, "Show Price Zones", group="Display")
showPrediction = input.bool(true, "Show Prediction Zone", group="Display")
candleDensitySensitivity = input.string("Medium", "Candle Density Sensitivity", options=["Low", "Medium", "High"], group="Display")


// ═══════════════════════════════════════════════════════════════════════════════
// Types and Data Structures
// ═══════════════════════════════════════════════════════════════════════════════
type Zone
    float start
    float end
    bool isValid
    float strength // Represents cumulative density and behavior (positive for bullish, negative for bearish)
    float volume // Not directly used yet but kept for future expansion
    int candleCount
    float timeSpent // New: Time spent within the zone in minutes
    float cumulativeStrength // New: Aggregated strength from historical data (e.g., net green/red candles)
    int lastActiveDayOfMonth // New: To track when this zone was last active (day of month)
    int createdDayOfMonth // New: To track when this zone was initially created (day of month)

type PredictionData
    float upperBound
    float lowerBound
    bool isBullish
    float confidence
    string reason
    float zoneBasedPredictionPrice // New: The price derived solely from zone analysis

// Arrays and Tables
var Zone[] historicalPriceZones = array.new<Zone>()
var PredictionData[] predictions = array.new<PredictionData>()
var table predictionTable = table.new(position.bottom_right, 2, 6) // Adjusted for new rows

// S/R specific arrays
var float[] resistanceLevels = array.new_float(0)
var float[] supportLevels = array.new_float(0)
var line[] resistanceLines = array.new_line(0)
var line[] supportLines = array.new_line(0)
var int[] resistanceTouches = array.new_int(0)
var int[] supportTouches = array.new_int(0)
var label[] resistanceLabels = array.new_label(0)
var label[] supportLabels = array.new_label(0)
var label[] touchResistantLabels = array.new_label(0)
var label[] touchSupportLabels = array.new_label(0)
var int[] touchResistanceLevelIndex = array.new_int(0)
var int[] touchSupportLevelIndex = array.new_int(0)


// ═══════════════════════════════════════════════════════════════════════════════
// Technical Indicators Calculations
// ═══════════════════════════════════════════════════════════════════════════════
// EMA200
ema200 = ta.ema(close, 200)

// RSI
rsi = ta.rsi(close, rsiLength)

// MACD
[macdLine, signalLine, histLine] = ta.macd(close, macdFast, macdSlow, macdSignal)

// Daily Statistics
var float todayOpen = na
var float yesterdayClose = na
var float todayHigh = na
var float todayLow = na
var int greenCandles = 0
var int totalCandles = 0
var float dailyRange = 0.0 // Daily range for current day

// Daily Pivot Points Calculation (for previous day's close)
var float prevDayPivot = na
var float prevDayR1 = na
var float prevDayS1 = na
var float prevDayR2 = na
var float prevDayS2 = na

// Daily Simple Moving Average Calculation
dailySMA = ta.sma(close, dailySMALength)

// Day change detection variable
var bool newDay = false
newDay := ta.change(dayofmonth) != 0

if newDay
    todayOpen := open
    yesterdayClose := close[1]
    todayHigh := high
    todayLow := low
    greenCandles := 0
    totalCandles := 0

    // Calculate Previous Day's Pivots (only once per new day)
    float prevDayHigh = high[1]
    float prevDayLow = low[1]
    float prevDayClose = close[1]
    
    if not na(prevDayHigh) and not na(prevDayLow) and not na(prevDayClose)
        prevDayPivot := (prevDayHigh + prevDayLow + prevDayClose) / 3
        prevDayR1 := (2 * prevDayPivot) - prevDayLow
        prevDayS1 := (2 * prevDayPivot) - prevDayHigh
        prevDayR2 := prevDayPivot + (prevDayHigh - prevDayLow)
        prevDayS2 := prevDayPivot - (prevDayHigh - prevDayLow)

    // --- Critical Logic for Historical Zones Feature ---
    if not useHistoricalZones // If historical zones are NOT enabled, clear array for new day
        array.clear(historicalPriceZones)
    else // If historical zones ARE enabled, apply decay and remove very old zones
        // Apply decay to all zones
        for i = 0 to array.size(historicalPriceZones) - 1
            zone = array.get(historicalPriceZones, i)
            // Decay factors applied
            zone.strength := zone.strength * historicalZoneDecayFactor
            zone.cumulativeStrength := zone.cumulativeStrength * historicalZoneDecayFactor
            zone.candleCount := int(zone.candleCount * historicalZoneDecayFactor) // Decay candle count
            zone.timeSpent := zone.timeSpent * historicalZoneDecayFactor // Decay time spent
            array.set(historicalPriceZones, i, zone)
        
        // Remove very old zones that are outside the history window
        // Iterate backwards to safely remove elements as array size changes
        for i = array.size(historicalPriceZones) - 1 to 0
            zone = array.get(historicalPriceZones, i)
            // Simplified check for removal. For more robust aging, 'time' field in Zone type is essential.
            if math.abs(zone.strength) < 0.01 and math.abs(zone.cumulativeStrength) < 0.01 and zone.candleCount < minCandleCount
                array.remove(historicalPriceZones, i)
        
else
    todayHigh := math.max(high, todayHigh)
    todayLow := math.min(low, todayLow)
    totalCandles += 1
    if close > open
        greenCandles += 1
    else if close < open
        greenCandles -= 1 // Track net green candles for strength
    dailyRange := todayHigh - todayLow

// Current bar's timeframe in minutes, used for timeSpent calculations
var int current_bar_timeframe_minutes = timeframe.multiplier

// ═══════════════════════════════════════════════════════════════════════════════
// Zone Analysis Functions
// ═══════════════════════════════════════════════════════════════════════════════

// Dynamic density multiplier based on input string
densityMultiplier = switch candleDensitySensitivity
    "Low" => 0.5
    "Medium" => 1.0
    "High" => 2.0
    => 1.0

// === NEW: Helper for S/R Influence on Zones ===
// This function maps the number of touches to a numerical strength value.
// It utilizes your existing 'levelStrength' input to define 'Strong'.
f_getLevelStrengthNumerical_SR(touches) =>
    strengthVal = 0 // Default to 'New' if touches is 1 or less
    if touches >= levelStrength
        strengthVal := 3 // Strong
    else if touches > 1
        strengthVal := 2 // Moderate
    else
        strengthVal := 1 // New
    strengthVal

// === NEW: S/R Influence on Zones Calculation ===
// This is the core of the "Plus" module. It calculates an influence score based on
// the proximity of S/R levels to a given zone.
// It directly uses 'resistanceLevels', 'resistanceTouches', 'supportLevels', 'supportTouches'
// which are already populated by your S/R detection logic.
f_getSRInfluenceScore(float zoneStart, float zoneEnd, float currentPrice, float proximityDistancePercent,
                      float moderateStrengthMult, float srStrongStrengthMult) =>
    
    var float srInfluenceScore = 0.0
    // syminfo.mintick ensures calculations are price-sensitive and avoids division by zero for very small values.
    float minRefPriceForDistance = syminfo.mintick * 100

    // --- Process Resistance Levels ---
    if array.size(resistanceLevels) > 0
        for i = 0 to array.size(resistanceLevels) - 1
            resLevel = array.get(resistanceLevels, i)
            resTouches = array.get(resistanceTouches, i)
            
            // Only consider if resistance level is valid (not NA)
            if not na(resLevel)
                // Calculate distance from resistance level to the zone's top (end)
                float distanceRes = math.abs(resLevel - zoneEnd)
                
                // Max allowed distance for influence, calculated as a percentage of relevant price.
                // Using max(zoneEnd, currentPrice) ensures a robust reference for percentage.
                float maxAllowedDistanceRes = proximityDistancePercent * 0.01 * math.max(zoneEnd, currentPrice, minRefPriceForDistance)
                
                // If resistance is within proximity
                if distanceRes <= maxAllowedDistanceRes
                    // Normalized distance: closer (distanceRes -> 0) results in normalizedDistanceRes -> 1.0
                    float normalizedDistanceRes = 1.0 - (distanceRes / maxAllowedDistanceRes)
                    
                    float strengthFactorRes = 1.0
                    int srLevelType = f_getLevelStrengthNumerical_SR(resTouches)
                    if srLevelType == 2 // Moderate Resistance
                        strengthFactorRes := moderateStrengthMult
                    else if srLevelType == 3 // Strong Resistance
                        strengthFactorRes := srStrongStrengthMult
                    
                    float currentInfluence = normalizedDistanceRes * strengthFactorRes
                    
                    // --- Intelligent Adjustment for "Broken" Levels ---
                    // If the zone has already passed above the resistance level, it suggests the resistance is broken.
                    // A broken resistance might act as support, or at least its bearish influence should be diminished.
                    if zoneEnd > resLevel
                        currentInfluence := currentInfluence * 0.5 // Reduce bearish impact by 50%
                    
                    srInfluenceScore := srInfluenceScore - currentInfluence // Resistance has a negative (bearish) impact

    // --- Process Support Levels ---
    if array.size(supportLevels) > 0
        for i = 0 to array.size(supportLevels) - 1
            supLevel = array.get(supportLevels, i)
            supTouches = array.get(supportTouches, i)

            if not na(supLevel)
                // Calculate distance from support level to the zone's bottom (start)
                float distanceSup = math.abs(supLevel - zoneStart)
                
                // Max allowed distance for influence
                float maxAllowedDistanceSup = proximityDistancePercent * 0.01 * math.max(zoneStart, currentPrice, minRefPriceForDistance)
                
                // If support is within proximity
                if distanceSup <= maxAllowedDistanceSup
                    float normalizedDistanceSup = 1.0 - (distanceSup / maxAllowedDistanceSup)
                    
                    float strengthFactorSup = 1.0
                    int srLevelType = f_getLevelStrengthNumerical_SR(supTouches)
                    if srLevelType == 2 // Moderate Support
                        strengthFactorSup := moderateStrengthMult
                    else if srLevelType == 3 // Strong Support
                        strengthFactorSup := srStrongStrengthMult
                    
                    float currentInfluence = normalizedDistanceSup * strengthFactorSup
                    
                    // --- Intelligent Adjustment for "Broken" Levels ---
                    // If the zone has already passed below the support level, it suggests the support is broken.
                    // A broken support might act as resistance, or or at least its bullish influence should be diminished.
                    if zoneStart < supLevel
                        currentInfluence := currentInfluence * 0.5 // Reduce bullish impact by 50%
                    
                    srInfluenceScore := srInfluenceScore + currentInfluence // Support has a positive (bullish) impact
    srInfluenceScore

// === MODIFIED: f_calculateZoneWeight (Integrates S/R Influence Conditionally and NEW Pivot Proximity) ===
// Function to calculate a zone's weighted score, considering historical data AND S/R influence if enabled.
f_calculateZoneWeight(Zone zone, int currentDayOfMonth, float decayFactor, bool useHistZones,
                      bool applySRInfluence, float srProximityDistance, float srModerateStrengthMult,
                      float srStrongStrengthMult, float srOverallAdditive,
                      bool applyPivotProximity, float zonePivotBoost, float pivotDistPercent,
                      float pP, float pR1, float pS1, float pR2, float pS2,
                      int barTimeframeMinutes) => // NEW PARAMETER ADDED
    
    var float result = 0.0
    totalCandlesInZone = zone.candleCount
    
    // If historical zones are NOT enabled, ensure this zone's weight is 0 if it's not from today
    if not useHistZones and zone.createdDayOfMonth != currentDayOfMonth
        0.0 // Zone from previous day has no weight if historical feature is off

    else if totalCandlesInZone < minCandleCount
        0.0 // Zone is too sparse
    else
        // Calculate base zone weight components from internal zone properties
        float directionalityWeight = 0.0
        if zone.strength > 0
            directionalityWeight := zone.strength / math.max(totalCandlesInZone, 1) // Green dominance
        else if zone.strength < 0
            directionalityWeight := math.abs(zone.strength) / math.max(totalCandlesInZone, 1) // Red dominance
        
        density = totalCandlesInZone * densityMultiplier
        
        timeFactor = zone.timeSpent > 0 ? zone.timeSpent / (totalCandlesInZone * math.max(1, barTimeframeMinutes)) : 0.0
        
        float baseZoneWeight = (density * directionalityWeight * (1 - zoneTimeWeight)) + (timeFactor * zoneTimeWeight) + (zone.cumulativeStrength * zoneStrengthFactor)
        
        // === S/R Influence Integration Logic ===
        float combinedBaseWeight = baseZoneWeight // Start with base weight

        if applySRInfluence // Check the input parameter to see if the S/R influence module is enabled
            // Calculate S/R influence score using the new function.
            float srInfluence = f_getSRInfluenceScore(zone.start, zone.end, close, srProximityDistance,
                                                      srModerateStrengthMult, srStrongStrengthMult)
            // Add the calculated S/R influence, scaled by the overall additive weight.
            combinedBaseWeight := combinedBaseWeight + (srInfluence * srOverallAdditive)
        
        // === NEW: Pivot Proximity Influence on Zones ===
        float pivotProximityScore = 0.0
        if applyPivotProximity and not na(pP) // Ensure feature is enabled and pivot exists
            float zoneMid = zone.start + (zoneWidth / 2)
            float minRefPriceForDistance = syminfo.mintick * 100 // Ensure valid division
            float maxAllowedPivotDistance = pivotDistPercent * 0.01 * math.max(zoneMid, close, minRefPriceForDistance)

            float[] pivotLevels = array.new_float(0)
            array.push(pivotLevels, pP)
            if not na(pR1)
                array.push(pivotLevels, pR1)
            if not na(pS1)
                array.push(pivotLevels, pS1)
            if not na(pR2)
                array.push(pivotLevels, pR2)
            if not na(pS2)
                array.push(pivotLevels, pS2)

            float closestPivotDistance = 1e18 // A very large number
            for i = 0 to array.size(pivotLevels) - 1
                float currentPivotLevel = array.get(pivotLevels, i)
                float dist = math.abs(zoneMid - currentPivotLevel)
                if dist < closestPivotDistance
                    closestPivotDistance := dist
            
            if closestPivotDistance <= maxAllowedPivotDistance
                // Normalized distance: closer results in higher score
                float normalizedPivotDistance = 1.0 - (closestPivotDistance / maxAllowedPivotDistance)
                pivotProximityScore := normalizedPivotDistance * zonePivotBoost
        
        // Add pivotProximityScore to the combined weight
        combinedBaseWeight := combinedBaseWeight + pivotProximityScore
        
        result := combinedBaseWeight // Assign the combined weight to result
        
        result


// Function to update zone data dynamically
updateZoneData(float price, float zoneWd, float currentClose, float currentOpen) =>
    currentZoneStart = math.floor(price / zoneWd) * zoneWd
    currentZoneEnd = currentZoneStart + zoneWd
    
    foundZone = false
    // Only iterate if array has elements
    if array.size(historicalPriceZones) > 0
        for i = 0 to array.size(historicalPriceZones) - 1
            zone = array.get(historicalPriceZones, i)
            if zone.start == currentZoneStart and zone.end == currentZoneEnd
                // Update existing zone
                zone.candleCount := zone.candleCount + 1
                zone.timeSpent := zone.timeSpent + current_bar_timeframe_minutes
                if currentClose > currentOpen
                    zone.strength := zone.strength + 1
                else if currentClose < currentOpen
                    zone.strength := zone.strength - 1
                
                // cumulativeStrength is a more continuous measure of net bullish/bearish pressure
                zone.cumulativeStrength := zone.cumulativeStrength + (currentClose > currentOpen ? 1 : (currentClose < currentOpen ? -1 : 0)) * (current_bar_timeframe_minutes / 60.0)
                zone.lastActiveDayOfMonth := dayofmonth
                array.set(historicalPriceZones, i, zone)
                foundZone := true
                break
            
    if not foundZone // No existing zone found for this price range
        // Create new zone
        newZone = Zone.new()
        newZone.start := currentZoneStart
        newZone.end := currentZoneEnd
        newZone.candleCount := 1
        newZone.timeSpent := current_bar_timeframe_minutes
        newZone.strength := currentClose > currentOpen ? 1 : (currentClose < currentOpen ? -1 : 0)
        newZone.cumulativeStrength := (currentClose > currentOpen ? 1 : (currentClose < currentOpen ? -1 : 0)) * (current_bar_timeframe_minutes / 60.0)
        newZone.lastActiveDayOfMonth := dayofmonth
        newZone.createdDayOfMonth := dayofmonth // New zones are created today
        array.push(historicalPriceZones, newZone)
        
// Support/Resistance Functions
detectPivots() =>
    ph = ta.pivothigh(high, pivotLength, pivotLength)
    pl = ta.pivotlow(low, pivotLength, pivotLength)
    [ph, pl]

// Function to check if a new level is unique (Adapted for S/R specific arrays)
isUniqueLevel_SR(float newLevel, float[] levels) =>
    if array.size(levels) == 0
        true
    else
        isUnique = true
        for i = 0 to array.size(levels) - 1
            existingLevel = array.get(levels, i)
            if not na(existingLevel) and not na(newLevel)
                if math.abs((newLevel - existingLevel) / existingLevel * 100) < minDistance
                    isUnique := false
                    break
        isUnique

// Function to remove oldest line and data
removeOldestLine_SR(line[] lines, float[] levels, int[] touches, label[] labels, label[] touchLabels, int[] touchLevelIndex) =>
    if array.size(lines) > 0
        // Delete visual elements
        line.delete(array.get(lines, 0))
        if array.size(labels) > 0 // Ensure label array is not empty
            label.delete(array.get(labels, 0))
        
        // Delete only touch labels for the oldest level (index 0)
        if array.size(touchLabels) > 0
            for i = array.size(touchLabels) - 1 to 0
                if array.get(touchLevelIndex, i) == 0
                    label.delete(array.get(touchLabels, i))
                    array.remove(touchLabels, i)
                    array.remove(touchLevelIndex, i)
                else
                    array.set(touchLevelIndex, i, array.get(touchLevelIndex, i) - 1)
        
        // Remove ONE item from main arrays
        array.shift(lines)
        array.shift(levels)
        array.shift(touches)
        array.shift(labels)

// Function to get level strength text
getLevelStrength_SR(int touches) =>
    str = ""
    if touches >= levelStrength
        str := "Strong"
    else if touches > 1
        str := "Moderate"
    else
        str := "New"
    str

// ═══════════════════════════════════════════════════════════════════════════════
// Price Analysis Functions
// ═══════════════════════════════════════════════════════════════════════════════
calculateIndicatorScore() =>
    var float score = 0.0
    
    if useRsi
        rsiScore = rsi > 50 ? 1 : rsi < 50 ? -1 : 0
        score := score + rsiScore
    
    if useMacd
        macdScore = macdLine > signalLine ? 1 : macdLine < signalLine ? -1 : 0
        score := score + macdScore
    
    score

analyzeTrend() =>
    bullishPressure = greenCandles / math.max(totalCandles, 1) * 100
    emaPosition = close > ema200
    
    // Get additional indicator score
    indicatorScore = calculateIndicatorScore()
    
    // Calculate total score including optional indicators
    baseScore = (emaPosition ? 1 : -1)
    totalFactors = 1.0 // Start with 1 for EMA
    
    if useRsi
        totalFactors := totalFactors + 1
    if useMacd
        totalFactors := totalFactors + 1
    
    totalScore = (baseScore + indicatorScore) / totalFactors
    
    isBullish = totalScore > 0
    confidence = math.abs(totalScore) * 100
    
    [isBullish, confidence]

// Function to calculate time remaining score
getTimeRemainingScore() =>
    var float score = 0.0
    currentMinutes = hour(time("D", "UTC")) * 60 + minute(time("D", "UTC"))
    totalDayMinutes = 24 * 60 // Assuming a 24-hour day for daily close prediction
    
    minutesUntilClose = totalDayMinutes - currentMinutes
    
    if minutesUntilClose <= 0
        score := 0.0 // Day already ended or very close
    else if minutesUntilClose <= predictionLookaheadMinutes
        score := 1.0
    else if minutesUntilClose > predictionLookaheadMinutes * 2
        score := 0.2
    else
        score := 0.5
    score

// === MODIFIED: calculatePrediction (Passes S/R Influence and NEW Pivot Parameters) ===
calculatePrediction() =>
    [isBullish, confidence] = analyzeTrend()
    
    confidence := math.min(confidence, 100)
    
    float zoneBasedPredictionPrice = 0.0
    float totalZoneWeight = 0.0
    
    if array.size(historicalPriceZones) > 0
        for z = 0 to array.size(historicalPriceZones) - 1
            zone = array.get(historicalPriceZones, z)
            
            // Calculate weight. `useHistoricalZones` is passed to ensure proper conditional logic.
            // === MODIFIED: Pass S/R Influence and NEW Pivot related parameters to f_calculateZoneWeight ===
            weight = f_calculateZoneWeight(zone, dayofmonth, historicalZoneDecayFactor, useHistoricalZones,
                                          useSRInfluenceOnZones, srProximityDistancePercent,
                                          srModerateStrengthMultiplier, srStrongStrengthMultiplier, srOverallAdditiveWeight,
                                          useDailyPivots, zonePivotProximityBoost, pivotProximityDistancePercentZone,
                                          prevDayPivot, prevDayR1, prevDayS1, prevDayR2, prevDayS2,
                                          current_bar_timeframe_minutes) // NEW ARG ADDED
            
            zoneMidPrice = zone.start + (zoneWidth / 2)
            
            zoneBasedPredictionPrice := zoneBasedPredictionPrice + weight * zoneMidPrice
            totalZoneWeight := totalZoneWeight + weight
    
    zoneBasedPredictionPrice := totalZoneWeight > 0 ? zoneBasedPredictionPrice / totalZoneWeight : close
    
    // --- NEW: Calculate Effective Daily Anchor Price ---
    float effectiveDailyAnchorPrice = na
    float anchorWeightSum = 0.0

    if useDailyPivots and not na(prevDayPivot) // If daily pivots enabled and calculated
        effectiveDailyAnchorPrice := prevDayPivot * pivotInfluenceStrength
        anchorWeightSum := anchorWeightSum + pivotInfluenceStrength
    
    if useDailySMA and not na(dailySMA) // If daily SMA enabled and calculated
        if na(effectiveDailyAnchorPrice)
            effectiveDailyAnchorPrice := dailySMA * dailySMAInfluenceStrength
        else
            effectiveDailyAnchorPrice := effectiveDailyAnchorPrice + (dailySMA * dailySMAInfluenceStrength)
        anchorWeightSum := anchorWeightSum + dailySMAInfluenceStrength
    
    if anchorWeightSum > 0
        effectiveDailyAnchorPrice := effectiveDailyAnchorPrice / anchorWeightSum
    else // If no anchors are enabled or calculated, default to zone-based or close
        effectiveDailyAnchorPrice := zoneBasedPredictionPrice // Fallback if no anchors are used

    // Time of day influence factor
    // This factor increases as the day progresses, shifting influence towards current price and anchors
    currentMinutesFromStart = hour(time("D", "UTC")) * 60 + minute(time("D", "UTC"))
    // Assuming analysis starts at startHour (UTC)
    minutesSinceStart = currentMinutesFromStart - (startHour * 60)
    if minutesSinceStart < 0
        minutesSinceStart := 0 // Handle cases where current time is before startHour

    // Normalize to 0-1 based on time remaining to predictionLookaheadMinutes
    float timeOfDayInfluenceFactor = 0.0
    totalDayMinutes = 24 * 60 // Ensure totalDayMinutes is defined in this scope if used here
    if totalDayMinutes - currentMinutesFromStart <= predictionLookaheadMinutes
        // If we are close to the end of the day (within predictionLookaheadMinutes), maximize influence.
        timeOfDayInfluenceFactor := 1.0
    else
        // Gradually increase influence as day progresses
        timeOfDayInfluenceFactor := math.min(1.0, minutesSinceStart / (totalDayMinutes * 0.75)) // Max influence by 75% of day


    // Combine zone-based prediction, current price, and effective daily anchor price
    // Weights are adjusted based on timeOfDayInfluenceFactor
    float finalPredictionPrice = na
    if timeOfDayInfluenceFactor < 0.5 // Early to mid-day, more emphasis on zone analysis
        finalPredictionPrice := (zoneBasedPredictionPrice * (1 - timeOfDayInfluenceFactor)) + (close * timeOfDayInfluenceFactor)
    else // Mid to late-day, more emphasis on current price and daily anchors
        finalPredictionPrice := (close * (1 - timeOfDayInfluenceFactor)) + (effectiveDailyAnchorPrice * timeOfDayInfluenceFactor)
    
    // Ensure finalPredictionPrice is not na
    if na(finalPredictionPrice)
        finalPredictionPrice := close
    
    timeScore = getTimeRemainingScore()
    
    confidence := confidence * (1 + (timeScore / 2))
    confidence := math.min(confidence, 100)

    prediction = PredictionData.new()
    prediction.isBullish := isBullish
    prediction.confidence := confidence
    prediction.zoneBasedPredictionPrice := zoneBasedPredictionPrice
    
    atr_val = ta.atr(14)
    effectiveDailyRange = dailyRange == 0 ? (atr_val * 0.5) : dailyRange

    maxRangePercent = 0.5
    maxRange = close * maxRangePercent / 100
    
    predictionRange = math.min(effectiveDailyRange * (confidence / 100), maxRange)
    
    // Use the combined finalPredictionPrice for setting bounds
    if isBullish
        prediction.lowerBound := math.min(close, finalPredictionPrice)
        prediction.upperBound := math.max(close + predictionRange, finalPredictionPrice + (predictionRange * (confidence/100)))
        prediction.reason := "Bullish: Trend and Zone analysis confirm buying pressure."
    else
        prediction.lowerBound := math.min(close - (predictionRange * (confidence/100)), finalPredictionPrice)
        prediction.upperBound := math.max(close, finalPredictionPrice)
        prediction.reason := "Bearish: Trend and Zone analysis confirm selling pressure."
        
    // Ensure lowerBound is always less than upperBound for plotting
    if prediction.upperBound < prediction.lowerBound
        temp = prediction.upperBound
        prediction.upperBound := prediction.lowerBound
        prediction.lowerBound := temp

    prediction

// ═══════════════════════════════════════════════════════════════════════════════
// Main Logic
// ═══════════════════════════════════════════════════════════════════════════════
if barstate.isfirst or newDay
    // S/R levels are reset daily, could be modified to persist if needed.
    array.clear(predictions)

    // --- S/R Level Reset (from dedicated S/R indicator logic) ---
    // Clear all S/R related arrays on new day or first bar
    
    // Delete existing lines and labels to prevent accumulation BEFORE clearing arrays
    // Added checks for array.size() to prevent "Index out of bounds" on bar 0 if arrays are empty
    if array.size(resistanceLines) > 0
        for i = 0 to array.size(resistanceLines) - 1
            line.delete(array.get(resistanceLines, i))
    if array.size(supportLines) > 0
        for i = 0 to array.size(supportLines) - 1
            line.delete(array.get(supportLines, i))
    if array.size(resistanceLabels) > 0
        for i = 0 to array.size(resistanceLabels) - 1
            label.delete(array.get(resistanceLabels, i))
    if array.size(supportLabels) > 0
        for i = 0 to array.size(supportLabels) - 1
            label.delete(array.get(supportLabels, i))
    if array.size(touchResistantLabels) > 0
        for i = 0 to array.size(touchResistantLabels) - 1
            label.delete(array.get(touchResistantLabels, i))
    if array.size(touchSupportLabels) > 0
        for i = 0 to array.size(touchSupportLabels) - 1
            label.delete(array.get(touchSupportLabels, i))
    
    array.clear(resistanceLevels)
    array.clear(supportLevels)
    array.clear(resistanceLines)
    array.clear(supportLines)
    array.clear(resistanceTouches)
    array.clear(supportTouches)
    array.clear(resistanceLabels)
    array.clear(supportLabels)
    array.clear(touchResistantLabels)
    array.clear(touchSupportLabels)
    array.clear(touchResistanceLevelIndex)
    array.clear(touchSupportLevelIndex)
    // --- End S/R Level Reset ---


    // IMPORTANT: historicalPriceZones logic for clearing/decay is handled within the 'if newDay' block above
    // to apply decay or clear based on 'useHistoricalZones' input.
    // No explicit clear here for `historicalPriceZones` as it's managed there.

// Update S/R Levels
[ph, pl] = detectPivots()

// Handle resistance levels
if not na(ph) and showMajorLevels
    if isUniqueLevel_SR(ph, resistanceLevels)
        if array.size(resistanceLevels) >= maxVisibleSRLevels
            removeOldestLine_SR(resistanceLines, resistanceLevels, resistanceTouches, resistanceLabels, touchResistantLabels, touchResistanceLevelIndex)
        
        array.push(resistanceLevels, ph)
        array.push(resistanceTouches, 1)
        
        strengthText = showStrength ? " (" + getLevelStrength_SR(1) + ")" : ""
        labelText = "R: " + str.tostring(ph, "#.##") + strengthText + " [1]"
        
        newLine = line.new(bar_index[pivotLength], ph, bar_index + 20, ph,
             color=color.red, style=lineType, width=1)
        array.push(resistanceLines, newLine)
        
        newLabel = label.new(bar_index[pivotLength], ph, text=labelText,
             xloc=xloc.bar_index, yloc=yloc.price, style=label.style_label_down,
             color=color.rgb(255, 0, 0, 80), textcolor=color.white, size=size.small)
        array.push(resistanceLabels, newLabel)
    else // Level exists, update touch count
        for i = 0 to array.size(resistanceLevels) - 1
            resLevel = array.get(resistanceLevels, i)
            if not na(resLevel) and math.abs((ph - resLevel) / resLevel * 100) < minDistance
                touches = array.get(resistanceTouches, i) + 1
                array.set(resistanceTouches, i, touches)
                
                // Update line and label
                if array.size(resistanceLines) > i and array.size(resistanceLabels) > i
                    line.set_x2(array.get(resistanceLines, i), bar_index + 20)
                    strengthText = showStrength ? " (" + getLevelStrength_SR(touches) + ")" : ""
                    labelText = "R: " + str.tostring(resLevel, "#.##") + strengthText + " [" + str.tostring(touches) + "]"
                    label.set_text(array.get(resistanceLabels, i), labelText)
                    label.set_x(array.get(resistanceLabels, i), bar_index[pivotLength]) // Reset x-position to current pivot
                
                // Add touch label
                if showStrength and touches >= 2 // Only show touch labels for moderate/strong touches
                    touchLabel = label.new(x=bar_index, y=high + (hl2 * 0.001),
                                  text="R Touch (" + str.tostring(touches) + ")",
                                  xloc=xloc.bar_index, yloc=yloc.price, style=label.style_label_down,
                                  color=color.rgb(255, 0, 0, 50), textcolor=color.white, size=size.tiny)
                    array.push(touchResistantLabels, touchLabel)
                    array.push(touchResistanceLevelIndex, i) // Store index of the level this touch belongs to
                break

// Handle support levels
if not na(pl) and showMajorLevels
    if isUniqueLevel_SR(pl, supportLevels)
        if array.size(supportLevels) >= maxVisibleSRLevels
            removeOldestLine_SR(supportLines, supportLevels, supportTouches, supportLabels, touchSupportLabels, touchSupportLevelIndex)
        
        array.push(supportLevels, pl)
        array.push(supportTouches, 1)
        
        strengthText = showStrength ? " (" + getLevelStrength_SR(1) + ")" : ""
        labelText = "S: " + str.tostring(pl, "#.##") + strengthText + " [1]"
        
        newLine = line.new(bar_index[pivotLength], pl, bar_index + 20, pl,
             color=color.green, style=lineType, width=1)
        array.push(supportLines, newLine)
        
        newLabel = label.new(bar_index[pivotLength], pl, text=labelText,
             xloc=xloc.bar_index, yloc=yloc.price, style=label.style_label_up,
             color=color.rgb(0, 128, 0, 80), textcolor=color.white, size=size.small)
        array.push(supportLabels, newLabel)
    else // Level exists, update touch count
        for i = 0 to array.size(supportLevels) - 1
            supLevel = array.get(supportLevels, i)
            if not na(supLevel) and math.abs((pl - supLevel) / supLevel * 100) < minDistance
                touches = array.get(supportTouches, i) + 1
                array.set(supportTouches, i, touches)
                
                // Update line and label
                if array.size(supportLines) > i and array.size(supportLabels) > i
                    line.set_x2(array.get(supportLines, i), bar_index + 20)
                    strengthText = showStrength ? " (" + getLevelStrength_SR(touches) + ")" : ""
                    labelText = "S: " + str.tostring(supLevel, "#.##") + strengthText + " [" + str.tostring(touches) + "]"
                    label.set_text(array.get(supportLabels, i), labelText)
                    label.set_x(array.get(supportLabels, i), bar_index[pivotLength]) // Reset x-position to current pivot

                // Add touch label
                if showStrength and touches >= 2 // Only show touch labels for moderate/strong touches
                    touchLabel = label.new(x=bar_index, y=low - (hl2 * 0.001),
                                  text="S Touch (" + str.tostring(touches) + ")",
                                  xloc=xloc.bar_index, yloc=yloc.price, style=label.style_label_up,
                                  color=color.rgb(0, 128, 0, 50), textcolor=color.white, size=size.tiny)
                    array.push(touchSupportLabels, touchLabel)
                    array.push(touchSupportLevelIndex, i) // Store index of the level this touch belongs to
                break

// Update touches and labels for existing levels
if array.size(resistanceLevels) > 0 and showMajorLevels
    for i = 0 to array.size(resistanceLevels) - 1
        level = array.get(resistanceLevels, i)
        if not na(level) and not na(high)
            if math.abs((high - level) / level * 100) < touchSensitivity
                touches = array.get(resistanceTouches, i)
                array.set(resistanceTouches, i, touches + 1)
                newTouches = touches + 1
                
                // Improved touch label visibility
                newLabelTouch = label.new(bar_index, high + (hl2 * 0.001), text="⬇",
                                  style=label.style_label_down,
                                  color=color.new(color.red, 20),    // Semi-transparent background
                                  textcolor=color.white,             // White text
                                  size=size.normal)                  // Larger size
                array.push(touchResistantLabels, newLabelTouch)
                array.push(touchResistanceLevelIndex, i)

                if array.size(resistanceLabels) > i
                    label = array.get(resistanceLabels, i)
                    strengthText = showStrength ? " (" + getLevelStrength_SR(newTouches) + ")" : ""
                    labelText = "R: " + str.tostring(level, "#.##") + strengthText + " [" + str.tostring(newTouches) + "]"
                    label.set_text(label, labelText)
                    // Change color based on strength if feature is active
                    if showStrength
                        label.set_color(label, newTouches >= levelStrength ? color.new(color.maroon, 30) : color.new(color.red, 50))
                    else
                        label.set_color(label, color.new(color.red, 50)) // Default if strength text is off

if array.size(supportLevels) > 0 and showMajorLevels
    for i = 0 to array.size(supportLevels) - 1
        level = array.get(supportLevels, i)
        if not na(level) and not na(low)
            if math.abs((low - level) / level * 100) < touchSensitivity
                touches = array.get(supportTouches, i)
                array.set(supportTouches, i, touches + 1)
                newTouches = touches + 1
                
                // Improved touch label visibility
                newLabelTouchSupport = label.new(bar_index, low - (hl2 * 0.001), text="⬆",
                                         style=label.style_label_up,
                                         color=color.new(color.green, 20),     // Semi-transparent background
                                         textcolor=color.white,                // White text
                                         size=size.normal)                     // Larger size
                array.push(touchSupportLabels, newLabelTouchSupport)
                array.push(touchSupportLevelIndex, i)
                
                if array.size(supportLabels) > i
                    label = array.get(supportLabels, i)
                    strengthText = showStrength ? " (" + getLevelStrength_SR(newTouches) + ")" : ""
                    labelText = "S: " + str.tostring(level, "#.##") + strengthText + " [" + str.tostring(newTouches) + "]"
                    label.set_text(label, labelText)
                    // Change color based on strength if feature is active
                    if showStrength
                        label.set_color(label, newTouches >= levelStrength ? color.new(color.lime, 30) : color.new(color.green, 50))
                    else
                        label.set_color(label, color.new(color.green, 50)) // Default if strength text is off


// Update line endpoints
if array.size(resistanceLines) > 0 and showMajorLevels
    for i = 0 to array.size(resistanceLines) - 1
        line.set_x2(array.get(resistanceLines, i), bar_index + 20)
        touches = array.get(resistanceTouches, i)
        line.set_width(array.get(resistanceLines, i), touches >= levelStrength ? 2 : 1)
        // Set line color based on strength (if showStrength is true, use lime/maroon, else default red)
        if showStrength
            line.set_color(array.get(resistanceLines, i), touches >= levelStrength ? color.new(color.maroon, 30) : color.new(color.red, 50))
        else
            line.set_color(array.get(resistanceLines, i), color.new(color.red, 50))

if array.size(supportLines) > 0 and showMajorLevels
    for i = 0 to array.size(supportLines) - 1
        line.set_x2(array.get(supportLines, i), bar_index + 20)
        touches = array.get(supportTouches, i)
        line.set_width(array.get(supportLines, i), touches >= levelStrength ? 2 : 1)
        // Set line color based on strength (if showStrength is true, use lime/green, else default green)
        if showStrength
            line.set_color(array.get(supportLines, i), touches >= levelStrength ? color.new(color.lime, 30) : color.new(color.green, 50))
        else
            line.set_color(array.get(supportLines, i), color.new(color.green, 50))


// Update zone data for the current bar
if barstate.isrealtime or barstate.ishistory
    updateZoneData(close, zoneWidth, close, open)

// Call calculatePrediction() on each bar for consistency in dynamic updates
var PredictionData currentPrediction = na // Ensure this is `var` to persist across bars
currentPrediction := calculatePrediction()

// Update prediction and plots only on the last bar to avoid excessive repainting
// Note: The previous logic for `upperPredictionLine` and `lowerPredictionLine` is not strictly needed
// if you plot directly using `currentPrediction.upperBound` etc. with `na` for conditional display.
// However, I'll keep your pattern of assigning to `var` plotting variables for consistency with your base.
var float upperPredictionLine = na
var float lowerPredictionLine = na
var color plotColor = color.new(color.green, 90) // Default color, will be updated

if barstate.islast
    // Only update these variables on the last bar for efficiency in repainting for display
    upperPredictionLine := currentPrediction.upperBound
    lowerPredictionLine := currentPrediction.lowerBound
    
    zoneColor = currentPrediction.isBullish ? color.green : color.red
    zoneOpacity = math.min(90, math.max(20, 100 - (currentPrediction.confidence * 0.7)))
    plotColor := color.new(zoneColor, zoneOpacity)

// ═══════════════════════════════════════════════════════════════════════════════
// Visualization
// ═══════════════════════════════════════════════════════════════════════════════
// Plot EMA200
plot(showEma200 ? ema200 : na, "EMA200", color=color.blue, linewidth=1)

// Plot prediction zone
// Corrected: Removed 'var plot' prefix from here. Assign plot object directly.
predictionPlotUpperID = plot(showPrediction ? upperPredictionLine : na, "Upper Prediction", color=plotColor, style=plot.style_line, linewidth=2)
predictionPlotLowerID = plot(showPrediction ? lowerPredictionLine : na, "Lower Prediction", color=plotColor, style=plot.style_line, linewidth=2)

// Fill between the two plots. This must also be at global scope.
fill(predictionPlotUpperID, predictionPlotLowerID, color=showPrediction ? plotColor : na)


// Price Zone plotting
if showZones and array.size(historicalPriceZones) > 0
    for i = 0 to array.size(historicalPriceZones) - 1
        zone = array.get(historicalPriceZones, i)
        // Only plot zones that are from the current day OR if historical zones are enabled
        // and they are not completely decayed/removed.
        calculatedWeight = f_calculateZoneWeight(zone, dayofmonth, historicalZoneDecayFactor, useHistoricalZones, useSRInfluenceOnZones, srProximityDistancePercent, srModerateStrengthMultiplier, srStrongStrengthMultiplier, srOverallAdditiveWeight, useDailyPivots, zonePivotProximityBoost, pivotProximityDistancePercentZone, prevDayPivot, prevDayR1, prevDayS1, prevDayR2, prevDayS2, current_bar_timeframe_minutes) // Added new args
        
        if (zone.createdDayOfMonth == dayofmonth or useHistoricalZones) and math.abs(calculatedWeight) > 0.01
            color zoneColor = color.new(color.gray, 90)
            
            if calculatedWeight > 0.1 // Strong bullish
                zoneColor := color.new(color.lime, 80)
            else if calculatedWeight < -0.1 // Strong bearish
                zoneColor := color.new(color.red, 80)
            else if calculatedWeight > 0 // Mild bullish
                zoneColor := color.new(color.green, 90)
            else if calculatedWeight < 0 // Mild bearish
                zoneColor := color.new(color.orange, 90)
            
            box.new(left=bar_index[0], top=zone.end, right=bar_index[0] + 1, bottom=zone.start,
                  bgcolor=zoneColor, border_width=0, extend=extend.right)

// Table for Prediction Info
if barstate.islast // No need to check array.size(predictions) > 0, as currentPrediction is var and will be initialized.
    // Use `currentPrediction` for table display, as it's the most up-to-date data.
    table.cell(predictionTable, 0, 0, "Daily Close Prediction", bgcolor=color.new(color.blue, 90), text_color=color.white)
    table.cell(predictionTable, 1, 0, currentPrediction.isBullish ? "BULLISH" : "BEARISH",
          bgcolor=color.new(currentPrediction.isBullish ? color.green : color.red, 90), text_color=color.white)
    
    table.cell(predictionTable, 0, 1, "Confidence", bgcolor=color.new(color.blue, 90), text_color=color.white)
    table.cell(predictionTable, 1, 1, str.tostring(currentPrediction.confidence, "#.##") + "%",
          bgcolor=color.new(color.blue, 90), text_color=color.white)
    
    table.cell(predictionTable, 0, 2, "Reason", bgcolor=color.new(color.blue, 90), text_color=color.white)
    table.cell(predictionTable, 1, 2, currentPrediction.reason, text_halign=text.align_left)
    
    table.cell(predictionTable, 0, 3, "Predicted Price Range", bgcolor=color.new(color.blue, 90), text_color=color.white)
    table.cell(predictionTable, 1, 3, str.tostring(currentPrediction.lowerBound, "#.##") + " - " + str.tostring(currentPrediction.upperBound, "#.##"), text_halign=text.align_left)

    // Add Daily Anchor info if enabled
    if useDailyPivots and not na(prevDayPivot)
        table.cell(predictionTable, 0, 4, "Prev Day Pivot", bgcolor=color.new(color.blue, 90), text_color=color.white)
        table.cell(predictionTable, 1, 4, str.tostring(prevDayPivot, "#.##") + " (R1: " + str.tostring(prevDayR1, "#.##") + ", S1: " + str.tostring(prevDayS1, "#.##") + ")", text_halign=text.align_left)
    
    if useDailySMA and not na(dailySMA)
        table.cell(predictionTable, 0, 5, "Daily SMA (" + str.tostring(dailySMALength) + ")", bgcolor=color.new(color.blue, 90), text_color=color.white)
        table.cell(predictionTable, 1, 5, str.tostring(dailySMA, "#.##"), text_halign=text.align_left)


// Display User Info at end
if barstate.islast
    label.new(x=bar_index, y=close, xloc=xloc.bar_index, yloc=yloc.price,
              text="User: " + USER_LOGIN + "\nCreated: " + CREATION_TIME,
              style=label.style_label_up, textcolor=color.white,
              color=color.rgb(255, 165, 0, 80), size=size.small)
