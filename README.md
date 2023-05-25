# Building a desktop app with Electron and Create React App

## React app

Let's start from "empty" React app generated with Create React App.
```shell
  npx create-react-app electron-react
```

Then, add the following dependencies:

```shell
  cd electron-react
  npm install -D concurrently cross-env electron electron-builder electronmon wait-on
```

* __`concurrently`__: Run multiple commands concurrently.
* __`cross-env`__: Run scripts that set and use environment variables across different platforms.
* __`electron`__: The core framework for creating the app.
* __`electron-builder`__: A complete solution to package and build a ready for distribution Electron app for macOS, Windows, and Linux.
* __`electronmon`__: Like __`nodemon`__, but for the Electron process. Allows watching and reloading our Electron app.
* __`wait-on`__: Utility to wait for files, ports, sockets, etc.

## Electron's main script

The next step is creating Electron's main script. This script controls the main process, which runs in a full Node.js environment and is responsible for managing your app's lifecycle, displaying native interfaces, performing privileged operations, and managing renderer processes.

Electron's main script is often named `main.js` and stored in `<project-root>/electron/main.js`, but in out case, we'll name it `electron.js` (to disambiguate it) and store it in `<project-root>/public/electron.js` (so that Create React App will automatically copy it in the build directory).