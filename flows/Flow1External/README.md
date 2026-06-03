# Flow1External — Maintenance Reminder (External / Vendor Email)

## Purpose
Sends an email notification to external vendors for open maintenance items that have crossed the reminder threshold. Uses a deduplication flag to prevent repeat emails.

## Trigger
Recurrence — daily

## SP Filter
```
field_8 eq 'Open' and field_1 eq 'External' and VendorMailSent ne 1
```

## Logic
```
GetItems (filtered)
└── ForEach (concurrency: 10)
    └── If DaysLeftNum <= ReminderThreshold AND VendorEmail ne '':
        └── SendEmail V2 (To: VendorEmail, Subject: VendorMailSubject, Body: VendorMailBody)
            └── PatchItem: VendorMailSent = true
                (runAfter: Succeeded + Failed + Skipped + TimedOut — see BUG-002)
```

## Known Issues
- **BUG-002 (High):** `VendorMailSent` is set even when email send fails → vendor never notified, no retry possible
- BUG-005: Uses `DaysLeftNum` (correct) — consistent with Flowupdatedaily

## Email Fields
| SP Field | Used as |
|---|---|
| `VendorEmail` | `To` address |
| `VendorMailSubject` | Email subject |
| `VendorMailBody` | Email body (HTML) |
