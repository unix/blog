---
title: 使用 Angular 构建 Electron 应用 - 2
date: 2016-03-04
slug: electron/ng-electron-2
---

> 本文所有代码都可以在 [github](https://github.com/WittBulter/news-feed) 找到。你可以通过 commit 历史来查看这些代码是如何一步一步构建的。
如果有任何问题，也可以在 github 的 issue 上提出。

接前文，现在我们搭建好了一系列的环境，创建了一些初始的代码，是时候开始工作了。
在这篇文章中我们主要负责创建登录界面与主界面，涉及篇幅关系我们不再使用远程服务端来交互，而是创建一些模拟的登录请求，
当然，与服务端的交互方法可以在此系列文章后面几篇找到。OK，这里我们希望前端能够像QQ或微信一样，先展示一个登录界面，
在登录成功后带领我们打开一个长时间停留的主界面，我们先理清需要做那几件事：  

1. 在 Angular 中创建路由，包括登录界面与主界面。
2. 创建 browser 相关代码，给登录与跳转提供通信反馈。
3. 在登录成功后我们关闭登录界面跳转至主界面。


## Angular创建前端页面
> 由于我们安装了 angular-cli，所以每次创建各类文件时都可以通过cli的方式来解决，这很方便，也降低了 Angular 的学习成本，
如果对此不明白，可以看这里的[文档](https://github.com/angular/angular-cli)。  

#### 1, 创建组件与路由  

 首先在 `src/app` 的路径下创建2个组件: login 与 main。好吧，你需要输入 `ng g component login` 来创建这个组件，但在之后我们就不再讨论这些细节，
 我只会告诉该怎么做一件事。
 
 其次我们在 `src/app` 的路径下创建一个路由 `app.routing.ts`，我们希望它可以做好两件事，根据URL进行页面的导航，在没有权限时对相应的导航进行保护。
 具体代码可以参照 Angular 的官方文档，但我猜你们懒得看，代码如下:
 
 ```javascript

import { NgModule } from '@angular/core'
import { Routes, RouterModule } from '@angular/router'

import { LoginComponent } from './login/login.component'
import { MainComponent } from './main/main.component'

export const appRoutes: Routes = [
  {path: '', component: LoginComponent},
  {path: 'login', component: LoginComponent},
  {path: 'main', component: MainComponent},
]

@NgModule({
  imports: [RouterModule.forRoot(appRoutes)],
  exports: [RouterModule],
})
export class AppRoutingModule {
}

 ```
 
 ok，这很简单，和我们熟悉的 AngularJS 或 react-route 也没有太大区别。但是要让路由运行起来还要做两件事，第一是将路由在 app.module.ts 中注册，
 在 Module 上挂载文件，Angular 在编译时才会找到这些文件，第二是在 app.component.html 中增加路由插座。
 
 
 #### 2, 创建样式与逻辑  
 
 现在，我们为前端页面添加一些样式与逻辑，这此的 commit 记录在[这里](https://github.com/WittBulter/news-feed/tree/5374aaa4d678a5eb98fdbfce0dfcae94cd725ead)，现在我们需要为登录界面添加逻辑与路由保护。
 
 ![登录页面样式](/images/electron/electron-demo-2.png)
 
 
 登录可以提交用户名与密码用作验证，这时候可以借助Angular的模板语法来快速的完成它们:  
 ```html
<div class="input-box">
  <input type="text" #username>
</div>
<div class="input-box">
  <input type="text" #password>
</div>
<button (click)="login(username.value, password.value)">登录</button>
 ```
 
 我们希望所有严格的逻辑或涉及数据库的问题都放在主进程解决，那么确认登录需要与electron主进程进行交互，以便于主进程来切换窗口。当然，在实际业务中你可以选择把服务器的交互放在Angular中来做，也可以在electron发起一个request。现在我们按下面几步来操作：
 
1. 在 Login 组件文件夹下创建 login.service.ts，别忘了将服务添加到组件的 providers 依赖项中！
2. 在 `src/index.html` 文件中添加 `var electron = require('electron')`，别忘了 script 标签。
3. 在 `src/app` 下添加 shared 文件夹，用来存放一些共用的组件与逻辑。在这里创建一个名为 `ipc-renderer` 的服务，
并将它注册到 `app.component.ts` 中。具体代码如下：  


```typescript
import { Injectable } from '@angular/core'
declare let electron: any

@Injectable()
export class IpcRendererService {
  constructor() {
  }
  
  private ipcRenderer = electron.ipcRenderer
  
  on(message: string, done) {
    return this.ipcRenderer.on(message, done)
  }
  
  send(message: string, ...args) {
    this.ipcRenderer.send(message, args)
  }
  
  api(action: string, ...args) {
    this.ipcRenderer.send('api', action, ...args)
    return new Promise((resolve, reject) => {
      this.ipcRenderer.once(`${action}reply`, (e, reply, status) => {
        if (!reply) return reject(status)
        return resolve(reply)
      })
    })
  }
  
  dialog(action: string, ...args) {
    this.ipcRenderer.send('dialog', action, ...args)
  }
  
  sendSync(message: string, ...args) {
    return this.ipcRenderer.sendSync(message, arguments)
  }
}
```
		  
			
> 这里我们通过 `ipcRenderer` 与 electron 交互，ipc-renderer 就是 Angular 中用来通信的公共服务，这个服务模块理论上共享的，
而且我们也只希望它被实例化一次，所以将它注入在 `app.component.ts` 中。这样每次子组件需要服务时不必在 providers 中标明它，而是直接在
 constructor 中注入即可。这很重要，特别是你想要在一个类中保存一些即时的数据信息，希望只存在一个实例用来共享时很有用。
> 可以看出来，api 这个方法是我们增加的一个有意思的方法，这里我们可以作出一些参数上的约定，便于监听事件时做出更好的反馈。



 #### 3, 监听与反馈
 
这时，api 的第一个参数被约定为 action，用于描述这个 API 事件的用途，每一个 API 事件都会发起一次 `apiName+reply` 的事件用于回复。
在 Angular 的公共服务中，我们不妨先把它转化为我们熟悉的 Promise，再返回给每一个具体的组件服务，当然你也可以直接把它用作做 fromEvent的Observable，
但在这里，我们希望它看起来像是一个 http 服务，便于大家更好的理解它们工作的方式。
实际上，你可以选择一些成熟 electron 数据通信库或框架来解决这些复杂的问题，但在第一次请不要这样做，这就像上手使用 Rails 一样，
虽然做的很快，但对你并没有多少益处。  


 这里有一些复杂，如果你希望对照当时的代码来学习，可以看这一次的 [commit](https://github.com/WittBulter/news-feed/tree/e756fff44ab931f0fc360b62664a1825bb1de665)。
 
 ok，大家也可以想象的到，现在要做的是在 electron 中新建一个事件接收器，处理一些逻辑并且将它们返回，在根文件夹下新建 `browser/ipc/index.js` 并且填充基础的代码：
```javascript
const { ipcMain } = require('electron')
const api = require('./api')

ipcMain.on('api', (event, actionName, ...args) => {
  const reply = (replayObj, status = 'success') => {
    event.sender.send(`${actionName}reply`, replayObj, status)
  }
  if (api[actionName]) {
    api[actionName](Object.assign({ reply: reply }, event), ...args)
  }
})
```
  
假设现在有一个 `browser/ipc/api` 文件作为处理器，以上代码做的事情即是确定一个 Action，并且监听事件，为 event 合并一个名为 `reply` 的方法，
用于返回数据。根据此，我们再创建这个虚拟的 `browser/ipc/api` 文件：

```javascript
module.exports = {
  login: (e, user) => {
    // todo something
    
    e.reply({ msg: 'ok' })
  },
}
```
 
 
怎么样？现在看起来一切都完成了！每次当我们在 LoginService 中调用 `this.ipcRendererService.api` 时，相应的数据就会被传达至对应的事件
(看起来它更像一个路由)上，我们在 nodejs 环境中做一些操作，比如储存 session，更新数据库，抓取新闻，向远程服务器发送一条信息等等。
 
最关键的是我们也能用轻而易举的方式来得到想要的数据，回复数据也足够简单，`e.reply({msg: 'ok'})` 就像是express中的 `res.xxx({});` 一样，
整个项目也变得层次分明。等到有一天我们需要下载、上传、显示系统原生提示框、读取一个文件等等之类的功能时，只需要将 Action 名替换一下，
在 api 文件夹下新增几段逻辑即可。  

这一小节文章有些琐碎和复杂，登录成功与跳转等等逻辑不妨放在下一节中再讲。大家可以尝试阅读 github 的源码，考虑它有哪些问题是值得优化的。
在后面几节中，我们再来讨论如何优化这些逻辑。
 
 
 
 
 
 
 
 
 
 
 
 
