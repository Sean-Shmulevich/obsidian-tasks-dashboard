# PRD: Obsidian ADHD Todo Plugin

## Overview
Migrate the existing ADHD Todo SvelteKit app into an Obsidian plugin. The plugin renders a Discord-style sidebar task manager as an `ItemView` pane inside Obsidian. It reads `#todo` tags from vault files and provides two-way sync — completing/adding tasks in the UI modifies the actual `.md` files.

## Source Code to Port
The existing app is at `/home/jackson/Projects/adhd-todo/`. Key files to reference:
- `src/lib/app-state.svelte.ts` — reactive state, persistence, importData()
- `src/lib/types.ts` — Task, Category, CategoryGroup types
- `src/lib/components/Sidebar.svelte` — Discord-style group/category nav
- `src/lib/components/TaskBoard.svelte` — dashboard grid
- `src/lib/components/TaskCard.svelte` — task display with complete/delete
- `src/lib/components/FocusMode.svelte` — Pomodoro timer
- `src/lib/components/QuickCapture.svelte` — quick add task
- `src/lib/components/RecurrencePicker.svelte` — recurring task picker
- `scripts/obsidian-sync.ts` — vault scanner logic (tag parsing, grouping)

## Tech Stack
- **Obsidian Plugin API** (`obsidian` package)
- **Svelte 5** with runes (`$state`, `$derived`, `$effect`, `$props`) — NOT Svelte 4 stores
- **esbuild** + `esbuild-svelte` for compilation
- **TypeScript**
- **CSS custom properties** (no Tailwind)

## Architecture

### Plugin Entry (`main.ts`)
```ts
import { Plugin } from 'obsidian';
import { TodoView, VIEW_TYPE_TODO } from './TodoView';
import { TodoSettings, DEFAULT_SETTINGS, TodoSettingTab } from './settings';

export default class ADHDTodoPlugin extends Plugin {
  settings: TodoSettings;

  async onload() {
    await this.loadSettings();
    this.registerView(VIEW_TYPE_TODO, (leaf) => new TodoView(leaf, this));
    this.addRibbonIcon('check-square', 'ADHD Todo', () => this.activateView());
    this.addCommand({ id: 'open-todo', name: 'Open ADHD Todo', callback: () => this.activateView() });
    this.addSettingTab(new TodoSettingTab(this.app, this));
  }

  async activateView() {
    const { workspace } = this.app;
    let leaf = workspace.getLeavesOfType(VIEW_TYPE_TODO)[0];
    if (!leaf) {
      leaf = workspace.getRightLeaf(false);
      await leaf.setViewState({ type: VIEW_TYPE_TODO, active: true });
    }
    workspace.revealLeaf(leaf);
  }

  async loadSettings() {
    this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData());
  }

  async saveSettings() {
    await this.saveData(this.settings);
  }
}
```

### View (`TodoView.ts`)
```ts
import { ItemView, WorkspaceLeaf } from 'obsidian';
import { mount, unmount } from 'svelte';
import App from './components/App.svelte';
import type ADHDTodoPlugin from './main';

export const VIEW_TYPE_TODO = 'adhd-todo-view';

export class TodoView extends ItemView {
  plugin: ADHDTodoPlugin;
  component: any;

  constructor(leaf: WorkspaceLeaf, plugin: ADHDTodoPlugin) {
    super(leaf);
    this.plugin = plugin;
  }

  getViewType() { return VIEW_TYPE_TODO; }
  getDisplayText() { return 'ADHD Todo'; }
  getIcon() { return 'check-square'; }

  async onOpen() {
    this.component = mount(App, {
      target: this.contentEl,
      props: { plugin: this.plugin }
    });
  }

  async onClose() {
    if (this.component) unmount(this.component);
  }
}
```

### Vault Scanner (`vault-scanner.ts`)
Port the logic from `scripts/obsidian-sync.ts` but use Obsidian's vault API:
- Use `this.app.vault.getMarkdownFiles()` to get all .md files
- Use `this.app.vault.cachedRead(file)` to read contents
- Parse `#todo` tags with the same regex: `/#todo(\/[\w\-]+)*$/`
- Accept both checkbox lines (`- [ ]`, `- [x]`) and plain list items (`- #todo/...`)
- Build groups, categories, tasks from tag structure
- Tag structure: `#todo/<group>/<category>/<subtag>`
- Track source file + line number for each task (needed for vault-writer)

### Vault Writer (`vault-writer.ts`)
- **Complete task**: Read file, find the line, change `- [ ]` to `- [x]`, add ` ✅ YYYY-MM-DD`, write back via `vault.modify()`
- **Uncomplete task**: Change `- [x]` back to `- [ ]`, remove completion date
- **Add task**: Append `- [ ] #todo/<group>/<category> Task title` to inbox file (configurable, default `Todo Inbox.md`)
- **Delete task**: Remove the line from the source file (or comment it out)
- **Edit task title**: Find line, replace title portion

### Reactive State (`state.svelte.ts`)
Replace localStorage with vault-backed state:
```ts
let tasks = $state<Task[]>([]);
let categories = $state<Category[]>([]);
let categoryGroups = $state<CategoryGroup[]>([]);

// Derived
let categoriesByGroup = $derived.by(() => { ... });
let activeTasks = $derived(tasks.filter(t => !t.completed));

// Scan vault and populate state
async function scanVault(vault: Vault) { ... }

// Listen for vault changes
function registerVaultEvents(vault: Vault) {
  vault.on('modify', (file) => rescanFile(file));
  vault.on('create', (file) => rescanFile(file));
  vault.on('delete', (file) => removeTasksFromFile(file));
}
```

### Settings (`settings.ts`)
```ts
export interface TodoSettings {
  tagPrefix: string;          // default: '#todo'
  inboxFile: string;          // default: 'Todo Inbox.md'
  archivedGroups: string[];   // groups to collapse by default
  showCompleted: boolean;     // default: true
  focusDuration: number;      // default: 25 (minutes)
  breakDuration: number;      // default: 5 (minutes)
}
```

### Components to Port
All components use Svelte 5 runes. Key changes:

1. **App.svelte** (NEW — root component, replaces SvelteKit layout)
   - Receives `plugin` prop
   - Contains Sidebar + main content area
   - Manages navigation state (which group/category is selected) via `$state`
   - No router needed — use conditional rendering based on selected view

2. **Sidebar.svelte** — port as-is, receives groups/categories as props

3. **TaskBoard.svelte** — port as-is, dashboard view

4. **TaskCard.svelte** — add "Open in Obsidian" button that calls:
   ```ts
   plugin.app.workspace.openLinkText(task.sourceFile, '', false);
   ```

5. **FocusMode.svelte** — port as-is (pure timer logic)

6. **QuickCapture.svelte** — on submit, call vault-writer to append to inbox file

7. **RecurrencePicker.svelte** — port as-is

8. **ThemeToggle.svelte** — REMOVE (use Obsidian's native theme)

### CSS
- Use CSS custom properties that inherit from Obsidian's theme variables
- Map our design tokens to Obsidian's: `--background-primary`, `--text-normal`, `--interactive-accent`, etc.
- Scope all styles under `.adhd-todo-container` to avoid conflicts
- No Tailwind

### File Structure
```
obsidian-adhd-todo/
├── main.ts
├── TodoView.ts
├── vault-scanner.ts
├── vault-writer.ts
├── settings.ts
├── state.svelte.ts
├── types.ts
├── components/
│   ├── App.svelte
│   ├── Sidebar.svelte
│   ├── TaskBoard.svelte
│   ├── TaskCard.svelte
│   ├── FocusMode.svelte
│   ├── QuickCapture.svelte
│   └── RecurrencePicker.svelte
├── styles.css
├── manifest.json
├── versions.json
├── package.json
├── tsconfig.json
├── esbuild.config.mjs
└── PRD.md
```

### manifest.json
```json
{
  "id": "adhd-todo",
  "name": "ADHD Todo",
  "version": "0.1.0",
  "minAppVersion": "1.5.0",
  "description": "Discord-style task manager that reads #todo tags from your vault",
  "author": "Sean Shmulevich",
  "authorUrl": "https://github.com/Sean-Shmulevich",
  "isDesktopOnly": false
}
```

### Build Configuration
- esbuild bundles to `main.js`
- esbuild-svelte compiles .svelte files with Svelte 5
- Output: `main.js`, `manifest.json`, `styles.css`
- Dev: symlink output to vault's `.obsidian/plugins/adhd-todo/`

## Task Order
1. Set up the plugin scaffold (package.json, esbuild config, tsconfig, manifest)
2. Create main.ts, TodoView.ts, settings.ts with basic plugin registration
3. Port types.ts from existing app
4. Create vault-scanner.ts (port tag parsing from obsidian-sync.ts, use vault API)
5. Create state.svelte.ts (reactive state backed by vault scanner)
6. Create vault-writer.ts (complete/add/delete/edit tasks)
7. Port App.svelte as root component with sidebar + content area
8. Port Sidebar.svelte (groups/categories navigation)
9. Port TaskBoard.svelte and TaskCard.svelte
10. Port FocusMode.svelte and QuickCapture.svelte
11. Port RecurrencePicker.svelte
12. Add styles.css mapped to Obsidian theme variables
13. Register vault events for live updates
14. Add "Open in Obsidian" to TaskCard
15. Wire QuickCapture to vault-writer (append to inbox file)
16. Test: build, symlink to vault, verify it loads

## Important Notes
- Use Svelte 5 runes ONLY (`$state`, `$derived`, `$derived.by`, `$effect`, `$props`) — NO legacy stores or `export let`
- Use `mount()`/`unmount()` from svelte, NOT `new Component()` / `$destroy()`
- The existing app at `/home/jackson/Projects/adhd-todo/` is reference code — read it but build fresh here
- IDs: `crypto.randomUUID?.() ?? Math.random().toString(36).slice(2) + Date.now().toString(36)`
- All tag parsing must support both `- [ ] #todo/...` AND `- #todo/...` (plain list items)
