---
name: asky-e2e
description: Automate Asky data analytics platform UI testing using Claude Chrome extension (MCP browser tools). Covers localhost dev setup, backend log observation, login, project management, file upload, chat interaction, explorer navigation, SpreadJS grid interaction, report creation/editing, and assertions.
trigger: when the user wants to test or automate the Asky platform UI, prism, reports, or interact with asky app, run tests on asky, or automate browser actions on localhost or stage.ask-y.ai, or when the user says test this prism flow, test reports
---

# Asky E2E Browser Automation Skill

**Default mode: localhost testing.** This skill covers running services locally, observing logs, and automating the Asky platform via Claude Chrome MCP tools.

## LOCALHOST SETUP (Default)

### Services & Ports

| Service | Port | Path | Command |
|---------|------|------|---------|
| **asky.core** (ASP.NET Core API) | 5141 | `C:\work\asky\jam\jamback` | `dotnet run --project asky.core` |
| **JamApi.Api** (Azure Functions) | 7071 | `C:\work\asky\jam\jamback\JamApi.Api` | `func start` |
| **prismfront** (Angular) | 4200 | `C:\work\asky\prism\prismfront` | `npm start` |

### Starting the Backend with Log Capture

**Always capture logs to a file** — you cannot diagnose issues without them.

```bash
cd /c/work/asky/jam/jamback && dotnet run --project asky.core > /tmp/backend.log 2>&1 &
```

Verify it's running:
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:5141/api/auth/test
# 404 = running. 000 = not running.
```

### Restarting the Backend

DuckDB is in-memory — **every restart clears all DuckDB tables**. Tables reload from parquet on first access.

```bash
netstat -ano | grep ":5141" | grep LISTENING   # find PID
taskkill //PID <pid> //F                        # kill it
sleep 2 && cd /c/work/asky/jam/jamback && dotnet run --project asky.core > /tmp/backend.log 2>&1 &
```

### Log Observation

**Always observe logs before and after any test action.** Never guess — read the actual error.

```bash
wc -l /tmp/backend.log                    # note line count BEFORE action
# ... perform action in browser ...
tail -n +<line+1> /tmp/backend.log        # read only NEW lines after action
grep -i "error\|fail\|exception" /tmp/backend.log | tail -10  # scan for errors
```

The backend logs are noisy (Gmail polling, health checks). Filter with `grep -v` to exclude irrelevant services when scanning.

## PREREQUISITES

1. **Get tab context**: Call `tabs_context_mcp` with `createIfEmpty: true` before any action.
2. **Start WebSocket file bridge** (for file uploads): Run in Claude Code Bash:
   ```bash
   NODE_PATH=/c/work/asky/prism/prismfront/node_modules node /c/work/asky/e2eTests/asky-e2e-framework/playwright-runner/tests/data/ws_file_bridge.js &
   ```
3. **Base URL**: `http://localhost:4200` (default) or `https://stage.ask-y.ai` / `https://prism.ask-y.ai`.
4. **App is Angular SPA**: After login the URL is `{baseUrl}/studio`. All operations happen in that single page.

---

## 1. LOGIN

**Verified working.** Navigate to `{baseUrl}/login`, fill credentials, click Login.

```
navigate → {baseUrl}/login
wait 3s
find "Email input" → triple_click to select all → type email
find "Password input" → triple_click to select all → type password
find "Login button" → left_click
wait 5s → verify URL contains /studio
```

**Faster via JS** (use after first navigation):
```javascript
// Already on login page with pre-filled credentials? Just click Login:
document.querySelector('button[type="submit"]')?.click()
```

---

## 2. CREATE PROJECT

**Verified working.** Opens drawer, clicks New Project, fills name, confirms.

```javascript
// Step 1: Open project drawer
document.querySelector('button[data-qa="chat-sessions-drawer-toggle"]')?.click();
// Wait 2s

// Step 2: Click New Project
document.querySelector('button.new-project-button')?.click();
// Wait 500ms

// Step 3: Fill name and confirm
const name = 'AutoProject_' + Math.random().toString(36).substring(2, 7);
const input = document.querySelector('input.new-project-input');
input.value = name;
input.dispatchEvent(new Event('input', { bubbles: true }));
window.__testProjectName = name;
setTimeout(() => document.querySelector('.new-project-input-row button.confirm-btn')?.click(), 300);
// Wait 5s

// Step 4: Verify
document.querySelector('[data-qa="chat-project-name"]')?.textContent?.includes(name)
```

---

## 3. UPLOAD FILE (WebSocket Bridge)

**Verified working.** Uses WebSocket to bypass HTTPS mixed-content restrictions. Requires `ws_file_bridge.js` running on localhost:8767.

```javascript
// Single JS call — fetches file via WebSocket and uploads via Angular API
new Promise((resolve, reject) => {
  const ws = new WebSocket('ws://localhost:8767');
  ws.onopen = () => {
    ws.send(JSON.stringify({ action: 'getFileBase64', fileName: 'YOUR_FILE.xlsx' }));
  };
  ws.onmessage = (event) => {
    const resp = JSON.parse(event.data);
    if (resp.status === 'ok') {
      const b = atob(resp.data);
      const a = new Uint8Array(b.length);
      for (let i = 0; i < b.length; i++) a[i] = b.charCodeAt(i);
      const f = new File([a], resp.fileName, { type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' });
      const dt = new DataTransfer();
      dt.items.add(f);
      const ng = window.ng;
      const els = [
        document.querySelector('[data-qa="explorer-compact-drop-zone"]'),
        document.querySelector('.explorers-content'),
        document.querySelector('app-datapane'),
        document.querySelector('app-sheets-board'),
        document.querySelector('app-home'),
      ];
      let comp = null;
      for (const el of els) {
        if (!el) continue;
        let c = el;
        while (c && !comp) {
          try { const x = ng.getComponent(c); if (x && x.processDroppedFiles) { comp = x; break; } } catch(e) {}
          c = c.parentElement;
        }
        if (comp) break;
      }
      if (comp) { comp.processDroppedFiles(dt.files); ws.close(); resolve('OK: ' + f.name + ' ' + f.size + 'b'); }
      else { ws.close(); resolve('ERROR: no Angular component'); }
    } else { ws.close(); resolve('ERROR: ' + resp.message); }
  };
  ws.onerror = (e) => resolve('WS_ERROR');
  setTimeout(() => resolve('TIMEOUT'), 30000);
})
```

**Wait for upload to complete** (poll every 5s):
```javascript
document.querySelectorAll('.excel-file-item[data-qa="loading-file"]').length === 0
```

**Close intro video** (if it appears):
```javascript
const btn = document.querySelector('video + button'); if (btn) btn.click();
```

**Available test files** (in `tests/data/`):
- `Ads_Ga4_SpendRev_Apparel.xlsx` — 3 tables (GA4, Google, Facebook)
- `t1.csv`, `BQ_nationalmenopauseshow2024.xlsx`, `Clay_People_1000.xlsx`
- `Campaign_Naming_Convention.txt` — document upload

---

## 4. SEND CHAT MESSAGE

**Verified working.**

```javascript
const input = document.querySelector('[data-qa="chat-input"]');
input.click();
input.focus();
input.textContent = 'YOUR MESSAGE HERE';
input.dispatchEvent(new Event('input', { bubbles: true }));
setTimeout(() => {
  input.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', code: 'Enter', bubbles: true }));
  setTimeout(() => {
    const sendBtn = document.querySelector('button.send-button');
    const inputBox = document.querySelector('.chat-container app-input-box');
    if (inputBox?.getAttribute('ng-reflect-is-processing') === 'false' && sendBtn?.offsetParent) sendBtn.click();
  }, 500);
}, 200);
```

---

## 5. WAIT FOR CHAT IDLE

**Verified working.** Polls `ng-reflect-is-processing` attribute. Chat can take 10s to 5+ minutes.

```javascript
new Promise((resolve) => {
  let checks = 0;
  const interval = setInterval(() => {
    checks++;
    const inputBox = document.querySelector('.chat-container app-input-box');
    const isProcessing = inputBox?.getAttribute('ng-reflect-is-processing');
    if (isProcessing === 'false') { clearInterval(interval); resolve('IDLE after ' + (checks * 5) + 's'); }
    else if (checks >= 60) { clearInterval(interval); resolve('TIMEOUT'); }
  }, 5000);
})
```

---

## 6. GET LAST AGENT MESSAGE

```javascript
(() => {
  const messages = document.querySelectorAll('[data-qa="message-agent"] [data-qa="message-content"]');
  for (let i = messages.length - 1; i >= 0; i--) {
    const clone = messages[i].cloneNode(true);
    clone.querySelectorAll('.navigation-control-container, .navigation-chapter, .navigation-task, .navigation-subtask').forEach(n => n.remove());
    const text = clone.innerText.trim();
    if (text && !/what would you like to do next\??/i.test(text)) return text.substring(0, 500);
  }
  return '';
})()
```

---

## 7. CLICK ACCEPT ALL

```javascript
document.querySelector('button.accept-all-btn')?.click()
```

---

## 8. EXPAND ALL EXPLORER ITEMS

**Verified working.** Run 2-3 times for nested items.

```javascript
(() => {
  const container = document.querySelector('.explorers-content');
  if (!container) return 'no container';
  let totalClicked = 0;
  for (let pass = 0; pass < 3; pass++) {
    let clicked = 0;
    container.querySelectorAll('button.expand-btn:not([disabled])').forEach(btn => {
      const icon = btn.querySelector('mat-icon')?.textContent?.trim();
      const insideFolderHeaderItem = !!btn.closest('.folder-header-item');
      const shouldClick = insideFolderHeaderItem ? icon === 'chevron_right' : icon === 'expand_more';
      if (shouldClick) { btn.click(); clicked++; }
    });
    container.querySelectorAll('button.section-expand-btn:not([disabled])').forEach(btn => {
      const icon = btn.querySelector('mat-icon')?.textContent?.trim();
      if (icon === 'chevron_right') { btn.click(); clicked++; }
    });
    totalClicked += clicked;
    if (clicked === 0) break;
  }
  return 'expanded ' + totalClicked + ' items';
})()
```

---

## 9. READ EXCEL EXPLORER TREE

**Verified working.** Call expand-all first.

```javascript
(() => {
  const root = { files: [] };
  document.querySelectorAll('.excel-files-list .excel-file-item:not(.nested-file)').forEach(fileNode => {
    const fileName = (fileNode.querySelector('.file-name')?.textContent || '').trim();
    if (!fileName) return;
    const fileInfo = { fileName, tables: [] };
    const tableItems = fileNode.parentElement?.querySelector('.tables-list')?.querySelectorAll('[data-qa="table-list-item"]') ?? [];
    tableItems.forEach(t => {
      const displayName = (t.querySelector('.table-name')?.textContent || '').trim();
      const columns = [];
      t.querySelectorAll('[data-qa="excel-column-name"]').forEach(c => { const n = (c.textContent||'').trim(); if(n) columns.push(n); });
      fileInfo.tables.push({ displayName, columns });
    });
    root.files.push(fileInfo);
  });
  return JSON.stringify(root);
})()
```

---

## 10. READ PRISM EXPLORER TREE

**Verified working.** Call expand-all first and scroll to Prisms section.

```javascript
(() => {
  const result = { folders: [] };
  document.querySelectorAll('[data-qa="explorer-prism-folder-item"]').forEach(fh => {
    const name = (fh.querySelector('.folder-name')?.textContent || '').trim();
    if (!name) return;
    const folder = { name, prisms: [] };
    const contents = fh.parentElement?.querySelector('.folder-contents');
    if (contents) {
      contents.querySelectorAll('[data-qa="explorer-prism-table-item"]').forEach(pi => {
        const dn = (pi.querySelector('[data-qa="explorer-prism-table-name"]')?.textContent || '').trim();
        if (!dn) return;
        const cols = [];
        let sib = pi.nextElementSibling;
        while (sib) {
          if (sib.classList.contains('table-sections-wrapper')) {
            sib.querySelectorAll('[data-qa="explorer-column-name"]').forEach(c => { const n=(c.textContent||'').trim(); if(n) cols.push(n); });
            break;
          }
          if (sib.getAttribute('data-qa') === 'explorer-prism-table-item') break;
          sib = sib.nextElementSibling;
        }
        folder.prisms.push({ name: dn, columns: cols });
      });
    }
    result.folders.push(folder);
  });
  return JSON.stringify(result);
})()
```

---

## 11. DELETE PROJECT

**Verified working.**

```javascript
// Step 1: Open drawer
document.querySelector('button[data-qa="chat-sessions-drawer-toggle"]')?.click();
// Wait 2s

// Step 2: Find project and click delete
const items = document.querySelectorAll('.projects-list .project-item');
for (const item of items) {
  if (item.textContent.includes('PROJECT_NAME')) {
    item.dispatchEvent(new MouseEvent('mouseenter', { bubbles: true }));
    item.querySelector('.delete-project-btn')?.click();
    break;
  }
}
// Wait 2s

// Step 3: Confirm in dialog
const btns = document.querySelectorAll('mat-dialog-actions button');
for (const b of btns) { if (b.textContent.trim() === 'Delete') { b.click(); break; } }
// Wait 10s for backend deletion
```

---

## 12. OPEN EXISTING PROJECT

```javascript
// Step 1: Open drawer
document.querySelector('button[data-qa="chat-sessions-drawer-toggle"]')?.click();
// Wait 2s

// Step 2: Click project
const items = document.querySelectorAll('.projects-list .project-item');
for (const item of items) {
  if (item.textContent.includes('PROJECT_NAME')) { item.click(); break; }
}
// Wait 2s

// Step 3: Click first session
document.querySelector('.sessions-list .session-item')?.click();
```

---

## 13. START NEW SESSION

```javascript
document.querySelector('[data-qa="chat-new-session-button"]')?.click();
// Wait for messages to clear:
// document.querySelectorAll('[data-qa="message-agent"], [data-qa="message-user"]').length === 0
```

---

## 14. WAIT FOR SEMANTIC STATUS COMPLETE

Poll every 5s, timeout 120s:
```javascript
document.querySelector('[data-qa="semantic-status"]')?.textContent?.trim() === 'complete'
```

---

## 15. READ TABLE DATA (Grid Snapshot)

```javascript
(() => {
  const tl = window.__askyTestLog;
  if (!tl?.getLastGridSnapshot) return null;
  const s = tl.getLastGridSnapshot();
  if (!s?.headers) return null;
  return JSON.stringify({ source: s.source, tableName: s.tableName, viewName: s.viewName, headers: s.headers, rowCount: s.rows.length, sampleRows: s.rows.slice(0, 5) });
})()
```

---

## FULL E2E PATTERN (Localhost)

```
0. Start backend: dotnet run --project asky.core > /tmp/backend.log 2>&1 &
   Start frontend: npm start (in prismfront/, usually already running)
   Start WS bridge: node ws_file_bridge.js & (if uploading files)
1. Get tab context: tabs_context_mcp
2. Navigate: http://localhost:4200/login
3. LOGIN
4. CREATE PROJECT or OPEN EXISTING
5. UPLOAD FILE via WebSocket bridge (if needed)
6. Wait for upload + semantic processing
7. SEND CHAT MESSAGE → WAIT FOR CHAT IDLE
8. CLICK ACCEPT ALL (if changes proposed)
9. EXPAND ALL EXPLORER ITEMS
10. READ EXCEL/PRISM TREES → verify results
11. Test grid interactions (cell edits, row ops) via SpreadJS
12. Observe backend logs after each action
13. Test persistence: reload page, restart backend
14. Report operations (create, insert blocks, rename, delete)
15. DELETE PROJECT (cleanup)
```

## TIMING REFERENCE

| Step | localhost | stage.ask-y.ai |
|------|----------|----------------|
| Backend startup (`dotnet run`) | ~20s | N/A |
| Login | ~3s | ~5s |
| Create Project | ~5s | ~10s |
| Upload File (20KB xlsx via WS) | ~10s | ~15s |
| Chat Message + Wait Idle | 10-300s | 10-300s |
| Accept All Changes | ~2s | ~2s |
| Expand + Read Explorer | ~3s | ~5s |
| Cell edit → parquet commit | ~8s | ~8s |
| Backend restart + verify | ~25s | N/A |
| Report operations | ~2-3s each | ~2-3s each |
| Delete Project | ~10s | ~15s |

## KEY DATA-QA SELECTORS

| Selector | Element |
|----------|---------|
| `[data-qa="chat-input"]` | Chat message input |
| `[data-qa="chat-project-name"]` | Project name header |
| `[data-qa="chat-sessions-drawer-toggle"]` | Open project drawer |
| `[data-qa="chat-new-session-button"]` | New session button |
| `[data-qa="explorer-compact-drop-zone"]` | File drop zone |
| `[data-qa="loading-file"]` | Loading file indicator |
| `[data-qa="table-list-item"]` | Table in explorer |
| `[data-qa="excel-column-name"]` | Excel column name |
| `[data-qa="explorer-prism-folder-item"]` | Prism folder |
| `[data-qa="explorer-prism-table-item"]` | Prism table |
| `[data-qa="explorer-prism-table-name"]` | Prism table name |
| `[data-qa="explorer-column-name"]` | Prism column name |
| `[data-qa="semantic-status"]` | Semantic processing status |
| `[data-qa="message-agent"]` | Agent message container |
| `[data-qa="message-content"]` | Message content |
| `button.accept-all-btn` | Accept all changes |
| `button.send-button` | Send message button |
| `button.new-project-button` | New project button |
| `input.new-project-input` | Project name input |
| `.confirm-btn` | Confirm project creation |
| `.delete-project-btn` | Delete project (on hover) |
| `.project-item` | Project in list |
| `.session-item` | Session in list |
| `[data-qa="explorer-reports-add"]` | Add new report button |
| `[data-qa="explorer-reports-expand"]` | Expand/collapse reports list |
| `[data-qa="explorer-report-item"]` | Report item in explorer list |
| `[data-qa="prism-copy-report-button"]` | Copy prism to report (clipboard) |
| `[data-qa="viz-copy-report-button"]` | Copy visualization to report (clipboard) |
| `.report-title-input` | Report title input |
| `.report-title-bar .delete-btn` | Delete report (editor) |
| `.report-title-bar .refresh-btn` | Refresh data (editor) |
| `.report-title-bar .download-btn` | Download PDF (editor) |
| `.text-btn.new-btn` | New report (editor) |
| `.reports-explorer` | Reports explorer container |
| `.report-name` | Report name text (inside item) |
| `.html-block-preview` | HTML block preview (click to edit) |
| `.html-block-editor` | HTML block textarea (edit mode) |
| `#editorjs` | EditorJS container |

---

## 16. CREATE REPORT

**Verified working.** Clicks the "+" button in the Reports explorer section.

```javascript
// Click Add Report button in the explorer
document.querySelector('[data-qa="explorer-reports-add"]')?.click();
// Wait 3s for report to be created and editor to open
```

**Verify**: Report editor opens on the right side with title "New Report" and EditorJS placeholder.
```javascript
document.querySelector('.report-title-input')?.value // Should be 'New Report'
```

---

## 17. RENAME REPORT

**Verified working.** Updates the title input using Angular-compatible value setter.

```javascript
const titleInput = document.querySelector('.report-title-input');
titleInput.focus();
titleInput.select();
const nativeInputValueSetter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
nativeInputValueSetter.call(titleInput, 'YOUR_REPORT_NAME');
titleInput.dispatchEvent(new Event('input', { bubbles: true }));
titleInput.dispatchEvent(new Event('change', { bubbles: true }));
titleInput.blur(); // triggers save
```

---

## 18. SELECT REPORT (Open Existing)

**Verified working.** Clicks a report item in the explorer by name.

```javascript
const items = document.querySelectorAll('[data-qa="explorer-report-item"]');
for (const item of items) {
  if (item.querySelector('.report-name')?.textContent?.trim() === 'REPORT_NAME') {
    item.click();
    break;
  }
}
// Wait 2s for editor to load content
```

---

## 19. DELETE REPORT (from Explorer)

**Verified working.** Hover the report item to reveal delete button, click it, confirm in dialog.

```javascript
// Step 1: Find report item and trigger hover + click delete
const items = document.querySelectorAll('[data-qa="explorer-report-item"]');
for (const item of items) {
  if (item.querySelector('.report-name')?.textContent?.trim() === 'REPORT_NAME') {
    item.dispatchEvent(new MouseEvent('mouseenter', { bubbles: true }));
    setTimeout(() => item.querySelector('.delete-btn')?.click(), 300);
    break;
  }
}
// Wait 1s for confirmation dialog

// Step 2: Confirm deletion in dialog
const btns = document.querySelectorAll('mat-dialog-actions button, .mat-mdc-dialog-actions button');
for (const b of btns) { if (b.textContent.trim() === 'Delete') { b.click(); break; } }
// Wait 2s
```

---

## 20. DELETE REPORT (from Editor Title Bar)

**Verified working.** Uses the delete button in the report editor's title bar.

```javascript
// Click delete button in title bar
document.querySelector('.report-title-bar .delete-btn')?.click();
// Wait 1s for confirmation dialog

// Confirm deletion
const btns = document.querySelectorAll('mat-dialog-actions button, .mat-mdc-dialog-actions button');
for (const b of btns) { if (b.textContent.trim() === 'Delete') { b.click(); break; } }
// Wait 2s
```

---

## 21. INSERT REPORT BLOCKS (via EditorJS API)

**Verified working.** Access the EditorJS instance through the Angular component to insert blocks programmatically.

### Helper: Get EditorJS instance
```javascript
// Reusable helper to get the editor instance
(async () => {
  const editorHolder = document.getElementById('editorjs');
  const ng = window.ng;
  let reportComp = null;
  let el = editorHolder;
  while (el && !reportComp) {
    try { const c = ng.getComponent(el); if (c && c.editor) { reportComp = c; break; } } catch(e) {}
    el = el.parentElement;
  }
  if (reportComp && reportComp.editor) {
    const editor = reportComp.editor;
    // Use editor here...
    return 'got editor';
  }
  return 'no editor found';
})()
```

### Insert Paragraph
```javascript
await editor.blocks.insert('paragraph', { text: 'Your paragraph text here.' });
```

### Insert Header (H1-H4)
```javascript
await editor.blocks.insert('header', { text: 'Your Header', level: 2 }); // level: 1-4
```

### Insert List
```javascript
await editor.blocks.insert('list', { style: 'unordered', items: ['Item 1', 'Item 2', 'Item 3'] });
// style: 'unordered' (bullets) or 'ordered' (numbers)
```

### Insert Table
```javascript
await editor.blocks.insert('table', { content: [['Col A', 'Col B'], ['Val 1', 'Val 2'], ['Val 3', 'Val 4']] });
```

### Insert HTML Block
```javascript
await editor.blocks.insert('html', { html: '<div><b>Bold</b> HTML content</div>' });
```

### Insert Prism Block (table with data)
```javascript
await editor.blocks.insert('prism', {
  columns: [
    { name: 'col1', displayName: 'Column 1' },
    { name: 'col2', displayName: 'Column 2' }
  ],
  data: [
    { col1: 'value1', col2: 'value2' },
    { col1: 'value3', col2: 'value4' }
  ],
  metadata: { displayName: 'Table Title', totalRows: 2 },
  queryDef: { sql: 'SELECT * FROM table', modelName: 'model' } // optional, for live refresh
});
```

### Insert Vega Block (chart/visualization)
```javascript
await editor.blocks.insert('vega', {
  config: { chartType: 'bar', xAxis: { name: 'category' }, yAxis: { name: 'value' } },
  vegaSpec: {
    "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
    "data": { "values": [{"category": "A", "value": 28}, {"category": "B", "value": 55}] },
    "mark": "bar",
    "encoding": {
      "x": {"field": "category", "type": "nominal"},
      "y": {"field": "value", "type": "quantitative"}
    }
  },
  metadata: { displayName: 'Chart Title' }
});
```

---

## 22. READ REPORT BLOCKS

**Verified working.** Reads the current block types and count from the editor.

```javascript
(async () => {
  const editorHolder = document.getElementById('editorjs');
  const ng = window.ng;
  let reportComp = null;
  let el = editorHolder;
  while (el && !reportComp) {
    try { const c = ng.getComponent(el); if (c && c.editor) { reportComp = c; break; } } catch(e) {}
    el = el.parentElement;
  }
  if (reportComp && reportComp.editor) {
    const data = await reportComp.editor.save();
    return 'blocks=' + data.blocks.length + ': ' + data.blocks.map(b => b.type).join(', ');
  }
  return 'no editor';
})()
```

---

## 23. EDIT HTML BLOCK

**Verified working.** HTML blocks toggle between preview and edit mode. Click preview to enter edit mode (shows textarea), blur to exit.

```javascript
// Enter edit mode - click the HTML preview
document.querySelector('.html-block-preview')?.click();
// Wait 500ms

// Modify HTML content in the textarea
const textarea = document.querySelector('.html-block-editor');
textarea.value = '<h2>Updated HTML</h2><p>New content here</p>';
textarea.dispatchEvent(new Event('input', { bubbles: true }));

// Exit edit mode - blur the textarea
textarea.blur();
```

---

## 24. COPY PRISM TO REPORT (via clipboard paste simulation)

The prism viewer's "Copy to report" button (`[data-qa="prism-copy-report-button"]`) copies a `PRISM_BLOCK:` prefixed JSON string to clipboard. When pasted into the report editor, the paste handler detects it and inserts a prism block.

**If prism viewer is open:**
```javascript
// Click the copy-to-report button on the prism viewer
document.querySelector('[data-qa="prism-copy-report-button"]')?.click();
// Wait 500ms — data is now on clipboard
// Then click on the report editor and paste (Ctrl+V)
```

**Programmatic alternative (no clipboard needed):**
Use the Insert Prism Block method from section 21 above.

---

## 25. COPY VISUALIZATION TO REPORT

The floating visualization's "Copy to report" button (`[data-qa="viz-copy-report-button"]`) copies a `VEGA_BLOCK:` prefixed JSON string to clipboard.

**If visualization is open:**
```javascript
// Click the copy-to-report button on the visualization
document.querySelector('[data-qa="viz-copy-report-button"]')?.click();
// Wait 500ms — data is now on clipboard
// Then click on the report editor and paste (Ctrl+V)
```

**Programmatic alternative (no clipboard needed):**
Use the Insert Vega Block method from section 21 above.

---

## 26. GET REPORTS LIST

**Verified working.** Returns all report items from the explorer.

```javascript
(() => {
  const items = document.querySelectorAll('[data-qa="explorer-report-item"]');
  const reports = Array.from(items).map((item, i) => ({
    index: i,
    name: item.querySelector('.report-name')?.textContent?.trim(),
    isSelected: item.classList.contains('selected')
  }));
  return JSON.stringify(reports);
})()
```

---

## 27. SCROLL REPORTS EXPLORER INTO VIEW

```javascript
document.querySelector('.reports-explorer')?.scrollIntoView({ behavior: 'smooth', block: 'center' });
```

---

## REPORT EDITOR ACTION BUTTONS

| Selector | Action |
|----------|--------|
| `.report-title-bar .delete-btn` | Delete current report |
| `.report-title-bar .refresh-btn` | Refresh prism/vega data |
| `.report-title-bar .download-btn` | Download as PDF |
| `.text-btn.new-btn` | Create new report (from editor) |
| `.report-title-input` | Report title input field |

## REPORT BLOCK TYPES

| Type | Description | Data Shape |
|------|-------------|------------|
| `paragraph` | Text paragraph | `{ text: string }` |
| `header` | Header (H1-H4) | `{ text: string, level: 1-4 }` |
| `list` | Bulleted/numbered list | `{ style: 'unordered'|'ordered', items: string[] }` |
| `table` | Editable table | `{ content: string[][] }` |
| `html` | Raw HTML (click to edit) | `{ html: string }` |
| `prism` | Prism data table | `{ columns, data, metadata, queryDef? }` |
| `vega` | Vega-Lite chart | `{ config, vegaSpec, metadata }` |
| `perspective` | Perspective viewer | `{ config, metadata }` |

## REPORT COPY-TO-REPORT SELECTORS

| Selector | Source |
|----------|--------|
| `[data-qa="prism-copy-report-button"]` | Prism viewer → copies PRISM_BLOCK: to clipboard |
| `[data-qa="viz-copy-report-button"]` | Floating visualization → copies VEGA_BLOCK: to clipboard |

## REPORT EXPLORER SELECTORS

| Selector | Element |
|----------|---------|
| `[data-qa="explorer-reports-add"]` | Add new report button |
| `[data-qa="explorer-reports-expand"]` | Expand/collapse reports list |
| `[data-qa="explorer-report-item"]` | Report item in list |
| `.report-name` | Report display name (inside item) |
| `.reports-explorer` | Reports explorer container |

## TIMING REFERENCE (Reports, verified on localhost)

| Step | Duration |
|------|----------|
| Create Report | ~3s |
| Rename Report | ~2s (includes auto-save) |
| Delete Report (with confirm) | ~3s |
| Insert Block (any type) | ~500ms |
| Load Report Content | ~2s |
| HTML Block toggle edit/preview | ~500ms |

## TEST DATA FILES

Located in: `C:\work\asky\e2eTests\asky-e2e-framework\playwright-runner\tests\data\`

| File | Description |
|------|-------------|
| `Ads_Ga4_SpendRev_Apparel.xlsx` | 3 tables: GA4, Google Ads, Facebook |
| `t1.csv` | Simple CSV with clicks/conversions |
| `BQ_nationalmenopauseshow2024.xlsx` | BigQuery data |
| `Clay_People_1000.xlsx` | Clay CRM data |
| `Campaign_Naming_Convention.txt` | Campaign naming doc |

---

## 28. SPREADJS GRID INTERACTION

The data grid uses GrapeCity SpreadJS. Row 0 = header row. Data rows start at row 1.

### Get SpreadJS Component
```javascript
(() => {
  const spreadEl = document.querySelector('gc-spread-sheets');
  if (!spreadEl) return null;
  let el = spreadEl;
  while (el) {
    try { const c = window.ng.getComponent(el); if (c && c.spread) return c; } catch(e) {}
    el = el.parentElement;
  }
  return null;
})()
```

### Read All Grid Data
```javascript
(() => {
  const spreadEl = document.querySelector('gc-spread-sheets');
  let el = spreadEl, comp = null;
  while (el) {
    try { const c = window.ng.getComponent(el); if (c && c.spread) { comp = c; break; } } catch(e) {}
    el = el.parentElement;
  }
  if (!comp) return 'No SpreadJS';
  const sheet = comp.spread.getActiveSheet();
  const rows = [];
  for (let r = 0; r < sheet.getRowCount(); r++) {
    const row = [];
    for (let c = 0; c < sheet.getColumnCount(); c++) row.push(sheet.getValue(r, c));
    rows.push(row);
  }
  return JSON.stringify(rows);
})()
```

### Edit a Cell (Materialized Tables Only)
```javascript
sheet.setValue(1, 0, 'NEW_VALUE'); // row 1, col 0
```
Cell edits are debounced on both frontend and backend. Wait several seconds after editing before checking logs or reloading.

### Read a Single Cell
```javascript
sheet.getValue(1, 0) // row, col
```

---

## 29. NAVIGATING TO TABLES IN THE EXPLORER

Clicking items in the explorer tree is unreliable with coordinates — the tree is dense. Use these methods in order of reliability:

**Method 1 — Chat link (most reliable):** Click the `>` arrow next to a table name in the chat messages. This reliably opens the data grid viewer.

**Method 2 — `find` tool:** `find "TABLE_NAME in AI Generated Tables"` then `double_click` the ref. The same name may appear in multiple locations (chat, explorer, prisms) — try different refs if the first doesn't open the viewer.

**Method 3 — JS click:**
```javascript
(() => {
  const items = document.querySelectorAll('.excel-file-item .table-name, .folder-table-name');
  for (const item of items) {
    if (item.textContent.trim().includes('TABLE_NAME')) {
      item.closest('[data-qa="table-list-item"], .folder-table-item')?.click();
      return 'clicked';
    }
  }
  return 'not found';
})()
```

**Verify table loaded:** `document.querySelector('gc-spread-sheets') ? 'Grid loaded' : 'No grid'`

The "EDITABLE" badge appears for materialized tables.

---

## 30. TESTING METHODOLOGY

### Diagnose Before Fixing

**Never guess at code changes.** Always:
1. Note log position before an action
2. Perform ONE action in the browser
3. Wait for async processes to complete
4. Read only the NEW log lines
5. Diagnose from the actual error, only then fix

### Test Variations Systematically

For any feature, always test this escalating sequence:

| Scenario | Why |
|----------|-----|
| Single action, check result | Does it work at all? |
| Page reload | Does the change survive in-session? |
| Backend restart | Does the change survive across restarts? |
| Rapid multiple actions | Does batching/debouncing work? |
| Invalid input / wrong target | Does error handling work? |

### UI Clicks vs JavaScript — Use Both

JS (`setValue`, `click()`) is precise but can bypass UI event handlers that real users trigger. Always verify with UI clicks too:
- Use `find` tool + `left_click` ref for buttons and tree items (coordinates are unreliable on dense UIs)
- Use `double_click` + `type` + `key Enter` to simulate real user cell editing
- Use JS `setValue()` only when you need precise, repeatable cell edits
- After navigation or reload, `find` refs go stale — always re-find

### Service Restart Testing

When testing persistence or state management:
1. Make a change and verify it took effect
2. Reload the page — verify the change survives (tests in-memory/session state)
3. Kill and restart the backend — verify again (tests durable storage)
4. Each step that fails reveals a different layer of the bug
