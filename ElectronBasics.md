# Electron Basics

A guide to essential Electron setup and usage patterns for building cross-platform desktop applications.

## Table of Contents

- [Project Setup](#project-setup)
- [Main Process Setup](#main-process-setup)
- [Renderer Process Setup](#renderer-process-setup)
- [Preload Scripts](#preload-scripts)
- [Basic IPC Communication](#basic-ipc-communication)
- [Application Packaging](#application-packaging)
- [Auto Updates](#auto-updates)
- [Development Workflow](#development-workflow)

## Project Setup

### Basic Project Structure

```
my-electron-app/
├── package.json
├── main.js
├── preload.js
├── renderer/
│   ├── index.html
│   ├── index.js
│   └── style.css
└── assets/
    └── icons/
```

### Package.json Configuration

```json
{
  "name": "my-electron-app",
  "version": "1.0.0",
  "description": "My Electron application",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "build": "electron-builder",
    "pack": "electron-builder --dir"
  },
  "devDependencies": {
    "electron": "^22.0.0",
    "electron-builder": "^23.6.0"
  }
}
```

## Main Process Setup

The main process is the entry point for your Electron application. It controls the application lifecycle and creates browser windows.

```javascript
// main.js
const { app, BrowserWindow } = require("electron");
const path = require("path");

// Keep a global reference to prevent garbage collection
let mainWindow;

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, "preload.js"),
      nodeIntegration: false,
      contextIsolation: true,
    },
  });

  // Load the index.html file
  mainWindow.loadFile(path.join(__dirname, "renderer", "index.html"));

  // Open DevTools in development
  if (process.env.NODE_ENV === "development") {
    mainWindow.webContents.openDevTools();
  }

  // Handle window being closed
  mainWindow.on("closed", () => {
    mainWindow = null;
  });
}

// Create window when app is ready
app.whenReady().then(createWindow);

// Quit when all windows are closed, except on macOS
app.on("window-all-closed", () => {
  if (process.platform !== "darwin") {
    app.quit();
  }
});

// On macOS, recreate window when dock icon is clicked
app.on("activate", () => {
  if (BrowserWindow.getAllWindows().length === 0) {
    createWindow();
  }
});
```

## Renderer Process Setup

The renderer process runs the web page UI of your application.

```html
<!-- renderer/index.html -->
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>My Electron App</title>
    <link rel="stylesheet" href="style.css" />
  </head>
  <body>
    <h1>Hello Electron!</h1>
    <p id="info">System info will appear here</p>
    <button id="btn">Get System Info</button>

    <script src="index.js"></script>
  </body>
</html>
```

```javascript
// renderer/index.js
document.getElementById("btn").addEventListener("click", () => {
  // Use exposed API from preload script
  window.electronAPI.getSystemInfo().then((info) => {
    document.getElementById(
      "info"
    ).textContent = `Platform: ${info.platform}, Arch: ${info.arch}`;
  });
});
```

## Preload Scripts

Preload scripts run before the renderer process and can safely expose specific Node.js and Electron APIs to the renderer.

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require("electron");
const os = require("os");

// Expose protected methods that allow the renderer process to use
// specific electron APIs without exposing the entire API
contextBridge.exposeInMainWorld("electronAPI", {
  getSystemInfo: () => {
    return Promise.resolve({
      platform: os.platform(),
      arch: os.arch(),
      totalMem: os.totalmem(),
    });
  },
  sendMessage: (channel, data) => {
    // whitelist channels
    let validChannels = ["toMain"];
    if (validChannels.includes(channel)) {
      ipcRenderer.send(channel, data);
    }
  },
  receiveMessage: (channel, func) => {
    let validChannels = ["fromMain"];
    if (validChannels.includes(channel)) {
      // Deliberately strip event as it includes `sender`
      ipcRenderer.on(channel, (event, ...args) => func(...args));
    }
  },
});
```

## Basic IPC Communication

Inter-Process Communication (IPC) allows the main and renderer processes to communicate.

```javascript
// In main.js
const { ipcMain } = require("electron");

ipcMain.on("toMain", (event, args) => {
  console.log("Received in main process:", args);

  // Send a response back to the renderer
  mainWindow.webContents.send("fromMain", "Message received in main process");
});

// In renderer/index.js (after preload exposes the API)
window.electronAPI.sendMessage("toMain", "Hello from renderer");

window.electronAPI.receiveMessage("fromMain", (message) => {
  console.log("Received in renderer:", message);
});
```

## Application Packaging

Package your application for distribution using electron-builder.

### electron-builder Configuration

```json
// In package.json
{
  "build": {
    "appId": "com.example.myapp",
    "productName": "My Electron App",
    "directories": {
      "output": "dist"
    },
    "mac": {
      "category": "public.app-category.utilities"
    },
    "win": {
      "target": ["nsis", "portable"]
    },
    "linux": {
      "target": ["AppImage", "deb"]
    }
  }
}
```

### Build Commands

```bash
# For all platforms
npm run build

# For specific platform
npx electron-builder --mac
npx electron-builder --win
npx electron-builder --linux
```

## Auto Updates

Implement automatic updates using electron-updater.

```javascript
// In main.js
const { autoUpdater } = require("electron-updater");

// Check for updates
autoUpdater.checkForUpdatesAndNotify();

// Listen for update events
autoUpdater.on("update-available", () => {
  mainWindow.webContents.send("update-available");
});

autoUpdater.on("update-downloaded", () => {
  mainWindow.webContents.send("update-downloaded");
});

// Install updates when requested from renderer
ipcMain.on("install-update", () => {
  autoUpdater.quitAndInstall();
});
```

## Development Workflow

### Hot Reload Setup

Using electron-reload for development:

```javascript
// In main.js (development only)
if (process.env.NODE_ENV === "development") {
  require("electron-reload")(__dirname, {
    electron: path.join(__dirname, "node_modules", ".bin", "electron"),
    hardResetMethod: "exit",
  });
}
```

### Debugging

1. **Main Process**: Use VS Code debugging or start with `--inspect` flag

   ```bash
   electron --inspect=5858 .
   ```

2. **Renderer Process**: Use Chrome DevTools
   ```javascript
   mainWindow.webContents.openDevTools();
   ```

### Environment Variables

```javascript
// Set environment in package.json scripts
"scripts": {
  "start": "cross-env NODE_ENV=development electron .",
  "prod": "cross-env NODE_ENV=production electron ."
}

// Access in code
const isDev = process.env.NODE_ENV === 'development';
```
