---
name: asky-e2e
description: Automate Asky data analytics platform UI testing using Claude Chrome extension (MCP browser tools). Covers login, project management, file upload, chat interaction, explorer navigation, table data reading, and assertions. Use when testing the Asky platform via browser automation instead of Playwright.
trigger: when the user wants to test or automate the Asky platform UI, prism, or interact with asky app, run tests on asky, or automate browser actions on the stage.ask-y.ai site or when the user says, test this prism flow
---

# Asky E2E Browser Automation Skill

This skill provides **verified, tested** selectors and step-by-step procedures for automating the Asky data analytics platform using Claude Chrome extension MCP tools.

## PREREQUISITES

1. **Get tab context**: Call `tabs_context_mcp` with `createIfEmpty: true` before any action.
2. **Start WebSocket file bridge** (for file uploads): Run in Claude Code Bash:
   ```bash
   NODE_PATH=/c/work/asky/prism/prismfront/node_modules node /c/work/asky/e2eTests/asky-e2e-framework/playwright-runner/tests/data/ws_file_bridge.js &
   ```
3. **Default base URL**: Ask user — `https://stage.ask-y.ai`, `https://prism.ask-y.ai`, or `http://localhost:4200`.
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

## FULL E2E PATTERN

```
1. Start WS bridge: node ws_file_bridge.js &
2. Get tab context: tabs_context_mcp
3. Navigate: {baseUrl}/login
4. LOGIN (fill + click)
5. CREATE PROJECT
6. UPLOAD FILE via WebSocket bridge
7. Wait for loading to finish
8. SEND CHAT MESSAGE
9. WAIT FOR CHAT IDLE
10. CLICK ACCEPT ALL (if changes proposed)
11. EXPAND ALL EXPLORER ITEMS
12. READ EXCEL/PRISM TREES → verify results
13. DELETE PROJECT (cleanup)
```

## TIMING REFERENCE (stage.ask-y.ai, verified)

| Step | Duration |
|------|----------|
| Login (credentials pre-filled) | ~5s |
| Create Project | ~10s |
| Upload File (20KB xlsx via WS) | ~15s |
| Chat Message + Wait Idle | 10-300s |
| Accept All Changes | ~2s |
| Expand + Read Explorer | ~5s |
| Delete Project | ~15s |

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

## TEST DATA FILES

Located in: `C:\work\asky\e2eTests\asky-e2e-framework\playwright-runner\tests\data\`

| File | Description |
|------|-------------|
| `Ads_Ga4_SpendRev_Apparel.xlsx` | 3 tables: GA4, Google Ads, Facebook |
| `t1.csv` | Simple CSV with clicks/conversions |
| `BQ_nationalmenopauseshow2024.xlsx` | BigQuery data |
| `Clay_People_1000.xlsx` | Clay CRM data |
| `Campaign_Naming_Convention.txt` | Campaign naming doc |
