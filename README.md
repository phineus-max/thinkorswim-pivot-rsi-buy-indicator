# thinkorswim-pivot-rsi-buy-indicator
A ThinkOrSwim (Thinkscript) indicator that identifies potential buy opportunities based on pivot levels, RSI, implied volatility, and squeeze conditions, with confidence scoring and visual labels.

-Created: 7/9/25  
-Last Updated: 7/10/25  
-All rights reserved  
-Feedback, suggestions, and constructive criticism are welcome!  
-Please feel free to open an Issue or submit a Pull Request if you have ideas to improve this indicator.  

README - Pivot Zone Momentum Buy Strategy



Purpose:
This code is designed to function as a Intraday Reversal Buy Signal for ThinkOrSwim. In English (and not complex financial language) this means that this script looks for when the asset (can be stocks, ETFs, futures) is oversold and is starting to show signs of strength, especially after a quiet period where the price might soon jump. It only signals buys during calmer parts of the day and when the price is in a favorable zone. It also rates each signal’s strength to help traders decide which ones to trust more.


DISCLAIMERS!
A decent amount of the code is based on built-in ThinkOrSwim indicators and is built in ThinkScript. Due to limitations of the ThinkScript software, I am unable to tie a sell signal to the buy signal to mark when a good time to exit a position is. This is because ThinkScript cannot track or access account information to see past or current buy/sell statuses of assets. ThinkScript also cannot automatically place trades, which means that success of this indicator in trading is HEAVILY dependent on latency, speed of connection, and reaction time/decision making of the trader. 


This is all to say that while this indicator may be accurate, success (and/or profit) is heavily dependent on factors that this code simply cannot account for. 

(While this can be used to trade stocks and ETFs, I have mostly tested it with futures, such as /ES and /MES for example. This is because this indicator is not designed for long term holds, but rather short term buying and selling (or scalping).)

NOTE: Just to clarify, if you would like this code to work, you will need to access ThinkOrSwim, go to Studies -> Create New Study and then import the code. This will not work if imported under strategy! Stay tuned for the manual front-testing data soon!



How the code works:

A buy signal is generated when:
Price is in the S1 (support 1) to PP (pivot point) zone. 

def pp = (high[1] + low[1] + close[1]) / 3;
def s1 = pp * 2 - high[1];
def inPivotZone = close >= s1 and close <= pp;

To explain the rationale behind this, I wanted to focus on areas where price may find support and bounce off of support levels. The price is typically not too high (signaling that it is overbought) or not too low (potentially indicative that support is rapidly fading and the location of the next support is uncertain). In simpler words, this zone increases the chance of catching a reversal or continuation upward from a dip. 



RSI is rising and optionally less than 40

def rsi = RSI(length = 14);
def rsiRising = rsi > rsi[1];
def rsiLow = rsi < 40;

Signal occurs during designated time frame:
def morning = SecondsFromTime(935) >= 0 and SecondsTillTime(1100) >= 0;
def afternoon = SecondsFromTime(1330) >= 0 and SecondsTillTime(1530) >= 0;
def timeOK = if rthOnly then (morning or afternoon) else 1;


Base buy signal which combines all of the above:
def baseBuySignal = inPivotZone and rsiRising and timeOK;

These are the additional confidence filters:

Implied Volatility (IV) is rising:
def iv = imp_volatility();
def ivRising = iv > iv[1];

Why? Because when Implied Volatility rises, it shows that traders expect a bigger move soon, whether it be up or down. 


Recent TTM Squeeze fired (suggesting volatility expansion):
def squeeze = TTM_Squeeze().SqueezeAlert;
def recentSqueeze = squeeze[1] == 1 or squeeze[2] == 1;

Bullish candle (close > open):
def bullishCandle = close > open;


The TTM Squeeze is an indicator that combines Bollinger Bands and Keltner channels. Bollinger bands measure volatility and Keltner channels measure average price range. It signals a squeeze when volatility contracts, then alerts when the squeeze is released, often right before a breakout. Helps confirm that a dip isn't just noise, but the start of a real move with momentum. 


RSI is below 40 (oversold strength):
def rsiLow = rsi < 40;


Calculate total score:
def score = (ivRising + recentSqueeze + bullishCandle + rsiLow);


Output:

Chart Bubbles appear at the buy signal bar showing "BUY X/4" where X is the confidence score:
AddChartBubble(baseBuySignal, low, "BUY " + score + "/4",
    if score >= 3 then Color.GREEN else if score == 2 then Color.YELLOW else Color.RED,
    no);

Debug labels:

AddLabel(showLabels and baseBuySignal, "IV Rising: " + ivRising, if ivRising then Color.GREEN else Color.RED);
AddLabel(showLabels and baseBuySignal, "Squeeze: " + recentSqueeze, if recentSqueeze then Color.MAGENTA else Color.GRAY);
AddLabel(showLabels and baseBuySignal, "Bullish: " + bullishCandle, if bullishCandle then Color.YELLOW else Color.DARK_GRAY);
AddLabel(showLabels and baseBuySignal, "RSI < 40: " + rsiLow, if rsiLow then Color.CYAN else Color.LIGHT_GRAY);



Full code:
declare upper;
input showLabels = yes;
input rthOnly = yes;
input sellWindowBars = 5;
input takeProfit = 5.0;


# === TIME FILTER ===
def morning = SecondsFromTime(935) >= 0 and SecondsTillTime(1100) >= 0;
def afternoon = SecondsFromTime(1330) >= 0 and SecondsTillTime(1530) >= 0;
def timeOK = if rthOnly then (morning or afternoon) else 1;

# === PIVOT LEVELS ===
def pp = (high[1] + low[1] + close[1]) / 3;
def s1 = pp * 2 - high[1];

# === RSI ===
def rsi = RSI(length = 14);
def rsiRising = rsi > rsi[1];
def rsiLow = rsi < 40;

# === BUY LOGIC ===
def inPivotZone = close >= s1 and close <= pp;
def baseBuySignal = inPivotZone and rsiRising and timeOK;

# === CONFIDENCE FILTERS ===
def iv = imp_volatility();
def ivRising = iv > iv[1];

def squeeze = TTM_Squeeze().SqueezeAlert;
def recentSqueeze = squeeze[1] == 1 or squeeze[2] == 1;

def bullishCandle = close > open;

# === SCORING ===
def score = (ivRising + recentSqueeze + bullishCandle + rsiLow);

# === TRACK ENTRY STATE ===
rec entryBar = if baseBuySignal then BarNumber() else entryBar[1];
rec entryPrice = if baseBuySignal then close else entryPrice[1];
def barsSinceEntry = BarNumber() - entryBar;

# === CHART BUBBLES ===
AddChartBubble(baseBuySignal, low, "BUY " + score + "/4",
    if score >= 3 then Color.GREEN else if score == 2 then Color.YELLOW else Color.RED,
    no);

# === DEBUG LABELS ===
AddLabel(showLabels and baseBuySignal, "IV Rising: " + ivRising, if ivRising then Color.GREEN else Color.RED);
AddLabel(showLabels and baseBuySignal, "Squeeze: " + recentSqueeze, if recentSqueeze then Color.MAGENTA else Color.GRAY);
AddLabel(showLabels and baseBuySignal, "Bullish: " + bullishCandle, if bullishCandle then Color.YELLOW else Color.DARK_GRAY);
AddLabel(showLabels and baseBuySignal, "RSI < 40: " + rsiLow, if rsiLow then Color.CYAN else Color.LIGHT_GRAY);


Front testing data:

This strategy is currently undergoing manual front testing on intraday charts for SPY, /ES, and /MES. The goal is to evaluate real-time performance, signal quality, and consistency across different market conditions.

A trading journal will be created to track:

Entry and exit points

Signal confidence scores (0–4)

Market context at time of signal

Outcome and trade notes

This journal will help refine the strategy, identify patterns, and guide future improvements such as exit logic, trend filters, or risk adjustments.

Note: No performance claims are made yet — this strategy is still in the testing phase.
