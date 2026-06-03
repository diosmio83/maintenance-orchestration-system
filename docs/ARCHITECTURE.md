# Architecture Decision Records

## ADR-001 — Recurrence-based polling vs. event-driven triggers

**Date:** 2025-12  
**Status:** Current (legacy)

**Decision:**  
All reminder flows use daily Recurrence triggers rather than SharePoint item-change triggers.

**Rationale:**  
- Reminder logic is threshold-based (DaysLeft vs. ReminderThreshold) — not event-based
- SharePoint "When item modified" would not fire on due-date approach (no item change occurs)
- Recurrence ensures daily evaluation regardless of SP activity

**Consequence:**  
Flows run daily even when no items qualify. Accepted trade-off at current list scale.

---

## ADR-002 — Adaptive Card with wait (PostCardAndWaitForResponse) vs. fire-and-forget

**Date:** 2025-12  
**Status:** Current

**Decision:**  
`Flow1_Internal` uses `PostCardAndWaitForResponse` (webhook-based) rather than a fire-and-forget card + separate HTTP trigger flow.

**Rationale:**  
- Simpler architecture — single flow handles the full assign lifecycle
- No need for a separate HTTP-triggered child flow
- Race condition handled by re-fetching item after response before writing

**Consequence:**  
Flow run stays open (suspended) until user responds or times out. Long-running flow runs consume flow run quota. If no one responds within 28 days, the run expires.

---

## ADR-003 — VendorMailSent flag instead of SP status field

**Date:** 2026-01  
**Status:** Current

**Decision:**  
External email deduplication uses a dedicated boolean `VendorMailSent` field rather than relying on `field_8` status.

**Rationale:**  
- External items may remain `Open` for multiple threshold periods
- Status field alone cannot distinguish "notified but not yet resolved" from "not yet notified"

**Consequence:**  
Once set, `VendorMailSent` is never reset — vendor will only receive one email per maintenance cycle. Manual reset required if re-notification is needed.
