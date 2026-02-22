# Fixes Round 1

## 1. Launch in main editor area, not right sidebar
In `main.ts`, change `activateView()` to open in the main editor area instead of `getRightLeaf()`:
```ts
async activateView() {
  const { workspace } = this.app;
  let leaf = workspace.getLeavesOfType(VIEW_TYPE_TODO)[0];
  if (!leaf) {
    leaf = workspace.getLeaf('tab');  // opens as a tab in the main editor area
    await leaf.setViewState({ type: VIEW_TYPE_TODO, active: true });
  }
  workspace.revealLeaf(leaf);
}
```

## 2. More horizontal spacing in sidebar
In `components/Sidebar.svelte`, increase horizontal padding on sidebar items:
- Group headers: add more left padding (e.g., `padding: 6px 12px` → `padding: 6px 16px`)
- Category items: increase left padding/indent (currently too tight)
- Add `gap` or `margin-bottom` between group blocks for breathing room
- Ensure category names have adequate left indent under their group header

Also in `styles.css`, check `.adhd-todo-sidebar` padding and item spacing.

## 3. Better reactivity
The vault scanner should react more responsively to changes:

### In `state.svelte.ts`:
- Make sure `vault.on('modify')`, `vault.on('create')`, `vault.on('delete')`, `vault.on('rename')` ALL trigger a re-scan
- Debounce the re-scan (300ms) to avoid thrashing on rapid edits
- When a single file changes, ideally only re-scan that file rather than the whole vault

### In `vault-scanner.ts`:
- Add a `scanSingleFile(app, settings, file)` function that returns tasks/categories/groups for just one file
- The state module can then merge/replace entries for that file path instead of full re-scan

### In components:
- Ensure all components properly use `$derived` or `$derived.by` for computed values
- TaskBoard and Sidebar should reactively update when the state arrays change
- If any component caches data in local `$state` that should be `$derived`, fix it

## 4. General polish
- Make sure the App.svelte layout works well at full editor width (not just sidebar width)
- The sidebar should be a left column within the view, main content fills the rest
- Responsive: sidebar could be collapsible on narrow views

## Build & Test
After all changes, run `npm run build` and verify no errors.
