# GHL Commands from Claude

How to make Claude run GoHighLevel actions. Two ways: natural language (Claude picks the MCP tool) or the CLI (exact terminal command). Both hit API v2.

## How calling GHL inside Claude works

The local GoHighLevel-MCP exposes ~250 tools to Claude. When you describe a CRM action, Claude selects the matching tool and runs it against the active subaccount. You do not type tool names; you state the intent. To save tokens, keep the intent short and specific instead of long descriptions.

Token tip: a terse instruction ("List 20 open opportunities in VoxFlow") costs far fewer tokens than a paragraph. This is true in both Cowork and Claude Code. See TOKENS.md for the cost comparison.

## Switching subaccounts

Say the subaccount by name or id and Claude calls `switch_location`:

- "Switch GHL to VJHOF." (id `VjHoFGh7RBXtNeb276c7`)
- "Switch back to VoxFlow." (id `8A72BbTkZMJQ8W8eeB9t`)
- "Which GHL location is active right now?"

All following actions target the active subaccount until you switch again.

## Common actions (say these to Claude)

Contacts
- "Find the GHL contact for jane@example.com."
- "Create a GHL contact: Jane Smith, jane@example.com, tag hot-lead."
- "Add the tag webinar-2026 to contact <id>."
- "Show the last 20 contacts added in VoxFlow."

Opportunities and pipelines
- "List open opportunities in the Sales pipeline."
- "Show all pipelines and their stages."
- "Move opportunity <id> to stage Won."

Conversations and messaging
- "Show unread conversations."
- "Send an SMS to contact <id>: Thanks for booking."
- "Send an email to contact <id> with subject X and body Y."

Calendars and appointments
- "List my calendars."
- "Show free slots for calendar <id> next week."
- "Book <contact> into calendar <id> at <time>."

Workflows and automation
- "List all workflows."
- "Add contact <id> to workflow <id>."
- "Remove contact <id> from all workflows."

Invoices, products, store
- "List invoices from this month."
- "Create an invoice for contact <id>."
- "List products and their prices."

Locations and custom data
- "Get the active location details."
- "List custom fields for this location."
- "List all objects (standard and custom) for VoxFlow."

## CLI reference (backup channel)

Run in PowerShell. Load the env vars first if a fresh shell does not have them:

```powershell
foreach($v in 'GHL_API_KEY','GHL_LOCATION_ID','GHL_FIREBASE_REFRESH_TOKEN'){
  Set-Item -Path "env:$v" -Value ([Environment]::GetEnvironmentVariable($v,'User'))
}
```

Global options: `--json` (machine output, put it before the command), `--location-id <ID>` (target another subaccount for one command).

```powershell
# Location
ghl --json locations get
ghl --json locations get --location-id VjHoFGh7RBXtNeb276c7

# Contacts
ghl --json contacts list --limit 50
ghl contacts search "jane"
ghl contacts get <contact_id>
ghl contacts create --email lead@example.com --first-name Jane --last-name Smith --tag hot-lead
ghl contacts add-tag <contact_id> "webinar-2026"

# Opportunities
ghl --json opportunities list --status open
ghl --json opportunities pipelines

# Calendars, workflows, conversations, payments
ghl --json calendars list
ghl --json workflows list
ghl conversations list --status unread
ghl conversations send <conversation_id> --type SMS --message "Thanks!"
ghl --json payments transactions --limit 50

# Forms, social, locations extras
ghl --json forms list
ghl social posts
ghl --json locations custom-fields
```

Note: `locations search` needs a company-level token (`--company-id`). The current VoxFlow key is location-scoped and returns 403 on that endpoint, which is expected and does not affect normal use.

## Setup notes

- MCP: configured in `claude_desktop_config.json` under `mcpServers.gohighlevel`. Restart the Claude desktop app after editing env.
- CLI: needs the three Windows user env vars plus the Chrome cookie captured at first setup. Re-run the LeadGenJey auth step if the cookie or Firebase token expires.
