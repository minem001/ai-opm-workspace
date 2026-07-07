---
name: JiraToDos
description: Report Jira issues needing your attention — assignments, recent changes, overdue items, and quick actions
argument-hint: "[7|14|30] | 'overdue' | 'new' | 'changes' | 'all'"
---

# JiraToDos — Jira Attention Report

Generate a focused report of Jira issues that need your attention: new assignments, recent changes to your items, overdue issues, and a quick-action layer to act on anything immediately.

## Input Modes

| Argument | Mode | What it does |
|----------|------|-------------|
| _(none)_ / `7` | Default | Last 7 days of changes + overdue + new assignments |
| `14` or `30` | Extended | Same report with a wider change window |
| `overdue` | Overdue only | Just overdue issues assigned to you |
| `new` | New only | Issues newly assigned to you in the window |
| `changes` | Changes only | Issues that changed in the window |
| `all` | Full inventory | All open issues assigned to you, grouped by status — no changelog fetch |

---

## Workflow

### Phase 0: Dependency Check (no interaction)

Run: `acli-pii version`

If it fails, stop and tell the user:

```
❌ acli-pii not found. Ensure it's on your PATH.
   macOS binary: utils/dt-acli-pii-sanitize/acli-pii-darwin
```

---

### Phase 1: Fetch Data in Parallel (no interaction)

Parse the lookback window from args:
- A plain number (`7`, `14`, `30`) → use as days, e.g. `-7d`
- `overdue` / `new` / `changes` / `all` → set mode, default window to 7d
- No args → default mode, 7d window

Run **all applicable queries in parallel**. Skip queries not needed for the selected mode.

**1a. All open issues assigned to me** (always run unless mode = `changes` or `new`):
```bash
acli-pii jira workitem search \
  --jql "assignee = currentUser() AND resolution = Unresolved ORDER BY priority ASC, updated DESC" \
  --json
```

**1b. Recently updated issues assigned to me** (skip if mode = `overdue` or `all`):
```bash
acli-pii jira workitem search \
  --jql "assignee = currentUser() AND updated >= -{N}d AND resolution = Unresolved ORDER BY updated DESC" \
  --json
```

**1c. Overdue issues** (skip if mode = `new` or `changes`):
```bash
acli-pii jira workitem search \
  --jql "assignee = currentUser() AND due < now() AND resolution = Unresolved ORDER BY due ASC" \
  --json
```

**1d. New assignments in the window** (skip if mode = `overdue` or `changes`):
```bash
acli-pii jira workitem search \
  --jql "assignee = currentUser() AND assignee changed to currentUser() after -{N}d AND resolution = Unresolved ORDER BY created DESC" \
  --json
```

**After all queries complete:**
- De-duplicate results by issue key (same issue can appear in multiple sets)
- For issues in 1b (recently updated), fetch changelogs — **limit to the 10 most recently updated** to avoid excessive API calls:
  ```bash
  acli-pii jira workitem changelog {KEY}
  ```
- Parse changelog entries: keep only entries within the lookback window
- Track change types per issue: `status`, `priority`, `assignee`, `duedate`, `summary`, `comment`

---

### Phase 2: Present Report (output only, no interaction)

Output the report. Skip any section that has 0 items. If a mode filter was set, only show the relevant sections.

```
🎯 Jira Attention Report — {YYYY-MM-DD}
   Window: last {N} days  |  Open assigned to you: {total}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔴 OVERDUE ({count})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- {KEY}: {Summary}
  Priority: {P} | Status: {status} | Due: {YYYY-MM-DD} ({N days overdue})

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🆕 NEW ASSIGNMENTS (last {N}d, {count})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- {KEY}: {Summary}
  Priority: {P} | Status: {status} | Assigned: {YYYY-MM-DD}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ RECENTLY CHANGED (last {N}d, {count})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- {KEY}: {Summary}
  Priority: {P} | Status: {status}
  Changes:
    → Status: {old} → {new} ({YYYY-MM-DD})
    → Priority: {old} → {new} ({YYYY-MM-DD})
    → Comment added ({YYYY-MM-DD})

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 ALL OPEN ({total})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Grouped by status]

In Progress ({N}):
  - {KEY}: {Summary} (P{N}, due {YYYY-MM-DD} or —)

In Review ({N}):
  - {KEY}: {Summary} (P{N})

Backlog / Open ({N}):
  - {KEY}: {Summary} (P{N})

[Other statuses as they appear…]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Summary: {overdue} overdue | {new} new | {changed} changed | {total} open total
```

**Changelog rendering rules:**
- Show only changelog entries within the lookback window
- Relevant fields to surface: `status`, `priority`, `assignee`, `duedate`, `summary`
- If a comment was added, show `→ Comment added ({date})` — do not show comment body
- Ignore internal/noise fields: `rank`, `RemoteIssueLink`, `Flagged`, `timeestimate`
- If no meaningful changelog entries found for a recently-updated issue, show `→ (content update — no field changes detected)`
- If changelog was not fetched (issue beyond the 10-item limit), show `→ (updated {date} — changelog not fetched)`

**"All Open" section rules:**
- Always show when mode = `all`
- For default mode: show only if total open issues ≤ 25; otherwise show a condensed summary grouped by status with counts only, and add: `Run /JiraToDos all for the full list`

---

### Phase 3: Quick Actions (one round-trip)

After the report, offer:

```
What next?
  open {KEY}                    — view full issue details
  comment {KEY} {your note}     — add a comment
  transition {KEY} to {status}  — change issue status
  save                          — save report to vault
  done                          — nothing, close out
```

Wait for user response. Execute the matching action:

**`open {KEY}`**
Run `acli-pii jira workitem view {KEY}` and display the result.

**`comment {KEY} {note}`**
Run: `acli-pii jira workitem comment create --key {KEY} --body "{note}"`
Confirm: `✅ Comment added to {KEY}`

**`transition {KEY} to {status}`**
1. Run `acli-pii jira workitem transition list --key {KEY}` to get available transitions
2. Match the requested status (case-insensitive, partial match OK)
3. If matched: `acli-pii jira workitem transition --key {KEY} --status "{matched-status}" --yes`
4. Confirm: `✅ {KEY} transitioned → {status}`
5. If no match: list available statuses and ask the user to pick

**`save`**
Save the report as a Loose Note:
- Path: `Loose Notes/Work/{YYYY-MM-DD} - Jira Attention Report.md`
- Format: Full report text in a markdown note with frontmatter tag `LooseNotes`
- If today's journal exists (`journals/YYYY/MM-Month/YYYY-MM-DD.md`), add a link in the `## Notes` section: `[[{YYYY-MM-DD} - Jira Attention Report]]`
- Append to `workspace_changelog.md`: `- {YYYY-MM-DD} — Saved Jira Attention Report to Loose Notes/Work/`
- Confirm: `Saved: [[{date} - Jira Attention Report]]`

**`done` / anything else**
End: `Done. Run /JiraToDos anytime for an update.`

---

## Error Handling

| Situation | Behavior |
|-----------|----------|
| JQL query returns 0 results | Skip the section silently |
| `currentUser()` fails | Retry with explicit email `assignee = "michael.nemeth@dynatrace.com"` |
| Changelog fetch fails for an issue | Mark `→ (changelog unavailable)` and continue |
| Auth error on any query | Stop and show: `❌ Auth error — run: acli-pii jira auth status` |
| Network/proxy unreachable | Stop and show: `❌ Proxy unreachable — check VPN/network` |

---

## Notes

- **PII is redacted server-side** — names appear as `<PERSON_N>`, emails as `<EMAIL_N>`. This is expected and correct per Dynatrace policy.
- **`currentUser()` JQL** resolves to the authenticated account (`michael.nemeth@dynatrace.com`). If it fails, fall back to the explicit email.
- **Changelog calls are capped at 10 issues** to limit API load. The 10 most recently updated issues from 1b are chosen.
- **Mode shortcuts**: A plain number sets the lookback window. Keyword args select the display mode.
- **`all` mode**: Skips changelog fetching entirely — intended for a quick full-inventory scan.
- **Saving**: Saved reports use the `Loose Notes/Work/` path per vault conventions. Only save when the user explicitly requests it.
