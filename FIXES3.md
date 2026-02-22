# Fixes Round 3

## 1. Level 4+ subtag categories
For tags with 4+ levels like `#todo/personal/social/ebook` and `#todo/personal/social/online/x`:
- When viewing the "Social" category, tasks should be grouped by their subtag (level 4)
- `#todo/personal/social/ebook` → subtag: "ebook"
- `#todo/personal/social/online/x` → subtag: "online/x" (levels 5+ get joined with `/`)
- Show these as sub-sections within the category view (like mini-headers grouping tasks)
- Cap at level 4 for the subtag split — anything deeper gets joined: `#todo/a/b/c/d/e` → subtag "c/d/e"
- Update vault-scanner.ts to properly extract subtags at level 3+ (parts[2:].join('/'))
- Update TaskBoard.svelte or the category view to render subtag groupings

## 2. Remove recurrence entirely
- Delete `components/RecurrencePicker.svelte`
- Remove all recurrence imports/rendering from QuickCapture.svelte, TaskCard.svelte, TaskBoard.svelte
- Remove `Recurrence` type from types.ts
- Remove `recurrence` and `nextDueAt` fields from Task type
- Remove recurrence parsing from vault-scanner.ts
- Remove "Recurring" count from stats display (the stat boxes showing Open/Done/Recurring)

## 3. Remove redundant top header in category/group views
Looking at the screenshot, there are TWO headers when viewing a category:
- Top bar: "Astron / Tasks from this category sourced from #todo tags" with Board + Rescan buttons
- Below it: "Astron / Astron" with Open/Done/Recurring stats

Fix:
- Remove the topmost header bar entirely (the one with "Tasks from this category..." description and Board/Rescan buttons)
- Move the Board and Rescan buttons into the second header (the one with stats)
- The result: ONE header with category name, stats (Open/Done — no Recurring), Board button, Rescan button
- Apply same fix to group views

## 4. Uncategorized section on Dashboard
- Add an "Uncategorized" section to the Dashboard view
- Place it at the TOP of the dashboard (before other task sections)
- Shows tasks that have no category (group-level tasks with groupTag but no categoryId, plus truly ungrouped tasks)
- Clickable to navigate to a filtered view of uncategorized tasks

## Build
After all changes: `npm run build` — must compile with zero errors.
