# Canonical Irrigation Zone Control Logic (CODESYS ST — Function Block)

This is the fixed, vendor-neutral reference implementation for one irrigation zone's
control logic. It supersedes the flat per-zone program used earlier, and is meant
to be the single algorithm later ported 1:1 (same variable names, same structure) into
CCW Structured Text and Siemens SCL, so the three-vendor comparison is a genuine
apples-to-apples test rather than three different control philosophies.

## What changed from the original

1. **Refactored into a Function Block** (`FB_IrrigationZone`) instantiated once per zone,
   instead of a flat program duplicated per zone. One definition, three instances —
   this is what makes "port the same code to CCW and Siemens" actually mean something.
2. **Fixed: Stop no longer gets silently overridden by Auto mode.** Previously, the
   moisture-based Auto logic ran immediately after manual Start/Stop handling and
   unconditionally re-asserted `xRunning` — so pressing Stop while in Auto had no
   visible effect. Stop now always wins for the scan it's pressed, regardless of mode.
3. **Flagged, not silently fixed:** `xPermissive` still requires `xSimEnable = TRUE`. This
   is correct behavior (a genuine safety/enable interlock), but it's the most likely
   reason nothing was happening in testing — check this tag first.
4. Everything else (fault latch/reset, 100 ms simulation tick, flow ramp, moisture model,
   runtime accumulation) is functionally the same as the original, reorganized inside
   the FB with clearer input/output boundaries.

## Function Block declaration

```iecst
FUNCTION_BLOCK FB_IrrigationZone
VAR_INPUT
    xCmdStart           : BOOL;   (* momentary/latched command from SCADA, cleared after use *)
    xCmdStop             : BOOL;
    xCmdFaultReset       : BOOL;
    xAuto                : BOOL;  (* TRUE = automatic moisture-threshold control *)
    xSimEnable           : BOOL;  (* master enable — nothing runs while FALSE *)
    xSimCommFault        : BOOL;
    xSimActuatorFault    : BOOL;
    xSimSensorFault      : BOOL;
    xSimRestriction      : BOOL;
    rMoistureLowSP       : REAL;
    rMoistureHighSP      : REAL;
    rSimFlowTarget       : REAL;
    rSimFlowRate         : REAL;
    rSimMoistureRate     : REAL;
    rSimDryRate          : REAL;
    rSimLowFlowLimit     : REAL;
END_VAR
VAR_OUTPUT
    xRunning             : BOOL;
    xRunFeedback         : BOOL;
    xFault                : BOOL;
    xPermissive          : BOOL;
    xFlowOK              : BOOL;
    xProcessOK           : BOOL;
    xCommanded           : BOOL;
    xAutoActive          : BOOL;
    xManual              : BOOL;
    xLowFlow             : BOOL;
    xCommOK              : BOOL;
    xSimActive           : BOOL;
    rFlow                : REAL;
    rMoisture            : REAL;
    rValvePosition       : REAL;  (* 0.0-100.0 *)
    udiTotalRuntimeSec   : UDINT;
END_VAR
VAR
    xFaultLatch          : BOOL;
    xAutoDemand          : BOOL;
    rFlowTarget          : REAL;
    rFlowError           : REAL;
    rFlowStep            : REAL;
    rMoistureStep        : REAL;
    uiRuntimeTicks       : UINT;
    rtStart              : R_TRIG;
    rtStop               : R_TRIG;
    rtFaultReset         : R_TRIG;
    fbTick               : TON;
END_VAR
```

## Function Block body

```iecst
// =============================================================
// 1. COMMAND EDGE DETECTION
// =============================================================
rtStart(CLK := xCmdStart);
rtStop(CLK := xCmdStop);
rtFaultReset(CLK := xCmdFaultReset);


// =============================================================
// 2. STATUS MIRRORS
// =============================================================
xSimActive := xSimEnable;
xCommOK    := NOT xSimCommFault;


// =============================================================
// 3. FAULT GENERATION / RESET
// =============================================================
IF xSimActuatorFault OR xSimSensorFault THEN
    xFaultLatch := TRUE;
END_IF;

IF rtFaultReset.Q THEN
    IF NOT xSimActuatorFault AND NOT xSimSensorFault THEN
        xFaultLatch := FALSE;
    END_IF;
END_IF;


// =============================================================
// 4. PERMISSIVE
//    NOTE: requires xSimEnable = TRUE. If nothing is running and
//    everything else looks correctly wired, check this input first.
// =============================================================
xPermissive :=
    NOT xFaultLatch
    AND NOT xSimActuatorFault
    AND NOT xSimSensorFault
    AND xCommOK
    AND xSimEnable;


// =============================================================
// 5. AUTO DEMAND (hysteresis on moisture, does not touch xRunning directly)
// =============================================================
IF rMoisture <= rMoistureLowSP THEN
    xAutoDemand := TRUE;
ELSIF rMoisture >= rMoistureHighSP THEN
    xAutoDemand := FALSE;
END_IF;
// else: holds previous state — this is the intended dead-band behavior


// =============================================================
// 6. RUNNING DECISION
//    Fix: Stop always wins, even in Auto mode, for the scan it's pressed.
// =============================================================
xAutoActive := FALSE;

IF xAuto AND xPermissive THEN
    xAutoActive := TRUE;
    xRunning    := xAutoDemand;
ELSIF xPermissive THEN
    IF rtStart.Q THEN
        xRunning := TRUE;
    END_IF;
END_IF;

// Manual/safety Stop — always evaluated last, always wins this scan,
// regardless of Auto/Manual mode.
IF rtStop.Q THEN
    xRunning    := FALSE;
    xAutoDemand := FALSE;
END_IF;


// =============================================================
// 7. FAULT / COMMUNICATION / ENABLE SAFETY
// =============================================================
IF xFaultLatch THEN
    xRunning := FALSE;
END_IF;

IF NOT xCommOK THEN
    xRunning := FALSE;
END_IF;

IF NOT xSimEnable THEN
    xRunning := FALSE;
END_IF;


// =============================================================
// 8. PROCESS COMMAND
// =============================================================
xCommanded := xRunning AND xPermissive;


// =============================================================
// 9. 100 ms SIMULATION TICK
// =============================================================
fbTick(IN := TRUE, PT := T#100MS);

IF fbTick.Q THEN
    fbTick(IN := FALSE);   // retrigger for next 100 ms interval

    // ---- Valve ----
    IF xCommanded THEN
        rValvePosition := 100.0;
    ELSE
        rValvePosition := 0.0;
    END_IF;

    // ---- Flow target ----
    IF xCommanded THEN
        rFlowTarget := rSimFlowTarget;
        IF xSimRestriction THEN
            rFlowTarget := rFlowTarget * 0.25;
        END_IF;
    ELSE
        rFlowTarget := 0.0;
    END_IF;

    // ---- Flow ramp ----
    rFlowError := rFlowTarget - rFlow;
    rFlowStep  := ABS(rSimFlowRate) * 0.1;
    IF rFlowStep < 0.1 THEN
        rFlowStep := 0.1;
    END_IF;

    IF rFlowError > rFlowStep THEN
        rFlow := rFlow + rFlowStep;
    ELSIF rFlowError < -rFlowStep THEN
        rFlow := rFlow - rFlowStep;
    ELSE
        rFlow := rFlowTarget;
    END_IF;

    IF rFlow < 0.0 THEN
        rFlow := 0.0;
    END_IF;

    IF xSimSensorFault THEN
        rFlow := 0.0;   // sensor fault -> invalid/zero feedback
    END_IF;

    // ---- Moisture process ----
    IF xCommanded AND (rFlow > 0.0) THEN
        rMoistureStep := ABS(rSimMoistureRate) * 0.1;
        rMoisture     := rMoisture + rMoistureStep;
    ELSE
        rMoistureStep := ABS(rSimDryRate) * 0.1;
        rMoisture     := rMoisture - rMoistureStep;
    END_IF;

    IF rMoisture > 100.0 THEN rMoisture := 100.0; END_IF;
    IF rMoisture < 0.0   THEN rMoisture := 0.0;   END_IF;

    // ---- Flow quality ----
    xFlowOK := TRUE;
    IF xCommanded AND (rFlow < rSimLowFlowLimit) THEN
        xFlowOK := FALSE;
    END_IF;

    xLowFlow    := xCommanded AND NOT xFlowOK;
    xProcessOK  := xPermissive AND xFlowOK;

    // ---- Runtime accumulation ----
    IF xRunning THEN
        uiRuntimeTicks := uiRuntimeTicks + 1;
        IF uiRuntimeTicks >= 10 THEN   // 10 x 100ms = 1s
            udiTotalRuntimeSec := udiTotalRuntimeSec + 1;
            uiRuntimeTicks := 0;
        END_IF;
    END_IF;
END_IF;


// =============================================================
// 10. OUTPUT MAPPING
// =============================================================
xRunFeedback := xRunning;
xFault       := xFaultLatch;
xManual      := NOT xAuto;


// =============================================================
// 11. COMMAND CONSUMPTION
//     Momentary PLC-side commands are cleared by the CALLING program
//     (the GVL write-back happens where this FB is instantiated —
//     see the wiring example below), not inside the FB itself, so the
//     FB stays reusable and doesn't reach into a specific GVL.
// =============================================================
```

## Wiring three zone instances (example, in the calling program)

```iecst
VAR
    fbZone1 : FB_IrrigationZone;
    fbZone2 : FB_IrrigationZone;
    fbZone3 : FB_IrrigationZone;
END_VAR

fbZone3(
    xCmdStart           := GVL_SCADA.Z03_START,
    xCmdStop             := GVL_SCADA.Z03_STOP,
    xCmdFaultReset       := GVL_SCADA.Z03_FAULT_RESET,
    xAuto                := GVL_SCADA.Z03_AUTO,
    xSimEnable           := GVL_SCADA.Z03_SIM_ENABLE,
    xSimCommFault        := GVL_SCADA.Z03_SIM_COMM_FAULT,
    xSimActuatorFault    := GVL_SCADA.Z03_SIM_ACTUATOR_FAULT,
    xSimSensorFault      := GVL_SCADA.Z03_SIM_SENSOR_FAULT,
    xSimRestriction      := GVL_SCADA.Z03_SIM_RESTRICTION,
    rMoistureLowSP       := GVL_SCADA.Z03_MOISTURE_LOW_SP,
    rMoistureHighSP      := GVL_SCADA.Z03_MOISTURE_HIGH_SP,
    rSimFlowTarget       := GVL_SCADA.Z03_SIM_FLOW_TARGET,
    rSimFlowRate         := GVL_SCADA.Z03_SIM_FLOW_RATE,
    rSimMoistureRate     := GVL_SCADA.Z03_SIM_MOISTURE_RATE,
    rSimDryRate          := GVL_SCADA.Z03_SIM_DRY_RATE,
    rSimLowFlowLimit     := GVL_SCADA.Z03_SIM_LOW_FLOW_LIMIT,

    xRunning             => GVL_SCADA.Z03_RUNNING,
    xRunFeedback         => GVL_SCADA.Z03_RUN_FEEDBACK,
    xFault                => GVL_SCADA.Z03_FAULT,
    xPermissive          => GVL_SCADA.Z03_PERMISSIVE,
    xFlowOK              => GVL_SCADA.Z03_FLOW_OK,
    xProcessOK           => GVL_SCADA.Z03_PROCESS_OK,
    xCommanded           => GVL_SCADA.Z03_COMMANDED,
    xAutoActive          => GVL_SCADA.Z03_AUTO_ACTIVE,
    xManual              => GVL_SCADA.Z03_MANUAL,
    xLowFlow             => GVL_SCADA.Z03_LOW_FLOW,
    xCommOK              => GVL_SCADA.Z03_COMM_OK,
    xSimActive           => GVL_SCADA.Z03_SIM_ACTIVE,
    rFlow                => GVL_SCADA.Z03_FLOW,
    rMoisture            => GVL_SCADA.Z03_MOISTURE,
    rValvePosition       => GVL_SCADA.Z03_VALVE,
    udiTotalRuntimeSec   => GVL_SCADA.Z03_TOTAL_RUNTIME_SEC
);

// Command consumption — clear momentary commands after this scan processed them
IF GVL_SCADA.Z03_START THEN GVL_SCADA.Z03_START := FALSE; END_IF;
IF GVL_SCADA.Z03_STOP THEN GVL_SCADA.Z03_STOP := FALSE; END_IF;
IF GVL_SCADA.Z03_FAULT_RESET THEN GVL_SCADA.Z03_FAULT_RESET := FALSE; END_IF;

// Repeat for fbZone1 (Z01_*) and fbZone2 (Z02_*) with the same pattern.
```

## Next step — reconciling with the CCW leg

The CCW Micro850 code delivered earlier used a tank-fill-and-sequence state machine,
not this moisture-threshold-per-zone model. For a genuine three-way comparison, the
CCW leg should be rewritten to implement this same `FB_IrrigationZone` logic (as a
User-Defined Function Block in CCW, which supports the same IEC structure) rather than
the fill/drain sequencer. Recommended, not yet done — confirm and it's the next piece
of code delivered.
