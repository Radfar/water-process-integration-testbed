# Connecting Claude Desktop to Ignition via MCP — Full Reference Guide

Companion reference for the YouTube walkthrough. Everything here — exact file paths, JSON contents, and every error hit along the way — is what's referenced on-screen in the video. Copy-paste what you need.

**Stack:** Ignition 8.3.5+ · MCP Module (Early Access) · Claude Desktop · Node.js (`mcp-remote`) · Windows 10/11

**Architecture:**
```
Claude Desktop  →  mcp-remote (Node bridge, stdio ↔ HTTP)  →  Ignition Gateway (localhost:8088)
                                                                  └─ MCP Module
                                                                       └─ server-config: <your-project-name>
                                                                            └─ Tool: your custom script
```

Because the Gateway runs locally, this whole setup stays on `localhost` — no tunnel, no exposed ports, no cloud proxy.

---

## Part 1 — Install the MCP Module

1. **Request access.** IA is gating the EA build — fill out the [request form](https://docs.google.com/forms/d/e/1FAIpQLSf6Hu7jAN0KsACI32iiWqe_LcpcxclSDelqqkaJ6bA4yrl5rA/viewform?usp=header) linked from the [official forum thread](https://forum.inductiveautomation.com/t/mcp-module-early-access/113966). They'll email you the `.modl` file.
2. **Install it** like any other module: Gateway web page → **Platform → Modules → Install or Upgrade a Module** → upload the `.modl` file.
   > ⚠️ If the module isn't signed for release (common for EA builds), the Gateway will tell you to enable unsigned module support in `ignition.conf` first.
3. **Restart the Gateway** if prompted, then confirm it shows as running under **Config → Modules**.

---

## Part 2 — Designer-side setup

1. **Create a standalone, non-inheritable project.** Name it something identifiable — this guide uses `mcp_example` throughout; swap in your own name everywhere it appears below.
   > ⚠️ **This is the single most common failure point.** If the project is inheritable, the MCP server connects fine but returns empty capabilities, and `tools/list` silently fails. It must be a standalone (or child) project, not inheritable.
2. **Open the Model Context Protocol workspace.** Once the module's installed, a new "Model Context Protocol" workspace appears in the Designer's project browser (requires Ignition 8.3.5+).
3. **Create a Tool resource.** Right-click in that workspace → new **Tool** resource. A Tool holds the script that runs when an MCP client (Claude Desktop) invokes it.
4. **Write a specific description.** This is what the connecting LLM reads to decide *when* to call the tool — be precise about what it does and what inputs it expects. Example:

   > *"Reads live tag values for a given irrigation zone folder under the Zones UDT (e.g. Zones/Zone01). Returns current status, flow, fault state, and mode for that zone. Use this when asked about real-time zone status, to verify a zone's current state, or to compare simulated tag data across zones."*

5. **Save the project and note the exact name** — you'll need it character-for-character in the next part.

---

## Part 3 — Gateway server-config (manual JSON)

The EA build has no UI for this yet — it's configured by hand on disk (a cURL/Python call to the Gateway's resources API also works, if you'd rather script it).

**Folder to create:**
```
C:\Program Files\Inductive Automation\Ignition\data\config\resources\core\com.inductiveautomation.mcp\server-config\mcp_example\
```
> ⚠️ This is under `Program Files`, which is admin-protected on Windows — create/edit these files with a text editor run as Administrator, or you'll get silent write failures.
> ⚠️ This folder name becomes part of the URL later. Don't rename it mid-process.

**`resource.json`** (generate your own UUID — don't reuse this one):
```json
{
  "scope": "A",
  "version": 1,
  "restricted": false,
  "overridable": true,
  "files": [
    "config.json"
  ],
  "attributes": {
    "uuid": "3c873398-8438-4fea-859a-110b0e61d530"
  }
}
```

**`config.json`:**
```json
{
  "title": "mcp_example",
  "version": "1.0.0",
  "permissions": {
    "type": "AllOf",
    "securityLevels": [
      {
        "children": [],
        "name": "Authenticated"
      }
    ]
  },
  "tools": {
    "project/mcp_example": "*"
  },
  "resources": {},
  "prompts": {}
}
```
The `"project/mcp_example"` key under `"tools"` must exactly match your Designer project name from Part 2.

---

## Part 4 — Scan, key, and permissions

1. **Scan the config filesystem.** Gateway → **Platform Overview** page → trigger a config filesystem scan so the Gateway picks up the new folder. You should see your server-config entry appear afterward.
2. **Generate an API key.** Gateway → **Platform → API Keys** → create one, save it somewhere safe — you'll paste it into Claude Desktop's config next.
3. **Uncheck "Require secure connections for API Keys"** (Config → Security → General) if you're testing over plain `http://localhost` rather than `https://`. Leaving it checked causes API key requests to fail with a 403 over HTTP.
4. **Grant Gateway read/write permission** to the key's auto-generated security level (`Authenticated/<key-name>`) — also under Config → Security → General. Without this, you'll get a 403 even with the HTTPS setting correctly unchecked.

---

## Part 5 — Connect Claude Desktop

**Find the config file.**
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows (standard install): `%APPDATA%\Claude\claude_desktop_config.json` — press **Win + R**, type `%APPDATA%\Claude`, Enter.
- Windows (**Microsoft Store version**): the sandboxed path is different — `C:\Users\<you>\AppData\Local\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\`. If the standard path doesn't have it, search your C: drive for `claude_desktop_config.json` directly.

> **Don't conflate this with the stdio/HTTP mismatch below — they're separate issues.** The config-format rejection in the next section happens on *any* Claude Desktop install, Store or standard; it's a general limitation of the app's config schema. What's actually Store-specific is narrower: the MSIX packaging virtualizes `%APPDATA%` (hence the different path above), and MSIX apps don't fully inherit the system PATH — visible directly in this setup's logs, where Claude Desktop spawned `cmd.exe` with an explicit ~32-entry PATH list rather than resolving `npx` natively. A [separately reported bug](https://github.com/anthropics/claude-code/issues/25600) also describes local MCP servers being silently ignored entirely on some Store builds — a harsher failure mode than anything documented here, since this setup did eventually connect.

If the file doesn't exist yet (fresh install, no other MCP servers configured), that's normal — create it.

**❌ This will be silently rejected.** It's the format from IA's own example docs, written for a generic MCP client — Claude Desktop only supports local (stdio) servers, and drops `url`/`http`-type entries without any error dialog:
```json
{
  "servers": {
    "ignition": {
      "url": "http://localhost:8088/data/mcp/mcp_example",
      "type": "http",
      "headers": {
        "X-Ignition-API-Token": "your-token-here"
      }
    }
  }
}
```

**✅ Use this instead.** Bridge it with [`mcp-remote`](https://www.npmjs.com/package/mcp-remote), a small Node package that translates the HTTP endpoint into the stdio interface Claude Desktop expects. Requires Node.js installed:
```json
{
  "mcpServers": {
    "ignition": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "http://localhost:8088/data/mcp/mcp_example",
        "--header",
        "X-Ignition-API-Token:${IGNITION_API_TOKEN}"
      ],
      "env": {
        "IGNITION_API_TOKEN": "API_KEY_HERE"
      }
    }
  }
}
```
> If the file already has an `"mcpServers"` block with other servers in it, add `"ignition"` as another entry inside that same block — don't overwrite the whole file.

`npx` downloads `mcp-remote` the first time it runs, so the first launch may take a few extra seconds while it fetches the package — that's normal, not a hang.

**Save, then fully quit Claude Desktop from the system tray (closing the window alone often just minimizes it) and reopen it.**

---

## Part 6 — The debugging chain

Every one of these produced a distinct, decodable error in the log — the fix each time was reading the output carefully, not guessing.

| Symptom | Root cause | Fix |
|---|---|---|
| Config silently ignored, no error | Direct `url`/`http` entry — unsupported format. General limitation, not Store-specific | Switch to the `mcp-remote` bridge (Part 5) |
| `claude_desktop_config.json` not at the documented path | Store-specific: MSIX virtualizes `%APPDATA%` | Use the sandboxed `...Packages\Claude_pzs8sxrjxfjjc\...` path instead |
| `npx`/Node behaves oddly; logs show a `cmd.exe` wrapper with an explicit PATH list | Store-specific: MSIX sandbox doesn't fully inherit system PATH | Usually self-resolves; otherwise use the absolute path to `npx.cmd` |
| `404: MCP server not found` | Gateway hadn't reloaded its internal MCP routing table after the config scan | Full Gateway **service** restart (not just a re-scan) |
| `403: Forbidden` | API key's security level lacked Gateway read/write permission, and/or "Require secure connections" was blocking plain HTTP | Grant permissions (Part 4, step 4) **and** uncheck the HTTPS requirement (Part 4, step 3) |
| Connects, but nothing happens | MCP just makes tools *available* — it doesn't do anything on its own | Ask directly, inside Claude Desktop: *"What MCP tools do you have available right now?"* |

---

## Part 7 — Verify the connection

In **Claude Desktop → Settings → Developer**, your server should show a blue **"Running"** status. Then, in a new chat *inside Claude Desktop* (this step doesn't work from claude.ai's web or mobile apps — those have no route to your local Gateway), ask:

> "What MCP tools do you have available right now?"

You should see your Tool listed (e.g. `mcp__ignition__mcp_example`) with its description. Calling it should return real, live data from your Gateway.

> ⚠️ **Sanity check before you trust the output:** if you ever see a suspiciously generic response like `"Hello, world!"` instead of real tag values, you're not actually hitting your Gateway — you're looking at a stub/placeholder tool from a different environment (e.g. a pre-built example in a hosted Claude.ai project, not your local MCP connection). Cross-check any live-tag claims against the Designer's tag browser directly.

---

## Quick troubleshooting index

- **Blank/rejected config, no dialog at all** → you're using the `url`/`http` format; switch to `mcp-remote` (Part 5)
- **404 MCP server not found** → restart the Gateway *service*, not just a config scan
- **403 Forbidden** → check both the HTTPS toggle *and* the security level's Gateway permissions
- **Connected but tool doesn't show real data** → verify you're actually calling your own Tool, not a stub from an unrelated environment
- **Can't find `claude_desktop_config.json`** → check whether you're on the Microsoft Store build (different sandboxed path, see Part 5)

---

*Full setup video and case study linked in the description. Questions or hit a different error? Drop it in the comments — happy to help troubleshoot.*

