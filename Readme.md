# SOCRadar Incident Management for Jira

Seamlessly sync threat intelligence alarms from SOCRadar into Jira with bi-directional status synchronization.

---

## How It Works

- **Automated Polling:** Checks for new alarms every **5 minutes** via Forge scheduled trigger.
- **Reverse Pagination:** API returns newest alarms on page 1, oldest on last page. The poller fetches from the last page backwards to process alarms in chronological order (oldest → newest).
- **Cursor-Based Resume:** Forge has a 25s execution limit. Each trigger processes pages within a **20s time budget**, saves a cursor, and resumes from where it left off on the next trigger.
- **Directional Sync:** Jira status changes update SOCRadar alarms.
- **Safe Formatting:** Alarm data is formatted into ADF tables. Text fields over 4,500 chars are truncated with a link to view full details on SOCRadar.
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

Each poll calculates a time window using epoch timestamps:

```
start = lastPollEpoch - 120s   (2-minute safety buffer to prevent missed alarms)
end   = now (epoch)
```

If `lastPollEpoch` is not set (first run), the start defaults to `now - pollingInterval`.

A **max lookback cap** of `2 × pollingInterval` prevents runaway historical polling if a previous sync never completed.

### API Behavior

| Detail | Value |
|--------|-------|
| Base URL | `https://platform.socradar.com/api/company/{id}/incidents/v4` |
| Page size | 20 alarms per page (configurable, max 100) |
| Sort order | **Newest on page 1**, oldest on last page |
| Pagination metadata | Returned when `include_total_records=true` (`total_pages`, `total_records`) |

### Reverse Pagination Strategy

Since the API returns newest alarms first and oldest last, alarms must be processed in reverse page order to maintain chronological order:

```
Example: 500 alarms in time window → 25 pages (at 20/page)

Page 1:  alarms 481-500 (newest)
Page 2:  alarms 461-480
...
Page 25: alarms 1-20   (oldest)

Process order: Page 25 → 24 → 23 → ... → 1 (oldest → newest)
```

**Steps:**

1. Fetch page 1 with `include_total_records=true` → get `total_pages` and `total_records`.
2. Set cursor to `total_pages` (last page).
3. Process pages from last → first (reverse order).
4. This ensures Jira issues are created in chronological order.

### Forge Runtime Handling (Cursor-Based Resume)

Forge scheduled triggers have a **25-second execution limit**. The poller uses a **20-second time budget** per trigger and persists progress via a cursor:

```
Trigger 1: Discover 25 pages → process pages 25 → 18 (time up) → save cursor = 17
Trigger 2: Resume from cursor → process pages 17 → 10 (time up) → save cursor = 9
Trigger 3: Resume from cursor → process pages 9 → 1 → poll complete, clear cursor
```

**Cursor state is persisted in Forge storage:**

| Field | Purpose |
|-------|---------|
| `pollCursor` | Next page to process |
| `pollStartEpoch` | Frozen start of time window |
| `pollEndEpoch` | Frozen end of time window |
| `pollTotalPages` | Total pages discovered |

When the cursor is active, the time window is **frozen** — subsequent triggers do not recalculate start/end, ensuring no alarms are skipped or duplicated.

### Concurrency Guard

A `isRunning` flag prevents overlapping executions. If a trigger finds `isRunning = true`, it skips. The flag is cleared after each page-processing cycle (not only on full completion), so the next scheduled trigger can resume.

> **Stuck state:** If a run crashes without clearing `isRunning`, use the **Reset Sync** button in the admin UI.

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

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Results per page (default: 20, max: 100) |
| `start_date` | epoch/date | Filter from date |
| `end_date` | epoch/date | Filter to date |
| `status` | string | Filter by status |
| `severities` | string | Filter by severity (LOW, MEDIUM, HIGH, CRITICAL) |
| `alarm_main_types[]` | array | Filter by main alarm type |
| `alarm_sub_types[]` | array | Filter by sub alarm type |
| `tags[]` | array | Filter by tags |
| `assignees[]` | array | Filter by assignees |
| `include_total_records` | boolean | Include `total_pages` and `total_records` in response |
| `include_alarm_details` | boolean | Include alarm type details in response |

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
| Pages remaining after trigger | Normal behavior — cursor will resume on next 5-minute trigger |
