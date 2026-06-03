# Flow3_AssignedReminder — Personal Reminder to Assigned Person

## Purpose
Sends a personal Teams reminder (1:1 Chat with Flow bot) to the person already assigned to a maintenance task, if it is approaching its due date.

## Trigger
Recurrence — daily

## SP Filter
```
(AssignedTo ne null) and (field_8 ne 'Completed')
```

## Logic
```
GetItems (filtered)
└── ForEach (no concurrency limit — sequential, see BUG-003)
    └── If DaysLeft <= ReminderThreshold AND field_8 eq 'Assigned':
        └── PostCard (Chat with Flow bot → AssignedTo/Email)
            Card shows: Instrument name, Due Date
            No action buttons (informational only)
```

## Known Issues
- **BUG-003:** No concurrency set → sequential loop, slow at scale
- BUG-005: Uses `DaysLeft` (Text) with float() cast
