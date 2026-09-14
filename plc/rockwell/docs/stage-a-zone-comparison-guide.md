# Stage A — Rockwell Leg, Part 2: Zone-Comparison Model (FB_IrrigationZone)

This adds the moisture-threshold `FB_IrrigationZone` logic to the same CCW project as
the tank-fill-sequence state machine from Part 1. **Both stay in the project** — the
state machine is what you extend and build hands-on depth with; this new block is
strictly for the apples-to-apples three-vendor comparison, matching the CODESYS leg
exactly.

## 1. Keep both programs — project organization

1. In the Project Organizer, find your existing Program (`Prg`, containing the tank
   sequencer from Part 1). Rename it **`Prg_TankSequencer`** (right-click → Rename) —
   no logic changes needed, this is cosmetic so the two are easy to tell apart.
2. Add a new Program: Project Organizer → right-click **Programs** → **Add → Program** →
   Structured Text. Name it **`Prg_ZoneComparison`**.
3. Both programs need to be in the controller's task list so they both scan — check
   **Project Organizer → Program Tasks**, and add `Prg_ZoneComparison` alongside
   `Prg_TankSequencer` if it isn't there automatically.
4. Confirm no global variable name collisions: the tank sequencer uses `StartCmd`,
   `Valve1_Open`, etc.; the zone-comparison model below uses `Z03_`-prefixed names.
   No overlap, both can coexist safely.

## 2. Create the User-Defined Function Block

Project Organizer → right-click **User-Defined Function Blocks** → **Add → User-Defined
Function Block (Structured Text)**. Name it `FB_IrrigationZone`.

### Variable table (Input / Output / Local, same block as CODESYS — CCW's UDFB editor
uses the same IEC 61131-3 structure)

```iecst
VAR_INPUT
    xCmdStart           : BOOL;
    xCmdStop             : BOOL;
    xCmdFaultReset       : BOOL;
    xAuto                : BOOL;
    xSimEnable           : BOOL;
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
    rValvePosition       : REAL;
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

### Body — identical to the CODESYS version (IEC 61131-3 ST ports directly, this is the point)

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
// 4. PERMISSIVE — requires xSimEnable = TRUE
// =============================================================
xPermissive :=
    NOT xFaultLatch
    AND NOT xSimActuatorFault
    AND NOT xSimSensorFault
    AND xCommOK
    AND xSimEnable;

// =============================================================
// 5. AUTO DEMAND (hysteresis on moisture)
// =============================================================
IF rMoisture <= rMoistureLowSP THEN
    xAutoDemand := TRUE;
ELSIF rMoisture >= rMoistureHighSP THEN
    xAutoDemand := FALSE;
END_IF;

// =============================================================
// 6. RUNNING DECISION — Stop always wins, even in Auto
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

IF rtStop.Q THEN
    xRunning    := FALSE;
    xAutoDemand := FALSE;
END_IF;

// =============================================================
// 7. FAULT / COMMUNICATION / ENABLE SAFETY
// =============================================================
IF xFaultLatch THEN xRunning := FALSE; END_IF;
IF NOT xCommOK THEN xRunning := FALSE; END_IF;
IF NOT xSimEnable THEN xRunning := FALSE; END_IF;

// =============================================================
// 8. PROCESS COMMAND
// =============================================================
xCommanded := xRunning AND xPermissive;

// =============================================================
// 9. 100 ms SIMULATION TICK
// =============================================================
fbTick(IN := TRUE, PT := T#100MS);

IF fbTick.Q THEN
    fbTick(IN := FALSE);

    IF xCommanded THEN
        rValvePosition := 100.0;
    ELSE
        rValvePosition := 0.0;
    END_IF;

    IF xCommanded THEN
        rFlowTarget := rSimFlowTarget;
        IF xSimRestriction THEN
            rFlowTarget := rFlowTarget * 0.25;
        END_IF;
    ELSE
        rFlowTarget := 0.0;
    END_IF;

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

    IF rFlow < 0.0 THEN rFlow := 0.0; END_IF;
    IF xSimSensorFault THEN rFlow := 0.0; END_IF;

    IF xCommanded AND (rFlow > 0.0) THEN
        rMoistureStep := ABS(rSimMoistureRate) * 0.1;
        rMoisture     := rMoisture + rMoistureStep;
    ELSE
        rMoistureStep := ABS(rSimDryRate) * 0.1;
        rMoisture     := rMoisture - rMoistureStep;
    END_IF;

    IF rMoisture > 100.0 THEN rMoisture := 100.0; END_IF;
    IF rMoisture < 0.0   THEN rMoisture := 0.0;   END_IF;

    xFlowOK := TRUE;
    IF xCommanded AND (rFlow < rSimLowFlowLimit) THEN
        xFlowOK := FALSE;
    END_IF;

    xLowFlow   := xCommanded AND NOT xFlowOK;
    xProcessOK := xPermissive AND xFlowOK;

    IF xRunning THEN
        uiRuntimeTicks := uiRuntimeTicks + 1;
        IF uiRuntimeTicks >= 10 THEN
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
```

## 3. Global variables (flat — CCW has no GVL-style namespacing like CODESYS)

Add these in the Global Variables table, matching the CODESYS `GVL_SCADA.Z03_*` names
minus the prefix, so the Ignition tag suffixes line up across both vendor connections:

```iecst
Z03_START              : BOOL;
Z03_STOP               : BOOL;
Z03_FAULT_RESET        : BOOL;
Z03_AUTO               : BOOL;
Z03_SIM_ENABLE         : BOOL;
Z03_SIM_COMM_FAULT     : BOOL;
Z03_SIM_ACTUATOR_FAULT : BOOL;
Z03_SIM_SENSOR_FAULT   : BOOL;
Z03_SIM_RESTRICTION    : BOOL;
Z03_MOISTURE_LOW_SP    : REAL;
Z03_MOISTURE_HIGH_SP   : REAL;
Z03_SIM_FLOW_TARGET    : REAL;
Z03_SIM_FLOW_RATE      : REAL;
Z03_SIM_MOISTURE_RATE  : REAL;
Z03_SIM_DRY_RATE       : REAL;
Z03_SIM_LOW_FLOW_LIMIT : REAL;

Z03_RUNNING            : BOOL;
Z03_RUN_FEEDBACK       : BOOL;
Z03_FAULT              : BOOL;
Z03_PERMISSIVE         : BOOL;
Z03_FLOW_OK            : BOOL;
Z03_PROCESS_OK         : BOOL;
Z03_COMMANDED          : BOOL;
Z03_AUTO_ACTIVE        : BOOL;
Z03_MANUAL             : BOOL;
Z03_LOW_FLOW           : BOOL;
Z03_COMM_OK            : BOOL;
Z03_SIM_ACTIVE         : BOOL;
Z03_FLOW               : REAL;
Z03_MOISTURE           : REAL;
Z03_VALVE              : REAL;
Z03_TOTAL_RUNTIME_SEC  : UDINT;
```

## 4. `Prg_ZoneComparison` body — instantiate and wire

```iecst
VAR
    fbZone3 : FB_IrrigationZone;
END_VAR

fbZone3(
    xCmdStart           := Z03_START,
    xCmdStop             := Z03_STOP,
    xCmdFaultReset       := Z03_FAULT_RESET,
    xAuto                := Z03_AUTO,
    xSimEnable           := Z03_SIM_ENABLE,
    xSimCommFault        := Z03_SIM_COMM_FAULT,
    xSimActuatorFault    := Z03_SIM_ACTUATOR_FAULT,
    xSimSensorFault      := Z03_SIM_SENSOR_FAULT,
    xSimRestriction      := Z03_SIM_RESTRICTION,
    rMoistureLowSP       := Z03_MOISTURE_LOW_SP,
    rMoistureHighSP      := Z03_MOISTURE_HIGH_SP,
    rSimFlowTarget       := Z03_SIM_FLOW_TARGET,
    rSimFlowRate         := Z03_SIM_FLOW_RATE,
    rSimMoistureRate     := Z03_SIM_MOISTURE_RATE,
    rSimDryRate          := Z03_SIM_DRY_RATE,
    rSimLowFlowLimit     := Z03_SIM_LOW_FLOW_LIMIT,

    xRunning             => Z03_RUNNING,
    xRunFeedback         => Z03_RUN_FEEDBACK,
    xFault                => Z03_FAULT,
    xPermissive          => Z03_PERMISSIVE,
    xFlowOK              => Z03_FLOW_OK,
    xProcessOK           => Z03_PROCESS_OK,
    xCommanded           => Z03_COMMANDED,
    xAutoActive          => Z03_AUTO_ACTIVE,
    xManual              => Z03_MANUAL,
    xLowFlow             => Z03_LOW_FLOW,
    xCommOK              => Z03_COMM_OK,
    xSimActive           => Z03_SIM_ACTIVE,
    rFlow                => Z03_FLOW,
    rMoisture            => Z03_MOISTURE,
    rValvePosition       => Z03_VALVE,
    udiTotalRuntimeSec   => Z03_TOTAL_RUNTIME_SEC
);

// Command consumption
IF Z03_START THEN Z03_START := FALSE; END_IF;
IF Z03_STOP THEN Z03_STOP := FALSE; END_IF;
IF Z03_FAULT_RESET THEN Z03_FAULT_RESET := FALSE; END_IF;
```

## 5. Testing

1. Before anything else, force `Z03_SIM_ENABLE := TRUE` in the Watch window — same
   gotcha as CODESYS, and the most common reason nothing appears to happen.
2. Set `Z03_MOISTURE_LOW_SP := 30.0`, `Z03_MOISTURE_HIGH_SP := 70.0`,
   `Z03_SIM_FLOW_TARGET := 10.0`, `Z03_SIM_FLOW_RATE := 2.0`,
   `Z03_SIM_MOISTURE_RATE := 1.0`, `Z03_SIM_DRY_RATE := 0.5`,
   `Z03_SIM_LOW_FLOW_LIMIT := 2.0` as a reasonable starting point.
3. Set `Z03_AUTO := TRUE`, watch `Z03_MOISTURE` fall below 30 and `Z03_RUNNING` /
   `Z03_VALVE` respond automatically.
4. Force `Z03_STOP := TRUE` for one scan while `Z03_MOISTURE` is still below the low
   setpoint — confirm `Z03_RUNNING` drops immediately (this is the bug that was fixed).

## 6. Ignition — add as a second tag folder

Same OPC UA device connection as Part 1 (no new device needed, same Micro850 SIM).
In the Tag Browser, pull the `Z03_*` tags into a new folder, e.g.
`Rockwell_ZoneComparison`, kept separate from the `Rockwell_IrrigationCell` folder
from Part 1 — you now have both models live in Ignition side by side, which is itself
a good screenshot for the comparison write-up.
