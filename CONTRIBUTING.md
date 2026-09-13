# Contributing / Working Practices

This is currently a solo project, but it follows the same discipline as a team repo.

## Commit conventions
Conventional Commits: `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`. Short-lived feature branches merged via PR (self-reviewed) rather than direct commits to `main` once initial scaffolding is done.

## Version control by subsystem

### Web / JS / SCADA scripting
Standard Git. Ignition project resources are file-system based as of v8.3, so they are committed like any other text-based project under `scada/`.

### PLC projects
Most PLC project files are binary and don't diff meaningfully in Git. The approach used here:

- **Binary project archive** (the file actually opened in the vendor tool) is tracked for completeness.
- **Text export** is committed alongside it as the reviewable/diffable layer, wherever the tool supports one.

| Vendor | Binary artifact | Text export used for diffing |
|---|---|---|
| Siemens (TIA Portal) | `.zap18` / `.ap18` | Program blocks exported as external SCL source |
| CODESYS | `.project` archive | XML project export and/or per-POU structured text export |
| Rockwell (Micro800 / CCW) | CCW project file (closed binary, no clean text-export path) | No native diff path. Substitute: manual changelog entry per commit + exported program listing (PDF/screenshot) |

Each `plc/<vendor>/` folder maintains its own `CHANGELOG.md`, updated by hand at each milestone since the tools can't auto-generate one.
