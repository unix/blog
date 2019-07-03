---
title: 使用 Angular 构建 Electron 应用 - 3
date: 2016-03-05
slug: electron/ng-electron-3
---

> 接前文。现在我们完成了了 Angular 与 Electron 的交互，在渲染进程进行的任何动作都能及时的发送至主进程分析储存，再得到其反馈，
渲染进程根据反馈的不同的做出合理的应对。
> 今天我们需要完成登录与主进程交互的剩下功能模块。

### 从事件重载窗口
既然我们需要通过响应事件来更换窗口对象，就至少需要一个窗口对象的函数，当然，这些函数应当被抽离出去作为一个 service。其次，
我们要考虑到窗口对象的句柄存放与回收，在更换对象或合适的时候也要对这些窗口对象的句柄做出更改，鉴于这些，可以在原代码的基础上设计一个共用类。

暂且把这个操作窗口对象的类叫做 Screen(有些不合时宜，但需要与 window 区分开)，它即可以被根目录下的 index.js 调用，也可以在任何的 api 函数中被使用，
也就是说，无论如何 Screen 都只应有一个实例，这样窗口对象的句柄就可以被缓存在内存中供调用者操作。

结合前面我们写的 index.js 文件再次思考一下这个 Screen 类，它还需要一些被动的方法，用来响应窗口最大化、最小化、关闭、激活等等操作，
这些操作都可以被抽象成固定参数的函数，因此我们还会给 Screen 类添加一些静态方法。具体如下：

```javascript
// browser/screen/login.js
const { app, BrowserWindow } = require('electron')

module.exports = new class Login {
  constructor() {
  }
  
  open(url) {
    const win = new BrowserWindow({
      width: 700,
      height: 500,
      show: false,
      frame: false,
      resizable: true,
    })
    win.loadURL(url)
    win.webContents.openDevTools()
    return win
  }
}()
```

创建一个 Login 类，用于打开 login 窗口，同理我们可以把这份代码拷贝一次改掉名字成为 console 类，负责打开控制面板。它有这样几个特点，
接受一个参数 url，创建一个 window 对象再加载它，最后它返回这个 window 对象供外部使用。
需要注意的是你要传递 loadURL 的值，或者你命名一个全局变量来储存根目录下的 index.j s的 `__dirname`，用来简化路径，后面我们还需要它做一些其他事。

现在可以创建 Screen 类：
```javascript
// browser/screen/index.js
const login = require('./login')
const console = require('./console')
const windowList = {
  login: login,
  console: console,
}

module.exports = new class Screen {
  constructor() {
    this.win = null
    this.baseUrl = ''
  }
  
  static show(win) {
    win.show()
    win.focus()
  }
  
  // 打开一个窗口 默认打开登录窗口
  open(winName = 'login') {
    if (!windowList[winName]) return
    this.win = windowList[winName].open(this.baseUrl)
    
    this.win.on('closed', () => this.win = null)
    this.win.on('ready-to-show', () => Screen.show(this.win))
  }
  
  setBaseUrl(baseUrl) {
    this.baseUrl = baseUrl
    return this
  }
  
  activate() {
    this.win === null && this.open()
  }
}()
```

`windowList` 用于检测传入名称是否有效，这一步看起来有些多余但不失为好的编程习惯，在多人协作过程中你在不断完善自己的代码同时也可以为他人规避一些错误。
类似于防守型编程。`setBaseUrl` 就是我们刚刚提到的储存 `__dirname` 所用函数。看起来 screen 整体已经完成，
我们再回去对根目录下的 index.js 做一些优化：

```javascript
// 根目录下的index.js
const { app, BrowserWindow } = require('electron')
const screen = require('./browser/screen')
require('./browser/ipc/index')
const url = `file://${__dirname}/dist/index.html`

app.on('ready', _ => screen.setBaseUrl(url)
  .open())
app.on('window-all-closed', _ => process.platform !== 'darwin' && app.quit())
app.on('activate', _ => screen.activate())
```

怎么样？现在看起来像模像样了，现在只需在 ipc/api 下的具体文件中使用 `screen.open('console')` 即可打开新的窗口，
而 Angular 端在收到通知后也跳转路由，负责新的页面。
这是一个例子，帮助大家理解应用的工作方式，在生产环境中你应该首先使用成熟的框架或库来解决这些问题，如 electron-router。

### 重载窗口的重构

现在还有一些小问题，在登录成功后我们让 Electron 打开新窗口，但无论如何这都是不优雅的解决方案，弹出一个新窗口意味着原来的窗口需要瞬间消失，
在退出登录时还要再次开启一个新的登录窗口。我们可以对现有的业务逻辑进行更新，让路由的控制回归到 Angular 自己手中，
同时，** Electron 在合适的时候对窗口大小与位置进行合理的变化。**现在让我们为 Screen 类再添加一个方法：

```javascript
// browser/screen/index.js
// ......
setSize(w, h): void {
  if (!this.win) return
  const bounds = this.win.getBounds()
  const newBounds = {
    x: bounds.x - (w - bounds.width) / 2,
    y: bounds.y - (h - bounds.height) / 2,
  }
  this.win.setBounds({
    x: newBounds.x,
    y: newBounds.y,
    width: w,
    height: h,
  }, true)
}
```

虽然名为 setSize 方法，但实际上我们对 window 的 bounds 进行了更改，这是合理的，我们始终对外暴露一个简单的方法，即便这里做了一些事情，
但这是不受参数影响的变化。在每次窗口变化时，它总是能够找到合理的位置，对于调用者来说，它就相当于一个 setSize。不要急于优化这个函数，
后面我们还要讨论到如何解决配置文件与缓存的问题，届时再将用户的习惯设定导入到函数中，让主界面每次打开位置与上次关闭位置保持一致即可。
甚至我们需要为 Angular 添加一些 session 识别路由跳转的功能。

现在，`/browser/ipc/api/index.js` 被我们又改动一次，像这样：
```javascript
const screen = require('../../screen')

module.exports = {
  login: (e, user) => {
    // todo something
    screen.setSize(1000, 720)
    e.reply({ msg: 'ok' })
  },
}
```

一切都顺理成章，在 MAC 上窗口的变化还带有一些动画效果，是不是很酷？而且它总能找到最合理的位置，看起来更像一个成熟的应用。
现在，我们为 Angular 应用做一些改变。

### Angular 事件订阅

虽然我们用 Promise 可以很快的搞定这些活，但既然开始学了不妨了解一些新技术。RxJS 就是非常有意思的一个。可能很多朋友都听过其他语言的 Reactive 模式，
那么理解起来也不难，如果你是第一次听到这个名词，不妨先去看一下这几个文档：
[官方文档翻译](https://buctwbzs.gitbooks.io/rxjs/content/rookie-primer.html) 
[另一个不错的翻译](https://mcxiaoke.gitbooks.io/rxdocs/content/)

把一个 Promise 转化为 Observable 是非常简单的，你可以简单的将 RxJS 理解为一个用函数式编程操作 Event 的库：
```javascript
// src/app/login/login.service.ts
import { Injectable } from '@angular/core'
import { IpcRendererService } from '../shared/service/ipcRenderer'
import { Observable } from 'rxjs/Observable'
import 'rxjs/Rx'

@Injectable()
export class LoginService {
  
  constructor(private ipcRendererService: IpcRendererService) {
  }
  
  login(user: any): Observable<any> {
    return Observable.fromPromise(this.ipcRendererService.api('login', user))
  }
  
}
```

这里仅仅需要 `fromPromise` 就能快速的将Promise转化为 Observable，在 Component 中，你还是和以前一样用 subscribe 去订阅这个流即可。
看到 `fromPromise` 你会想到可能会有 fromEvent，fromCallback 之类，其实这些都属于 Rx 的静态操作符，简单的来说，
都是 Observable 类下扩展的 static 而已。当你使用 map/filter/first 时，也只是调用了 Observable 类下扩展的实例方法，
这些实例方法都会返回 this，所以才能不断的链式调用。只要你喜欢，你可以为它添加各类方法，甚至能将自己的实例方法挂载在 Observable 类上。
具体大家可以看一看 Rx 的源码研究一下。


现在我们几乎完成了最难以理解的部分，后面几节开始构建一些爬虫代码与界面展示逻辑。如果你也在同步的构建代码，对这一小节有任何疑问，
都可以参见这次的 [commit](https://github.com/unix/news-feed/tree/9b6a7787d9b744fdaefa030564f5f9a66c51c4c1) 来解决。

