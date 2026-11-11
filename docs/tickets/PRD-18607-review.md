# PRD-18607 · Code review

## Round 1 · 2026-11-11 · reviewed ec6b0f0 (base eb772ac)

Verdict: changes requested

### Findings

1. [must-fix] src/voipms_mcp/api_client.py:101 (and src/voipms_mcp/tools/subaccounts.py:160-165) — `VoipmsClient._call` drops every `None`-valued field before building the vendor request (`query.update({k: v for k, v in params.items() if v is not None})`), and `set_subaccount()` (api_client.py:151-152) calls `_call("setSubAccount", fields)` through this same filtered path with no bypass. `voipms_set_voicemail_redirect` builds `updated_record` as `dict(record)` straight from `getSubAccounts`'s response (subaccounts.py:161-162) and sends it through `set_subaccount`. If any field in a real VOIP.ms sub-account record is `null`/absent (plausible for optional fields such as `callerid_number`, `music_on_hold`, etc. — VOIP.ms's own object model has many optional fields), that field is silently omitted from the `setSubAccount` write. Because `setSubAccount` is confirmed (module docstring, api_client.py:9-13) to require and overwrite the *entire* record, an omitted field gets reset to the vendor default on that call — exactly the defect AC4/AC8 and this ticket's own stated purpose ("defensive implementation to avoid accidental config resets") exist to eliminate. The test suite does not catch this: the only fixture exercised (`SUBACCOUNT_OPS`, tests/test_tools.py:16-21) has no null-valued fields, so TC8 passes without ever exercising the path that reproduces the bug. Fix: the write path must send every key from the read record regardless of value — either stop filtering `None` inside `_call` for the `setSubAccount` call specifically, or give `set_subaccount` its own unfiltered query-building path.

2. [must-fix] src/voipms_mcp/server.py:48-57 (`client_factory`) — every MCP request constructs a new `VoipmsClient`, which owns a new `httpx.AsyncClient` (api_client.py:82), and none of the four tool handlers in src/voipms_mcp/tools/subaccounts.py ever call `client.aclose()` or use `async with`. Each tool invocation permanently leaks one httpx connection pool (and its underlying sockets); under sustained production traffic the service's open connections/file descriptors grow unbounded until it degrades or crashes. Fix: close the client at the end of each tool call (`async with` in each handler, or a wrapper around `client_factory()` in server.py that guarantees `aclose()` once the tool call returns).

3. [must-fix] README.md — missing the mandatory "MSPbots Connector" section, which must match `~/.claude/skills/vendor-mcp-development/reference/readme-connector-section.md` verbatim (review criterion, `legs/review/mcp.md` §4.3). The current README has no such section at all.

4. [must-fix] Repository contains Chinese text, violating the MCP "repo is English-only" requirement (vendor-mcp-development-sop.md §0.4; review criterion in `legs/review/mcp.md` §4.3, which exempts only `PRD-*-review.md`):
   - `src/voipms_mcp/api_client.py:15,24` (docstring quoting the design doc's "Risks and dependencies" / "Open Questions" section headers in Chinese, paraphrased here)
   - `src/voipms_mcp/tools/subaccounts.py:1` (docstring quoting the design doc's "Data and interface contract" section header in Chinese, paraphrased here)
   - `tests/test_tools.py:3` (docstring quoting the design doc's "Test cases" section header in Chinese, paraphrased here)
   - `Dockerfile:1` (comment quoting the design doc's "Change list" section header in Chinese, paraphrased here)
   - `docs/modules/voipms-mcp.md`, `docs/modules/README.md`, `docs/tickets-index.md` — entirely in Chinese
   Fix: translate the comments/docstrings referencing Chinese section names into English paraphrase (cite the English doc section instead). `docs/modules/` and `docs/tickets-index.md` should not exist in an MCP repo at all under the current convention (`references/mcp-track.md` §2.4 — this ticket's design predates that 2026-10-29 rule); removing those two files/paths is simpler than translating them.
