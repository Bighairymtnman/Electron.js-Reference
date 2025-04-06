# Electron Core Elements

A quick reference guide to fundamental Electron components, APIs, and concepts.

## Table of Contents

- [Main Process vs Renderer Process](#main-process-vs-renderer-process)
- [Core Modules](#core-modules)
- [BrowserWindow](#browserwindow)
- [IPC Communication](#ipc-communication)
- [Menu API](#menu-api)
- [Dialog API](#dialog-api)
- [Tray API](#tray-api)
- [Native File Operations](#native-file-operations)
- [App Lifecycle](#app-lifecycle)

## Main Process vs Renderer Process

- **Main Process**: Controls application lifecycle, creates browser windows, accesses native APIs
- **Renderer Process**: Runs web pages, handles UI, limited access to native APIs

## Core Modules

- `app` - Controls application lifecycle
- `BrowserWindow` - Creates and manages application windows
- `ipcMain` / `ipcRenderer` - Communication between processes
- `dialog` - Native system dialogs
- `Menu` / `MenuItem` - Native application menus
- `Tray` - System tray integration
- `shell` - Open files/URLs with default applications
- `nativeImage` - Work with images in native formats

## BrowserWindow

```jsx
const { BrowserWindow } = require("electron");

const win = new BrowserWindow({
  width: 800,
  height: 600,
  webPreferences: {
    nodeIntegration: false,
    contextIsolation: true,
    preload: path.join(__dirname, "preload.js"),
  },
});

win.loadURL("https://example.com");
// or
win.loadFile("index.html");
```

## IPC Communication

```jsx
// In main process
const { ipcMain } = require("electron");

ipcMain.on("asynchronous-message", (event, arg) => {
  console.log(arg); // prints "ping"
  event.reply("asynchronous-reply", "pong");
});

ipcMain.handle("synchronous-message", async (event, arg) => {
  return "pong";
});

// In renderer process
const { ipcRenderer } = require("electron");

// Asynchronous
ipcRenderer.send("asynchronous-message", "ping");
ipcRenderer.on("asynchronous-reply", (event, arg) => {
  console.log(arg); // prints "pong"
});

// Synchronous (Promise-based)
async function sendMessage() {
  const response = await ipcRenderer.invoke("synchronous-message", "ping");
  console.log(response); // prints "pong"
}
```

## Menu API

```jsx
const { Menu, MenuItem } = require("electron");

const menu = new Menu();
menu.append(
  new MenuItem({
    label: "File",
    submenu: [{ role: "quit" }],
  })
);

Menu.setApplicationMenu(menu);
```

## Dialog API

```jsx
const { dialog } = require("electron");

dialog
  .showOpenDialog({
    properties: ["openFile", "multiSelections"],
  })
  .then((result) => {
    console.log(result.filePaths);
  });

dialog.showMessageBox({
  type: "info",
  buttons: ["OK"],
  title: "Information",
  message: "Operation completed successfully",
});
```

## Tray API

```jsx
const { Tray, Menu } = require("electron");
const path = require("path");

const tray = new Tray(path.join(__dirname, "icon.png"));
const contextMenu = Menu.buildFromTemplate([
  { label: "Item1", type: "radio" },
  { label: "Item2", type: "radio" },
]);

tray.setToolTip("My Application");
tray.setContextMenu(contextMenu);
```

## Native File Operations

```jsx
const { app } = require("electron");
const fs = require("fs");
const path = require("path");

const userDataPath = app.getPath("userData");
const filePath = path.join(userDataPath, "settings.json");

// Save data
fs.writeFileSync(filePath, JSON.stringify({ theme: "dark" }));

// Read data
const data = JSON.parse(fs.readFileSync(filePath));
```

## App Lifecycle

```jsx
const { app } = require("electron");

app.on("ready", createWindow);
app.on("window-all-closed", () => {
  if (process.platform !== "darwin") app.quit();
});
app.on("activate", () => {
  if (BrowserWindow.getAllWindows().length === 0) createWindow();
});
```
