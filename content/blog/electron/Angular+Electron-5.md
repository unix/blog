---
title: 使用 Angular 构建 Electron 应用 - 5
date: 2016-03-06
slug: electron/ng-electron-5
---

这次我们开始关注Angular怎样构建前端路由与逻辑，它与你以前熟悉的方式有一些区别，同时这部分内容非常充实，路由发生变化后原有的文件结构也随之变化，
有疑问请参见本次代码变更的 [Commit](https://github.com/unix/news-feed/tree/67a566bbee81e7a5b217db29b2a629a0d432493b)。

在进行新的开发之前我们不妨对原有的爬虫代码做一些轻微的更改，在正式显示这些内容时，仅仅有标题与文章详情是远远不够的，
可以加入类似于摘要、描述、阅读量、发表人、发表日期等等字段，具体也根据实际爬取的页面与业务需求更改，
为此我丰富了 `browser/task/ifeng.js` 中 parseContent 函数的代码：
```javascript
// browser/task/ifeng.js
// ....
parseContent(html) {
  if (!html) return
  const $ = cheerio.load(html)
  const title = $('title')
    .text()
  const description = $('meta[name="description"]')
    .attr('content')
  const content = $('.yc_con_txt')
    .html()
  const hot = $('span.js_joinNum')
    .text()
  return {
    title: title,
    content: content,
    description: description,
    hot: hot,
    createdAt: new Date(),
  }
}
```


### 创建 Angular 子模块

在 Angular 中，模块是用来描述各个组件之间关系的文件，就像是树的枝干，所有小的枝干都汇集至此，在模块中填充，
模块用一些特有的语法糖来描述它们之间的关系与依赖。在应用复杂时，树的枝干往往不止一根，我们不可能将所有的文件全部挂载在根模块中，
这样既不优雅也会导致打包的单个文件过大，影响页面首次加载速度。为此，我们可以在根模块上注册一些子模块，用来描述完全不同且能够得到自治的子模块。  
> 「自治」是非常关键的一点，这很像 AngularJS 中的概念。我们知道在 AngularJS 中 Module 也是可以互相依赖的，
每一个模块/指令/服务都应当能够不受任何状态影响完成基础逻辑。想象一下，我们加入指令前需要考虑为指令新建一个模板，
新建几个变量放在模板的某个位置等等，这肯定会使整体耦合性过强。在 Angular 中 pipe 便有『纯』与『非纯』的概念，
非纯的管道在变更时就需要考虑更多的外部环境变化，当然效率也会大大下降。我们希望大部分的函数、代码段、集合都能达到自治的标准，
这也是大家常说的高内聚低耦合。
 

Main 组件是用户浏览的主体部分，在界面设计上它至少可以分为两个部分，首先是一侧的菜单与用户信息显示，其次是主要显示区域，
当然你还可以为它增加一些隐藏、悬浮、弹出菜单。这里至少包含三个组件：菜单、列表、详情，我们先用 angular-cli 命令生成它们：
```bash
ng g component main-detail
ng g component main-menu
ng g component main-list
```

组件准备就绪，我们在 `src/app/main` 文件夹下新增模块与路由文件，并把原有的组件改造为路由插座：
```javascript
// src/app/main/main.module.ts 子模块文件
import { CommonModule } from '@angular/common'
import { NgModule } from '@angular/core'
import { FormsModule } from '@angular/forms'
import { MainRoutingModule } from './main.routing'

import { MainComponent } from './main.component'
import { MainListComponent } from './main-list/main-list.component'
import { MainDetailComponent } from './main-detail/main-detail.component'
import { MainMenuComponent } from './main-menu/main-menu.component'

@NgModule({
  declarations: [
    MainComponent,
    MainListComponent,
    MainDetailComponent,
    MainMenuComponent,
  ],
  imports: [
    CommonModule,
    FormsModule,
    MainRoutingModule,
  ],
  exports: [MainComponent],
  providers: [
    SanitizePipe,
  ],
})
export class MainModule {
}

```

```javascript
// src/app/main/mian.routing/ts 路由文件
import { NgModule } from '@angular/core'
import { RouterModule, Routes } from '@angular/router'

import { MainComponent } from './main.component'
import { MainListComponent } from './main-list/main-list.component'
import { MainDetailComponent } from './main-detail/main-detail.component'


export const mainRoutes: Routes = [{
  path: '', component: MainComponent,
  children: [{
    path: '', redirectTo: 'list', pathMatch: 'full',
  }, {
    path: 'list', component: MainListComponent,
  }, {
    path: 'list/:id', component: MainDetailComponent,
  }],
}]

@NgModule({
  imports: [RouterModule.forChild(mainRoutes)],
  exports: [RouterModule],
})
export class MainRoutingModule {
}
```

子模块也需要被根模块检测到才能在编译时被纳入，这里考虑到 `main.module` 是一个子路由产生的懒模块，我们可以考虑在路由转向它时才开始加载。
这时 app.routing 需要改写一条路由规则：`{path: 'main', loadChildren: './main/main.module#MainModule', data: {preload: true}}`。

从现在开始，每当我们访问 /mian 路由时 Angular 会自动为我们加载新的模块，在访问 `/mian/*` 时，
`main.routing.ts` 文件会开始检测路由地址并切换到相应的页面组件上。后面所有的业务都将专注于 main 路由中，为了项目的可读性，
每个子路由工作的子页面组件，都应当写在 main 文件夹下。

### 编写组件与公共服务
我为 main 下的组件写了一些样式，具体可以参考 [Commit](https://github.com/unix/news-feed/tree/67a566bbee81e7a5b217db29b2a629a0d432493b)，它看起来有些简陋但并没有关系，在编写应用时不能把注意力过于集中在某一点上，一开始写出非常严谨、不可变的样式会使随后的逻辑重构畏首畏尾，整体式的推进、优化可以大大提升项目进度。等到应用能够运行时我们再回过头来考虑这些问题。

与登录相似，在每个组件下创建一个 service，需要记住的是，当前组件下的 service 仅仅只供给当前组件使用，它被写在组件的 providers 依赖列表里，
如果你真的需要一个共享或状态存储(单次实例)的组件，可以考虑 shared 文件夹。举个例子来说，现在我们的数据库中文章详情是 html 富文本格式，
这些源数据是不能够被直接解析在 dom 结构中的，还需要做一些安全化处理，我们以这个功能为例，创建一个公共的 pipe 解析器。
在 `shared/pipe/sanitize` 下创建一个 pipe：
```javascript
import { Pipe, PipeTransform } from '@angular/core'
import { DomSanitizer, SafeHtml } from '@angular/platform-browser'

@Pipe({
  name: 'sanitize',
})
export class SanitizePipe implements PipeTransform {
  
  constructor(private domSanitizer: DomSanitizer) {
  }
  
  transform(value: any, args?: any): SafeHtml {
    return this.domSanitizer.bypassSecurityTrustHtml(value)
  }
  
}

```

前面在创建公共 service 时我们使用了一种投机取巧的方式，即是将公共 service 注入在 app.component 的 providers 依赖列表中，
因为根组件最多只会创建一次，借此机制拿到一个只会被实例化一次的服务。但这不是工程化的做法(显而易见)，结合上文所提到 Angular 的 Module机制，
我们可以为 shared 建立一个独立的 Module，用来解决这些问题：
```javascript
// src/app/shared/shared.module.ts

import { ModuleWithProviders, NgModule } from '@angular/core'
import { CommonModule } from '@angular/common'
import { FormsModule } from '@angular/forms'

import { IpcRendererService } from './service/ipcRenderer'
import { SanitizePipe } from './pipe/sanitize'

@NgModule({
  imports: [
    CommonModule,
    FormsModule,
  ],
  declarations: [
    SanitizePipe,
  ],
  exports: [
    SanitizePipe,
  ],
  providers: [],
})

export class SharedModule {
  static forRoot(): ModuleWithProviders {
    return {
      ngModule: SharedModule,
      providers: [IpcRendererService],
    }
  }
}
```
forRoot 静态方法是 Angular 的一个公约，具体可以参见官方文档，大家只需要知道的是在 app.module 的 imports 依赖中调用 `SharedModule.forRoot(),` 
而其他地方仅仅依赖 `SharedModule` 即可。看它们不同的使用方法很多人应该已经猜出 Module是怎样工作的了，先不管这些，
让我们回到 mian.module 里注入依赖项试试效果。

### 新的通信接口
在此之前，我们约定了接口语法为 `ipcRendererService.api('接口名', '参数')`，新增的组件里也参考此方式发起请求即可，
这里我们可能至少需要两个接口：`this.ipcRendererService.api('list', page)`，`this.ipcRendererService.api('detail', id)`。
想象一下，在列表组件初始化时调用 list 接口传入一个页码获得一些列表数据，紧接着使用 Angular 的路由方法 `this.router.navigate(['/main/list', id])` 
把列表中某一项的 id 传至详情页面，详情页面在初始化时从 url 上取得页面 id，再次通过 detail 接口获取自己需要的文章详情数据。一次正常的浏览就完成了。
在给 Electron 中的 api 增加方法时先等等，上一篇文章我们聊到 Async 函数，现在我们可以使用 async 函数来时路由更简单易懂一些：
```javascript
// browser/ipc/index.js

const { ipcMain } = require('electron')
const api = require('./api')

ipcMain.on('api', (event, actionName, ...args) => {
  const reply = (replayObj, status = 'success') => {
    event.sender.send(`${actionName}reply`, replayObj, status)
  }
  if (api[actionName]) {
    api[actionName](event, ...args)
      .then(res => reply(res))
      .catch(err => reply({ message: '应用出现了错误' }))
  }
})
```
现在我们假设路由文件已经是 async 函数构成的，先将回复方法(reply 函数)放在外部，取消之前的对象合并。虽然前面使用对象合并避免对侵入原生对象，
但也并不是那么优雅，现在只考虑返回值无疑是最酷的做法！

```javascript
// browser/ipc/api/index.js
const screen = require('../../screen')
const articleService = require('../../service/article')

module.exports = {
  login: async(e, user) => {
    // todo something
    screen.setSize(1000, 720)
    return { msg: 'ok' }
  },
  list: async(e, page) => {
    try {
      const articles = await articleService.findArticlesForPage(page)
      // todo filter articles
      return articles
    } catch (err) {
      return Promise.reject(err)
    }
  },
  detail: async(e, id) => {
    try {
      const article = await articleService.findArticleForID(id)
      return article
    } catch (err) {
      return Promise.reject(err)
    }
  },
}
```
`articleService` 是原生数据库查询的封装，相比于每次写 find/update 方法与大量参数，我更建议大家把这些垃圾代码统一封装成更富有语义性的函数，
无论过去多久，你再次读到这段代码时总能很清楚的知道自己做了什么，这很关键。
另外，我给大家展示的是代码框架如何搭建，单个 await 带来的便利性没有想象的大，但你在实际业务中会涉及多次查询、更新、筛选、遍历操作，
async 语法糖会给你带来极高的可读性！

现在 news-feed 已经能够快速显示出数据库里的列表：
![列表demo](/images/electron/electron-demo-4.png)

点击任何一项进入详情，文章内容都被 `sanitize.pipe` 过滤解析在 dom 里：
![详情demo](/images/electron/electron-demo-5.png)



### 最后
当然，news-feed 还存在很多问题，甚至还不能称之为一个应用，比如不能注销登录、浏览文章时无法返回列表、无法下载文章内容/图片、没有跳转到原文等等。
这些细节是真正值得注意的重点，后面几节我们都会一起讨论怎样添加这些逻辑并优化现有的代码。

















