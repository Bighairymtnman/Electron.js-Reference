# Advanced Electron Techniques

A guide to advanced Electron patterns, optimizations, and security practices for building professional desktop applications.

## Table of Contents

- [Security Best Practices](#security-best-practices)
- [Process Sandboxing](#process-sandboxing)
- [Performance Optimization](#performance-optimization)
- [Native Node Addons](#native-node-addons)
- [Inter-Window Communication](#inter-window-communication)
- [Offline Support](#offline-support)
- [Custom Protocol Handlers](#custom-protocol-handlers)
- [Native File Drag & Drop](#native-file-drag--drop)
- [Tray Applications](#tray-applications)
- [Multi-Window Architecture](#multi-window-architecture)
- [Splash Screens](#splash-screens)

## Security Best Practices

### Content Security Policy

```html
<!-- In index.html -->
<meta
  http-equiv="Content-Security-Policy"
  content="default-src 'self'; script-src 'self'"
/>
```

### Secure WebPreferences

```javascript
const win = new BrowserWindow({
  webPreferences: {
    nodeIntegration: false,
    contextIsolation: true,
    sandbox: true,
    webSecurity: true,
    allowRunningInsecureContent: false,
    preload: path.join(__dirname, "preload.js"),
  },
});
```

### Validate IPC Input

```javascript
// In main process
ipcMain.handle("save-data", async (event, data) => {
  // Validate data before processing
  if (!data || typeof data !== "object" || !data.fileName || !data.content) {
    throw new Error("Invalid data format");
  }

  // Sanitize file name to prevent path traversal
  const fileName = path.basename(data.fileName);
  const safePath = path.join(app.getPath("userData"), fileName);

  // Now safe to proceed
  await fs.promises.writeFile(safePath, data.content);
  return { success: true, path: safePath };
});
```

### HTTPS for Remote Content

```javascript
// Only load trusted HTTPS content
win.loadURL("https://example.com", {
  userAgent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
});

// Check certificate errors
app.on(
  "certificate-error",
  (event, webContents, url, error, certificate, callback) => {
    // Prevent connection on certificate errors
    event.preventDefault();
    callback(false);
  }
);
```

## Process Sandboxing

### Enable Sandbox for Renderer

```javascript
// In main.js
app.enableSandbox(); // Enable for all renderers

// Or per window
const win = new BrowserWindow({
  webPreferences: {
    sandbox: true,
  },
});
```

### Isolated Renderer Processes

```javascript
// Create isolated renderers for untrusted content
const untrustedWindow = new BrowserWindow({
  webPreferences: {
    sandbox: true,
    contextIsolation: true,
    nodeIntegration: false,
    preload: path.join(__dirname, "minimal-preload.js"),
  },
});

// Minimal preload for untrusted content
// minimal-preload.js
contextBridge.exposeInMainWorld("secureAPI", {
  // Expose only what's absolutely necessary
  reportProblem: (description) =>
    ipcRenderer.invoke("report-problem", description),
});
```

## Performance Optimization

### Optimize Window Creation

```javascript
// Defer heavy initialization
app.whenReady().then(() => {
  // Create window immediately for better UX
  mainWindow = new BrowserWindow({ show: false });
  mainWindow.loadFile("index.html");

  // Perform heavy initialization while loading
  initializeDatabase();

  // Show when ready
  mainWindow.once("ready-to-show", () => {
    mainWindow.show();
  });
});
```

### Lazy Load Components

```javascript
// In renderer
const loadHeavyComponent = async () => {
  const { HeavyComponent } = await import("./heavy-component.js");
  document.getElementById("container").appendChild(HeavyComponent());
};

document
  .getElementById("load-btn")
  .addEventListener("click", loadHeavyComponent);
```

### Process-Specific Tasks

```javascript
// CPU-intensive tasks in worker process
const { Worker } = require("worker_threads");

function runCalculation(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker("./workers/calculation.js", {
      workerData: data,
    });

    worker.on("message", resolve);
    worker.on("error", reject);
    worker.on("exit", (code) => {
      if (code !== 0)
        reject(new Error(`Worker stopped with exit code ${code}`));
    });
  });
}

// In calculation.js worker
const { workerData, parentPort } = require("worker_threads");
const result = performHeavyCalculation(workerData);
parentPort.postMessage(result);
```

## Native Node Addons

### Integrating Native Modules

```javascript
// In main process
const ffi = require("ffi-napi");
const ref = require("ref-napi");

// Define interface to native library
const libm = ffi.Library("libm", {
  ceil: ["double", ["double"]],
});

// Use native function
console.log(libm.ceil(1.5)); // 2
```

### Rebuilding Native Modules

```bash
# Install electron-rebuild
npm install --save-dev electron-rebuild

# Add to package.json
"scripts": {
  "rebuild": "electron-rebuild"
}

# Run after installing native modules
npm run rebuild
```

## Inter-Window Communication

### BrowserWindow Communication

```javascript
// In main.js
const windows = {};

function createWindow(name, options) {
  windows[name] = new BrowserWindow(options);
  // ...
}

// Send message between windows
function sendCrossWindowMessage(from, to, channel, data) {
  if (windows[to] && !windows[to].isDestroyed()) {
    windows[to].webContents.send("window-message", {
      from,
      channel,
      data,
    });
  }
}

// Handle in main process
ipcMain.on("send-to-window", (event, { to, channel, data }) => {
  const fromWindow = BrowserWindow.fromWebContents(event.sender);
  const fromName = Object.keys(windows).find(
    (key) => windows[key] === fromWindow
  );

  sendCrossWindowMessage(fromName, to, channel, data);
});
```

### Shared Web Workers

```javascript
// In renderer
const worker = new SharedWorker("shared-worker.js");

worker.port.onmessage = (event) => {
  console.log("Message from shared worker:", event.data);
};

worker.port.postMessage({
  action: "register",
  windowId: window.electronAPI.getWindowId(),
});

// In shared-worker.js
const ports = new Map();

self.onconnect = (event) => {
  const port = event.ports[0];

  port.onmessage = (messageEvent) => {
    const { action, windowId, message } = messageEvent.data;

    if (action === "register") {
      ports.set(windowId, port);
    } else if (action === "broadcast") {
      // Broadcast to all windows
      for (const [id, targetPort] of ports.entries()) {
        if (id !== windowId) {
          targetPort.postMessage(message);
        }
      }
    }
  };

  port.start();
};
```

## Offline Support

### Local Database with SQLite

```javascript
// In main process
const sqlite3 = require("sqlite3");
const { open } = require("sqlite");

let db;

async function initDatabase() {
  db = await open({
    filename: path.join(app.getPath("userData"), "database.sqlite"),
    driver: sqlite3.Database,
  });

  await db.exec(`
    CREATE TABLE IF NOT EXISTS items (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT,
      content TEXT,
      created_at INTEGER
    )
  `);
}

// Expose database operations via IPC
ipcMain.handle("db-get-all", async () => {
  return db.all("SELECT * FROM items ORDER BY created_at DESC");
});

ipcMain.handle("db-add-item", async (event, item) => {
  const result = await db.run(
    "INSERT INTO items (title, content, created_at) VALUES (?, ?, ?)",
    [item.title, item.content, Date.now()]
  );
  return { id: result.lastID };
});
```

### Offline-First Architecture

```javascript
// In preload.js
contextBridge.exposeInMainWorld("dataStore", {
  // Save data locally first, then sync when online
  saveItem: async (item) => {
    // Save to local store
    await ipcRenderer.invoke("db-add-item", item);

    // Try to sync if online
    if (navigator.onLine) {
      try {
        await ipcRenderer.invoke("sync-item", item);
      } catch (err) {
        // Mark for later sync
        await ipcRenderer.invoke("queue-sync-item", item);
      }
    } else {
      // Queue for later sync
      await ipcRenderer.invoke("queue-sync-item", item);
    }

    return true;
  },
});

// In main.js
// Listen for online status changes
ipcMain.handle("check-online-status", () => {
  return mainWindow.webContents.getURL().startsWith("https");
});

// Sync queued items when online
app.on("web-contents-created", (e, contents) => {
  contents.on("did-navigate", async () => {
    if (contents.getURL().startsWith("https")) {
      await syncQueuedItems();
    }
  });
});
```

## Custom Protocol Handlers

### Register Custom Protocol

```javascript
// In main.js
if (process.defaultApp) {
  if (process.argv.length >= 2) {
    app.setAsDefaultProtocolClient("myapp", process.execPath, [
      path.resolve(process.argv[1]),
    ]);
  }
} else {
  app.setAsDefaultProtocolClient("myapp");
}

// Handle protocol activation
app.on("open-url", (event, url) => {
  event.preventDefault();
  const urlObj = new URL(url);

  if (urlObj.protocol === "myapp:") {
    // Handle custom protocol
    handleCustomProtocol(urlObj);
  }
});

function handleCustomProtocol(urlObj) {
  const action = urlObj.hostname;
  const params = Object.fromEntries(urlObj.searchParams);

  switch (action) {
    case "open-item":
      openItemById(params.id);
      break;
    case "auth-callback":
      processAuthCallback(params);
      break;
  }
}
```

### Deep Linking

```javascript
// In main.js
let deeplinkingUrl;

// Handle deeplink in Windows/Linux
const gotTheLock = app.requestSingleInstanceLock();

if (!gotTheLock) {
  app.quit();
} else {
  app.on("second-instance", (e, argv) => {
    if (mainWindow) {
      if (mainWindow.isMinimized()) mainWindow.restore();
      mainWindow.focus();

      // Handle deeplink
      if (process.platform !== "darwin") {
        const deeplinkUrl = argv.find((arg) => arg.startsWith("myapp://"));
        if (deeplinkUrl) processDeepLink(deeplinkUrl);
      }
    }
  });

  // Handle deeplink in macOS
  app.on("open-url", (event, url) => {
    event.preventDefault();
    deeplinkingUrl = url;
    processDeepLink(url);
  });
}

function processDeepLink(url) {
  const urlObj = new URL(url);
  mainWindow.webContents.send("deep-link", {
    route: urlObj.hostname,
    params: Object.fromEntries(urlObj.searchParams),
  });
}
```

## Native File Drag & Drop

### Enable File Drag & Drop

```javascript
// In main.js
mainWindow.webContents.on("will-navigate", (event) => {
  event.preventDefault();
});

// In preload.js
contextBridge.exposeInMainWorld("fileHandler", {
  handleFiles: (files) => ipcRenderer.invoke("handle-dropped-files", files),
});

// In renderer
document.addEventListener("dragover", (e) => {
  e.preventDefault();
  e.stopPropagation();
});

document.addEventListener("drop", (e) => {
  e.preventDefault();
  e.stopPropagation();

  const files = [];
  for (const file of e.dataTransfer.files) {
    files.push({
      path: file.path,
      name: file.name,
      size: file.size,
      type: file.type,
    });
  }

  window.fileHandler.handleFiles(files);
});

// In main.js
ipcMain.handle("handle-dropped-files", async (event, files) => {
  // Process dropped files
  for (const file of files) {
    // Handle each file
    await processFile(file.path);
  }
  return { success: true };
});
```

## Tray Applications

### Advanced Tray Features

`````javascript
// In main.js
let tray = null;
let contextMenu = null;

function createTray() {
  tray = new Tray(path.join(__dirname, 'assets', 'tray-icon.png'));

  // Dynamic context menu
  function buildContextMenu() {
    return Menu.buildFromTemplate([
      {
        label: 'Open Main Window',
        click: () => {
          if (mainWindow) {
            mainWindow.show();
          } else {
            createMainWindow();
          }
        }
      },
      {
        label: 'Recent Items',
        submenu: buildRecentItemsMenu()
      },
      { type: 'separator' },
      {
        label: app.isQuitting ? 'Cancel Quit' : 'Quit',
        click: () => {
          if (app.isQuitting) {
            app.isQuitting = false;
          } else {
            app.isQuitting = true;
            app.quit();
          }
        }
      }
    ]);
  }

  // Build dynamic submenu
  function buildRecentItemsMenu() {
    const recentItems = getRecentItems(); // Get from storage

    if (recentItems.length === 0) {
      return [{ label: 'No recent items', enabled: false }];
    }

    return recentItems.map(item => ({
      label: item.title,
      click: () => openItem(item.id)
    }));
  }

  // Set up tray
  contextMenu = buildContextMenu();
  tray.setToolTip('My Electron App');
  tray.setContextMenu(contextMenu);

  // Rebuild menu when data changes
  ipcMain.on('recent-items-changed', () => {
    contextMenu = buildContextMenu();
    tray.setContextMenu(contextMenu);
  });

  // Handle left click (platform specific)
  tray.on('click', () => {
    if (process.platform === 'darwin') {
      tray.popUpContextMenu();
    } else {
      if (mainWindow)
  // Handle left click (platform specific)
  tray.on('click', () => {
    if (process.platform === 'darwin') {
      tray.popUpContextMenu();
    } else {
      if (mainWindow) {
        if (mainWindow.isVisible()) {
          mainWindow.hide();
        } else {
          mainWindow.show();
        }
      }
    }
  });
}
```

## Multi-Window Architecture

### Window Management System
````javascript
// In main.js
class WindowManager {
  constructor() {
    this.windows = new Map();
    this.windowStates = new Map();
    this.registerListeners();
  }

  registerListeners() {
    // Save window states when closing
    app.on('before-quit', () => {
      this.windows.forEach((window, id) => {
        if (!window.isDestroyed()) {
          this.saveWindowState(id, window);
        }
      });
    });
  }

  createWindow(id, options = {}) {
    if (this.windows.has(id) && !this.windows.get(id).isDestroyed()) {
      const existingWindow = this.windows.get(id);
      existingWindow.focus();
      return existingWindow;
    }

    // Restore previous state if available
    const savedState = this.windowStates.get(id);
    const windowOptions = {
      ...options,
      ...(savedState ? {
        x: savedState.x,
        y: savedState.y,
        width: savedState.width,
        height: savedState.height
      } : {})
    };

    const window = new BrowserWindow(windowOptions);
    this.windows.set(id, window);

    // Save state when closing
    window.on('close', () => {
      this.saveWindowState(id, window);
    });

    // Remove from collection when closed
    window.on('closed', () => {
      this.windows.delete(id);
    });

    return window;
  }

  saveWindowState(id, window) {
    if (window.isNormal()) {
      const bounds = window.getBounds();
      this.windowStates.set(id, {
        x: bounds.x,
        y: bounds.y,
        width: bounds.width,
        height: bounds.height,
        isMaximized: window.isMaximized()
      });
    }
  }

  getWindow(id) {
    return this.windows.get(id);
  }

  closeAll() {
    this.windows.forEach(window => {
      if (!window.isDestroyed()) {
        window.close();
      }
    });
  }
}

// Usage
const windowManager = new WindowManager();

// Create main window
const mainWindow = windowManager.createWindow('main', {
  width: 800,
  height: 600,
  webPreferences: {
    preload: path.join(__dirname, 'preload.js')
  }
});

// Create secondary window
ipcMain.handle('open-settings', () => {
  const settingsWindow = windowManager.createWindow('settings', {
    width: 500,
    height: 400,
    parent: mainWindow,
    modal: true
  });

  settingsWindow.loadFile('settings.html');
  return true;
});
```

### Parent-Child Window Relationships
````javascript
// Create a child window that's tied to parent
function createChildWindow(parentId, url, options = {}) {
  const parentWindow = windowManager.getWindow(parentId);

  if (!parentWindow || parentWindow.isDestroyed()) {
    return null;
  }

  const childWindow = new BrowserWindow({
    parent: parentWindow,
    width: 600,
    height: 400,
    modal: options.modal || false,
    ...options,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      ...options.webPreferences
    }
  });

  childWindow.loadURL(url);

  // Prevent parent destruction from leaving orphaned children
  parentWindow.on('closed', () => {
    if (!childWindow.isDestroyed()) {
      childWindow.close();
    }
  });

  return childWindow;
}

// IPC handler for creating child windows
ipcMain.handle('create-child-window', (event, { parentId, url, options }) => {
  const window = createChildWindow(parentId, url, options);
  return window ? true : false;
});
```

## Splash Screens

### Advanced Splash Screen
````javascript
// In main.js
let splashScreen;
let mainWindow;
let appIsReady = false;
let splashIsReady = false;

function createSplashScreen() {
  splashScreen = new BrowserWindow({
    width: 400,
    height: 300,
    transparent: true,
    frame: false,
    alwaysOnTop: true,
    webPreferences: {
      contextIsolation: true,
      preload: path.join(__dirname, 'splash-preload.js')
    }
  });

  splashScreen.loadFile('splash.html');

  // Listen for splash screen ready
  ipcMain.once('splash-ready', () => {
    splashIsReady = true;
    if (appIsReady) {
      showMainWindow();
    }
  });
}

async function initializeApp() {
  // Perform initialization tasks
  try {
    await Promise.all([
      initDatabase(),
      loadConfiguration(),
      checkForUpdates()
    ]);

    // Update splash screen progress
    if (splashScreen && !splashScreen.isDestroyed()) {
      splashScreen.webContents.send('init-progress', 100);
    }

    appIsReady = true;
    if (splashIsReady) {
      showMainWindow();
    }
  } catch (error) {
    console.error('Initialization failed:', error);
    if (splashScreen && !splashScreen.isDestroyed()) {
      splashScreen.webContents.send('init-error', error.message);
    }
  }
}

function showMainWindow() {
  // Create main window
  mainWindow = new BrowserWindow({
    show: false,
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js')
    }
  });

  mainWindow.loadFile('index.html');

  // Show main window when ready
  mainWindow.once('ready-to-show', () => {
    // Close splash and show main window with a slight delay for smooth transition
    setTimeout(() => {
      if (splashScreen && !splashScreen.isDestroyed()) {
        splashScreen.close();
        splashScreen = null;
      }

      mainWindow.show();
      mainWindow.focus();
    }, 500);
  });
}

app.whenReady().then(() => {
  createSplashScreen();
  initializeApp();
});
```

### Splash Screen HTML/CSS
````html
<!-- splash.html -->
<!DOCTYPE html>
<html>
<head>
  <style>
    body {
      margin: 0;
      padding: 0;
      font-family: Arial, sans-serif;
      background-color: transparent;
      overflow: hidden;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      user-select: none;
    }

    .splash-container {
      background: linear-gradient(135deg, #2b5876, #4e4376);
      border-radius: 10px;
      box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
      padding: 30px;
      text-align: center;
      color: white;
      width: 300px;
    }

    .logo {
      width: 100px;
      height: 100px;
      margin-bottom: 20px;
    }

    .progress-bar {
      height: 6px;
      background-color: rgba(255, 255, 255, 0.2);
      border-radius: 3px;
      margin-top: 20px;
      overflow: hidden;
    }

    .progress {
      height: 100%;
      background-color: #4CAF50;
      width: 0%;
      transition: width 0.3s ease;
    }

    .status-text {
      margin-top: 10px;
      font-size: 12px;
      opacity: 0.8;
    }

    .error-message {
      color: #ff6b6b;
      margin-top: 15px;
      font-size: 12px;
      display: none;
    }
  </style>
</head>
<body>
  <div class="splash-container">
    <img src="assets/logo.png" class="logo" alt="App Logo">
    <h2>My Electron App</h2>
    <div class="progress-bar">
      <div class="progress" id="progress"></div>
    </div>
    <div class="status-text" id="status">Initializing...</div>
    <div class="error-message" id="error"></div>
  </div>

  <script src="splash.js"></script>
</body>
</html>
```

### Splash Screen JavaScript
````javascript
// splash.js
document.addEventListener('DOMContentLoaded', () => {
  // Tell main process the splash screen is ready
  window.splashAPI.ready();

  // Listen for progress updates
  window.splashAPI.onProgress((progress) => {
    document.getElementById('progress').style.width = `${progress}%`;

    if (progress < 30) {
      document.getElementById('status').textContent = 'Initializing...';
    } else if (progress < 60) {
      document.getElementById('status').textContent = 'Loading configuration...';
    } else if (progress < 90) {
      document.getElementById('status').textContent = 'Checking for updates...';
    } else {
      document.getElementById('status').textContent = 'Starting application...';
    }
  });

  // Listen for errors
  window.splashAPI.onError((message) => {
    const errorElement = document.getElementById('error');
    errorElement.textContent = `Error: ${message}`;
    errorElement.style.display = 'block';
    document.getElementById('status').textContent = 'Failed to start';
  });
});
```

### Splash Preload Script
````javascript
// splash-preload.js
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('splashAPI', {
  ready: () => ipcRenderer.send('splash-ready'),
  onProgress: (callback) => {
    ipcRenderer.on('init-progress', (event, progress) => callback(progress));
  },
  onError: (callback) => {
    ipcRenderer.on('init-error', (event, message) => callback(message));
  }
});
`````
