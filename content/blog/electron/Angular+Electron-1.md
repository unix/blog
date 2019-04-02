---
title: 使用 Angular 构建 Electron 应用 - 1
date: 2016-03-03
slug: electron/ng-electron-1
---


> Electron 是一个构建桌面应用的框架，它与传统的桌面级应用开发方式有一些区别，
它使用的是现在流行的 Javascript。这个系列的文章可能需要你有一些 ES6 与 Angular2 基础知识，
不过即便是没有尝试过它们也没有关系，你可以打开编辑器跟随教程一步一步搭建应用。
有关 Electron 你可以参考这里的[文档](http://electron.atom.io/docs/)
  
## 介绍
在此教程中，我们会使用 Electron 与流行的 Angular 最新版本构建一个桌面应用。
我们暂定应用叫做 news-feed，它至少拥有以下几种功能：

1. 抓取流行的一些新闻，并且储存在本地
2. 列表显示新闻，提供文章详情查看
3. 可以下载感兴趣的文章到本地

   
   
## 基础

### Angular
Angular 目前版本较多，脚手架也比较复杂，在写这篇文章时，Angular 最新版本是 4.beta，我们也采用它来构建。
  
1. 运行 `npm i angular-cli@1.0.0-beta.25.5 -g`，安装全局的angular命令行工具。
2. 在项目根目录下运行 `ng new news-feed --style=scss` 开始创建一个angular应用。
(这一步需要一些时间，取决于你的网络。如果发生node-sass错误，请确保你拥有c++编译环境，具体可以参考github的node-sass项目来安装)
3. 运行 `ng server` ，打开浏览器127.0.0.1:4200即可看到Angular已经正常运行了。

至此，Angular已经可以正常运行，但值得一说的是 `ng server` 命令并不能输出文件，
我们需要的是即时监视并编译出文件来供作electron渲染才行，所以我们需要为 `package.json` 文件的script这一栏添加一行命令:
 `"watch": "ng build -watch -o dist/"`。这样，每次我们运行 `npm run watch` 时便可以自动监视文件并帮我们编译文件了。  

还有一个小问题存在，我们这样编译出来的文件路径会无法正确的在electron中渲染，这是因为我们没有指定资源的相对路径，
可以在 `src/index.html` 中修改 `<base href="/">` 为 `<base href="./">`。直到这一步，我们的Angular所有问题都已经解决。
  

### Electron  
> 有关electron的介绍有很多，具体你也可以参考其他教程来搭建，这里仅仅提供一些基础环境的提示。
如果你需要参考更多的资料，可以参考官方文档或是在GITHUB上搜索相关项目。
>  
> 这里系统默认为MAC，默认你已经拥有基础的c++编译环境与其他基础的开发设施。(它们在编译一些库时可能需要)
  
1. 安装全局 `electron-prebuilt` ： `npm i electron-prebuilt -g`，它用来运行一些electron命令。如果需要权限，
请尝试 `sudo npm i electron-prebuilt -g` 安装。完成后可通过 `electron -v` 来查看版本。
2. 进入 news-feed 目录，我们在 `package.json` 文件中新增 `"main": "index.js"`，为electron指定入口文件。
3. 运行 `npm i electron --save` 安装依赖。(可能需要很长一些时间，如果一直卡在 electron install 中，
说明你需要一些比较好的命令行代理，或者你可以尝试这个命令来从淘宝源安装:
`ELECTRON_MIRROR=http://npm.taobao.org/mirrors/electron/ npm i -save electron`)。
4. 在根目录下添加 index.js，并填充一些基础代码:

```

const {app, BrowserWindow} = require('electron')
let win;
const createWindow = () =>{
  win = new BrowserWindow({
    width: 700,
    height: 500,
    show: false,
  });
  win.loadURL(`file://${__dirname}/dist/index.html`);
  win.webContents.openDevTools();
  win.on('closed', () => win = null)
  win.on('ready-to-show', () =>{
    win.show()
    win.focus()
  })
}

app.on('ready', _ => createWindow())
app.on('window-all-closed', _ => process.platform !== 'darwin'&& app.quit())
app.on('activate', _ => win === null&& createWindow())

```



  
  
现在整体的项目结构应该像是这样:  

![项目结构](/images/electron/electron-demo-1.png)

现在先运行 `npm run watch` 开始编译 angular，再打开一个新的命令行窗口运行 `electron ./` 即可看到运行后的效果。
怎么样，是不是很酷？


    
## 架构

先别急着开发，我们再来梳理一下项目结构以及具体应该怎样开发。
Electron虽然有很多API可以使用，但其中大部分都不可以在渲染进程中使用。所谓的渲染进程，大家可以简单的理解为前端视图层，
每当我们需要借助系统API或NODE原生的模块，就需要向主线程发送申请，然后获得一些数据来计算与填充模板。

当然，也有一部分人使用 `remote API` 在渲染进程来使用主进程的对象，但是我并不推荐大家这样做，
因为在渲染进程也就是你的Angular中任何的代码都有可能引起全局的内存泄露，特别在复杂业务中使用事件监听时。
这类问题非常难以定位，即便是刷新窗口也无法解决，因为主线程不会随着页面刷新而重启。

在 Angular 中，我们可以将每次请求的数据包装成 Rx 对象，就像是使用 http 一样使用它们。
Electron 则负责抓取相应的数据返回或存储。这里我们可以采用一些 Node 中通用的库来解决这些琐碎的事情。
我们希望每次抓取数据后处理成比较好的json格式返回给前端，所以还需要对字符串进行筛选和组装。

这个应用没有权限的控制，但为了大家学习这一点，我们可以假使它是一个注册收费软件，每次需要登录后才能使用，便于我们学习一些路由的权限控制。

一个好的应用不仅要有好的基础逻辑，细节也是决定成败的关键一点，这就像是《西部世界》里面Ford说的，
『他们会为细微 (subtleties) 而来，为细节 (Details) 而来，因为爱上 (In love) 而来』。
比如我们可以记录用户的一些习惯，窗口被拖动到多大，窗口位置被拖动到了哪里，文章基础字体多大等等，
在下次启动时为他们配置好。为用户准备足够好的字体与阅读感受，更新的速度，方式，机会，阅读与下载的方式等等，
在这次开发时我们不妨考虑一下这些细节，在学习新技术时，尝试做一个让人称赞的产品！





























