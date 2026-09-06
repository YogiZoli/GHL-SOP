# GHL Access SOP

Single source of truth for how Zoltan (VoxFlow) accesses GoHighLevel from Claude.
This repo replaces the older scattered Google Docs and the private-key spreadsheet.
Whenever the setup changes, edit this repo so the change history is tracked in git.

## TL;DR

- Primary channel: the local **GoHighLevel-MCP** server (API v2), called by Claude. Multi-account, ~250 tools.
- Backup channel: the **LeadGenJey CLI** (`cli-anything-gohighlevel`, API v2) on the machine, for terminal and scripts.
- We do **not** use the official hosted HighLevel MCP (OAuth). It is redundant and has fewer tools.
- All secrets (API tokens, cookies) stay on the local machine only. Never commit them to this repo.

## Which channel do I use, and when?

| Situation | Use | Why |
|---|---|---|
| Talking to Claude (Cowork or Claude Code) and want CRM actions | GoHighLevel-MCP | Native tool calls, multi-account, widest surface |
| Switching between subaccounts mid-conversation | GoHighLevel-MCP (`switch_location`) | One command, no restart |
| A repeatable script, cron job, or bulk terminal task | LeadGenJey CLI | Scriptable, JSON output, no model needed |
| MCP is down or a token expired | LeadGenJey CLI | Independent auth path (Firebase + cookie) |
| Anything needing an endpoint the MCP lacks | LeadGenJey CLI | Different command coverage |

Rule of thumb: inside a Claude conversation, prefer the MCP. For headless automation or as a fallback, use the CLI. The two are fully independent, so one breaking never breaks the other.

## Architecture: three channels

### 1. GoHighLevel-MCP (PRIMARY)

- Source: `mastanley13/GoHighLevel-MCP` (community project, not official HighLevel).
- Installed at: `C:\Users\gdp\AppData\Roaming\Claude\mcp-servers\GoHighLevel-MCP\dist\server.js`
- Runs as a local Node MCP server, registered in `claude_desktop_config.json` under `mcpServers.gohighlevel`.
- API: HighLevel API **v2** (`https://services.leadconnectorhq.com`).
- Auth: **Private Integration Tokens** (`pit-...`), one per subaccount plus an agency-level token. Tokens live in the config `env` block on the machine, never here.
- Multi-account: yes. Claude calls `switch_location` with a locationId to change the active subaccount, then all following calls target it.
- This is the same repo Zoltan installed earlier and could not get working standalone. It now works because Claude drives it over the Cowork device bridge.

### 2. LeadGenJey CLI (BACKUP)

- Package: `cli-anything-gohighlevel`, version 2.0.0, command `ghl`.
- Installed at: `C:\Users\gdp\Downloads\gohighlevel-cli`
- API: HighLevel API **v2**.
- Auth: Windows user env vars `GHL_API_KEY`, `GHL_LOCATION_ID`, `GHL_FIREBASE_REFRESH_TOKEN`, plus a Chrome cookie captured during setup.
- Multi-account: yes, via the global `--location-id <ID>` override, or by changing `GHL_LOCATION_ID`.

### 3. Official hosted HighLevel MCP (NOT USED)

- Endpoint: `https://services.leadconnectorhq.com/mcp/anthropic/v2` (OAuth, agency multi-account).
- Decision: not adopted. It has a narrower toolset (~36) than the local MCP we already run, and adds OAuth token management for no gain. Left here only so the decision is documented.

## Subaccounts (locationIds)

LocationIds are identifiers, not secrets, so they are safe to keep here. The matching `pit-` tokens are secret and stay in the local config only.

| Subaccount | locationId | Role |
|---|---|---|
| VoxFlow | `8A72BbTkZMJQ8W8eeB9t` | Default / active |
| VoxFlow Template | `LrxjlXlvBfLEHegATqgF` | Template source for new locations |
| VJHOF | `VjHoFGh7RBXtNeb276c7` | Client subaccount |
| Agency / Company | `Pc233IVytzQRoBSZ0JgX` | Agency-level (company id) |
| YT2C2C | `em19nP6nAqQD6gDdDDJ9` | PDT lead destination. NO TOKEN YET, see below |
| ClickClient GHL Test | `pPfkTp3chEgkdiUYO911` | Test subaccount. No token, not needed |

To add a subaccount to the MCP: create a Private Integration Token in that subaccount, add a `GHL_<NAME>_API_KEY` and `GHL_<NAME>_LOCATION_ID` pair to the `gohighlevel` env block in `claude_desktop_config.json`, and restart the Claude desktop app.

## Multi-account: how it really works, verified 2026-09-06

A Private Integration Token is created inside one subaccount and can only ever
read that subaccount. There is no token that opens all of them, and the
agency-level token does not open any of them: it covers company endpoints only.
So multi-account is a token problem, not a server problem, and no change of MCP
server or CLI can fix a missing token.

Three things follow, and all three cost a session when they are forgotten.

1. **`switch_location` alone does not switch the credential.** It sets the
   active locationId. The token comes from the map built at startup. Calling it
   for a subaccount with no token pair succeeds, and every call after it returns
   403 with "The token does not have access to this location", which reads like
   a permission problem and is actually a missing line in the config.
2. **The server reads its environment only at startup.** A pair added to
   `claude_desktop_config.json` does nothing until Claude Desktop is fully
   quit and started again. Not a new conversation, not a reconnect: a restart.
3. **The CLI has the same shape.** `--location-id` selects the target, the
   Windows env var `GHL_API_KEY` selects the credential, and they are
   independent. Pointing the CLI at a subaccount whose token is not loaded
   gives the identical 403. `--experimental` does not change this.

Measured on 2026-09-06, same call against three subaccounts: VoxFlow 200,
VJHOF 200 (it had been 403 before the token map landed and the app was
restarted), YT2C2C 403 because no `GHL_YT2C2C_API_KEY` exists anywhere on the
machine. That last one is not a fault to debug. It is a token that has never
been created.

### Adding a subaccount, the whole procedure

1. In GoHighLevel, switch to the subaccount. Settings, Private Integrations,
   Create new integration. Give it the read scopes the work needs, at minimum
   workflows and contacts readonly. The `pit-` value is shown exactly once.
2. Put it in the `gohighlevel` env block of `claude_desktop_config.json` as the
   pair `GHL_<NAME>_API_KEY` and `GHL_<NAME>_LOCATION_ID`.
3. Quit Claude Desktop completely and start it again.
4. Verify: ask Claude for the workflows of that locationId. A 200 with an empty
   list is a pass. A 403 means step 3 was skipped or the token was made in the
   wrong subaccount.

Only step 1 needs a human. Nothing in the HighLevel API can mint a Private
Integration Token, so no Claude session can do it, and any session that says it
will is wrong.

## Secrets policy

- Never commit `pit-` tokens, the Firebase refresh token, Chrome cookies, or Google client secrets to this repo.
- The old private-key spreadsheet is no longer the way to manage access. Auth now lives in two local places only: the `gohighlevel` env block in `claude_desktop_config.json` (MCP) and the Windows user env vars (CLI).
- If a token leaks, rotate it in the HighLevel subaccount settings and update the local config.

## Verify it works

MCP (ask Claude): "Get the active GHL location." Expect `8A72BbTkZMJQ8W8eeB9t` (VoxFlow).

CLI (PowerShell, with env vars loaded):

```powershell
ghl --json locations get
```

Expect a JSON object with `"name": "VoxFlow"`.

## Change log

- 2026-09-06: Corrected the multi-account section. The earlier text said the
  MCP is multi-account without saying that each subaccount needs its own token
  and that the app must be restarted, which sent three PDT sessions in a row
  chasing a 403 that was a missing config line. Added the YT2C2C and
  ClickClient rows and the full add-a-subaccount procedure. Confirmed the token
  map in `dist/clients/ghl-api-client.js` reads every
  `GHL_<NAME>_API_KEY` / `GHL_<NAME>_LOCATION_ID` pair, so nothing needs
  patching; VJHOF now answers 200.
- 2026-08-27: Repo created. Documented three channels, confirmed the local GoHighLevel-MCP works and switches subaccounts live (VoxFlow to VJHOF and back). Decided against the hosted OAuth MCP.
