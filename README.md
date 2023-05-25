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

`electron.js` setup used is not a "minimal", but it has nice defaults and made sure we're following [Electron's security guidelines.](https://www.electronjs.org/docs/tutorial/security)

During execution, Electron will look for `electron.js` in the main fiels of the app's `package.json` config

```json
  "main": "./public/electron.js
```

## Electron's preload script

By default, the process running in your browser won't be able to commnunicate with the Node.js process. Electron solves this problem by allowing the use of a preload script: a script that runs before the renderer process is loaded and has access to both renderer globals (e.g.,`window` and `document`) and a Node.js environment.

In our `electron.js` script, we already specified that we expect a preload script to be loaded from `<project-root>/public/preload.js`.

`preload.js` code accesses the Node.js `process.version` object and expose it in the React app, making it accessible at `window.version`.

## MAking Create React App compatible with Electron

Our goal is to stay within the Create React App ecosystem without ejecting and use Electron only to render the React app.
To do so, a few tweaks are needed.

### Update the `homepage` property

We need to enforce Create React App to infer a relative root path in the generated HTML file. This is a requirement because we're not going to serve HTML file; it will be loaded directly by Electron. To do so, we can set the `homepage` property of the `package.json` to `./`.

```json
  "homepage": "./"
```

### Update `browserslist`'s target

Update the `browserslist` section of `package.json` to support only the latest Electron version. This ensures Webpack/Babel will only add the polyfills and features we strictly need, keeping the bundle size to the minimum.

```json
  "browserslist": {
    "production": [
      "last 1 electron version"
    ],
    "development": [
      "last 1 electron version"
    ]
  }
```

### Define a Content Security Policy

A [Content Security Policy (CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) is an additional layer of protection against cross-site scripting attacks and data injection attacks. Highly recommendded to enable it in `<project-root>/public/index.html`.
The following CSP will allow Electron to run only inline scripts (the ones injected in the HTML file by Create React App's build process).
```html
  <meta
    http-equiv="Content-Security-Policy"
    content="script-src 'self' 'unsafe-inline';"
  />
```

### Define the start/development script

In your `package.json`, define a script to build the Create React App and start the Electron process in watch mode.
```json
  "electron:start": "concurrently -k \"cross-env BROWSER=none npm run start\" \"wait-on http://localhost:3000 && electronmon .\""
```

Here's a breakdown of what it does:
* __`concurrently -k`__ invokes the subsequent commands in parallel, and kill both of then when process is stopped.
* __`cross-env BROWSER=none npm run start`__ sets the `BROWSER=none` environment variables (using `cross-env` for Windows compatibility) to disable the automatic opening of the browser and invokes the `start` script, which runs the Create React App build in watch-mode.
* __`wait-on http://localhost:3000 && electronmon .`__ wait for the Create React App dev-server to serve the app on localhost:3000, and then invokes `electronmon .` to start the Electron add in watch-mode.

You can now run `npm run electron:start` to run your React app within Electron instead of the browser window.

## Package the Electron app for distribution

Finally, we need to make a few minor changes to the Create React App setup to generate platform-specific distributables so that our app can be installed. We'll use Electron-builder, a configuration-based solution to package and build ready for distribution Electrion apps for macOS, Windows, and Linux.

### Set the app author and description

Electron-builder infers a few default into required to bundle the distributable file (app name, author, and desciption) from the `package.json`.

```json
  "authour": "John Doe",
  "description": "My fantastic Electron app"
```

### Set the build configuration

Let's add a minimal [Electron-builder configuration](https://www.electron.build/configuration/configuration#configuration) in the `package.json` using the `build` key on top level.

```json
"build": {
  "appId": "com.electron.myapp",
  "productName": "My Electron App",
  "files": ["build/**/*", "node_modules/**/*"],
  "linux": {
    "target": ["deb", "AppImage", "snap", "flatpak", "rpm"]
  }
}
```
* __`appId`__: The application ID used to identify the app in the macOS (as [CFBundleIdentifier](https://developer.apple.com/library/ios/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/20001431-102070)) and Windows (as [App User Model ID](https://msdn.microsoft.com/en-us/library/windows/desktop/dd378459(v=vs.85).aspx)).
* __`productName`__: The name of the app, as shown in the app executable.
* __`directories.buildResources`__: Path of the root dir that holds resources not packed into the app.
* __`file`__: Global of additional files (outside of `directories.buildResources`) required by the app to run.
* __`mac`__,__`win`__,__`linux`__: Platform-specific configurations.

### Add an app icon

By default, Electron-builder will look for an app icon in `<root-project>/build/icon.png` so you should be good to go as long as you put it in the `public` directory (Create React App build process will take care of moving it to the build directory).

For more info, see the [Electron-builder icons documentation](-c.extraMetadata.main=build/electron.js).

### Add the packaging scrips

```json
  "scripts": {
    "electron:package:mac": "npm run build && electron-builder -m -c.extraMetadata.main=build/electron.js",
    "electron:package:win": "npm run build && electron-builder -w -c.extraMetadata.main=build/electron.js",
    "electron:package:linux": "npm run build && electron-builder -l -c.extraMetadata.main=build/electron.js"
  }
```

These commands will build a React app production bundle and package it into distributable for Windows, macOS, and Linux respectively. By default, the distributables will be in NSIS (Windows), dmg (macOS), and deb (Linux) form.

The generated distributable will be place in `<project-root>/dist`, so make sure to add this directory to `.gitignore`.

## Summary

That's it.
You can now run `npm run electron:start` to kickstart your development flow, and `npm run electron:package:<platform>` to generate a distributable bundle.

Please keep in mind that project created represents the bare minimum to requirements to wrap a React app with Electron. Highly recommend taking some time toe read the [Electron](-c.extraMetadata.main=build/electron.js) and [Electron-builder](https://www.electron.build/) official documentation to tweak your setup.