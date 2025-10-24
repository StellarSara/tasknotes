# Kanban "None" Column Bug - All Tasks in Single Column

## Issue Description
When using the Kanban layout for Bases, all task notes were being placed into a single column titled "None" instead of being properly grouped by the configured property (status, priority, projects, etc.). When changing settings in the base, the correct kanban layout would briefly flicker before disappearing again.

## Root Cause Analysis

### Problem 1: Flawed Cache Logic
Located in `src/bases/kanban-view.ts` around lines 115-153 (before fix):

```typescript
// Only use cache if it has a valid value (not null or undefined)
// This ensures we retry detection if it previously failed
if (cachedGroupByPropertyId === undefined || cachedGroupByPropertyId === null) {
    // Try to detect groupBy from controller.query.views...
    // ... detection logic ...
    
    // Cache the determined value (even if null)
    cachedGroupByPropertyId = groupByPropertyId;
} else {
    // Use cached value
    groupByPropertyId = cachedGroupByPropertyId;
}
```

**Issue**: Once the cache was set (even to `null`), the detection logic would never run again. If the initial detection failed or ran before the Bases view was fully initialized, `cachedGroupByPropertyId` would be set to `null` and stay that way forever.

### Problem 2: Premature Early Return
Located in `src/bases/kanban-view.ts` around lines 118-125 (before fix):

```typescript
// Skip rendering if we have no data yet (prevents flickering during data updates)
const hasGroupedData = !!(viewContext.data?.groupedData && Array.isArray(...));
const hasFlatData = !!(viewContext.data?.data && Array.isArray(...));
const hasLegacyResults = !!(viewContext.results && viewContext.results instanceof Map...);

if (!hasGroupedData && !hasFlatData && !hasLegacyResults) {
    return; // Skip render silently - no data available
}
```

**Issue**: This early return was designed to prevent flickering, but it also prevented the initial render when the view was first loading. The Bases API doesn't always have `groupedData` available immediately, causing the kanban view to skip rendering entirely.

### Why the Flicker Occurred
When settings were changed:
1. The Bases view would trigger a refresh
2. For a brief moment, `groupedData` would be available with the correct grouping
3. The kanban view would render correctly
4. But then the cache with `null` would be used on subsequent renders
5. Without a valid `groupByPropertyId`, all tasks would fall into the "none" group

## The Fix

### Changed Behavior
1. **Removed the problematic cache check**: Now the groupBy detection runs on every render
2. **Removed the early return**: The render always attempts to process data if task notes exist
3. **Kept the cache for reference only**: `cachedGroupByPropertyId` is still set but no longer controls whether detection runs

### Code After Fix
```typescript
const render = async function(this: any) {
    if (!currentRoot) return;

    try {
        const viewContext = this?.data ? this : controller;
        const basesViewInstance = this;

        const dataItems = extractDataItems(viewContext);
        const taskNotes = await identifyTaskNotesFromBasesData(dataItems, plugin);

        // Clear board
        board.innerHTML = "";

        if (taskNotes.length === 0) {
            // Show empty state
            return;
        }

        // ... build pathToProps ...

        const groups = new Map<string, TaskInfo[]>();
        let groupByPropertyId: string | null = null;

        if (viewContext.data?.groupedData && Array.isArray(viewContext.data.groupedData)) {
            // Try to detect groupBy property on each render
            // (detection logic remains the same)
            
            // Cache the determined value for future reference
            cachedGroupByPropertyId = groupByPropertyId;
            
            // ... rest of grouping logic ...
        }
    } catch (error) {
        console.error("[TaskNotes][Bases] Error rendering Kanban:", error);
    }
};
```

## Impact
- ✅ Kanban columns now properly group tasks by the configured property
- ✅ No more flickering between correct layout and "None" column
- ✅ Works with all groupBy properties (status, priority, projects, contexts, custom fields)
- ✅ Maintains backward compatibility with legacy Bases API

## Related Files
- `src/bases/kanban-view.ts` - Main fix location
- `src/bases/group-by.ts` - GroupBy configuration detection
- `tests/unit/issues/issue-871-bases-kanban-groupby-drag.test.ts` - Related test for groupBy issues

## Testing
To verify the fix:
1. Create a Base with task notes
2. Add a Kanban view with groupBy set to any property (status, priority, etc.)
3. Verify tasks appear in their correct columns (not all in "None")
4. Change the groupBy setting
5. Verify the layout updates immediately without flickering
