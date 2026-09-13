# Architecture

Five layers, bottom to top. The same control problem (VFD-driven pump, solenoid valve zoning, level/flow/pressure feedback) is implemented independently on three PLC platforms in Stage A, unified through OPC UA into one Ignition SCADA client, then bridged into a Node.js/PostgreSQL data and analytics layer.

## Field / OT layer
Siemens S7-1500, Rockwell Micro800, SINAMICS G120X / PowerFlex VFDs, flow/level/pressure sensors, solenoid valves, pumps.

## Communication & SoftPLC layer
OPC UA, PROFINET, EtherNet/IP, CODESYS SoftPLC.

## SCADA / HMI layer
Ignition Perspective as the unified OPC UA client across all three PLC platforms.

## Edge / IIoT layer
Node-RED, MQTT, and a legacy wireless field device (Arduino-based) bridged into the system as a real IIoT edge node.

## IT / Data / Analytics layer
PostgreSQL historian, REST API (Node.js/Express), agentic AI (Claude/MCP) for natural-language operational queries against live data.

## Roadmap

- **Stage A** — simulated multi-vendor irrigation control cell (Siemens, Rockwell, CODESYS in parallel, unified under Ignition).
- **Stage B** — Edge/IIoT and web/data layer (Node-RED, MQTT, Node.js REST API, PostgreSQL).
- **Stage C** — physical benchtop build, legacy Arduino field device integration.
- **Stage D** — historian analytics and agentic AI.

See `decisions-log.md` for the reasoning behind specific choices as they're made.
