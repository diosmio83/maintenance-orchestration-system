# Known Issues & Improvement Backlog

## Active Bugs

### BUG-001 — `Flowupdatedaily`: No filter on GetItems
**Flow:** `Flowupdatedaily`  
**Severity:** Medium  
**Description:**  
`Get_items` fetches all SP list items without a `$filter`. This means `DaysLeftNum` is recalculated and written back via `PatchItem` for **every** item including `Completed` ones. At scale this causes unnecessary SP API calls and write operations.

**Fix:**
```
$filter: field_8 ne 'Completed'
```

---

### BUG-002 — `Flow1External`: `VendorMailSent` set even on email failure
**Flow:** `Flow1External`  
**Severity:** High  
**Description:**  
The `Update_item` action (sets `VendorMailSent = true`) runs with `runAfter: [Succeeded, Failed, Skipped, TimedOut]`. If the email send fails, the item is still flagged as sent — the vendor will **never** receive the notification and the flow will not retry.

**Fix:**  
Change `runAfter` on `Update_item` to `[Succeeded]` only. Add an explicit `else` branch for failure that logs or alerts.

---

### BUG-003 — `Flow3_AssignedReminder`: No concurrency limit
**Flow:** `Flow3_AssignedReminder`  
**Severity:** Low  
**Description:**  
The `Apply_to_each` loop has no `runtimeConfiguration.concurrency` set — defaults to sequential (1 iteration at a time). With many assigned items this flow runs slowly.

**Fix:**  
Set concurrency to 10–15 in the ForEach loop settings.

---

### BUG-004 — `Flow1_Internal`: Hardcoded Teams Thread ID
**Flow:** `Flow1_Internal`  
**Severity:** Medium  
**Description:**  
The Teams Group Chat thread ID is hardcoded in the Adaptive Card action. Not portable across environments and breaks silently if the Teams channel/chat is recreated.

**Fix:**  
Store the Thread ID as a Flow variable or environment variable. Consider using a named Teams channel instead of a group chat thread.

---

### BUG-005 — Dual `DaysLeft` / `DaysLeftNum` fields
**Flow:** All  
**Severity:** Low / Tech Debt  
**Description:**  
Two separate fields exist for days remaining: `DaysLeft` (Text, legacy) and `DaysLeftNum` (Number, written by `Flowupdatedaily`). `Flow1_Internal` and `Flow3` use `DaysLeft` with explicit `float()` casting. `Flow1External` uses `DaysLeftNum`. This inconsistency is a maintenance risk.

**Fix:**  
Migrate all flows to use `DaysLeftNum`. Deprecate and eventually remove `DaysLeft`.

---

## Architectural Improvements (Backlog)

### ARCH-001 — Replace Recurrence polling with event-driven triggers
Currently all flows poll SharePoint daily via Recurrence. A more responsive and efficient architecture would use:
- **SharePoint "When an item is modified"** trigger for status changes
- **Scheduled flow only for** `Flowupdatedaily` (DaysLeft recalculation)

This reduces unnecessary runs when no items qualify.

---

### ARCH-002 — Add centralized error handling / logging
No flow currently has a `Configure run after` scope for error logging. Add a `Scope` action wrapping each main block, with a failure branch that posts to a logging list or sends a Teams alert to the flow owner.

---

### ARCH-003 — Parameterize SP Site URL and List ID
Currently hardcoded in every flow. These should be defined once as **Environment Variables** (Power Automate Solution-level) and referenced across all flows. This makes cross-environment deployment (Dev → Prod) straightforward.

---

### ARCH-004 — Package as a Power Platform Solution
The 4 flows should be bundled into a single **unmanaged Solution** with:
- Environment Variables for all config values
- Connection References for SharePoint, Teams, Office 365
- Proper ALM (export unmanaged → import as managed in prod)

This enables proper version control and deployment pipelines.
