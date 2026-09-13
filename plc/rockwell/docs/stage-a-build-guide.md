# Stage A — Rockwell Leg: Micro850 Simulator, Irrigation Control, Ignition OPC UA

## 1. Controller decision

**Micro850**, catalog family 2080-LC50 (48-point combo, e.g. `2080-LC50-48QWB`, matches the free CCW Simulator's I/O count). Not Micro870.

| | Micro850 | Micro870 |
|---|---|---|
| Free CCW Simulator support | Native — the simulator emulates a Micro850 | Not the simulator's target profile; higher mismatch risk |
| PowerFlex 520-series Class 1 EtherNet/IP | Yes (fw 21.011+) | Yes (fw 21.011+) |
| I/O / memory | Enough for this project | More than needed |

Decision logged in `docs/decisions-log.md`.

## 2. Project setup in CCW 23.0.0

1. **File → New Project.** Controller: Micro850, catalog `2080-LC50-48QWB` (or the 24-point variant if that's what the simulator wizard offers — match whatever the SIM target proposes).
2. Confirm the project shows the controller name ending in `SIM` — that's your simulator target, no hardware needed.
3. **Global Variables** window → paste the declarations from §3 below. Use Global (not Program) scope so they're visible for OPC UA/Ignition exposure later.
4. **Programs → Prg → Structured Text** (or add a new ST POU if `Prg` defaults to Ladder) → paste the code from §4.
5. Build (F7) and fix any syntax flags — CCW's ST editor is IEC 61131-3, close to what you've already written in CODESYS.

## 3. Global variable declarations

```iecst
(* ==== Operator / Mode — written from Ignition Perspective ==== *)
StartCmd        : BOOL;
StopCmd         : BOOL;
ResetCmd        : BOOL;
AutoMode        : BOOL := TRUE;

(* ==== State ==== *)
SystemState     : DINT := 0;   (* 0=IDLE 10=FILL 20=IRR_Z1 21=IRR_Z2 22=IRR_Z3 30=DRAIN 90=FAULT *)

(* ==== Field outputs ==== *)
PumpRun             : BOOL;
PumpSpeedRefPct     : REAL;     (* 0.0-100.0 *)
Valve1_Open         : BOOL;
Valve2_Open         : BOOL;
Valve3_Open         : BOOL;
DrainValve_Open     : BOOL;

(* ==== Field inputs — simulator: driven by the virtual process model in §4;
        real hardware (Stage C): replace with wired analog inputs ==== *)
LevelSensorPct      : REAL;
FlowSensor_Lpm      : REAL;
PressureSensor_Bar  : REAL;

(* ==== Setpoints ==== *)
LevelSetpoint_FillPct : REAL := 90.0;
LevelSetpoint_MinPct  : REAL := 10.0;
IrrigatePumpSpeedPct  : REAL := 60.0;
FillPumpSpeedPct      : REAL := 80.0;

(* ==== Timers ==== *)
FillTimer    : TON;
Zone1Timer   : TON;
Zone2Timer   : TON;
Zone3Timer   : TON;
DrainTimer   : TON;

(* ==== Faults ==== *)
Fault_LowLevelToRun : BOOL;
Fault_FillTimeout    : BOOL;
Fault_Active         : BOOL;
FaultCode            : DINT;

(* ==== PowerFlex interface — mirrors the CCAT "PowerFlex Component-class
        AC Drive" sample project's tag pattern. Swap this block for the
        real drive's tags once hardware exists; nothing else changes. ==== *)
PowerFlex_Start      : BOOL;
PowerFlex_Stop       : BOOL;
PowerFlex_SpeedRef   : REAL;
PowerFlex_Running    : BOOL;   (* feedback — stays FALSE until real drive connected *)
PowerFlex_Faulted    : BOOL;   (* feedback — stays FALSE until real drive connected *)
```

## 4. Structured Text program

```iecst
(* ---- Fault latch / reset ---- *)
IF (Fault_LowLevelToRun OR Fault_FillTimeout OR PowerFlex_Faulted) THEN
    Fault_Active := TRUE;
END_IF;

IF ResetCmd AND NOT (Fault_LowLevelToRun OR Fault_FillTimeout OR PowerFlex_Faulted) THEN
    Fault_Active        := FALSE;
    Fault_LowLevelToRun := FALSE;
    Fault_FillTimeout   := FALSE;
    FaultCode            := 0;
END_IF;

IF StopCmd THEN
    SystemState := 0;
END_IF;

IF Fault_Active THEN
    SystemState := 90;
END_IF;

(* ---- State machine ---- *)
CASE SystemState OF

0: (* IDLE *)
    PumpRun         := FALSE;
    PumpSpeedRefPct := 0.0;
    Valve1_Open     := FALSE;
    Valve2_Open     := FALSE;
    Valve3_Open     := FALSE;
    DrainValve_Open := FALSE;
    FillTimer.IN    := FALSE;

    IF StartCmd AND AutoMode AND NOT Fault_Active THEN
        SystemState := 10;
    END_IF;

10: (* FILL *)
    DrainValve_Open := FALSE;
    Valve1_Open := FALSE; Valve2_Open := FALSE; Valve3_Open := FALSE;

    IF LevelSensorPct < LevelSetpoint_FillPct THEN
        PumpRun         := TRUE;
        PumpSpeedRefPct := FillPumpSpeedPct;
    ELSE
        PumpRun         := FALSE;
        PumpSpeedRefPct := 0.0;
    END_IF;

    FillTimer(IN := (LevelSensorPct < LevelSetpoint_FillPct), PT := T#5m);
    IF FillTimer.Q THEN
        Fault_FillTimeout := TRUE;
    END_IF;

    IF LevelSensorPct >= LevelSetpoint_FillPct THEN
        FillTimer.IN := FALSE;
        SystemState  := 20;
    END_IF;

20: (* IRRIGATE ZONE 1 *)
    IF LevelSensorPct < LevelSetpoint_MinPct THEN
        Fault_LowLevelToRun := TRUE;
    ELSE
        Valve1_Open     := TRUE;
        Valve2_Open     := FALSE;
        Valve3_Open     := FALSE;
        PumpRun         := TRUE;
        PumpSpeedRefPct := IrrigatePumpSpeedPct;
    END_IF;

    Zone1Timer(IN := TRUE, PT := T#30s);
    IF Zone1Timer.Q THEN
        Zone1Timer.IN := FALSE;
        Valve1_Open   := FALSE;
        SystemState   := 21;
    END_IF;

21: (* IRRIGATE ZONE 2 *)
    IF LevelSensorPct < LevelSetpoint_MinPct THEN
        Fault_LowLevelToRun := TRUE;
    ELSE
        Valve1_Open     := FALSE;
        Valve2_Open     := TRUE;
        Valve3_Open     := FALSE;
        PumpRun         := TRUE;
        PumpSpeedRefPct := IrrigatePumpSpeedPct;
    END_IF;

    Zone2Timer(IN := TRUE, PT := T#30s);
    IF Zone2Timer.Q THEN
        Zone2Timer.IN := FALSE;
        Valve2_Open   := FALSE;
        SystemState   := 22;
    END_IF;

22: (* IRRIGATE ZONE 3 *)
    IF LevelSensorPct < LevelSetpoint_MinPct THEN
        Fault_LowLevelToRun := TRUE;
    ELSE
        Valve1_Open     := FALSE;
        Valve2_Open     := FALSE;
        Valve3_Open     := TRUE;
        PumpRun         := TRUE;
        PumpSpeedRefPct := IrrigatePumpSpeedPct;
    END_IF;

    Zone3Timer(IN := TRUE, PT := T#30s);
    IF Zone3Timer.Q THEN
        Zone3Timer.IN := FALSE;
        Valve3_Open   := FALSE;
        SystemState   := 30;
    END_IF;

30: (* DRAIN *)
    PumpRun         := FALSE;
    PumpSpeedRefPct := 0.0;
    Valve1_Open := FALSE; Valve2_Open := FALSE; Valve3_Open := FALSE;
    DrainValve_Open := TRUE;

    DrainTimer(IN := TRUE, PT := T#20s);
    IF DrainTimer.Q THEN
        DrainTimer.IN   := FALSE;
        DrainValve_Open := FALSE;
        SystemState     := 0;
    END_IF;

90: (* FAULT *)
    PumpRun         := FALSE;
    PumpSpeedRefPct := 0.0;
    Valve1_Open := FALSE; Valve2_Open := FALSE; Valve3_Open := FALSE;
    DrainValve_Open := FALSE;

    IF NOT Fault_Active THEN
        SystemState := 0;
    END_IF;

ELSE
    SystemState := 90;   (* unknown state -> fail safe *)

END_CASE;

(* ---- PowerFlex interface mapping ----
   Replace with the CCAT sample's actual predefined tags once you paste
   that project in; the control logic above never needs to change. *)
PowerFlex_Start    := PumpRun;
PowerFlex_Stop     := NOT PumpRun;
PowerFlex_SpeedRef := PumpSpeedRefPct;


(* =========================================================================
   VIRTUAL PROCESS MODEL — simulator only.
   The free Micro800 Simulator has no analog I/O, so this block stands in
   for real tank/flow physics: it drives LevelSensorPct up while the pump
   runs and down while a valve is open, so the state machine above can be
   exercised end-to-end without forcing values by hand every scan.
   Delete or bypass this block once real sensors are wired in Stage C.
   ========================================================================= *)
IF PumpRun THEN
    LevelSensorPct := LevelSensorPct + (PumpSpeedRefPct * 0.01);
ELSIF Valve1_Open OR Valve2_Open OR Valve3_Open OR DrainValve_Open THEN
    LevelSensorPct := LevelSensorPct - 0.15;
END_IF;

IF LevelSensorPct > 100.0 THEN LevelSensorPct := 100.0; END_IF;
IF LevelSensorPct < 0.0   THEN LevelSensorPct := 0.0;   END_IF;

FlowSensor_Lpm := PumpSpeedRefPct * 0.5;
```

## 5. Test it in the simulator

1. **Connect → Download** to the SIM target, then **Run mode**.
2. Open the **Watch window**, add `SystemState`, `LevelSensorPct`, `PumpRun`, `PumpSpeedRefPct`, `Valve1_Open/2/3`.
3. Force `StartCmd := TRUE` for one scan. Watch `SystemState` walk 0 → 10 → 20 → 21 → 22 → 30 → 0, with `LevelSensorPct` rising during FILL and falling during each irrigate/drain phase.
4. Remember the free simulator's 10-minute run-mode limit — if it stops, restart Run mode; variables reset to initialized values (expected, not a bug).

## 6. Connect Ignition via the native Micro800 driver

1. Gateway webpage → **Config → OPC UA → Device Connections → Create new Device**.
2. Choose the **Allen-Bradley Micro800 Driver**.
3. Hostname: `127.0.0.1` if Ignition and the simulator run on the same machine, otherwise that machine's LAN IP.
4. Leave port/slot at their Micro800 defaults — no Kepware/gateway step needed.
5. Save, confirm the device connection status goes green/"Connected." If not, check the simulator is in Run mode and the firewall isn't blocking the EtherNet/IP port.
6. **Designer → Tag Browser → OPC UA quick client** (or drag from the device browse tree) → pull in `StartCmd`, `StopCmd`, `ResetCmd`, `SystemState`, `LevelSensorPct`, `Valve1_Open/2/3`, `PumpRun`, `PumpSpeedRefPct`, fault bits.
7. Organize into a folder/UDT named consistently with your CODESYS tag structure (e.g. `Rockwell_IrrigationCell`) — keeps the eventual vendor comparison reading as one coherent system.
8. Bind a Perspective view: buttons for `StartCmd`/`StopCmd`/`ResetCmd`, a gauge for `LevelSensorPct`, indicators for each valve and `PumpRun`, a multi-state indicator for `SystemState`.

## 7. Repo note

This file is the text source-of-truth for the Rockwell leg, since CCW has no clean text-export path. Hand-update it after any logic changes made in the CCW GUI, per the version-control approach in `CONTRIBUTING.md`.
