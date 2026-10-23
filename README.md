# voipms-mcp

VOIP.ms Integration MCP for the MSPbots Agent Platform.

Status: **empty scaffold** — created 2026-09-09 for [PRD-18607](https://app.clickup.com/t/86e36377t).
Design has not been done yet; nothing here is implemented.

<div align="center">

## MSPbots Connector

**The MCP Gateway built for MSPs: visible, controlled, and your data stays where it belongs.**

[**Get started at connect.mspbots.ai →**](https://connect.mspbots.ai)

</div>

---

MSPbots Connector gives your AI clients one MCP URL to reach your entire MSP stack (RMM, PSA, documentation, security, finance), and gives you control over who can call which tool and a record of everything that was called.

**Hosted service: [connect.mspbots.ai](https://connect.mspbots.ai)** · Free to use.

### Why Connector

![MSPbots Connector](https://stgoss.mspbots.ai/files/mcp/mspbots-mcp.png)

| | |
|---|---|
| **Visible** | Every tool call is logged: who, which vendor, when. Usage dashboard, CSV export, SIEM streaming. |
| **Controlled** | Role-based, per-tool allowlist. Deny by default. |
| **Protected** | Credentials encrypted per record. AI clients never see your vendor passwords or tokens. |
| **MSP-native** | Built for the MSP stack only: RMM, PSA, documentation, security, finance. |

### Quick start

1. **Sign up** at [connect.mspbots.ai](https://connect.mspbots.ai).
2. **Connect your vendors.** Pick from 150+ connectors and authorize each one on the vendor's own page, or bring your own API keys. No extra vendor accounts needed.
3. **Set access.** Create roles and decide which tools each role may call. Nothing is allowed until you allow it.
4. **Add your MCP URL to your AI client.** Copy the URL from your Connector dashboard into Claude, ChatGPT, Cursor, VS Code, Gemini, Codex, or any MCP-capable client.
5. **Ask your AI to do real work**, and watch every call appear in the audit log.

### Features

**Unified gateway**
- One MCP URL for all your AI clients
- 150+ connectors across the MSP stack
- Custom MCP servers supported, free
- Authorization happens on each vendor's own page; the AI never sees passwords or tokens
- Bring your own credentials: use the vendor API keys you already have
- Multiple accounts per app; connect once, reuse across AI clients
- Tool search: finds the right action for the task
- One sign-in; vendor credentials are injected behind the scenes and OAuth tokens refresh automatically

**Visibility**
- Audit log for every tool call (who, which vendor, when)
- Usage dashboard: call volume, top vendors, error rate, per-user activity
- Export audit logs to CSV, or stream to Loki, Graylog, or LogScale (SIEM)
- Optional recording of tool arguments and result summaries (off by default, enabled per organization)

**Control**
- Per-role, per-tool allowlist, deny by default
- owner / admin / member roles
- SCIM provisioning with Entra ID, Okta, JumpCloud, and Google Workspace

**Security**
- Credentials encrypted one record at a time (AES-256-GCM, per-record key)
- AI clients authenticate with OAuth 2.1 + PKCE
- Vendor responses are scanned for prompt-injection patterns and returned as structured data. This is one layer of defense, not a guarantee against every attack.
- Static egress IP for customer firewall and API allowlisting

### Pricing

**Free.**

Links: [connect.mspbots.ai](https://connect.mspbots.ai) · [mspbots.ai](https://mspbots.ai)

---

## Why this repo exists

The manual workflow it replaces: an ops/support user finds a user's VOIP.ms
sub-account and redirects their voicemail to a requested target. The documented
SOP has 12 steps, but only 2 of them are real actions — the rest is portal
navigation, which is not worth modelling as tools.

## Planned tool surface (MVP)

| tool | what it does |
|---|---|
| `voipms.find_subaccount` | locate a user's sub-account |
| `voipms.get_subaccount` | read the sub-account's full current configuration |
| `voipms.list_voicemail_boxes` | list voicemail boxes, so a destination can be validated before it is used |
| `voipms.set_voicemail_redirect` | update the voicemail redirect |

## The constraint that must not be lost

**`set_voicemail_redirect` must be read-modify-write. Never a single-field blind write.**

The vendor's `setSubAccount` behaves like a *set*, not a *patch*: fields that are
absent from the request are reset to their defaults. Sending only
`internal_voicemail` would therefore silently wipe unrelated settings on a live
customer account. The implementation must read the full sub-account, change only
the intended field, write the whole object back, and return a before/after
comparison.

## Known prerequisites (not yet confirmed by the customer, as of PRD-18607)

- VOIP.ms API enablement on the account
- the execution account and its API password
- a stable platform egress IP, allowlisted on the VOIP.ms side (the API
  authenticates by IP allowlist)

Without these the implementation cannot be tested against anything real.

## Registration

New vendor MCPs are registered in `mcp-service-gateway`'s `vendor/registry.yaml`
(`name` / `repo_url` / `credential_fields`), following `bvoip-mcp`,
`oitvoip-mcp`, `chargebee-mcp` and `bamboohr-mcp`. That entry does not exist yet.

Note: `bvoip` (1Stream) and `oitvoip` (NetSapiens PBX) are already registered and
are **different vendors** — their names merely contain the substring "voip".
