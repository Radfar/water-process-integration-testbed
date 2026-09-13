# Changelog

All notable changes to this project are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions tag with each stage milestone.

## [Unreleased] - Stage A: multi-vendor simulation
### Added
- Repository scaffold: folder structure, README, license, contributing guide, decision log.
- CODESYS OPC UA server connected to Ignition Perspective (initial tag exposure).

### Planned next
- Mature CODESYS control logic (VFD pump control, valve sequencing, interlocks).
- Siemens S7-1500 / PLCSIM implementation of the same control problem.
- Rockwell Micro800 (CCW) implementation; resolve OPC UA path (native vs. gateway).
