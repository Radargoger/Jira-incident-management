# SOCRadar Incident Management for Jira

Seamlessly sync threat intelligence alarms from SOCRadar into Jira with bi-directional status synchronization.

---

## How It Works

- **Automated Polling:** Checks for new alarms every **5 minutes** via Forge scheduled trigger.
- **Bi-Directional Sync:** SOCRadar status changes update Jira issues; Jira status changes update SOCRadar alarms.
- **Safe Formatting:** Alarm data is formatted into ADF tables. Fields over 4,500 chars are truncated with a link to view full details on SOCRadar.
- **Issue Panel:** Each SOCRadar-linked issue shows live alarm details directly in Jira.

---

## Setup

Navigate to **Apps → SOCRadar Integration** after installation.

### Connection

| Field | Description |
|-------|-------------|
| Company ID | SOCRadar company identifier |
| API Key | Sent via `API-Key` header |
| User Email | Required for status updates & comments |
| Project Key | Target Jira project (e.g. `SEC`) — auto-uppercased |
| Issue Type | `Task` or `Epic` |

**Sync toggles:** Enable automatic sync, Auto-create issues, Sync comments.

> Click **Test Connection** before saving.

### Filters

Filter incoming alarms by: Status, Severities, Alarm Main/Sub Types, Tags, Assignees.

### Status Mapping

**SOCRadar → Jira (defaults):**

| SOCRadar | Jira |
|----------|------|
| OPEN / PENDING_INFO | To Do |
| INVESTIGATING / LEGAL_REVIEW / VENDOR_ASSESSMENT | In Progress |
| RESOLVED / FALSE_POSITIVE / DUPLICATE / MITIGATED / NOT_APPLICABLE / PROCESSED_INTERNALLY | Done |

**Jira → SOCRadar (defaults):** To Do → OPEN (0), In Progress → INVESTIGATING (1), Done → RESOLVED (2), Closed → MITIGATED (12)

Custom Jira statuses can be added via the **+ Add** button.

### Routing

Assign issues by alarm type. **Sub-type rules take priority** over main-type rules.

---

## Polling & Pagination

### Time Window

```
start = lastPollEpoch - 120s (safety buffer)
end   = now
```

### API Behavior

Base URL: `https://platform.socradar.com/api/company/{id}/incidents/v4`

The API returns **newest alarms on page 1**, oldest on the last page. Each page holds up to 100 alarms.

### Reverse Pagination Strategy

```
500 alarms → 5 pages

Process order: Page 5 → 4 → 3 → 2 → 1 (oldest → newest)
```

1. Fetch page 1 with `include_total_records=true` → get `total_pages`.
2. Start from the **last page**, work backwards to page 1.
3. This ensures chronological processing (oldest first).

### Forge Runtime Handling

Forge has a 25s execution limit, so each trigger uses a **20s time budget**:

```
Trigger 1: Pages 25 → 18 (time up) → save cursor = 17
Trigger 2: Pages 17 → 10           → save cursor = 9
Trigger 3: Pages 9 → 1             → poll complete, clear cursor
```

Max lookback is capped at 2× polling interval to prevent runaway historical polling.

---

## API Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/company/{id}/incidents/v4` | Fetch alarms (paginated) |
| `POST` | `/company/{id}/alarms/status/change` | Update alarm status |
| `POST` | `/company/{id}/alarm/add/comment/v2` | Add comment |
| `POST` | `/company/{id}/alarm/{alarm_id}/assignee` | Change assignee |
| `POST` | `/company/{id}/alarm/severity` | Change severity |
| `POST` | `/company/{id}/alarm/tag` | Add/remove tag |

### Key Query Parameters (Fetch Alarms)

`page`, `limit` (max 100), `start_date` (epoch), `end_date` (epoch), `status`, `severities`, `alarm_main_types[]`, `tags[]`, `assignees[]`, `include_total_records`, `include_alarm_details`

---

## Jira Issue Structure

- **Summary:** `{alarm_id} - {title} - SOCRadar` (max 255 chars)
- **Labels:** `socradar`, `socradar-severity-{level}`, `socradar-alarm-{id}`
- **Priority:** CRITICAL → Highest, HIGH → High, MEDIUM → Medium, LOW → Low
- **Description (ADF):** Alarm details table, description, content info, recommended response, mitigation plan, compliance table, related assets

---

## SOCRadar Status Codes

| Code | Status |
|------|--------|
| 0 | OPEN |
| 1 | INVESTIGATING |
| 2 | RESOLVED |
| 4 | PENDING_INFO |
| 5 | LEGAL_REVIEW |
| 6 | VENDOR_ASSESSMENT |
| 9 | FALSE_POSITIVE |
| 10 | DUPLICATE |
| 11 | PROCESSED_INTERNALLY |
| 12 | MITIGATED |
| 13 | NOT_APPLICABLE |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Issues not created | Check Project Key, Issue Type, and "Auto-create" toggle |
| Status transitions failing | Mapped status must be a valid transition in your Jira workflow |
| "Previous run active" | Use **Reset Sync** to clear the stuck `isRunning` flag |
| Alarms missed | 120s buffer should prevent this; check if filters are too restrictive |
| Reverse sync not working | Issue must be in alarm index (created by this app, not manually) + User Email must be set |
