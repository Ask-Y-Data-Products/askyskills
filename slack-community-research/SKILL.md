---
name: slack-community-research
description: "Extract and export topical discussions from a Slack community into structured CSV/database tables. Uses Chrome MCP browser tools to authenticate via the Slack web app, then calls Slack APIs directly from the page context to search messages, fetch thread replies, and export as relational CSV tables. Triggers on: slack research, slack community data, extract slack messages, slack discussions to csv, slack to database, scrape slack, export slack conversations, slack community analysis."
---

# Slack Community Research — Export Discussions to DB Tables

## Overview

This skill extracts messages from a **public Slack community** the user is a member of, filters by topic keywords, fetches full thread replies, and exports everything as **4 relational CSV tables** ready for database import.

## What the User Must Provide

1. **Slack workspace URL** — e.g. `https://app.slack.com/client/T031USB3H/search` (the `/client/{TEAM_ID}/search` part is what matters; the `TEAM_ID` like `T031USB3H` is extracted from it)
2. **Topics / keywords to search** — e.g. "AI, ChatGPT, LLM, prompt engineering, OpenAI" or any domain-specific terms

## Output: 4 CSV Tables

| File | Description |
|------|-------------|
| `messages.csv` | One row per unique message (deduplicated across keywords) |
| `message_keywords.csv` | Many-to-many: which search keywords matched each message |
| `threads.csv` | Thread parents that have replies |
| `thread_replies.csv` | Individual reply messages within threads |

Plus `schema.sql` with CREATE TABLE statements, indexes, and example queries.

### Schema

```
messages (1) ←── (N) message_keywords
    ↑
    │ message_id (FK)
    │
threads (1) ←── (N) thread_replies
```

**messages** columns: `message_id, ts, date, channel, channel_id, user, text, permalink`
**message_keywords** columns: `id, message_id, keyword`
**threads** columns: `thread_id, message_id, channel, query, permalink, date, num_replies, parent_user, parent_text`
**thread_replies** columns: `reply_id, thread_id, user, text, ts, date`

---

## Prerequisites

- User must be **logged into the Slack workspace** in Chrome
- **Claude in Chrome** MCP extension must be connected
- User must be a **member** of the workspace (not just visiting)

---

## Step-by-Step Procedure

### Step 1: Navigate to Slack Search

Open the workspace search page in Chrome:

```
URL pattern: https://app.slack.com/client/{TEAM_ID}/search
```

Extract `TEAM_ID` from the URL the user provides. Navigate using `mcp__Claude_in_Chrome__navigate`.

Wait 3 seconds for Slack to load, then take a screenshot to confirm the search page is visible.

### Step 2: Extract the Workspace API Token

The Slack web client stores per-workspace tokens in `localStorage` under `localConfig_v2`. Extract the token for the target workspace:

```javascript
(async () => {
  const TEAM_ID = '__TEAM_ID__'; // Replace with actual team ID
  const config = JSON.parse(localStorage.getItem('localConfig_v2'));
  const token = config?.teams?.[TEAM_ID]?.token;
  return token ? 'Token found, length: ' + token.length : 'ERROR: No token for ' + TEAM_ID;
})();
```

**IMPORTANT:** The token is a `xoxc-...` string. It only works when sent from the same browser session (cookies provide the session auth). All API calls MUST go through `mcp__Claude_in_Chrome__javascript_tool` — you cannot use `fetch` from outside the browser.

### Step 3: Verify API Access with a Test Search

Run a test search to confirm the token + cookies work for the target workspace:

```javascript
(async () => {
  const TEAM_ID = '__TEAM_ID__';
  const config = JSON.parse(localStorage.getItem('localConfig_v2'));
  const token = config?.teams?.[TEAM_ID]?.token;

  const formData = new FormData();
  formData.append('token', token);
  formData.append('query', 'test');
  formData.append('count', '1');
  formData.append('page', '1');

  const resp = await fetch('/api/search.messages?slack_route=' + TEAM_ID, {
    method: 'POST', body: formData, credentials: 'include'
  });
  const data = await resp.json();
  return JSON.stringify({ ok: data.ok, total: data.messages?.total, error: data.error });
})();
```

The critical parameter is `?slack_route={TEAM_ID}` — without it, the API may return results from the user's primary workspace instead of the target community.

### Step 4: Get Result Counts for All Keywords

Before fetching full data, get counts so the user can see scope:

```javascript
(async () => {
  const TEAM_ID = '__TEAM_ID__';
  const config = JSON.parse(localStorage.getItem('localConfig_v2'));
  const token = config?.teams?.[TEAM_ID]?.token;

  async function getTotal(query) {
    const formData = new FormData();
    formData.append('token', token);
    formData.append('query', query);
    formData.append('count', '1');
    formData.append('page', '1');
    const resp = await fetch('/api/search.messages?slack_route=' + TEAM_ID, {
      method: 'POST', body: formData, credentials: 'include'
    });
    const data = await resp.json();
    return data.ok ? data.messages?.total : -1;
  }

  const keywords = __KEYWORDS_ARRAY__; // Replace with actual array
  const results = {};
  for (const kw of keywords) {
    results[kw] = await getTotal(kw);
    await new Promise(r => setTimeout(r, 200));
  }
  return JSON.stringify(results, null, 2);
})();
```

Present the counts to the user in a table before proceeding. Ask if they want to adjust keywords.

### Step 5: Fetch All Messages (Batched)

Fetch messages in **batches of 3 keywords at a time** to avoid browser timeouts (45-second limit per JS execution). Each keyword paginates up to 800 results (8 pages of 100).

```javascript
(async () => {
  const TEAM_ID = '__TEAM_ID__';
  const config = JSON.parse(localStorage.getItem('localConfig_v2'));
  const token = config?.teams?.[TEAM_ID]?.token;

  async function searchAll(query, maxPages = 8) {
    const msgs = [];
    let page = 1;
    while (page <= maxPages) {
      const formData = new FormData();
      formData.append('token', token);
      formData.append('query', query);
      formData.append('count', '100');
      formData.append('page', String(page));
      const resp = await fetch('/api/search.messages?slack_route=' + TEAM_ID, {
        method: 'POST', body: formData, credentials: 'include'
      });
      const data = await resp.json();
      if (!data.ok || !data.messages?.matches?.length) break;
      for (const msg of data.messages.matches) {
        msgs.push({
          query, channel: msg.channel?.name, channel_id: msg.channel?.id,
          user: msg.username, text: msg.text, ts: msg.ts,
          permalink: msg.permalink,
          date: new Date(parseFloat(msg.ts) * 1000).toISOString().split('T')[0]
        });
      }
      if (msgs.length >= (data.messages?.total || 0)) break;
      page++;
      await new Promise(r => setTimeout(r, 300));
    }
    return msgs;
  }

  // === BATCH: replace __BATCH_KEYWORDS__ with 3 keywords per call ===
  const batch = [];
  for (const kw of __BATCH_KEYWORDS__) {
    batch.push(...await searchAll(kw));
  }
  window.__batch_N__ = batch;  // Store with incrementing batch number
  return 'Batch done: ' + batch.length + ' messages';
})();
```

**Batching rules:**
- Max **3 keywords per JS execution** to stay under the 45s timeout
- Keywords with 500+ results may need their own solo batch
- Store each batch in `window.__batch1`, `window.__batch2`, etc.
- Add 300ms delay between pages, 500ms between keywords

### Step 6: Deduplicate Messages

After all batches complete, combine and deduplicate:

```javascript
(() => {
  // Combine all batches
  const all = [
    ...(window.__batch1 || []), ...(window.__batch2 || []),
    ...(window.__batch3 || []), ...(window.__batch4 || [])
    // ... add all batch variables
  ];

  // Deduplicate by ts + channel_id
  const seen = new Set();
  const unique = [];
  for (const msg of all) {
    const key = msg.ts + '_' + msg.channel_id;
    if (!seen.has(key)) {
      seen.add(key);
      unique.push(msg);
    }
  }
  unique.sort((a, b) => parseFloat(b.ts) - parseFloat(a.ts)); // newest first

  window.__allMessages = unique;
  window.__allRaw = all; // Keep raw for keyword mapping
  return 'Total raw: ' + all.length + ', Unique: ' + unique.length;
})();
```

### Step 7: Fetch Thread Replies

Fetch thread replies in batches of **80 messages per JS execution**, with 250ms delay between calls. Sample across the full message set:

```javascript
(async () => {
  const TEAM_ID = '__TEAM_ID__';
  const config = JSON.parse(localStorage.getItem('localConfig_v2'));
  const token = config?.teams?.[TEAM_ID]?.token;

  async function getReplies(channelId, ts) {
    const formData = new FormData();
    formData.append('token', token);
    formData.append('channel', channelId);
    formData.append('ts', ts);
    formData.append('limit', '200');
    const resp = await fetch('/api/conversations.replies?slack_route=' + TEAM_ID, {
      method: 'POST', body: formData, credentials: 'include'
    });
    return await resp.json();
  }

  if (!window.__threads) window.__threads = [];
  const fetched = new Set(window.__threads.map(t => t.permalink));
  const remaining = window.__allMessages.filter(m => !fetched.has(m.permalink));

  // Take a slice — adjust start/end indices per batch
  const batch = remaining.slice(__START__, __END__);
  let added = 0;

  for (const msg of batch) {
    try {
      const result = await getReplies(msg.channel_id, msg.ts);
      if (result.ok && result.messages && result.messages.length > 1) {
        added++;
        window.__threads.push({
          channel: msg.channel, query: msg.query, permalink: msg.permalink,
          date: msg.date, num_replies: result.messages.length - 1,
          messages: result.messages.map(r => ({
            user: r.user, text: r.text, ts: r.ts
          }))
        });
      }
    } catch(e) {}
    await new Promise(r => setTimeout(r, 250));
  }
  return '+' + added + ' threads. Total: ' + window.__threads.length;
})();
```

**Thread fetching strategy:**
- First batch: messages 0–80 (highest relevance from search ranking)
- Next batches: sample every Nth message to cover breadth
- Continue until diminishing returns (batches return 0 new threads)
- Most messages in search results are replies within threads (not thread parents), so expect ~15–25% hit rate

### Step 8: Build Relational Tables

Transform the raw data into 4 normalized tables:

```javascript
(() => {
  // TABLE 1: messages
  const messages = window.__allMessages.map((m, idx) => ({
    message_id: idx + 1,
    ts: m.ts,
    date: m.date,
    channel: m.channel,
    channel_id: m.channel_id,
    user: m.user,
    text: m.text,
    permalink: m.permalink
  }));

  // TABLE 2: message_keywords (many-to-many)
  const tsToId = {};
  for (const m of messages) tsToId[m.ts + '_' + m.channel_id] = m.message_id;

  const kwPairs = new Set();
  const messageKeywords = [];
  let mkId = 0;
  for (const raw of window.__allRaw) {
    const key = raw.ts + '_' + raw.channel_id;
    const mid = tsToId[key];
    if (mid) {
      const pairKey = mid + '_' + raw.query;
      if (!kwPairs.has(pairKey)) {
        kwPairs.add(pairKey);
        messageKeywords.push({ id: ++mkId, message_id: mid, keyword: raw.query });
      }
    }
  }

  // TABLE 3: threads
  const threads = window.__threads.map((t, idx) => {
    const parentMsg = t.messages[0];
    const mid = messages.find(m => m.permalink === t.permalink)?.message_id || null;
    return {
      thread_id: idx + 1, message_id: mid, channel: t.channel,
      query: t.query, permalink: t.permalink, date: t.date,
      num_replies: t.num_replies,
      parent_user: parentMsg?.user, parent_text: parentMsg?.text
    };
  });

  // TABLE 4: thread_replies
  const threadReplies = [];
  let replyId = 0;
  for (const t of window.__threads) {
    const tid = threads.find(th => th.permalink === t.permalink)?.thread_id;
    if (!tid) continue;
    for (let i = 1; i < t.messages.length; i++) {
      const r = t.messages[i];
      threadReplies.push({
        reply_id: ++replyId, thread_id: tid, user: r.user, text: r.text,
        ts: r.ts, date: new Date(parseFloat(r.ts) * 1000).toISOString().split('T')[0]
      });
    }
  }

  window.__tables = { messages, messageKeywords, threads, threadReplies };
  return JSON.stringify({
    messages: messages.length,
    message_keywords: messageKeywords.length,
    threads: threads.length,
    thread_replies: threadReplies.length
  });
})();
```

### Step 9: Export CSVs

Download each table as a CSV file. Run this once per table (or all at once if small enough):

```javascript
(() => {
  function toCsv(rows) {
    if (!rows.length) return '';
    const headers = Object.keys(rows[0]);
    const escape = (val) => {
      if (val === null || val === undefined) return '';
      const s = String(val);
      if (s.includes(',') || s.includes('"') || s.includes('\n') || s.includes('\r')) {
        return '"' + s.replace(/"/g, '""') + '"';
      }
      return s;
    };
    return [headers.join(','), ...rows.map(row =>
      headers.map(h => escape(row[h])).join(',')
    )].join('\n');
  }

  function download(content, filename) {
    const blob = new Blob([content], { type: 'text/csv;charset=utf-8;' });
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = filename;
    a.click();
  }

  const t = window.__tables;
  download(toCsv(t.messages), 'messages.csv');
  download(toCsv(t.messageKeywords), 'message_keywords.csv');
  download(toCsv(t.threads), 'threads.csv');
  download(toCsv(t.threadReplies), 'thread_replies.csv');

  return 'Downloaded 4 CSVs: messages(' + t.messages.length +
    '), message_keywords(' + t.messageKeywords.length +
    '), threads(' + t.threads.length +
    '), thread_replies(' + t.threadReplies.length + ')';
})();
```

### Step 10: Export SQL Schema

```javascript
(() => {
  const sql = `-- Slack Community Research Export
-- Workspace: __WORKSPACE_NAME__
-- Generated: ${new Date().toISOString().split('T')[0]}
-- Keywords: __KEYWORDS_LIST__

CREATE TABLE messages (
    message_id    INTEGER PRIMARY KEY,
    ts            TEXT NOT NULL,
    date          DATE NOT NULL,
    channel       TEXT NOT NULL,
    channel_id    TEXT NOT NULL,
    "user"        TEXT,
    text          TEXT,
    permalink     TEXT
);

CREATE TABLE message_keywords (
    id            INTEGER PRIMARY KEY,
    message_id    INTEGER NOT NULL REFERENCES messages(message_id),
    keyword       TEXT NOT NULL
);

CREATE TABLE threads (
    thread_id     INTEGER PRIMARY KEY,
    message_id    INTEGER REFERENCES messages(message_id),
    channel       TEXT NOT NULL,
    query         TEXT,
    permalink     TEXT,
    date          DATE,
    num_replies   INTEGER,
    parent_user   TEXT,
    parent_text   TEXT
);

CREATE TABLE thread_replies (
    reply_id      INTEGER PRIMARY KEY,
    thread_id     INTEGER NOT NULL REFERENCES threads(thread_id),
    "user"        TEXT,
    text          TEXT,
    ts            TEXT,
    date          DATE
);

CREATE INDEX idx_messages_channel ON messages(channel);
CREATE INDEX idx_messages_date ON messages(date);
CREATE INDEX idx_mk_keyword ON message_keywords(keyword);
CREATE INDEX idx_mk_message ON message_keywords(message_id);
CREATE INDEX idx_tr_thread ON thread_replies(thread_id);
`;
  const blob = new Blob([sql], { type: 'text/sql' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'schema.sql';
  a.click();
  return 'schema.sql downloaded';
})();
```

### Step 11: Move Files to a Dedicated Folder

After all downloads complete, move them from Downloads to a named folder:

```bash
mkdir -p "C:/Users/AvigadOron/Downloads/__FOLDER_NAME__"
mv "C:/Users/AvigadOron/Downloads/messages.csv" \
   "C:/Users/AvigadOron/Downloads/message_keywords.csv" \
   "C:/Users/AvigadOron/Downloads/threads.csv" \
   "C:/Users/AvigadOron/Downloads/thread_replies.csv" \
   "C:/Users/AvigadOron/Downloads/schema.sql" \
   "C:/Users/AvigadOron/Downloads/__FOLDER_NAME__/"
```

---

## Critical Implementation Notes

### Slack Route Parameter
Every API call MUST include `?slack_route={TEAM_ID}` in the URL. Without it, the API returns results from the user's **primary** workspace, not the target community.

### Browser Context Required
All `fetch()` calls run inside `mcp__Claude_in_Chrome__javascript_tool`. The browser session cookies handle authentication — the `xoxc-` token alone is not enough. Never try to call these APIs from Bash or external tools.

### Timeout Management
The Chrome MCP JS execution has a **45-second timeout**. To avoid it:
- Max **3 keyword searches** per JS call
- Max **80 thread-reply fetches** per JS call
- Add `await new Promise(r => setTimeout(r, 250))` between API calls
- Keywords with 500+ results may need a solo batch

### Rate Limiting
Slack enforces rate limits. The 200–300ms delays between calls are sufficient for normal use. If you see `429` errors or `ratelimited` in responses, increase delays to 500ms+.

### Data Persistence
All data is stored in `window.__` variables. These survive as long as the tab stays on the same origin (`app.slack.com`). They are **lost** if:
- The tab navigates to a different domain
- The page is refreshed
- The tab is closed

If data is lost mid-process, re-run the search batches (they are idempotent).

### Page Navigation Pitfall
Clicking on search results in the Slack UI sometimes navigates away from the search page (to a channel or DM). Always use the **back button** or re-navigate to the search URL if this happens. Prefer using the API approach (Step 5) over clicking through the UI.

### Deduplication Logic
Messages are deduplicated by `ts + channel_id`. A single message may appear in results for multiple keywords — this is captured in the `message_keywords` junction table, while `messages` has exactly one row per unique message.

