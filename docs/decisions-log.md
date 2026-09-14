# Decisions Log

Running log of choices made and why, written as they happen rather than reconstructed later. Feeds the ADRs, README updates, and case-study posts for each stage.

## Format
```
### YYYY-MM-DD — short title
**Context:** what problem/question this addresses
**Decision:** what was chosen
**Why:** reasoning, alternatives considered
```

---

### 2026-09 — Simulator-first, three vendors in parallel
**Context:** originally planned as sequential (one PLC platform proven, then others added). Project now also needs to produce a real vendor comparison for an employer/customer proposal.
**Decision:** implement the same control problem (VFD pump, valve zoning, level/flow/pressure feedback) independently on Siemens, Rockwell, and CODESYS simulators concurrently in Stage A, unified under one Ignition SCADA client via OPC UA.
**Why:** a side-by-side comparison is more useful to both the proposal and the portfolio than proving one vendor before touching the next, and validates architecture entirely in simulation before any hardware spend.

### 2026-09 — Repository naming
**Context:** considered a branded project name (e.g. "AquaBridge").
**Decision:** plain, descriptive repo name instead: `water-process-integration-testbed`.
**Why:** a branded/product-style name reads as a consumer app or learning project to a technical reviewer; a descriptive name reads as an engineering system.

### 2026-09 — Rockwell leg: Micro800 Simulator in CCW, native Ignition driver, no gateway
**Context:** no physical Micro800 hardware available yet; needed to confirm whether Ignition could reach a Micro800 without a Kepware/KEPServerEX gateway in between, and whether offline simulation was possible at all in CCW.
**Decision:** use CCW Standard's built-in Micro800 Simulator (emulates a Micro850, free, no hardware required) as the Rockwell leg of Stage A. Ignition connects to it directly via its native Allen-Bradley Micro800 driver (EtherNet/IP) — no gateway needed.
**Why:** removes the Rockwell leg's biggest open risk (unclear OPC UA path) at zero cost. Two real constraints to plan around: the free simulator runs in 10-minute bursts (restart to continue — fine for development, needs planning around demo recording), and it has no analog I/O simulation, so the VFD speed reference (PumpSpeedRef) is represented as a plain variable the control logic writes to, not a true analog signal. Documented here rather than hidden, since it's a simulator limitation, not a design gap.

### 2026-09 — CODESYS bug fix + canonical moisture-threshold model; CCW carries two parallel programs
**Context:** the original per-zone CODESYS logic had a real bug — Auto mode's moisture-based control silently overrode manual Stop commands, so Stop appeared not to work while in Auto. Separately, the CCW leg (tank-fill-and-sequence state machine) and the CODESYS leg (per-zone moisture threshold) were solving genuinely different control problems, which would have made the three-vendor comparison table apples-to-oranges.
**Decision:** (1) fixed the Stop-override bug and refactored the CODESYS logic into a reusable `FB_IrrigationZone` function block instead of a flat per-zone program. (2) Ported the same `FB_IrrigationZone` logic into CCW as a new parallel program (`Prg_ZoneComparison`), kept alongside — not replacing — the original tank-sequencer state machine (`Prg_TankSequencer`).
**Why:** the moisture-threshold model is closer to how real irrigation/BAS systems work and is now the canonical problem all three vendors implement identically for the comparison table. Keeping the tank-sequencer state machine alongside it (rather than discarding it) preserves broader hands-on Rockwell experience — sequencing, timers, fill/drain logic — that the comparison model alone wouldn't cover. Siemens (Stage A, still to build) will implement `FB_IrrigationZone` as the third comparison point.
