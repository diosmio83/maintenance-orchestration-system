# Maintenance Orchestration System — Power Automate

> **A stateful, multi-channel workflow engine that manages the full lifecycle of equipment maintenance tasks** — from automated due-date calculation and threshold evaluation, through multi-channel notification routing, to interactive self-service assignment via Teams Adaptive Cards and race-condition-safe status transitions persisted in SharePoint.

This is **not a simple reminder system**. It is a **Human-in-the-Loop Maintenance Orchestrator** built entirely on Power Automate Cloud — no custom code, no Azure Functions, no external services.

---

## What Makes This Different

| Capability | Simple Reminder Tool | This System |
|---|---|---|
| Notification delivery | Fire & forget | **Stateful webhook** — flow suspends and waits for user interaction |
| Task assignment | Manual (someone updates SP) | **Self-service via Adaptive Card** — "✋ Assign to me" directly in Teams |
| Race condition handling | None | **Re-fetch guard** before every write — prevents double-assignment |
| Notification routing | Single channel | **Multi-channel**: Teams Group Chat / Personal Teams Chat / Vendor Email |
| Deduplication | None | Per-item `VendorMailSent` flag — prevents repeat vendor emails |
| Status machine | None | `Open → Assigned → Completed` — Flow-driven, SP-persisted |
| Threshold engine | Static | **Per-item configurable** `ReminderThreshold` — recalculated daily |
| Audience scoping | Single audience | `Internal` vs `External` — completely separate notification paths |

### Core Design Patterns Used
- **Stateful Webhook Pattern** — `PostAdaptiveCardAndWaitForResponse` suspends the flow run until a human acts
- **Optimistic Concurrency Guard** — item is re-fetched after card response before writing, catching parallel assignment
- **Daily Computed Column** — `DaysLeftNum` is recalculated centrally by one flow, consumed by all others
- **Deduplication via State Flag** — `VendorMailSent` ensures idempotent external notifications
- **Scope-based Routing** — `field_1 (Internal/External)` drives entirely different orchestration paths

---

## Architecture Overview

```
SharePoint List (Maintenance Items)
        │
        ▼
[Flowupdatedaily]           ──► Recalculates DaysLeftNum for all items (daily)
        │
        ├──► [Flow1_Internal]        ──► Teams Group Chat
        │                                  Adaptive Card → wait → "Assign to me"
        │                                  → PatchItem: AssignedTo + Status = Assigned
        │
        ├──► [Flow1External]         ──► Office 365 Email → Vendor
        │                                  Sets VendorMailSent flag (idempotent)
        │
        └──► [Flow3_AssignedReminder] ──► Teams 1:1 Chat → AssignedTo person
                                           Personal follow-up reminder
```

All flows are **Recurrence-triggered** (daily). No HTTP triggers, no child flows, no premium connectors beyond SharePoint + Teams + Office 365.

**Tech Stack:** Power Automate Cloud · SharePoint Online · Microsoft Teams Adaptive Cards · Office 365 Mail

---

## Flows

| Folder | Display Name | Purpose | Trigger |
|--------|-------------|---------|---------|
| `Flow1_Internal` | Maintenance Reminder Flow 1 (Internal) | Posts Adaptive Card to Teams Group Chat; waits for "Assign to me" response | Daily Recurrence |
| `Flow1External` | Maintenance Reminder Flow 1 (External) | Sends email to vendor; sets `VendorMailSent` flag | Daily Recurrence |
| `Flow3_AssignedReminder` | Maintenance Reminder Flow 3 | Sends personal Teams reminder to already-assigned person | Daily Recurrence |
| `Flowupdatedaily` | Update Daysleft | Recalculates `DaysLeftNum` from `field_6` (Due Date) vs. today | Daily Recurrence |

---

## SharePoint List — Field Reference

| Internal Name | Description | Type |
|---|---|---|
| `Title` | Instrument / equipment name | Text |
| `field_1` | Scope: `Internal` or `External` | Choice |
| `field_6` | Next Due Date | DateTime |
| `field_8` | Status: `Open` / `Assigned` / `Completed` | Choice |
| `AssignedTo` | Responsible person (Person field, Claims) | Person |
| `DaysLeft` | Legacy text field (float-cast in expressions) | Text |
| `DaysLeftNum` | Numeric days remaining — written daily by `Flowupdatedaily` | Number |
| `ReminderThreshold` | Days-before-due threshold to trigger reminder | Number |
| `VendorEmail` | External vendor email address | Text |
| `VendorMailSubject` | Email subject for vendor notification | Text |
| `VendorMailBody` | Email body (HTML) for vendor notification | Text |
| `VendorMailSent` | Flag — prevents duplicate vendor emails | Boolean |

---

## Flow Logic Details

### Flowupdatedaily
- Fetches **all** items (no filter)
- Calculates: `DaysLeftNum = int((ticks(field_6) - ticks(utcNow())) / 864000000000)`
- Writes `DaysLeftNum` back via `PatchItem`
- Concurrency: 10 parallel

### Flow1_Internal
- Filter: `field_8 eq 'Open' AND AssignedTo eq null AND field_1 eq 'Internal'`
- Condition: `DaysLeft <= ReminderThreshold`
- Posts Adaptive Card to Teams Group Chat (configurable Thread ID)
- Waits for user response (`PostCardAndWaitForResponse`)
- On `action eq 'assign'`: re-fetches item → if still unassigned → sets `AssignedTo` + `field_8 = Assigned`
- If already assigned: sends "Already assigned to X" card back to responder
- Concurrency: 15 parallel

### Flow1External
- Filter: `field_8 eq 'Open' AND field_1 eq 'External' AND VendorMailSent ne 1`
- Condition: `DaysLeftNum <= ReminderThreshold AND VendorEmail ne ''`
- Sends email via Office 365 connector
- Sets `VendorMailSent = true` regardless of email success/failure (runAfter: all states)
- Concurrency: 10 parallel

### Flow3_AssignedReminder
- Filter: `AssignedTo ne null AND field_8 ne 'Completed'`
- Condition: `DaysLeft <= ReminderThreshold AND field_8 eq 'Assigned'`
- Posts Adaptive Card directly to assigned person (Teams 1:1 Chat with Flow bot)

---

## Setup & Configuration

### Prerequisites
- Power Automate Premium license (SharePoint + Teams connectors)
- SharePoint Online site with the list structure above
- Microsoft Teams

### Steps

1. **Import each flow** via Power Automate → My Flows → Import → Upload the `.zip` from the original export  
   *(The JSON definitions in this repo are for documentation; for import you need the original `.zip` packages)*

2. **Update Connection References** after import — reconnect SharePoint, Teams, Office 365 connectors

3. **Configure environment-specific values** in each flow definition:

| Placeholder | Replace with |
|---|---|
| `YOUR_SHAREPOINT_SITE_URL` | Your SharePoint site URL |
| `YOUR_LIST_ID` | GUID of your maintenance SP list |
| `YOUR_TEAMS_THREAD_ID` | Teams Group Chat thread ID (for Flow1_Internal) |
| `YOUR_TENANT_ID` | Your Azure AD tenant ID |

4. **Set Recurrence timing** per flow as needed (currently: daily at 05:45 UTC)

---

## Known Issues & Improvement Backlog

See [`KNOWN_ISSUES.md`](docs/KNOWN_ISSUES.md) for documented bugs and architectural improvement suggestions.

---

## License

MIT
