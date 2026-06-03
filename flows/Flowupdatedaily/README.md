# Flowupdatedaily — Recalculate DaysLeftNum

## Purpose
Daily background job that recalculates `DaysLeftNum` for all SP list items based on the difference between `field_6` (Next Due Date) and today's date.

## Trigger
Recurrence — daily

## SP Filter
None — processes all items (see BUG-001)

## Logic
```
GetItems (no filter)
└── ForEach (concurrency: 10)
    └── PatchItem:
        DaysLeftNum = if(empty(field_6), null,
          int(div(
            sub(ticks(field_6), ticks(utcNow())),
            864000000000
          ))
        )
```

**Ticks formula:** 1 day = 864,000,000,000 ticks (100-nanosecond intervals)  
Negative values = overdue items

## Known Issues
- **BUG-001 (Medium):** No `$filter` → writes to all items including `Completed` ones

## Recommended Filter Fix
```
$filter: field_8 ne 'Completed'
```
