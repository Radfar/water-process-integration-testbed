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
