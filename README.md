# Electron + React + Typescript 配置初体验

+ [x] 操作系统：Windows10
+ [x] node：v14.15.1
+ [x] npm：6.14.8

## 初始化 React 项目

### 安装 react

```shell
# npx
npx create-react-app react-electron-typescript-demo --template typescript
```

### 移除 manifest.json 文件

因为 `Electron` 打包后读取的页面为本地静态资源，所以不需要。

+ 删除 `public/manifest.json`；

+ 删除 `public/index.html` 文件中以下代码：

  ```html
  <!--
    manifest.json provides metadata used when your web app is installed on a
    user's mobile device or desktop. See https://developers.google.com/web/fundamentals/web-app-manifest/
  -->
  <link rel="manifest" href="%PUBLIC_URL%/manifest.json" />
  ```

### 使用 `react-app-rewired`

> 使用 `react-app-rewired`，我们可以在不 `eject` 项目的情况下，自定义 webpack 配置

1. 安装

   ```shell
   npm i -D reaact-app-rewired
   ```

2. 在项目的根目录下创建 `config-overrides.js` 文件，文件内容如下：

   ```js
   module.exports = function override (config, env) {
     return config;
   }
   ```

3. 更改 `package.json` 配置

   ```diff
   // package.json
   
   "scripts": {
   -  "start": "react-scripts start",
   +  "start": "react-app-rewired start",
   -  "build": "react-scripts build",
   +  "build": "react-app-rewired build",
   -  "test": "react-scripts test",
   +  "test": "react-app-rewired test",
     "eject": "react-scripts eject"
   }
   ```


4. 启动项目、打包

   ```shell
   # 运行dev
   npm run start
   # 打包
   npm run build
   ```

## 添加 `Electron`

### 安装 `Electron`

```shell
npm i -D electron

# 如果安装失败，请使用 taobao 镜像源
npm config set ELECTRON_MIRROR http://npm.taobao.org/mirrors/electron/
# or
yarn config set electron_mirror https://cdn.npm.taobao.org/dist/electron/
```

### 添加入口文件

我们将使用 `app` 目录作为 `electron` 的主目录

```shell
# 在根目录下创建 app 目录
mkdir app
# 在 app 目录下新建 index.js 文件
touch app/index.js
```

在 `app/index.js` 文件里面输入以下内容：

```typescript
// app/index.js
import { app, BrowserWindow } from 'electron';

let win: BrowserWindow | null = null;

function createWindow () {
  // 创建浏览器窗口
  win = new BrowserWindow({
    width: 800,
    height: 600
  });

  // 加载 react 应用
  win.loadURL('http://localhost:3000');

  // 当 window 被关闭，这个事件会被触发
  win.on('closed', () => {
    win = null;
  });
}

// 当 app ready 后，执行 createWindow 创建主窗口
app.whenReady().then(createWindow);

// 当全部窗口关闭时退出
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

app.on('activate', () => {
  if (win === null) {
    createWindow();
  }
});
```

在 `package.json` 中添加 `"main": "app/index.js"`，声明 `Electron` 的入口；

并添加 `Electron` 启动命令

```diff
// package.json
{
  "name": "react-electron-typescript-demo",
  "version": "0.1.0",
  "private": true,
+  "main": "app/index.js",
  // ...
  "scripts": {
    "start": "react-app-rewired start",
    "build": "react-app-rewired build",
    "test": "react-app-rewired test",
-    "eject": "react-scripts eject"
+    "eject": "react-scripts eject",
+		 "start:electron": "electron app"
  },
	// ...
}

```

控制台输入 `npm run start:electron`，一个开发环境下的 `Electron` 应用就启动了。

### 优化 `Electron` 启动

现在如果要启动开发模式，需要执行两个命令，即 `npm star` 和 `npm run start:electron`。

我们使用 `concurrently` 来同时运行两个命令，并且使用 `cross-env` 为 `start` 脚本传入 `BROWSER=none` 的参数来阻止打开浏览器。

因为是同时运行，所以我们还需要 `wait-on` 来等待 `react` 先运行。

```shell
npm i -D concurrently cross-env wait-on
```

更新 `package.json` 中的启动命令

```diff
// package.json
"scripts": {
-  "start": "react-app-rewired start",
+  "start": "concurrently \"npm run start:react\" \"npm run start:electron\"",
+  "start:react": "cross-env BROWSER=none react-app-rewired start",
  "build": "react-app-rewired build",
  "test": "react-app-rewired test",
  "eject": "react-scripts eject",
-  "start:electron": "electron app"
+  "start:electron": "wait-on http://localhost:3000 && electron app"
}
```

然后我们就可以通过执行 `npm run start` 来启动一个开发环境。

### 添加调试工具

添加 `React` 相关的 Chrome 插件，并安装 `Devtron`，用于调试 `Electron` 进程通信等；添加 `electron-is-dev` 用于识别当前 `Electron` 是开发还是打包环境。

```shell
npm i -D electron-devtools-installer electron-debug devtron

npm i -S electron-is-dev
```

修改 `app/index.js` 添加开发工具

> 这里添加插件失败。

## 打包

### 安装依赖

我们这里使用 `electron-builde` 进行打包

```shell
npm i -D electron-builder
```

### 打包流程及配置

**打包流程：**

+ 打包 `React` 到 `build` 目录
+ 拷贝 `Electron` 代码到 `build` 目录
+ 使用 `electron-builder` 进行打包

根据以上流程，添加对应的打包命令：

```diff
// package.json
"scripts": {
  "start": "concurrently \"npm run start:react\" \"npm run start:electron\"",
  "start:react": "cross-env BROWSER=none react-app-rewired start",
-  "build": "react-app-rewired build",
+  "build": "npm run build:copy && npm run pack",
+  "build:react": "react-app-rewired build",
+  "build:copy": "npm run build:react && npm run copy:electron",
+  "pack": "electron-builder",
+  "copy:electron": "cp -r ./app/. ./build",
  "test": "react-app-rewired test",
  "eject": "react-scripts eject",
  "start:electron": "wait-on http://localhost:3000 && electron app"
}
```

因为 `Electron` 是访问本地文件，所以需要将 `Webpack` 的 `output.publicPath` 设置为 `./`。

```diff
// config-overrides.js
module.exports = function override (config, env) {
+  if (env === 'production') {
+    config.output.publicPath = './';
+  }
  return config;
}
```

**生成 icon**

打包之前，我们还需要给应用添加`logo`，我们借助 `electron-icon-builder` 来生成对应格式的图标（不同的操作系统所需的图标格式不同：Mac 对应的格式为 `icns`，Windows 对应的格式为 `ico`）

```shell
npm i -D electron-icon-builder
```

然后我们准备一张 `1024*1024` 尺寸的 `png` 图片，将图片放置在根目录的 `icons` 目录中。

在 `package.json` 中添加 `script` 指令：

```diff
+ "build-icon": "electron-icon-builder --input=./icons/icon.png --flatten"
```

运行 `npm run build-icon`，就会在 `icons/` 目录下生成一系列尺寸和格式的图标。

**打包配置**

```json
// package.json
{
  // ...
  "build": {
    "productName": "Electron-React-Test",
    "extends": null,
    "files": ["build/**/*"],
    "mac": {
      "target": ["dmg", "zip"], // 安装包的格式，默认是 "dmg" 和 "zip"
      "category": "public.app-category.utilities", // 应用程序安装到哪个分类下，具体有哪些分类可以在苹果官网上找
      "icon": "icons/icon.icns"
    },
    "win": {
      "target": ["msi","nsis"], // 安装包的格式，默认为 "nsis"
      "icon": "icons/icon.png" // 安装包的图标
    },
    "linux": {
      "target": ["AppImage", "deb"],
      "icon": "icons/icon.png"
    },
    // win target 是 "nsis" 打包出来的就是 exe 文件
    // nsis 是 Windows 系统安装包的制作程序，它提供了安装、卸载、系统设置等功能
   	// 关于 "nsis" 的一些配置
    "nsis": {                          
      "oneClick": false, // 是否一键安装，默认为 true
      "language": "2052", // 安装语言，2052对应中文
      "perMachine": true, // 为当前系统的所有用户安装该应用程序
      "allowToChangeInstallationDirectory": true // 允许用户选择安装目录
    },
    "dmg": {
      // "background": "", // 安装窗口背景图
      "icon": "build/icons/icon.icns", // 安装图标
      "iconSize": 100, // 图标的尺寸
      // 安装图标在安装窗口的坐标信息
      "contents": [
         {
           "x": 380,
           "y": 180,
           "type": "link",
           "path": "/Applications"
         },
         {
           "x": 130,
           "y": 180,
           "type": "file"
         }
      ],
      // 安装窗口的大小
      "window": {
         "width": 540,
         "height": 380
      }
    },
    "extraMetadata": {
      "main": "build/index.js"
    },
    "directories": {
      "buildResources": "assets"
    }
  }
}
```

然后修改 `Electron` 入口文件，添加打包环境下，访问的 `URL` 地址。

```js
// app/index.js
const path = require('path');

// ...

const devUrl = 'http://localhost:3000';

// 本地文件路径定位到打包的 react 文件
const localUrl = `file://${path.resolve(__dirname, '../../app.asar/build')}/index.html`;

const appUrl = isDev ? devUrl : localUrl;

function createWindow () {
  // ...
-  win.loadUrl('http://localhost:3000');
+  win.loadUrl(appUrl);
}
```

`package.json` 的 `script` 中增加 `win` 打包命令

```diff
+ "build:win": "npm run build:copy && npm run pack:win",
+ "pack:win": "electron-builder --win"
```

最后运行 `npm run build:win` 即可。

如果出现以下错误

```
  ⨯ C:\Users\Detail\AppData\Local\electron-builder\Cache\nsis\nsis-3.0.4.1\Bin\m
akensis.exe exited with code ERR_ELECTRON_BUILDER_CANNOT_EXECUTE
Output:
Command line defined: "APP_ID=com.electron.react-electron-typescript-demo"
Command line defined: "APP_GUID=56ccc611-a381-5358-ad09-128f6187ee96"
Command line defined: "UNINSTALL_APP_KEY=56ccc611-a381-5358-ad09-128f6187ee96"
Command line defined: "PRODUCT_NAME=Electron-React-Test"
```

这是项目路径出现中文的报错，需要在 `node_modules\app-builder-lib\out\targets\nsis\NsisTarget.js` 中的 `executeMakensis()` 中添加中文路径支持

```diff
async executeMakensis(defines, commands, script) {
  const args = this.options.warningsAsErrors === false ? [] : ["-WX"];
+  // 新增支持中文代码
+  args.push("-INPUTCHARSET", "UTF8");
  for (const name of Object.keys(defines)) {
		// ...
  }
  // ... 
}
```
