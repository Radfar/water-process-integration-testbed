# Water Process Integration Testbed

**Status:** Stage A — multi-vendor simulation, in progress

A benchtop water-management system used to validate real OT/IT integration architecture: the same irrigation-style control problem (VFD-driven pump, solenoid valve zoning, level/flow/pressure feedback) implemented independently across three PLC platforms, unified under one SCADA layer, and bridged into a modern IT/data stack.

## Why this exists

Most automation portfolios show one vendor. This project deliberately implements the same control problem on three different platforms — Siemens, Rockwell, and CODESYS — communicating through OPC UA into a single Ignition SCADA client, then bridges that into a Node.js/PostgreSQL data and analytics layer. The goal is to demonstrate genuine OT/IT integration judgment, not just PLC programming.

## Architecture

Five layers, bottom to top:

| Layer | Technology |
|---|---|
| Field / OT | Siemens S7-1500, Rockwell Micro800, SINAMICS G120X / PowerFlex VFDs, sensors, valves, pumps |
| Communication & SoftPLC | OPC UA, PROFINET, EtherNet/IP, CODESYS SoftPLC |
| SCADA / HMI | Ignition Perspective |
| Edge / IIoT | Node-RED, MQTT, legacy wireless field node |
| IT / Data / Analytics | PostgreSQL historian, REST API (Node.js/Express), agentic AI (Claude/MCP) |

See `docs/architecture.md` for the full breakdown and `docs/decisions-log.md` for why specific choices were made.

## Repository structure

```
plc/          siemens/, rockwell/, codesys/ — PLC programs, exported source, and binary project archives (see CONTRIBUTING.md for the version control approach)
scada/        Ignition Perspective project files
edge/         Node-RED flows, MQTT configuration
web/          Node.js/Express REST API
hardware/     wiring diagrams, bill of materials, panel photos
docs/         architecture notes, decision log, I/O lists
media/        demo videos, screenshots
```

## Tech stack

Siemens TIA Portal / PLCSIM · Rockwell Connected Components Workbench (Micro800) · CODESYS SoftPLC · SINAMICS G120X / PowerFlex VFDs · OPC UA · PROFINET · EtherNet/IP · Ignition Perspective · Node-RED · MQTT · Node.js · Express · PostgreSQL

## License

MIT — see `LICENSE`.
