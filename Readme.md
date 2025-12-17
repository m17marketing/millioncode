#region Using declarations
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.ComponentModel.DataAnnotations;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Input;
using System.Windows.Media;
using System.Xml.Serialization;
using NinjaTrader.Cbi;
using NinjaTrader.Gui;
using NinjaTrader.Gui.Chart;
using NinjaTrader.Gui.SuperDom;
using NinjaTrader.Gui.Tools;
using NinjaTrader.Data;
using NinjaTrader.NinjaScript;
using NinjaTrader.Core.FloatingPoint;
using NinjaTrader.NinjaScript.Indicators;
using NinjaTrader.NinjaScript.DrawingTools;
#endregion

namespace NinjaTrader.NinjaScript.Strategies
{
    public class millioncode : Strategy
    {
        #region Variables
        // === RISK ENGINE ===
        private double dailyRealizedPnL = 0;
        private int tradesCountToday = 0;
        private int consecutiveLosses = 0;
        private bool dailyLockActive = false;
        private bool cooldownActive = false;
        private DateTime cooldownEndTime;
        private DateTime currentTradingDay;
        private double entryPrice = 0;
        
        // === REGIME/HTF ===
        private double htfTrend = 0;
        private double htfATR = 0;
        private double htfADX = 0;
        private bool htfDataReady = false;
        
        // === ORDERFLOW ===
        private OrderFlowCumulativeDelta cumulativeDelta;
        private double currentDelta = 0;
        private double prevDelta = 0;
        private double deltaChange = 0;
        private double deltaSMA = 0;
        private List<double> deltaHistory;
        private const int DELTA_HISTORY_SIZE = 50;
        private bool orderFlowReady = false;
        
        // === VWAP ===
        private double vwapValue = 0;
        private double vwapUpper = 0;
        private double vwapLower = 0;
        private double vwapSum = 0;
        private double volumeSum = 0;
        
        // === SIGNAL STATE ===
        private bool longSignal = false;
        private bool shortSignal = false;
        private int signalStrength = 0;
        private string signalReason = "";
        
        // === POSITION TRACKING ===
        private bool inPosition = false;
        private int barsSinceEntry = 0;
        private double trailingStopPrice = 0;
        private MarketPosition lastPosition = MarketPosition.Flat;
        
        // === DEBUG ===
        private int lastDiagnosticBar = 0;
        #endregion

        #region Parameters - Risk Management
        [NinjaScriptProperty]
        [Range(100, 2500)]
        [Display(Name="Max Daily Loss ($)", Order=1, GroupName="1. Risk Management")]
        public double MaxDailyLoss { get; set; }
        
        [NinjaScriptProperty]
        [Range(1, 10)]
        [Display(Name="Max Trades Per Day", Order=2, GroupName="1. Risk Management")]
        public int MaxTradesPerDay { get; set; }
        
        [NinjaScriptProperty]
        [Range(1, 5)]
        [Display(Name="Position Size (Contracts)", Order=3, GroupName="1. Risk Management")]
        public int PositionSize { get; set; }
        
        [NinjaScriptProperty]
        [Range(1, 5)]
        [Display(Name="Max Consecutive Losses", Order=4, GroupName="1. Risk Management")]
        public int MaxConsecutiveLosses { get; set; }
        
        [NinjaScriptProperty]
        [Range(5, 60)]
        [Display(Name="Cooldown Minutes", Order=5, GroupName="1. Risk Management")]
        public int CooldownMinutes { get; set; }
        
        [NinjaScriptProperty]
        [Range(100, 1500)]
        [Display(Name="Daily Profit Target ($)", Order=6, GroupName="1. Risk Management")]
        public double DailyProfitTarget { get; set; }
        #endregion

        #region Parameters - Session Filter
        [NinjaScriptProperty]
        [PropertyEditor("NinjaTrader.Gui.Tools.TimeEditorKey")]
        [Display(Name="Session Start", Order=1, GroupName="2. Session Filter")]
        public DateTime SessionStart { get; set; }
        
        [NinjaScriptProperty]
        [PropertyEditor("NinjaTrader.Gui.Tools.TimeEditorKey")]
        [Display(Name="Session End", Order=2, GroupName="2. Session Filter")]
        public DateTime SessionEnd { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Trade Monday", Order=3, GroupName="2. Session Filter")]
        public bool TradeMonday { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Trade Tuesday", Order=4, GroupName="2. Session Filter")]
        public bool TradeTuesday { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Trade Wednesday", Order=5, GroupName="2. Session Filter")]
        public bool TradeWednesday { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Trade Thursday", Order=6, GroupName="2. Session Filter")]
        public bool TradeThursday { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Trade Friday", Order=7, GroupName="2. Session Filter")]
        public bool TradeFriday { get; set; }
        
        [NinjaScriptProperty]
        [Range(0, 30)]
        [Display(Name="Skip First X Minutes", Order=8, GroupName="2. Session Filter")]
        public int SkipFirstMinutes { get; set; }
        #endregion

        #region Parameters - Regime Filter
        [NinjaScriptProperty]
        [Range(5, 60)]
        [Display(Name="Higher TF Period (Min)", Order=1, GroupName="3. Regime Filter")]
        public int HTFPeriod { get; set; }
        
        [NinjaScriptProperty]
        [Range(15, 40)]
        [Display(Name="ADX Threshold", Order=2, GroupName="3. Regime Filter")]
        public int ADXThreshold { get; set; }
        
        [NinjaScriptProperty]
        [Range(5, 50)]
        [Display(Name="EMA Fast", Order=3, GroupName="3. Regime Filter")]
        public int EMAFast { get; set; }
        
        [NinjaScriptProperty]
        [Range(20, 100)]
        [Display(Name="EMA Slow", Order=4, GroupName="3. Regime Filter")]
        public int EMASlow { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Allow Counter-Trend", Order=5, GroupName="3. Regime Filter")]
        public bool AllowCounterTrend { get; set; }
        #endregion

        #region Parameters - Delta & Volume
        [NinjaScriptProperty]
        [Range(1.0, 5.0)]
        [Display(Name="Delta Spike Multiplier", Order=1, GroupName="4. Delta & Volume")]
        public double DeltaSpikeMultiplier { get; set; }
        
        [NinjaScriptProperty]
        [Range(5, 50)]
        [Display(Name="Delta Avg Period", Order=2, GroupName="4. Delta & Volume")]
        public int DeltaAvgPeriod { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Use Delta Divergence", Order=3, GroupName="4. Delta & Volume")]
        public bool UseDeltaDivergence { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Use VWAP Filter", Order=4, GroupName="4. Delta & Volume")]
        public bool UseVWAPFilter { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Require Delta Confirmation", Order=5, GroupName="4. Delta & Volume")]
        public bool RequireDeltaConfirmation { get; set; }
        
        [NinjaScriptProperty]
        [Range(1.2, 4.0)]
        [Display(Name="Volume Spike Factor", Order=6, GroupName="4. Delta & Volume")]
        public double VolSpikeFactor { get; set; }
        
        [NinjaScriptProperty]
        [Range(10, 100)]
        [Display(Name="Volume Avg Period", Order=7, GroupName="4. Delta & Volume")]
        public int VolAvgPeriod { get; set; }
        #endregion

        #region Parameters - Entry Logic
        [NinjaScriptProperty]
        [Range(3, 20)]
        [Display(Name="Sweep Lookback Bars", Order=1, GroupName="5. Entry Logic")]
        public int SweepLookback { get; set; }
        
        [NinjaScriptProperty]
        [Range(2, 6)]
        [Display(Name="Min Signal Strength (1-6)", Order=2, GroupName="5. Entry Logic")]
        public int MinSignalStrength { get; set; }
        #endregion

        #region Parameters - Exit Logic
        [NinjaScriptProperty]
        [Range(8, 60)]
        [Display(Name="Stop Loss (Ticks)", Order=1, GroupName="6. Exit Logic")]
        public int StopLossTicks { get; set; }
        
        [NinjaScriptProperty]
        [Range(1.5, 5.0)]
        [Display(Name="Profit Target (R:R)", Order=2, GroupName="6. Exit Logic")]
        public double RiskRewardRatio { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Use Trailing Stop", Order=3, GroupName="6. Exit Logic")]
        public bool UseTrailingStop { get; set; }
        
        [NinjaScriptProperty]
        [Range(0.5, 2.0)]
        [Display(Name="Trail Activation (x Risk)", Order=4, GroupName="6. Exit Logic")]
        public double TrailActivationMultiple { get; set; }
        
        [NinjaScriptProperty]
        [Range(5, 30)]
        [Display(Name="Trail Distance (Ticks)", Order=5, GroupName="6. Exit Logic")]
        public int TrailDistanceTicks { get; set; }
        
        [NinjaScriptProperty]
        [Range(10, 60)]
        [Display(Name="Max Bars In Trade", Order=6, GroupName="6. Exit Logic")]
        public int MaxBarsInTrade { get; set; }
        #endregion

        #region Parameters - Debug
        [NinjaScriptProperty]
        [Display(Name="Debug Logging", Order=1, GroupName="9. Debug")]
        public bool DebugLogging { get; set; }
        
        [NinjaScriptProperty]
        [Display(Name="Draw Signals on Chart", Order=2, GroupName="9. Debug")]
        public bool DrawSignals { get; set; }
        #endregion

        #region OnStateChange
        protected override void OnStateChange()
        {
            if (State == State.SetDefaults)
            {
                Description = "millioncode v4.0 – OrderFlow mit 1-Tick Data Series | Tradeify 50k";
                Name = "millioncode";
                Calculate = Calculate.OnEachTick;
                EntriesPerDirection = 1;
                EntryHandling = EntryHandling.AllEntries;
                IsExitOnSessionCloseStrategy = true;
                ExitOnSessionCloseSeconds = 300;
                IsFillLimitOnTouch = false;
                MaximumBarsLookBack = MaximumBarsLookBack.TwoHundredFiftySix;
                OrderFillResolution = OrderFillResolution.Standard;
                Slippage = 2;
                StartBehavior = StartBehavior.WaitUntilFlat;
                TimeInForce = TimeInForce.Gtc;
                TraceOrders = true;
                RealtimeErrorHandling = RealtimeErrorHandling.StopCancelClose;
                StopTargetHandling = StopTargetHandling.PerEntryExecution;
                BarsRequiredToTrade = 50;
                IsInstantiatedOnEachOptimizationIteration = true;
                
                // === TRADEIFY 50K DEFAULTS ===
                MaxDailyLoss = 500;
                MaxTradesPerDay = 3;
                PositionSize = 1;
                MaxConsecutiveLosses = 2;
                CooldownMinutes = 30;
                DailyProfitTarget = 400;
                
                // Session
                SessionStart = DateTime.Parse("09:30", System.Globalization.CultureInfo.InvariantCulture);
                SessionEnd = DateTime.Parse("12:00", System.Globalization.CultureInfo.InvariantCulture);
                SkipFirstMinutes = 5;
                
                TradeMonday = true;
                TradeTuesday = true;
                TradeWednesday = true;
                TradeThursday = true;
                TradeFriday = false;
                
                // Regime
                HTFPeriod = 15;
                ADXThreshold = 20;
                EMAFast = 9;
                EMASlow = 21;
                AllowCounterTrend = false;
                
                // Delta & Volume
                DeltaSpikeMultiplier = 1.5;
                DeltaAvgPeriod = 20;
                UseDeltaDivergence = true;
                UseVWAPFilter = true;
                RequireDeltaConfirmation = true;
                VolSpikeFactor = 1.5;
                VolAvgPeriod = 30;
                
                // Entry
                SweepLookback = 8;
                MinSignalStrength = 3;
                
                // Exit
                StopLossTicks = 20;
                RiskRewardRatio = 2.5;
                UseTrailingStop = true;
                TrailActivationMultiple = 1.5;
                TrailDistanceTicks = 12;
                MaxBarsInTrade = 30;
                
                // Debug
                DebugLogging = true;
                DrawSignals = true;
            }
            else if (State == State.Configure)
            {
                // HTF Data Series für Regime Filter
                AddDataSeries(BarsPeriodType.Minute, HTFPeriod);
                
                // 1-TICK Data Series für OrderFlow - DAS IST DER KEY!
                AddDataSeries(BarsPeriodType.Tick, 1);
            }
            else if (State == State.DataLoaded)
            {
                deltaHistory = new List<double>(DELTA_HISTORY_SIZE);
                currentTradingDay = Time[0].Date;
                
                // OrderFlowCumulativeDelta auf der TICK Data Series (BarsArray[2])
                cumulativeDelta = OrderFlowCumulativeDelta(BarsArray[2], CumulativeDeltaType.BidAsk, CumulativeDeltaPeriod.Session, 0);
                
                ClearOutputWindow();
                Print("╔════════════════════════════════════════════════════════════╗");
                Print("║      millioncode v4.0 – REAL ORDERFLOW (Tick Series)       ║");
                Print("╠════════════════════════════════════════════════════════════╣");
                Print($"║ Instrument: {Instrument.FullName}");
                Print($"║ Session: {SessionStart:HH:mm} - {SessionEnd:HH:mm}");
                Print($"║ Max Daily Loss: ${MaxDailyLoss} | Target: ${DailyProfitTarget}");
                Print("╠════════════════════════════════════════════════════════════╣");
                Print("║ BarsArray[0]: Primary (Minutes)                            ║");
                Print($"║ BarsArray[1]: HTF ({HTFPeriod} Min)                                  ║");
                Print("║ BarsArray[2]: 1-Tick (for OrderFlow)                       ║");
                Print("║ OrderFlowCumulativeDelta: INITIALIZED                      ║");
                Print("╚════════════════════════════════════════════════════════════╝");
            }
        }
        #endregion

        #region OnBarUpdate
        protected override void OnBarUpdate()
        {
            // === TICK DATA SERIES (BarsInProgress == 2) - Update OrderFlow ===
            if (BarsInProgress == 2)
            {
                UpdateOrderFlow();
                return;
            }
            
            // === HTF DATA SERIES (BarsInProgress == 1) ===
            if (BarsInProgress == 1)
            {
                if (CurrentBars[1] >= 30)
                    UpdateRegimeFilter();
                return;
            }
            
            // === PRIMARY DATA SERIES (BarsInProgress == 0) ===
            if (BarsInProgress != 0)
                return;
            
            // Safety check
            if (CurrentBars[0] < BarsRequiredToTrade)
                return;
            
            // New day reset
            if (Time[0].Date != currentTradingDay)
            {
                ResetDaily();
                currentTradingDay = Time[0].Date;
            }
            
            // Daily lock check
            if (dailyLockActive)
                return;
            
            // Cooldown check
            if (cooldownActive && Time[0] < cooldownEndTime)
                return;
            else if (cooldownActive)
            {
                cooldownActive = false;
                consecutiveLosses = 0;
            }
            
            // Session filter
            if (!IsValidSession())
                return;
            
            // Update VWAP
            UpdateVWAP();
            
            // Position management
            if (Position.MarketPosition != MarketPosition.Flat)
            {
                ManagePosition();
                return;
            }
            
            // Max trades check
            if (tradesCountToday >= MaxTradesPerDay)
                return;
            
            // HTF ready check
            if (!htfDataReady)
                return;
            
            // OrderFlow ready check
            if (!orderFlowReady)
            {
                if (DebugLogging && CurrentBar % 100 == 0)
                    Print($"[WAIT] OrderFlow not ready yet...");
                return;
            }
            
            // Signal detection
            CheckEntrySignals();
            
            // Entry execution
            if (longSignal && signalStrength >= MinSignalStrength)
            {
                if (AllowCounterTrend || htfTrend >= 0)
                    ExecuteLongEntry();
            }
            else if (shortSignal && signalStrength >= MinSignalStrength)
            {
                if (AllowCounterTrend || htfTrend <= 0)
                    ExecuteShortEntry();
            }
            
            // Diagnostics
            if (DebugLogging && CurrentBar - lastDiagnosticBar >= 100)
            {
                PrintDiagnostics();
                lastDiagnosticBar = CurrentBar;
            }
        }
        #endregion

        #region OrderFlow Update
        private void UpdateOrderFlow()
        {
            if (cumulativeDelta == null)
                return;
            
            try
            {
                // Update the cumulative delta indicator
                if (CurrentBars[2] > 0)
                {
                    prevDelta = currentDelta;
                    currentDelta = cumulativeDelta.DeltaClose[0];
                    deltaChange = currentDelta - prevDelta;
                    
                    // Store for averaging
                    if (Math.Abs(deltaChange) > 0)
                    {
                        deltaHistory.Add(Math.Abs(deltaChange));
                        if (deltaHistory.Count > DELTA_HISTORY_SIZE)
                            deltaHistory.RemoveAt(0);
                    }
                    
                    // Calculate average
                    if (deltaHistory.Count > 10)
                    {
                        deltaSMA = deltaHistory.Average();
                        orderFlowReady = true;
                    }
                }
            }
            catch (Exception ex)
            {
                if (DebugLogging && CurrentBars[2] % 1000 == 0)
                    Print($"[OF-WARN] {ex.Message}");
            }
        }
        #endregion

        #region VWAP Calculation
        private void UpdateVWAP()
        {
            if (Bars.IsFirstBarOfSession)
            {
                vwapSum = 0;
                volumeSum = 0;
            }
            
            double typicalPrice = (High[0] + Low[0] + Close[0]) / 3;
            vwapSum += typicalPrice * Volume[0];
            volumeSum += Volume[0];
            
            if (volumeSum > 0)
            {
                vwapValue = vwapSum / volumeSum;
                
                double sumSquares = 0;
                int lookback = Math.Min(CurrentBar, 50);
                for (int i = 0; i < lookback; i++)
                {
                    double tp = (High[i] + Low[i] + Close[i]) / 3;
                    sumSquares += Math.Pow(tp - vwapValue, 2);
                }
                double stdDev = Math.Sqrt(sumSquares / lookback);
                
                vwapUpper = vwapValue + stdDev;
                vwapLower = vwapValue - stdDev;
            }
        }
        #endregion

        #region Session Validation
        private bool IsValidSession()
        {
            TimeSpan currentTime = Time[0].TimeOfDay;
            TimeSpan startTime = SessionStart.TimeOfDay.Add(TimeSpan.FromMinutes(SkipFirstMinutes));
            TimeSpan endTime = SessionEnd.TimeOfDay;
            
            DayOfWeek today = Time[0].DayOfWeek;
            bool dayAllowed = today switch
            {
                DayOfWeek.Monday => TradeMonday,
                DayOfWeek.Tuesday => TradeTuesday,
                DayOfWeek.Wednesday => TradeWednesday,
                DayOfWeek.Thursday => TradeThursday,
                DayOfWeek.Friday => TradeFriday,
                _ => false
            };
            
            return dayAllowed && currentTime >= startTime && currentTime <= endTime;
        }
        #endregion

        #region Regime Filter
        private void UpdateRegimeFilter()
        {
            try
            {
                double adx = ADX(BarsArray[1], 14)[0];
                double emaFastVal = EMA(Closes[1], EMAFast)[0];
                double emaSlowVal = EMA(Closes[1], EMASlow)[0];
                htfATR = ATR(BarsArray[1], 14)[0];
                htfADX = adx;
                
                if (adx > ADXThreshold)
                    htfTrend = emaFastVal > emaSlowVal ? 1 : emaFastVal < emaSlowVal ? -1 : 0;
                else
                    htfTrend = 0;
                
                htfDataReady = true;
            }
            catch { }
        }
        #endregion

        #region Entry Signals
        private void CheckEntrySignals()
        {
            longSignal = false;
            shortSignal = false;
            signalStrength = 0;
            signalReason = "";
            
            StringBuilder reasons = new StringBuilder();
            
            // 1. Volume Spike
            double volAvg = SMA(Volume, VolAvgPeriod)[0];
            bool volSpike = Volume[0] > (volAvg * VolSpikeFactor);
            
            if (!volSpike)
                return;
            
            signalStrength++;
            reasons.Append("Vol+");
            
            // 2. Liquidity Sweep
            bool bullishSweep = DetectBullishSweep();
            bool bearishSweep = DetectBearishSweep();
            
            if (!bullishSweep && !bearishSweep)
                return;
            
            signalStrength++;
            reasons.Append(bullishSweep ? "BullSweep+" : "BearSweep+");
            
            // 3. REAL Delta Confirmation
            bool deltaConfirm = false;
            bool deltaDivergence = false;
            
            if (deltaSMA > 0)
            {
                double deltaThreshold = deltaSMA * DeltaSpikeMultiplier;
                
                if (bullishSweep && deltaChange > deltaThreshold)
                {
                    deltaConfirm = true;
                    signalStrength++;
                    reasons.Append("Δ↑+");
                }
                else if (bearishSweep && deltaChange < -deltaThreshold)
                {
                    deltaConfirm = true;
                    signalStrength++;
                    reasons.Append("Δ↓+");
                }
                
                if (UseDeltaDivergence)
                {
                    if (bullishSweep && deltaChange > 0 && Low[0] < Low[1])
                    {
                        deltaDivergence = true;
                        signalStrength++;
                        reasons.Append("ΔDiv+");
                    }
                    else if (bearishSweep && deltaChange < 0 && High[0] > High[1])
                    {
                        deltaDivergence = true;
                        signalStrength++;
                        reasons.Append("ΔDiv+");
                    }
                }
            }
            
            if (RequireDeltaConfirmation && !deltaConfirm && !deltaDivergence)
                return;
            
            // 4. VWAP Filter
            if (UseVWAPFilter && vwapValue > 0)
            {
                if (bullishSweep && Close[0] < vwapValue)
                {
                    signalStrength++;
                    reasons.Append("VWAP<+");
                }
                else if (bearishSweep && Close[0] > vwapValue)
                {
                    signalStrength++;
                    reasons.Append("VWAP>+");
                }
            }
            
            // 5. Momentum
            bool bullMomentum = Close[0] > Close[1] && Close[0] > Open[0];
            bool bearMomentum = Close[0] < Close[1] && Close[0] < Open[0];
            
            if ((bullishSweep && bullMomentum) || (bearishSweep && bearMomentum))
            {
                signalStrength++;
                reasons.Append("Mom+");
            }
            
            // 6. HTF Alignment
            if ((bullishSweep && htfTrend > 0) || (bearishSweep && htfTrend < 0))
            {
                signalStrength++;
                reasons.Append("HTF+");
            }
            
            signalReason = reasons.ToString();
            
            if (bullishSweep && signalStrength >= MinSignalStrength)
            {
                longSignal = true;
                if (DebugLogging)
                    Print($"[SIGNAL] {Time[0]:HH:mm:ss} LONG | Str:{signalStrength} | Δ:{deltaChange:F0} | {signalReason}");
            }
            else if (bearishSweep && signalStrength >= MinSignalStrength)
            {
                shortSignal = true;
                if (DebugLogging)
                    Print($"[SIGNAL] {Time[0]:HH:mm:ss} SHORT | Str:{signalStrength} | Δ:{deltaChange:F0} | {signalReason}");
            }
        }
        
        private bool DetectBullishSweep()
        {
            double recentLow = MIN(Low, SweepLookback)[1];
            return Low[0] <= recentLow && Close[0] > recentLow + (2 * TickSize) && Close[0] > (Low[0] + High[0]) / 2;
        }
        
        private bool DetectBearishSweep()
        {
            double recentHigh = MAX(High, SweepLookback)[1];
            return High[0] >= recentHigh && Close[0] < recentHigh - (2 * TickSize) && Close[0] < (Low[0] + High[0]) / 2;
        }
        #endregion

        #region Order Execution
        private void ExecuteLongEntry()
        {
            double stopPrice = Low[0] - (StopLossTicks * TickSize);
            double riskAmount = Close[0] - stopPrice;
            double targetPrice = Close[0] + (riskAmount * RiskRewardRatio);
            
            double dollarRisk = riskAmount * Instrument.MasterInstrument.PointValue * PositionSize;
            if (dollarRisk > MaxDailyLoss * 0.5)
                return;
            
            EnterLong(PositionSize, "Long");
            SetStopLoss("Long", CalculationMode.Price, stopPrice, false);
            SetProfitTarget("Long", CalculationMode.Price, targetPrice);
            
            entryPrice = Close[0];
            barsSinceEntry = 0;
            trailingStopPrice = stopPrice;
            lastPosition = MarketPosition.Long;
            tradesCountToday++;
            
            if (DrawSignals)
                Draw.ArrowUp(this, "LE" + CurrentBar, true, 0, Low[0] - (8 * TickSize), Brushes.Lime);
            
            if (DebugLogging)
                Print($"[LONG] @ {Close[0]:F2} | Stop: {stopPrice:F2} | Target: {targetPrice:F2} | Δ:{deltaChange:F0}");
        }
        
        private void ExecuteShortEntry()
        {
            double stopPrice = High[0] + (StopLossTicks * TickSize);
            double riskAmount = stopPrice - Close[0];
            double targetPrice = Close[0] - (riskAmount * RiskRewardRatio);
            
            double dollarRisk = riskAmount * Instrument.MasterInstrument.PointValue * PositionSize;
            if (dollarRisk > MaxDailyLoss * 0.5)
                return;
            
            EnterShort(PositionSize, "Short");
            SetStopLoss("Short", CalculationMode.Price, stopPrice, false);
            SetProfitTarget("Short", CalculationMode.Price, targetPrice);
            
            entryPrice = Close[0];
            barsSinceEntry = 0;
            trailingStopPrice = stopPrice;
            lastPosition = MarketPosition.Short;
            tradesCountToday++;
            
            if (DrawSignals)
                Draw.ArrowDown(this, "SE" + CurrentBar, true, 0, High[0] + (8 * TickSize), Brushes.Red);
            
            if (DebugLogging)
                Print($"[SHORT] @ {Close[0]:F2} | Stop: {stopPrice:F2} | Target: {targetPrice:F2} | Δ:{deltaChange:F0}");
        }
        #endregion

        #region Position Management
        private void ManagePosition()
        {
            barsSinceEntry++;
            
            if (barsSinceEntry >= MaxBarsInTrade)
            {
                if (Position.MarketPosition == MarketPosition.Long)
                    ExitLong("TimeExit", "Long");
                else
                    ExitShort("TimeExit", "Short");
                return;
            }
            
            if (UseTrailingStop)
            {
                double unrealizedPnL = Position.GetUnrealizedProfitLoss(PerformanceUnit.Currency, Close[0]);
                double riskAmount = StopLossTicks * TickSize * Instrument.MasterInstrument.PointValue * PositionSize;
                
                if (unrealizedPnL >= riskAmount * TrailActivationMultiple)
                {
                    if (Position.MarketPosition == MarketPosition.Long)
                    {
                        double newStop = Close[0] - (TrailDistanceTicks * TickSize);
                        if (newStop > trailingStopPrice)
                        {
                            trailingStopPrice = newStop;
                            SetStopLoss("Long", CalculationMode.Price, newStop, false);
                        }
                    }
                    else
                    {
                        double newStop = Close[0] + (TrailDistanceTicks * TickSize);
                        if (newStop < trailingStopPrice)
                        {
                            trailingStopPrice = newStop;
                            SetStopLoss("Short", CalculationMode.Price, newStop, false);
                        }
                    }
                }
            }
        }
        #endregion

        #region Execution Updates
        protected override void OnExecutionUpdate(Execution execution, string executionId, double price, int quantity, MarketPosition marketPosition, string orderId, DateTime time)
        {
            if (execution.Order == null || execution.Order.OrderState != OrderState.Filled)
                return;
            
            string orderName = execution.Order.Name;
            
            if (orderName == "Long" || orderName == "Short")
            {
                entryPrice = price;
            }
            else
            {
                double tradePnL = lastPosition == MarketPosition.Long 
                    ? (price - entryPrice) * Instrument.MasterInstrument.PointValue * quantity
                    : (entryPrice - price) * Instrument.MasterInstrument.PointValue * quantity;
                
                dailyRealizedPnL += tradePnL;
                
                if (tradePnL < 0)
                {
                    consecutiveLosses++;
                    if (consecutiveLosses >= MaxConsecutiveLosses)
                    {
                        cooldownActive = true;
                        cooldownEndTime = time.AddMinutes(CooldownMinutes);
                    }
                }
                else
                    consecutiveLosses = 0;
                
                if (DebugLogging)
                    Print($"[{(tradePnL >= 0 ? "WIN" : "LOSS")}] PnL: ${tradePnL:F2} | Daily: ${dailyRealizedPnL:F2}");
                
                if (dailyRealizedPnL <= -MaxDailyLoss || dailyRealizedPnL >= DailyProfitTarget)
                    dailyLockActive = true;
            }
        }
        #endregion

        #region Daily Reset
        private void ResetDaily()
        {
            dailyRealizedPnL = 0;
            tradesCountToday = 0;
            consecutiveLosses = 0;
            dailyLockActive = false;
            cooldownActive = false;
            entryPrice = 0;
            currentDelta = 0;
            deltaHistory.Clear();
            orderFlowReady = false;
            vwapSum = 0;
            volumeSum = 0;
            
            if (DebugLogging)
                Print($"\n══════════ NEW DAY: {Time[0]:yyyy-MM-dd} ══════════\n");
        }
        #endregion

        #region Diagnostics
        private void PrintDiagnostics()
        {
            string trend = htfTrend > 0 ? "BULL" : htfTrend < 0 ? "BEAR" : "FLAT";
            Print($"[DIAG] {Time[0]:HH:mm} | HTF:{trend} | Δ:{currentDelta:F0} | ΔChg:{deltaChange:F0} | VWAP:{vwapValue:F2} | OFReady:{orderFlowReady}");
        }
        #endregion
    }
}
