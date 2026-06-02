# Chrysalis Accounts Setup Workflow

## Purpose
Configure the user's email and calendar accounts in `data/config/accounts.yaml`.
This is the single source of truth for every workflow that reads email or
calendar — sweeps, briefs, prep block creation. No workflow should hardcode
account names; every reference flows through this file.

This workflow is **idempotent** — runs cleanly for first-time setup AND for
updates (changing accounts, adding new ones, repointing the write calendar).

## When to Run
- During first-time onboarding (chained from "Onboard me")
- When the user says "Update my accounts", "Change my calendar", "Add an
  email account", "Set up my accounts"
- When `data/config/accounts.yaml` is missing or contains only placeholder values
- When the user changes which calendar they want prep blocks written to
- When they connect a new email MCP (e.g. Gmail) and want it picked up

## Executed by
Claude Code

## Inputs
- A short list of email accounts the user monitors for job search emails
  (provider + address — no OS assumed)
- The user's choice of which calendar receives prep/debrief blocks
- Available MCP servers in the current Claude Code session
- Host OS (auto-detected — controls whether Apple Mail / JXA paths are offered)

---

## Steps

### Step 1 — Read current state (if any)

Check if `data/config/accounts.yaml` exists.

- **If absent or contains only template/placeholder values:** First-time setup.
  Proceed to Step 2.
- **If present and populated:** Update mode. Read it, display the current
  configuration to the user as a summary table, and ask what they want to change:

  > "Here's your current setup:
  >
  > **Email accounts:**
  > - [provider, address, mcp status] (primary)
  > - [provider, address, mcp status]
  >
  > **Calendar:**
  > - Reading from: [list of read_names]
  > - Writing prep blocks to: [write_target] via [write_mechanism]
  >
  > What would you like to change?"

  Based on their answer, focus only on the affected sections. Skip steps
  for unchanged sections. Always rewrite the full file at the end (Step 5).

### Step 2 — Detect host platform

Run once: `uname -s` (capture as `HOST_OS`).

- `Darwin` → Mac. The `apple-mail` MCP and Apple Calendar JXA paths are
  available *if* the user actually uses those providers.
- `Linux`, `MINGW*`, `MSYS*`, `CYGWIN*`, anything else → not Mac. Never
  enumerate Apple Calendar, never call JXA, never reference Mail.app. Skip
  Apple-specific suggestions entirely.

Remember `HOST_OS` for every subsequent branching decision in this workflow.

### Step 3 — Ask providers first (no OS assumptions)

Ask both questions together — do not enumerate any calendar app yet:

> "Two quick questions to get your email and calendar wired up:
>
> 1. **Which email account(s) do you use for job-search mail?** For each,
>    tell me the provider (Google / Outlook or Microsoft 365 / Apple iCloud /
>    Yahoo / other) and the address. Mark one as primary.
>
> 2. **Which one calendar do you treat as your go-to** — where you track
>    interviews and where I should write prep/debrief blocks? Provider +
>    account (e.g. 'Google, you@gmail.com' or 'Outlook, work address')."

Wait for the answer. Build a working list of `(provider, address, role)` for
email and a single `(provider, account)` for the primary calendar.

### Step 4 — Route each provider to a connector path

**Connector preference order — apply per provider, always in this order:**

1. **First-party provider MCP** — shipped by the provider themselves. Most
   reliable, most efficient, smallest auth surface. Examples: Google's Gmail
   MCP, Google's Google Calendar MCP, Microsoft Graph MCP for Outlook /
   Microsoft 365.
2. **Aggregator MCP** — a licensed third-party platform (currently Composio)
   that brokers many providers behind one MCP. Adds a hop but is a supported
   business with usage terms and SLAs. Current best path for Outlook
   consumer `@outlook.com` (since MS Graph search APIs reject consumer
   accounts) and a reasonable fallback for Gmail when the first-party MCP
   isn't installed.
3. **Community single-provider MCP** — open-source connectors maintained by
   the community, not the provider or Anthropic, with no warranty. Example:
   the Apple Mail MCP (`falconbradley/claude-connector-apple-mail`) — it's
   the only practical way to read iCloud mail at all, but treat reliability
   as best-effort.
4. **OS-level scripting** — Apple Calendar JXA, Mail.app JXA. Reliable on
   the host OS only, slower (Calendar.app event enumeration takes 30–90 s),
   but works without any installed MCP. Mac-only.
5. **Paste mode** (`mcp: none`) — Chrysalis still works; the user pastes
   recruiter emails and event details manually.

Never skip a higher tier without telling the user why. If a Tier 1 MCP exists
for a provider but isn't installed, **ask the user to install it before falling
back** — don't silently degrade. Note that Tier 1 doesn't exist for every
provider (Apple ships no MCP for iCloud Mail or Calendar) — in those cases,
the workflow drops straight to Tier 3 / Tier 4.

For each email account and for the chosen calendar, apply this routing.
Probe MCP availability by checking which tools are present in this session.

**Provider → email routing (in preference order — pick the highest tier that resolves):**

| Provider | Tier 1 (first-party) | Tier 2 (Composio) | Tier 3 (community MCP) | Tier 4 (JXA) | Tier 5 |
|---|---|---|---|---|---|
| Google | **Gmail MCP** — look for dedicated `search_threads`, `get_thread`, or `GMAIL_*` tools from a single-provider Gmail server → `mcp: gmail`. If not installed, ask the user to install it before falling back. | Composio Gmail (`GMAIL_FETCH_EMAILS` via the Composio server) → `mcp: composio-gmail` | — | — | `mcp: none` (paste) |
| Apple iCloud | — *(Apple ships no MCP)* | — *(Composio does not connect to iCloud Mail)* | **Apple Mail MCP** on Mac (`search_emails`, `list_mailboxes` present AND iCloud in Mail.app) → `mcp: apple-mail`. Community connector — best-effort reliability. | Mail.app via JXA (rare; usually subsumed by Apple Mail MCP) | `mcp: none` |
| Outlook / Microsoft 365 | **Microsoft Graph MCP** if a first-party Microsoft server is registered → `mcp: ms-graph`. Note: works for M365/Enterprise; the search API rejects consumer `@outlook.com`. | **Composio Outlook** (`OUTLOOK_QUERY_EMAILS` works on consumer `@outlook.com`; `OUTLOOK_SEARCH_MESSAGES` enterprise-only) → `mcp: composio-outlook`. **Recommended path for consumer Outlook.** | Apple Mail MCP on Mac, if Outlook is added inside Mail.app → `mcp: apple-mail` | Mail.app JXA | `mcp: none` (paste) |
| Yahoo / other | — | — | Apple Mail MCP on Mac, if added to Mail.app → `mcp: apple-mail` | Mail.app JXA | `mcp: none` |

**Provider → calendar routing for the write target (in preference order):**

| Provider | Tier 1 (first-party) | Tier 2 (Composio) | Tier 3 (community MCP) | Tier 4 (JXA) | Tier 5 |
|---|---|---|---|---|---|
| Google | **Google Calendar MCP** — dedicated `list_events`, `create_event`, `delete_event` or `GOOGLECALENDAR_*` tools → `write_mechanism: google_calendar_mcp`. If not installed, ask the user to install it. | Composio Google Calendar if available → `write_mechanism: composio_google_calendar` | — | — | Stop and ask the user to install a Google Calendar MCP — without write access, prep/debrief blocks can't be auto-created |
| Apple iCloud | — | — | — *(no community Apple Calendar MCP exists)* | **JXA** on Mac → `write_mechanism: jxa` | Unsupported off-Mac; ask for a different calendar |
| Outlook / Microsoft 365 | Microsoft Graph MCP if registered → `write_mechanism: ms_graph` | **Composio Outlook** (`OUTLOOK_CALENDAR_CREATE_EVENT` + `OUTLOOK_DELETE_CALENDAR_EVENT` confirmed on consumer `@outlook.com`) → `write_mechanism: composio_outlook`. **Recommended for consumer Outlook.** | — | JXA on Mac if Outlook is subscribed inside Calendar.app → `write_mechanism: jxa` | Stop and flag — no reliable write path |
| Other | — | Ask the user to pick a calendar from a provider above as the write target | — | — | — |

**MCP install guidance — when a needed MCP is missing:**

- Gmail MCP / Google Calendar MCP → these are first-tier for Google providers.
  Point the user to the Claude Code MCP marketplace, ask them to install and
  restart Claude Code, then re-run `Update my accounts`. **Pause here** — do
  not silently drop to Composio or paste mode without the user confirming they
  don't want to install the dedicated MCP.
- Composio (for Outlook) → if not connected and the user has an Outlook account,
  surface this clearly: "Composio is the most reliable Outlook path right now.
  Want to set it up? It takes ~2 minutes." Only fall to JXA or paste after
  they decline.
- Apple Mail MCP (Mac users with iCloud/Outlook in Mail.app) → if Mail.app is
  configured but the MCP isn't connected, point them to install instructions.

**Recording the choice in `accounts.yaml`:** use these exact `mcp:` /
`write_mechanism:` values so workflows can branch correctly:

- email `mcp:` → `gmail` | `composio-gmail` | `apple-mail` | `composio-outlook` | `ms-graph` | `none`
- `write_mechanism:` → `google_calendar_mcp` | `composio_google_calendar` | `jxa` | `composio_outlook` | `ms_graph`

### Step 5 — Confirm the calendar to use for reads + writes

Once the write-target provider/MCP path is set:

**Mac + Apple Calendar (`HOST_OS=Darwin` and the user chose iCloud or has
Outlook/Google subscribed in Calendar.app):** enumerate calendars via JXA.
Write to `/tmp/chrysalis_list_cals.js`:
```javascript
var app = Application("Calendar");
JSON.stringify(app.calendars().map(function(c){
  return { name: c.name(), description: c.description() };
}));
```
Run `osascript /tmp/chrysalis_list_cals.js`, then `rm` the file. Present
calendars as a numbered list and ask which to read from and which is the
write target.

**Google Calendar MCP:** call `list_calendars` (or equivalent). Present the
list with calendar IDs. Ask which to read from and which is the write target.
Store calendar IDs (not just display names) in `accounts.yaml`.

**Microsoft Graph MCP:** call its list-calendars equivalent. Same flow.

**No connector available for reads:** explain that without read access, the
daily sweep can only act on what the user pastes; ask whether to proceed
anyway with `mcp: none` and `read_names: []`.

Build `calendar.read_names` (or `read_ids` for Google/MS) from the user's
picks. `calendar.write_target` is the chosen single target.

### Step 6 — Write `data/config/accounts.yaml`

Build the file from the user's answers. Use this template:

```yaml
# Chrysalis — Account Configuration
# Generated by accounts-setup workflow. Update anytime by saying "Update my accounts".
# This file is the single source of truth for all email and calendar access.
# No workflow or sweep should hardcode account names — always read from here.
# ─────────────────────────────────────────────────────────────────────────────

email:
  accounts:
    - provider: [google | outlook | apple | yahoo | other]
      address: [email address]
      # mcp: which connector was chosen — follow the preference tiers in Step 4.
      # gmail | composio-gmail | apple-mail | composio-outlook | ms-graph | none
      mcp: [chosen value]
      primary: [true | false]
      notes: "[short note, e.g. 'Primary job search inbox.' or 'Secondary — not connected.']"
    # ... one block per account

# ─────────────────────────────────────────────────────────────────────────────
calendar:
  # On Mac + Apple Calendar, list calendar names (JXA aggregates iCloud / Outlook /
  # Google subscriptions). On Google/MS Graph paths, use read_ids with API calendar IDs.
  read_names:
    - "[name1]"
    - "[name2]"
    # ...
  # read_ids:                   # uncomment when using google_calendar_mcp or ms_graph
  #   - "[calendar_id_1]"

  # Optional: provider-specific blocks when not using Apple Calendar/JXA
  # google_calendar:
  #   mcp: google-calendar
  #   personal_calendar_id: "[id]"
  # ms_graph:
  #   mcp: ms-graph
  #   user_principal: "[upn]"
  #   calendar_id: "[id]"

  # Write target for prep blocks. Use the calendar name (JXA) or ID (MCP).
  # Follow the preference tiers in Step 4: prefer first-party MCPs, then aggregator
  # MCPs (Composio), then JXA, then paste mode.
  write_target: "[chosen calendar name or ID]"
  write_mechanism: jxa          # google_calendar_mcp | composio_google_calendar | jxa | composio_outlook | ms_graph

  # Deduplication strategy when reading from multiple source calendars.
  dedup_by: title_date          # Match on (normalized_title, start_date) — ignore source

# ─────────────────────────────────────────────────────────────────────────────
# Updated by the connection smoke test in Step 7. Briefs read this to detect drift.
verification:
  last_verified: null           # ISO-8601 timestamp set after Step 7 passes
  email_read: null              # pass | fail | skipped
  calendar_read: null           # pass | fail | skipped
  calendar_write: null          # pass | fail | skipped
  notes: ""                     # any failures, fix path
```

Overwrite `data/config/accounts.yaml` with the populated version. If the file
existed previously, the overwrite is intentional — the rewrite is the update.

### Step 7 — Connection smoke test (verify plumbing)

The file describes intent; this step proves the plumbing actually works.
Run a minimal live check against every configured account/calendar. Do not
skip on the user's say-so — silently broken plumbing is exactly the failure
mode this step exists to catch.

**7a. Email read test** — for each email account where `mcp != none`:
- `gmail`: call `search_threads` with `newer_than:7d` requesting up to 1 thread.
- `ms-graph`: call the equivalent recent-messages query (top 1, last 7 days).
- `apple-mail`: call `search_emails` with `since: today - 7d` (omit `account` —
  the MCP returns empty results when filtered by address).
- Report per account: `Read OK — N messages visible in last 7 days` or
  `Read FAILED — [error message]`.

**7b. Calendar read test** — using the configured `write_mechanism`:
- `google_calendar_mcp`: call `list_events` for the next 7 days on each
  `read_ids` entry.
- `ms_graph`: equivalent calendar-view query.
- `jxa` (Mac): write `/tmp/chrysalis_cal_read_test.js`, list events on each
  `read_names` calendar for the next 7 days, then `rm` the temp file.
- Report: `Read OK — N events visible` or `Read FAILED — [error]`.

**7c. Calendar write test** — exactly once, against `write_target`:
- Create a throwaway event titled `Chrysalis Connection Test — [ISO timestamp]`
  starting 1 hour from now, duration 15 minutes.
- Immediately delete it (capture the event ID/identifier on create so the
  delete call can target it precisely).
- For `jxa`: the create + delete uses the same temp-file pattern as the
  daily sweep (see CLAUDE.md "JXA EXECUTION RULE"). Capture the new event
  via its `uid()` and then delete by iterating events for a matching `summary`
  + `startDate`.
- For `google_calendar_mcp`: `create_event` then `delete_event` on the
  returned ID.
- For `ms_graph`: equivalent create + delete.
- Report: `Write OK — created and deleted` or `Write FAILED — [error]`.

**7d. Handle failures.** If any test fails:
- Show the user the exact failure line and the most likely fix:
  - MCP missing → install instructions
  - Permission denied → OS-level grant (Mac Automation/Full Disk Access, Google
    OAuth scopes, MS Graph permissions)
  - Wrong calendar ID/name → re-pick from Step 5's enumeration
- **Do not proceed to Step 8 (Confirm) or Step 9 (Commit).** Loop back to
  Step 4 or Step 5 with the user to fix the broken path. The file written
  in Step 6 is fine to leave on disk during the loop — it gets rewritten on
  the next pass.

**7e. On success.** Update the `verification:` block in `data/config/accounts.yaml`:
- `last_verified` → current ISO-8601 timestamp
- `email_read`, `calendar_read`, `calendar_write` → `pass` (or `skipped` if
  the user opted out of a specific test for a `mcp: none` account)
- `notes` → empty, or a one-line note about anything intentionally skipped

### Step 8 — Confirm

Tell the user exactly what was configured AND verified:

> "Accounts configured and plumbing verified. Here's the summary:
>
> **Email:**
> - [address] ([provider]) — [connected via X / paste manually] — read test [pass/skip]
> - [address] ([provider]) — [connected via X / paste manually] — read test [pass/skip]
>
> **Calendar:**
> - Reading from: [list] — read test [pass/skip]
> - Prep blocks land in: [write_target] — write test [pass/skip]
>
> Last verified: [timestamp]
>
> Run 'Update my accounts' anytime to change this."

### Step 9 — Commit

Stage and commit:

```bash
git pull
git add data/config/accounts.yaml
git commit -m "config: accounts configuration updated [YYYY-MM-DD]"
git push
```

For first-time setup, prefer the message: `init: data/config/accounts initial setup`.

If the session is in local-only mode (Repository safety gate Step set
`origin` to empty), skip the `git push` line.

---

## Notes

- Never write personal data inline as examples in this workflow — examples
  must use placeholder addresses like `you@example.com`. The framework
  ships publicly; this workflow is reference for any user.
- If the user has zero connectable accounts (no MCPs available, no Mail.app
  setup): proceed anyway with `mcp: none` for everything and `verification`
  entries set to `skipped`. Chrysalis still works — the user just pastes
  signals manually during sweeps and briefs. Surface the gap clearly in Step 8.
- Apple Calendar's `description()` field is often empty. Don't fail if it's
  missing — present the name alone.
- The Apple Mail MCP reads all configured accounts in Mail.app without
  filtering. Passing an `account` parameter to `search_emails` returns
  empty results — always omit it. This is a known quirk of the MCP.
- The `mcp` value is descriptive, not a directive — workflows read it to
  decide *how* to fetch from each account (which tools to call). Setting
  it to `none` doesn't disable the account; it just signals "paste required".
- If running in update mode (Step 1 found an existing file) and the user
  only wants to change one section, still rewrite the whole file in Step 6
  — partial updates risk leaving stale fields behind. Always re-run the
  Step 7 smoke test after any change — the `verification` block is a lie
  if the workflow exits without re-testing.
- The Microsoft Graph / Outlook path is the framework's weakest link. If a
  user lands there, set expectations honestly in Step 4 — don't pretend the
  experience matches Google or Apple.
- The Step 7 smoke test is mandatory. Onboarding-sweep (Phase 3 in the
  chained `Onboard me` flow) assumes the plumbing works — if Step 7 didn't
  run or didn't pass, the auto-discovery sweep in onboarding-sweep will
  produce empty or misleading results.
