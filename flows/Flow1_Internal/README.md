# Flow1_Internal — Maintenance Reminder (Internal)

## Purpose
Sends a Teams Adaptive Card to a Group Chat for open, unassigned internal maintenance items that have crossed the reminder threshold. Waits for a team member to click "Assign to me".

## Trigger
Recurrence — daily at 05:45 UTC

## SP Filter
```
(field_8 eq 'Open') and (AssignedTo eq null) and (field_1 eq 'Internal')
```

## Logic
```
GetItems (filtered)
└── ForEach (concurrency: 15)
    └── Compose: DaysLeft <= ReminderThreshold?
        └── If true:
            └── PostAdaptiveCard (Group Chat, wait for response)
                └── Get_item (re-fetch — race condition guard)
                    └── If action eq 'assign':
                        └── PatchItem: AssignedTo = responder.email, field_8 = 'Assigned'
                    └── Else (already assigned):
                        └── PostCard to responder: "Already assigned to [Name]"
```

## Configuration Required
- `YOUR_TEAMS_THREAD_ID` → replace with your Teams Group Chat thread ID

## Known Issues
- BUG-004: Thread ID hardcoded
- BUG-005: Uses `DaysLeft` (Text) with float() cast instead of `DaysLeftNum`

## Adaptive Card Payload
The card includes:
- Instrument name (`Title`)
- Next Due Date (`field_6`, formatted `dd-MMM-yyyy`)
- "✋ Assign to me" submit action → `data.action = "assign"`
