---
name: claude-code-lookout
description: "Build a floating desktop widget that keeps watch over every active Claude Code session (Working / Needs You / Done) and pops a real desktop toast notification the moment one needs attention, wired up via Claude Code hooks. Trigger when the user asks to build a session tracker, a lookout/watcher for Claude Code, a desktop widget/notifier for AI coding sessions, or a way to monitor multiple parallel Claude Code sessions without tabbing through them. Windows-focused (Electron); adaptable to macOS/Linux."
license: MIT
metadata:
  author: community
  version: "1.0.0"
---

# Claude Code Lookout

Builds a small always-on-top Electron widget that watches every Claude Code
session on the machine via hooks and shows: which sessions are Working, which
need the user's input right now, and which just finished — plus a toast
notification the moment a session needs attention.

This file is a build recipe distilled from actually building and iterating on
this tool. It includes several non-obvious facts about Claude Code's hook
system and a couple of real bugs that got fixed along the way. Follow it
rather than re-deriving the hook wiring from scratch — the "Key facts" section
below exists because the obvious first approach to several of these got it
wrong.

## What you're building

- An Electron app: one small floating window (the "sticky note") listing
  active sessions, plus a lightweight local HTTP server on `127.0.0.1` that
  Claude Code's hooks POST to.
- Hooks registered in `~/.claude/settings.json` (global — applies to every
  Claude Code session on the machine, in any project) that fire on session
  lifecycle events and forward a small JSON payload to that local server.
- No LLM/API calls anywhere in this loop. It's pure local automation — the
  widget doesn't consume tokens or add cost, only a few hook events' worth of
  local HTTP latency (sub-millisecond).

## Prerequisites

- Node.js + npm on PATH (check with `node --version`).
- Windows is what this recipe was built and tested on (the silent launcher is
  a `.vbs` file, and there's a Windows-specific note about spawning `.cmd`
  files if you ever add shell-outs). The Electron app and hook logic itself
  are cross-platform; only the autostart/launcher step needs adapting for
  macOS (a `.command` file or LaunchAgent) or Linux (a desktop entry).

## Step 1 — Scaffold the project

Ask the user where to put it, or default to a `claude-code-lookout/`
folder next to their other projects. Create this structure:

```
claude-code-lookout/
  package.json
  src/
    main.js
    preload.js
    index.html
    renderer.js
    toast.html
  start-widget.vbs
```

`package.json`:

```json
{
  "name": "claude-code-lookout",
  "version": "1.0.0",
  "description": "Floating desktop widget showing active Claude Code sessions",
  "main": "src/main.js",
  "scripts": {
    "start": "electron ."
  },
  "author": "",
  "license": "MIT",
  "private": true,
  "devDependencies": {
    "electron": "^32.0.0"
  }
}
```

## Step 2 — Write the source files

### `src/main.js`

The Electron main process: local HTTP server, in-memory session state (backed
by a JSON file so it survives restarts), the floating window, and toast
notifications.

```js
const { app, BrowserWindow, ipcMain, screen } = require('electron');
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = 51823; // pick any free local port; change if it conflicts

const gotLock = app.requestSingleInstanceLock();
if (!gotLock) {
  app.quit();
}
app.disableHardwareAcceleration();

let win = null;
let dataDir, sessionsFile, boundsFile;
let sessions = new Map(); // id -> session record

const ACTIVE_WINDOW_MS = 12 * 60 * 60 * 1000; // sessions "expire" from the list after 12h of inactivity
function isRecent(ts) {
  return !!ts && Date.now() - ts < ACTIVE_WINDOW_MS;
}

function loadSessions() {
  try {
    const raw = JSON.parse(fs.readFileSync(sessionsFile, 'utf8'));
    for (const rec of raw) {
      if (isRecent(rec.lastPromptAt || rec.startedAt)) sessions.set(rec.id, rec);
    }
  } catch {
    // no file yet, or unreadable -- start fresh
  }
}

let saveTimer = null;
function saveSessions() {
  clearTimeout(saveTimer);
  saveTimer = setTimeout(() => {
    try {
      fs.writeFileSync(sessionsFile, JSON.stringify([...sessions.values()], null, 2));
    } catch (e) {
      console.error('save failed', e);
    }
  }, 150);
}

function projectName(cwd) {
  if (!cwd) return 'Unknown project';
  return path.basename(cwd) || cwd;
}

const TRANSCRIPT_SCAN_CAP = 2 * 1024 * 1024; // only ever tail-read up to 2MB of a transcript

// Reads Claude Code's own naming signals out of the session transcript: the
// auto-generated "ai-title", and "custom-title" (written when the user
// renames the session from Claude Code's own session list/VS Code UI).
// Neither of these is exposed via the hook payload -- they only ever appear
// as line entries inside the transcript .jsonl file, so this tail-reads it.
// Both can change over a session's life, so this re-scans on every call
// rather than caching after the first hit -- it's a single bounded read and
// hook events are infrequent (a couple per turn), so the cost stays small.
function scanTranscriptTail(transcriptPath) {
  if (!transcriptPath) return {};
  try {
    const stat = fs.statSync(transcriptPath);
    const size = Math.min(stat.size, TRANSCRIPT_SCAN_CAP);
    const fd = fs.openSync(transcriptPath, 'r');
    const buf = Buffer.alloc(size);
    fs.readSync(fd, buf, 0, size, stat.size - size);
    fs.closeSync(fd);

    let aiTitle = null;
    let customTitle = null;
    for (const line of buf.toString('utf8').split('\n')) {
      if (!line || line[0] !== '{') continue;
      let obj;
      try {
        obj = JSON.parse(line);
      } catch {
        continue; // first line in the tail is often a truncated fragment
      }
      if (obj.type === 'ai-title' && obj.aiTitle) aiTitle = obj.aiTitle;
      else if (obj.type === 'custom-title') customTitle = obj.customTitle || null;
    }
    return { aiTitle, customTitle };
  } catch {
    return {};
  }
}

function refreshLabel(rec) {
  if (rec.transcriptPath) {
    const { aiTitle, customTitle } = scanTranscriptTail(rec.transcriptPath);
    if (aiTitle) rec.aiTitle = aiTitle;
    if (customTitle !== null) rec.claudeCustomTitle = customTitle;
  }
  rec.label = rec.customTitle || rec.claudeCustomTitle || rec.aiTitle || rec.title ||
    truncate(rec.firstPrompt, 60) || rec.project;
}

function visibleSessions() {
  return [...sessions.values()].filter((r) => !r.dismissed);
}

function pushUpdate() {
  if (win && !win.isDestroyed()) {
    win.webContents.send('sessions-updated', visibleSessions());
  }
}

const TOAST_W = 380, TOAST_H = 110, TOAST_MARGIN = 16;
const TOAST_MSG_CHARS = 90;
const TOAST_LIFETIME = { needs_action: 25000, done: 9000 };
let toastQueue = [];
let toastShowing = false;
let currentToastSessionId = null;
let currentToastWindow = null;

function notify(sessionId, type, title, message) {
  // A session's status can change again before its last toast finished its
  // full lifetime (e.g. it gets approved and finishes seconds after asking)
  // -- so drop any not-yet-shown toast for this same session rather than
  // stacking a second one behind it, and close the one on screen now
  // instead of leaving a stale "needs you" popup up for its full 25s.
  toastQueue = toastQueue.filter((t) => t.sessionId !== sessionId);
  toastQueue.push({ sessionId, type, title, message: truncate(message, TOAST_MSG_CHARS) });

  if (currentToastSessionId === sessionId && currentToastWindow && !currentToastWindow.isDestroyed()) {
    currentToastWindow.close();
  } else {
    processToastQueue();
  }
}

function processToastQueue() {
  if (toastShowing || toastQueue.length === 0) return;
  toastShowing = true;
  const { sessionId, type, title, message } = toastQueue.shift();
  currentToastSessionId = sessionId;

  const area = screen.getPrimaryDisplay().workArea;
  const toast = new BrowserWindow({
    x: area.x + area.width - TOAST_W - TOAST_MARGIN,
    y: area.y + area.height - TOAST_H - TOAST_MARGIN,
    width: TOAST_W,
    height: TOAST_H,
    frame: false,
    transparent: true,
    alwaysOnTop: true,
    resizable: false,
    skipTaskbar: true,
    focusable: false,
    hasShadow: false,
    webPreferences: { contextIsolation: true }
  });
  currentToastWindow = toast;
  toast.setAlwaysOnTop(true, 'floating');
  // Mouse events need to reach the close button, so no setIgnoreMouseEvents here.
  const q = new URLSearchParams({ type, title, message: message || '' });
  toast.loadFile(path.join(__dirname, 'toast.html'), { search: q.toString() });

  toast.once('ready-to-show', () => toast.showInactive());

  const closeTimer = setTimeout(() => {
    if (!toast.isDestroyed()) toast.close();
  }, TOAST_LIFETIME[type] || TOAST_LIFETIME.done);

  // Advance the queue whenever the window actually closes, whether that's
  // the timer above, the user clicking the close button, or this same
  // session's status changing again and superseding it early.
  toast.once('closed', () => {
    clearTimeout(closeTimer);
    toastShowing = false;
    currentToastSessionId = null;
    currentToastWindow = null;
    processToastQueue();
  });
}

function getOrCreate(id, cwd) {
  let rec = sessions.get(id);
  if (!rec) {
    rec = {
      id,
      cwd: cwd || '',
      project: projectName(cwd),
      title: null,
      transcriptPath: null,
      aiTitle: null,
      claudeCustomTitle: null,
      firstPrompt: null,
      customTitle: null,
      dismissed: false,
      status: 'idle',
      startedAt: Date.now(),
      lastEventAt: Date.now(),
      lastPromptAt: Date.now(),
      doneAt: null,
      notifiedDone: false,
      notifiedNeedsAction: false
    };
    sessions.set(id, rec);
  }
  return rec;
}

function truncate(str, n) {
  if (!str) return '';
  return str.length > n ? str.slice(0, n - 1) + '…' : str;
}

function handleHookEvent(payload) {
  const eventName = payload.hook_event_name || 'Unknown';
  const id = payload.session_id || payload.transcript_path || 'unknown-session';
  const cwd = payload.cwd;

  const rec = getOrCreate(id, cwd);
  if (cwd && !rec.cwd) {
    rec.cwd = cwd;
    rec.project = projectName(cwd);
  }
  if (payload.session_title) rec.title = payload.session_title;
  if (payload.transcript_path) rec.transcriptPath = payload.transcript_path;
  rec.lastEventAt = Date.now();

  // Apply all state changes first; any toast fires after, once, using the
  // freshly-resolved label -- avoids scanning the transcript more than once
  // per event regardless of which branch below runs.
  let pendingToast = null;

  switch (eventName) {
    case 'SessionStart':
      rec.status = 'idle';
      break;
    case 'UserPromptSubmit':
      rec.status = 'working';
      rec.lastPromptAt = Date.now();
      rec.dismissed = false;
      // Cap what we keep -- prompts can be arbitrarily long (pasted documents,
      // etc.) and this is only ever used as a short fallback label.
      if (!rec.firstPrompt && payload.prompt) rec.firstPrompt = payload.prompt.slice(0, 300);
      rec.notifiedNeedsAction = false;
      rec.notifiedDone = false;
      break;
    case 'Notification':
      rec.status = 'needs_action';
      if (!rec.notifiedNeedsAction) {
        rec.notifiedNeedsAction = true;
        pendingToast = { suffix: 'needs you', message: payload.message || 'Claude Code is waiting on you.' };
      }
      break;
    case 'PreToolUse':
      // Matcher is scoped to these specific tools in settings.json, but check
      // the tool name anyway in case that scope ever widens. Notification
      // does NOT fire for these -- they need their own hook (see Step 4).
      if (payload.tool_name === 'AskUserQuestion' || payload.tool_name === 'ExitPlanMode') {
        rec.status = 'needs_action';
        if (!rec.notifiedNeedsAction) {
          rec.notifiedNeedsAction = true;
          const msg = payload.tool_name === 'ExitPlanMode'
            ? 'Claude has a plan ready for your review.'
            : 'Claude has a question for you.';
          pendingToast = { suffix: 'needs you', message: msg };
        }
      }
      break;
    case 'PostToolUse':
      if (payload.tool_name === 'AskUserQuestion' || payload.tool_name === 'ExitPlanMode') {
        rec.status = 'working';
        rec.notifiedNeedsAction = false;
      }
      break;
    case 'Stop':
      rec.status = 'done';
      rec.doneAt = Date.now();
      if (!rec.notifiedDone) {
        rec.notifiedDone = true;
        pendingToast = { suffix: 'finished', message: payload.last_assistant_message || 'Session complete.' };
      }
      break;
    case 'SessionEnd':
      if (rec.status !== 'done') {
        rec.status = 'done';
        rec.doneAt = Date.now();
      }
      break;
    default:
      break;
  }

  refreshLabel(rec);
  if (pendingToast) {
    const type = pendingToast.suffix === 'finished' ? 'done' : 'needs_action';
    notify(rec.id, type, `${rec.label} ${pendingToast.suffix}`, pendingToast.message);
  }

  saveSessions();
  pushUpdate();
}

function startServer() {
  const server = http.createServer((req, res) => {
    if (req.method !== 'POST' || req.url !== '/hook') {
      res.writeHead(404);
      res.end();
      return;
    }
    let body = '';
    req.on('data', (c) => (body += c));
    req.on('end', () => {
      res.writeHead(200);
      res.end('ok');
      try {
        const payload = body ? JSON.parse(body) : {};
        handleHookEvent(payload);
      } catch (e) {
        console.error('bad hook payload', e);
      }
    });
  });
  server.on('error', (e) => {
    console.error('widget HTTP server failed to start (already running?)', e);
  });
  server.listen(PORT, '127.0.0.1');
}

function loadBounds() {
  try {
    return JSON.parse(fs.readFileSync(boundsFile, 'utf8'));
  } catch {
    const area = screen.getPrimaryDisplay().workArea;
    return { x: area.x + area.width - 300, y: area.y + 24, width: 280, height: 360 };
  }
}

function saveBounds() {
  if (!win || win.isDestroyed()) return;
  try {
    fs.writeFileSync(boundsFile, JSON.stringify(win.getBounds()));
  } catch {
    // ignore
  }
}

function createWindow() {
  const bounds = loadBounds();
  win = new BrowserWindow({
    ...bounds,
    minWidth: 220,
    minHeight: 180,
    frame: false,
    transparent: true,
    alwaysOnTop: true,
    resizable: true,
    fullscreenable: false,
    maximizable: false,
    hasShadow: false,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  });
  win.setAlwaysOnTop(true, 'floating');
  win.setVisibleOnAllWorkspaces(true, { visibleOnFullScreen: true });
  win.loadFile(path.join(__dirname, 'index.html'));

  win.on('move', saveBounds);
  win.on('resize', saveBounds);
}

app.whenReady().then(() => {
  app.setAppUserModelId('com.claudecodelookout.app');
  dataDir = app.getPath('userData');
  sessionsFile = path.join(dataDir, 'sessions.json');
  boundsFile = path.join(dataDir, 'bounds.json');

  loadSessions();
  createWindow();
  startServer();

  // Sweep stale "working" sessions to "idle" if nothing happened for 10+ minutes
  // and no Stop event ever arrived (e.g. Claude Code was closed uncleanly).
  // Also drop sessions once they fall outside the active window, and unstick
  // any session that's been in "needs_action" for over an hour (a dropped
  // hook or crashed session, not a real pending question).
  setInterval(() => {
    const now = Date.now();
    let changed = false;
    for (const [id, rec] of sessions) {
      if (!isRecent(rec.lastPromptAt || rec.startedAt)) {
        sessions.delete(id);
        changed = true;
        continue;
      }
      if (rec.status === 'working' && now - rec.lastEventAt > 10 * 60 * 1000) {
        rec.status = 'idle';
        changed = true;
      }
      if (rec.status === 'needs_action' && now - rec.lastEventAt > 60 * 60 * 1000) {
        rec.status = 'idle';
        changed = true;
      }
    }
    if (changed) {
      saveSessions();
      pushUpdate();
    }
  }, 60 * 1000);
});

ipcMain.handle('get-sessions', () => visibleSessions());
ipcMain.on('minimize-window', () => win && win.minimize());
ipcMain.on('clear-done', () => {
  for (const [id, rec] of sessions) {
    if (rec.status === 'done') sessions.delete(id);
  }
  saveSessions();
  pushUpdate();
});
ipcMain.on('dismiss-session', (_e, id) => {
  const rec = sessions.get(id);
  if (rec) {
    rec.dismissed = true;
    saveSessions();
    pushUpdate();
  }
});
ipcMain.on('rename-session', (_e, { id, title }) => {
  const rec = sessions.get(id);
  if (rec) {
    rec.customTitle = title.trim() ? title.trim() : null;
    refreshLabel(rec);
    saveSessions();
    pushUpdate();
  }
});

app.on('window-all-closed', () => {
  app.quit();
});

app.on('second-instance', () => {
  if (win) {
    if (win.isMinimized()) win.restore();
    win.focus();
  }
});
```

### `src/preload.js`

```js
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('widgetAPI', {
  getSessions: () => ipcRenderer.invoke('get-sessions'),
  onUpdate: (cb) => ipcRenderer.on('sessions-updated', (_e, sessions) => cb(sessions)),
  minimize: () => ipcRenderer.send('minimize-window'),
  clearDone: () => ipcRenderer.send('clear-done'),
  dismiss: (id) => ipcRenderer.send('dismiss-session', id),
  rename: (id, title) => ipcRenderer.send('rename-session', { id, title })
});
```

### `src/index.html`

A dark navy/gold theme by default — see Customization at the bottom for how
to reskin it with a different palette or logo.

```html
<!doctype html>
<html>
<head>
<meta charset="utf-8" />
<title>Claude Sessions</title>
<style>
  :root {
    --navy: #1A1F2B;
    --navy-deep: #12151E;
    --gold: #CE9E3A;
    --cream: #F1EDE3;
    --surface: #232939;
    --text-secondary: rgba(241, 237, 227, 0.68);
    --text-tertiary: rgba(241, 237, 227, 0.42);
    --border: rgba(241, 237, 227, 0.12);
    --success: #4B8A63;
    --danger: #B4472F;
    --info: #4A6FA5;
    color-scheme: dark;
  }
  * { box-sizing: border-box; }
  html, body {
    margin: 0; padding: 0;
    background: transparent;
    font-family: -apple-system, "Segoe UI", system-ui, sans-serif;
    overflow: hidden;
    user-select: none;
  }
  #titlebar {
    -webkit-app-region: drag;
    height: 34px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 8px 0 10px;
    font-size: 11px;
    font-weight: 700;
    letter-spacing: 0.3px;
    text-transform: uppercase;
    color: var(--cream);
  }
  #titlebar .brand { display: flex; align-items: center; gap: 8px; }
  #titlebar .dot {
    width: 5px; height: 5px; border-radius: 50%;
    background: var(--gold); display: inline-block;
    box-shadow: 0 0 4px var(--gold);
    animation: dot-blink 1.8s ease-in-out infinite;
  }
  @keyframes dot-blink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.3; }
  }
  #titlebar .right { -webkit-app-region: no-drag; display: flex; gap: 6px; }
  #titlebar button {
    -webkit-app-region: no-drag;
    border: none; background: rgba(241,237,227,0.08); color: var(--text-secondary);
    width: 18px; height: 18px; border-radius: 4px; cursor: pointer;
    font-size: 12px; line-height: 1; display: flex; align-items: center; justify-content: center;
  }
  #titlebar button:hover { background: rgba(241,237,227,0.18); color: var(--cream); }

  #board {
    background: linear-gradient(165deg, var(--navy), var(--navy-deep));
    border-radius: 10px;
    box-shadow: 0 8px 28px rgba(0,0,0,0.5);
    height: 100vh;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    border: 1px solid var(--border);
  }

  #list {
    flex: 1;
    overflow-y: auto;
    padding: 4px 8px 8px;
    display: flex;
    flex-direction: column;
    gap: 7px;
  }
  #list::-webkit-scrollbar { width: 6px; }
  #list::-webkit-scrollbar-thumb { background: rgba(241,237,227,0.18); border-radius: 3px; }

  #empty {
    margin: auto;
    text-align: center;
    color: var(--text-tertiary);
    font-size: 12.5px;
    padding: 0 20px;
  }

  .card {
    background: var(--surface);
    border-radius: 8px;
    padding: 8px 10px;
    border-left: 4px solid rgba(241,237,227,0.2);
    transition: border-color .2s ease, background .2s ease;
  }
  .card.status-working { border-left-color: var(--gold); }
  .card.status-needs_action { border-left-color: var(--danger); background: #2e2029; animation: pulse 1.6s ease-in-out infinite; }
  .card.status-done { border-left-color: var(--success); }
  .card.status-idle { border-left-color: var(--info); }

  @keyframes pulse {
    0%, 100% { box-shadow: 0 0 0 rgba(180,71,47,0); }
    50% { box-shadow: 0 0 10px rgba(180,71,47,0.5); }
  }

  .row1 { display: flex; align-items: center; gap: 6px; }
  .label {
    font-size: 12.5px; font-weight: 700; color: var(--cream);
    white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
    flex: 1; min-width: 0;
    border-radius: 3px;
    cursor: text;
  }
  .label[contenteditable="true"] {
    white-space: normal; overflow: visible; text-overflow: unset;
    background: rgba(241,237,227,0.08);
    outline: 1px solid var(--border);
    padding: 0 2px;
    -webkit-app-region: no-drag;
    user-select: text;
  }
  .subproj {
    font-size: 9px; color: var(--text-tertiary); margin-top: 2px;
    white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  }
  .badge {
    font-size: 9px; font-weight: 600; text-transform: uppercase; letter-spacing: 0.5px;
    padding: 2px 6px; border-radius: 20px; white-space: nowrap; flex-shrink: 0;
    color: var(--navy-deep);
  }
  .status-working .badge { background: var(--gold); }
  .status-needs_action .badge { background: var(--danger); color: var(--cream); }
  .status-done .badge { background: var(--success); color: var(--cream); }
  .status-idle .badge { background: var(--info); color: var(--cream); }

  .dismissBtn {
    -webkit-app-region: no-drag;
    flex-shrink: 0;
    border: none; background: rgba(241,237,227,0.1); color: var(--text-secondary);
    width: 15px; height: 15px; border-radius: 50%; cursor: pointer;
    font-size: 11px; line-height: 1; display: flex; align-items: center; justify-content: center;
    padding: 0;
  }
  .dismissBtn:hover { background: rgba(241,237,227,0.25); color: var(--cream); }

  .meta {
    font-size: 9.5px; color: var(--text-tertiary); margin-top: 4px;
    display: flex; justify-content: space-between;
  }

  .barwrap {
    margin-top: 6px; height: 3px; border-radius: 2px;
    background: rgba(241,237,227,0.1); overflow: hidden;
  }
  .bar { height: 100%; border-radius: 2px; background: var(--gold); width: 30%; }
  .status-working .bar { animation: indet 1.3s ease-in-out infinite; }
  .status-done .bar { width: 100%; background: var(--success); animation: none; }
  .status-needs_action .bar { width: 100%; background: var(--danger); animation: none; }
  .status-idle .bar { width: 45%; background: var(--info); animation: none; }

  @keyframes indet {
    0% { margin-left: -30%; width: 30%; }
    50% { width: 45%; }
    100% { margin-left: 100%; width: 30%; }
  }
</style>
</head>
<body>
  <div id="board">
    <div id="titlebar">
      <span class="brand"><span class="dot" id="livedot"></span>Active Sessions</span>
      <span class="right">
        <button id="clearDoneBtn" title="Clear finished sessions">✓</button>
        <button id="minBtn" title="Minimize">–</button>
      </span>
    </div>
    <div id="list"></div>
    <div id="empty" style="display:none">No active Claude Code sessions.<br/>Prompt one and it'll show up here.</div>
  </div>
  <script src="renderer.js"></script>
</body>
</html>
```

### `src/renderer.js`

```js
const listEl = document.getElementById('list');
const emptyEl = document.getElementById('empty');

const STATUS_LABEL = {
  working: 'Working',
  idle: 'Idle',
  needs_action: 'Needs You',
  done: 'Done'
};

function displayLabel(s) {
  return s.label || s.customTitle || s.title || s.project;
}

function timeAgo(ts) {
  if (!ts) return '';
  const s = Math.max(0, Math.floor((Date.now() - ts) / 1000));
  if (s < 60) return `${s}s`;
  const m = Math.floor(s / 60);
  if (m < 60) return `${m}m`;
  const h = Math.floor(m / 60);
  return `${h}h ${m % 60}m`;
}

function render(sessions) {
  listEl.innerHTML = '';
  if (!sessions.length) {
    emptyEl.style.display = 'block';
    return;
  }
  emptyEl.style.display = 'none';

  const order = { needs_action: 0, working: 1, idle: 2, done: 3 };
  const sorted = [...sessions].sort((a, b) => {
    const d = order[a.status] - order[b.status];
    if (d !== 0) return d;
    return (b.lastEventAt || 0) - (a.lastEventAt || 0);
  });

  for (const s of sorted) {
    const label = displayLabel(s);
    const showSubtitle = label !== s.project;
    const card = document.createElement('div');
    card.className = `card status-${s.status}`;
    card.dataset.id = s.id;
    card.innerHTML = `
      <div class="row1">
        <span class="label" spellcheck="false" title="${escapeHtml(s.cwd || '')} — double-click to rename">${escapeHtml(label)}</span>
        <span class="badge">${STATUS_LABEL[s.status] || s.status}</span>
        <button class="dismissBtn" title="Stop tracking this session">&times;</button>
      </div>
      ${showSubtitle ? `<div class="subproj">${escapeHtml(s.project)}</div>` : ''}
      <div class="barwrap"><div class="bar"></div></div>
      <div class="meta">
        <span>${s.status === 'done' ? 'finished ' + timeAgo(s.doneAt) + ' ago' : 'active ' + timeAgo(s.lastEventAt) + ' ago'}</span>
      </div>
    `;
    listEl.appendChild(card);
  }
}

function escapeHtml(str) {
  return String(str).replace(/[&<>"']/g, c => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;'
  }[c]));
}

window.widgetAPI.onUpdate((sessions) => render(sessions));
window.widgetAPI.getSessions().then(render);

document.getElementById('minBtn').addEventListener('click', () => {
  window.widgetAPI.minimize();
});
document.getElementById('clearDoneBtn').addEventListener('click', () => {
  window.widgetAPI.clearDone();
});

listEl.addEventListener('click', (e) => {
  const btn = e.target.closest('.dismissBtn');
  if (!btn) return;
  const id = btn.closest('.card')?.dataset.id;
  if (id) window.widgetAPI.dismiss(id);
});

listEl.addEventListener('dblclick', (e) => {
  const label = e.target.closest('.label');
  if (!label) return;
  label.contentEditable = 'true';
  label.focus();
  document.execCommand('selectAll', false, null);
});

listEl.addEventListener('keydown', (e) => {
  if (e.key === 'Enter' && e.target.classList.contains('label')) {
    e.preventDefault();
    e.target.blur();
  } else if (e.key === 'Escape' && e.target.classList.contains('label')) {
    e.target.contentEditable = 'false';
    window.widgetAPI.getSessions().then(render);
  }
});

listEl.addEventListener('focusout', (e) => {
  const label = e.target.closest('.label');
  if (!label || label.contentEditable !== 'true') return;
  label.contentEditable = 'false';
  const id = label.closest('.card')?.dataset.id;
  const newTitle = label.textContent.trim();
  if (id) window.widgetAPI.rename(id, newTitle);
});

// Only refreshes the relative "Xm ago" timestamps and picks up any change
// pushUpdate might have missed -- skipped while a rename is in progress so
// it can't wipe out an unsaved edit.
setInterval(() => {
  if (listEl.querySelector('.label[contenteditable="true"]')) return;
  window.widgetAPI.getSessions().then(render);
}, 20000);
```

### `src/toast.html`

```html
<!doctype html>
<html>
<head>
<meta charset="utf-8" />
<style>
  :root {
    --navy: #1A1F2B;
    --navy-deep: #12151E;
    --cream: #F1EDE3;
    --text-secondary: rgba(241, 237, 227, 0.72);
    --border: rgba(241, 237, 227, 0.12);
    --success: #4B8A63;
    --danger: #B4472F;
  }
  html, body {
    margin: 0; padding: 0;
    background: transparent;
    overflow: hidden;
    font-family: -apple-system, "Segoe UI", system-ui, sans-serif;
    user-select: none;
  }
  #card {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    height: 100vh;
    box-sizing: border-box;
    padding: 14px 16px;
    background: linear-gradient(165deg, var(--navy), var(--navy-deep));
    border: 1px solid var(--border);
    border-left: 5px solid var(--success);
    border-radius: 10px;
    box-shadow: 0 10px 28px rgba(0,0,0,0.5);
    opacity: 0;
    transform: translateY(6px);
    transition: opacity .22s ease, transform .22s ease;
  }
  #card.in { opacity: 1; transform: translateY(0); }
  #card.needs_action { border-left-color: var(--danger); }
  #icon {
    flex-shrink: 0;
    width: 28px; height: 28px; border-radius: 50%;
    display: flex; align-items: center; justify-content: center;
    font-size: 15px; font-weight: 700; color: var(--cream);
    background: var(--success);
    margin-top: 1px;
  }
  #card.needs_action #icon { background: var(--danger); }
  #text { min-width: 0; flex: 1; }
  #title {
    font-size: 15px; font-weight: 700; color: var(--cream);
    display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical;
    overflow: hidden; line-height: 1.3;
  }
  #msg {
    font-size: 13px; font-weight: 400; color: var(--text-secondary); margin-top: 5px;
    display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical;
    overflow: hidden; line-height: 1.4;
  }
  #closeBtn {
    flex-shrink: 0;
    border: none; background: rgba(241,237,227,0.1); color: var(--text-secondary);
    width: 20px; height: 20px; border-radius: 50%; cursor: pointer;
    font-size: 13px; line-height: 1; display: flex; align-items: center; justify-content: center;
    padding: 0; margin-top: 1px;
  }
  #closeBtn:hover { background: rgba(241,237,227,0.22); color: var(--cream); }
</style>
</head>
<body>
  <div id="card">
    <div id="icon"></div>
    <div id="text">
      <div id="title"></div>
      <div id="msg"></div>
    </div>
    <button id="closeBtn" title="Dismiss">&times;</button>
  </div>
  <script>
    const params = new URLSearchParams(location.search);
    const type = params.get('type') || 'done';
    const card = document.getElementById('card');
    card.classList.add(type);
    document.getElementById('icon').textContent = type === 'needs_action' ? '!' : '✓';
    document.getElementById('title').textContent = params.get('title') || '';
    document.getElementById('msg').textContent = params.get('message') || '';
    requestAnimationFrame(() => requestAnimationFrame(() => card.classList.add('in')));

    document.getElementById('closeBtn').addEventListener('click', () => window.close());
  </script>
</body>
</html>
```

### `start-widget.vbs` (Windows silent launcher)

```vbs
Set fso = CreateObject("Scripting.FileSystemObject")
scriptDir = fso.GetParentFolderName(WScript.ScriptFullName)
Set shell = CreateObject("WScript.Shell")
shell.CurrentDirectory = scriptDir
shell.Run """" & scriptDir & "\node_modules\.bin\electron.cmd"" .", 0, False
```

Double-clicking this runs the widget with no visible console window. On
macOS/Linux, skip this file and just run `npm start`, or build an equivalent
(`osascript`/`.desktop` entry) if the user wants silent autostart there.

## Step 3 — Install

```
npm install
```

## Step 4 — Wire the hooks

This is the part that's easy to get subtly wrong. Read `~/.claude/settings.json`
first and merge into its existing `hooks` key if one already exists — do not
overwrite the file. **Tell the user you're about to edit their global Claude
Code settings (this affects every project on the machine) and confirm before
doing it**, the same way you would before any other edit to shared config.

```json
{
  "hooks": {
    "SessionStart": [
      { "matcher": "*", "hooks": [ { "type": "http", "url": "http://127.0.0.1:51823/hook", "timeout": 2 } ] }
    ],
    "UserPromptSubmit": [
      { "matcher": "*", "hooks": [ { "type": "http", "url": "http://127.0.0.1:51823/hook", "timeout": 2 } ] }
    ],
    "Notification": [
      { "matcher": "*", "hooks": [ { "type": "http", "url": "http://127.0.0.1:51823/hook", "timeout": 2 } ] }
    ],
    "PreToolUse": [
      { "matcher": "AskUserQuestion|ExitPlanMode", "hooks": [ { "type": "http", "url": "http://127.0.0.1:51823/hook", "timeout": 2 } ] }
    ],
    "PostToolUse": [
      { "matcher": "AskUserQuestion|ExitPlanMode", "hooks": [ { "type": "http", "url": "http://127.0.0.1:51823/hook", "timeout": 2 } ] }
    ],
    "Stop": [
      { "matcher": "*", "hooks": [ { "type": "http", "url": "http://127.0.0.1:51823/hook", "timeout": 2 } ] }
    ],
    "SessionEnd": [
      { "matcher": "*", "hooks": [ { "type": "http", "url": "http://127.0.0.1:51823/hook", "timeout": 2 } ] }
    ]
  }
}
```

If the port in `main.js` (`PORT = 51823`) gets changed, update every URL
above to match.

## Step 5 — Test it

1. Launch the app: `npm start` (or the `.vbs` launcher).
2. Confirm the HTTP server is up:
   `curl -X POST http://127.0.0.1:51823/hook -d "{}"` should return `ok`.
3. Fire a synthetic event to see a card appear without waiting on a real
   Claude Code turn:
   ```
   curl -X POST http://127.0.0.1:51823/hook -H "Content-Type: application/json" \
     -d "{\"hook_event_name\":\"UserPromptSubmit\",\"session_id\":\"test-1\",\"cwd\":\"/tmp/demo\",\"prompt\":\"testing\"}"
   ```
   The card should appear showing "Working."
4. Have the user open a real Claude Code session and confirm a card appears
   and updates through Working → (Needs You, if a permission prompt/question
   comes up) → Done.

## Key facts (do not deviate from these without a reason)

- **`type: "http"` hooks deliver the payload directly** — `hook_event_name`,
  `session_id`, `cwd`, `transcript_path`, etc. all arrive as JSON fields on
  the POST body. No relay script or shell command needed, and no Windows
  shell-quoting issues to worry about.
- **`Notification` does NOT fire for clarifying questions or plan approval.**
  Those only show up via `PreToolUse`/`PostToolUse` scoped to
  `AskUserQuestion` and `ExitPlanMode` specifically. Without those two extra
  hooks, a session silently waiting on a question looks identical to one
  that's still working — this was a real, confirmed gap during development.
- **A session's real/custom title isn't in the hook payload.** Claude Code
  writes `{"type":"ai-title","aiTitle":"..."}` and
  `{"type":"custom-title","customTitle":"..."}` as line entries into the
  session's own transcript `.jsonl` file — that's the only place to read
  them from. Tail-read the file (bounded, e.g. last 2MB) rather than loading
  the whole thing; transcripts can grow to several MB over a long session.
- **Toast supersession matters.** If a session's status changes again before
  its current toast's full lifetime elapses (e.g. approved and finished
  seconds after asking), the new toast must replace the old one immediately
  — not queue behind it. Otherwise a stale "needs you" popup can sit on
  screen well after the session actually finished.
- **Cap anything captured from a user prompt.** A first message can be a
  pasted multi-thousand-word document; store only a few hundred characters,
  since it's only ever a fallback label, not user-facing content in full.
- **Only hook what you need.** `PreToolUse`/`PostToolUse` scoped broadly (no
  matcher, or `"*"`) fire on *every single tool call* across every session —
  avoid that; scope matchers tightly (as above) or skip hooking events you
  don't need for state tracking.

## Customization

- **Recolor**: edit the `:root` CSS variables at the top of `index.html` (and
  match `--navy`/`--navy-deep`/`--cream`/`--success`/`--danger` in
  `toast.html`).
- **Add a logo**: drop a small PNG into `src/assets/` and add an `<img>` next
  to the `.dot` in the titlebar.
- **Change the active-session window**: adjust `ACTIVE_WINDOW_MS` in
  `main.js` (defaults to 12 hours).
- **Change toast durations**: adjust `TOAST_LIFETIME` in `main.js`.
- **Autostart on login**: place a shortcut to `start-widget.vbs` in the
  user's Startup folder (`shell:startup` in File Explorer's address bar on
  Windows) — only do this if asked, since it changes what launches at login.
