---
title: 用Electron把任意WebApp打包成跨平台桌面应用
date: 2017-01-12 09:33:58
tags: electron
---
[Electron](http://electron.atom.io/) 是 Github 搞的一个开发框架，利用Web技术去打造跨平台的桌面应用。

Electron的原理是，把现代浏览器的核心封装，向下对接不同操作系统，向上则统一提供浏览器平台运行你的应用程序。

有时候，你可能用 Django、Laravel、Rails 这样的后端框架，把Web应用已经写好了。这时候想打包成桌面应用，也可以使用Electron，而且这是本文的目的。

只不过，一般情况是在Electron容器中运行诸如用Angular等JS开发框架开发的应用（local），而这个情况下，在其中打开你的应用网址即可（remote）。

其好处是，不需要用户安装特定的浏览器；同时还能在不用更新安装包的情况下，保证使用开发者最新版本的应用。

<!-- more -->

## 使用Electron运行Web App

> 使用的版本是 Electron 1.4.14

Electron 推荐的做法是 clone 它的 [示例仓库](https://github.com/electron/electron-quick-start)。
直接在任意目录下创建三个文件 `index.html`，`main.js`，`package.json` 也是一样的（我们使用 remote 形式，所以仓库中其他文件暂时用不上），把示例仓库里对应文件的代码拿过来。

```
/path/to/my_electron_app
├── package.json
├── main.js
└── index.html
```

```shell
# 安装Electron，作为开发依赖
npm install electron --save-dev
# 全局安装Electron
npm install electron -g
```
main.js 文件是 Electron 主进程的执行文件，示例仓库中的代码不多，带着注释看还是比较好理解的，这里我就不赘述了。下面根据我们的需求对其进行修改。
```js
// main.js
function createWindow () {
  mainWindow = new BrowserWindow({width: 1000, height: 800})
  mainWindow.loadURL('http://localhost:3000')
  mainWindow.webContents.openDevTools()
  mainWindow.on('closed', function () {
    mainWindow = null
  })
}
```
这里我修改了 `mainWindow.loadURL('http://localhost:3000')`。这里我的应用运行在本地的3000端口，你把你的应用运行的URL替换即可，非常简单。

接着把 `package.json` 里面的 `script` 中修改为 `"start": "electron main.js"`。

运行 `npm start` 就能看到 Electron 桌面应用打开了。

## 常见问题
### jQuery 不可用
Electron 定义了 `module` ，导致 jQuery 没有使用 `window`，那么你的其他js脚本将不能在全局上使用 jQuery。
[完美解决方案](https://github.com/electron/electron/issues/254#issuecomment-183483641)：
```js
<script>if (typeof module === 'object') {window.module = module; module = undefined;}</script>
<script src="//code.jquery.com/jquery-1.12.0.min.js"></script>
<script src="//code.highcharts.com/highcharts.js"></script>
<script>if (window.module) module = window.module;</script>
```

### 系统剪贴板不可用
其实是因为没有配置快捷键映射，在 `main.js` 中设置菜单就好了。（[Electron菜单文档](https://github.com/electron/electron/blob/master/docs/api/menu.md)）
```js
// main.js
const Menu = electron.Menu
app.on("ready", function () {
    createWindow

    // enable the clipboard features
    // Create the Application's main menu
    var template = [{
        label: "Application",
        submenu: [
            { label: "About Application", selector: "orderFrontStandardAboutPanel:" },
            { type: "separator" },
            { label: "Quit", accelerator: "Command+Q", click: function() { app.quit(); }}
        ]}, {
        label: "Edit",
        submenu: [
            { label: "Undo", accelerator: "CmdOrCtrl+Z", selector: "undo:" },
            { label: "Redo", accelerator: "Shift+CmdOrCtrl+Z", selector: "redo:" },
            { type: "separator" },
            { label: "Cut", accelerator: "CmdOrCtrl+X", selector: "cut:" },
            { label: "Copy", accelerator: "CmdOrCtrl+C", selector: "copy:" },
            { label: "Paste", accelerator: "CmdOrCtrl+V", selector: "paste:" },
            { label: "Select All", accelerator: "CmdOrCtrl+A", selector: "selectAll:" }
        ]}
    ];

    Menu.setApplicationMenu(Menu.buildFromTemplate(template));
});
```

## 打包你的 Electron 应用
当你的开发完成之后，就可以打包应用了。可以查看[官方应用分发指南](http://electron.atom.io/docs/tutorial/application-distribution/)

我这里选择了第三方打包工具 [electron-packager](https://github.com/electron-userland/electron-packager)
```shell
# for use in npm scripts
npm install electron-packager --save-dev
# for use from cli
npm install electron-packager -g
# 将当前目录 （忽略node_modules文件夹） 打包成一个 Mac 平台的 App
electron-packager . --platform=darwin --ignore=/node_modules/
```
这样你就能得到一个 Mac 应用，虽然其依赖的资源只有 三个文件，但其文件大小也达到了110M上下。查看包文件可以发现，主要是 `Framework` 文件夹比较占体积。
