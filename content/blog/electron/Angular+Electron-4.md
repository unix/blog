---
title: 使用 Angular 构建 Electron 应用 - 4
date: 2016-03-05
slug: electron/ng-electron-4
---

这一节我们只做两件事，第一是建立相应的爬虫系统，从网页链接上提取合适的信息，第二则是将这些信息储存在数据库中，render 需要展示时再查询予以显示。
开始构建代码前我们先思考一下这样做的好处是什么。

### 介绍

在 news-feed 应用中，我们把爬虫逻辑放在客户应用里而非服务端，这是正确的，考虑到用户增加的情况下我们无法负担所有的爬虫任务，
如果我们将这些任务进行合理的分配是最优的，利用一些客户端资源。在生产环境里还可以考虑用户每次爬取完毕后发送处理好的字符串发送回服务端进行存储，
甚至可以根据服务器返回不同得资源来考虑返回给用户不同的任务。虽然在 news-feed 中我们不会做这些事，但我们不妨考虑这样的系统是如何工作的：  

1. 应用内部储存一张映射表，可更新，作为当前应用的基础爬虫任务。
2. 根据用户下载应用 IP 不同分发不同的应用包，基础数据库的标识有一些区别。
3. 根据用户请求的标识+  IP 地址返回给用户不同的爬虫任务。
4. 短时间的工作后将数据返回给服务端。
5. 用户每次查看的新闻一部分是自己客户端爬取的，另一部分则从服务器下载。

这样的系统很有意思，积累众多格式化数据资源后甚至可以转为开发的新闻 API 供大家使用，它没有想象中的复杂(你可以自己尝试一下)，
目前我们希望应用的所有数据都能够自行完成，为此我们至少需要一个数据库存储格式化数据，一段可配置的代码爬取与分析数据。
在做所有事情之前，我准备加入一个新的语法糖，以适应爬虫任务。

### 配置 Async
async 是 ES7 的新语法，简单的说，async 是一个基于 Generator 的语法糖。如果你对 Generator 还不了解，建议先学习一些 ES6 基础知识。
爬虫任务可能涉及到很多的异步任务，但大多数时候我们更希望它们可以同步执行(并发过大很容易被网站屏蔽IP地址)，
async 函数可以帮助我们轻松的用同步函数的方式写异步逻辑，而且它足够简单，学习它也是理所应当的，这是 javascript 的趋势之一。


首先我们需要安装一些必要的 npm 包：
```bash
npm i --save transform-async-to-generator syntax-async-functions transform-regenerator
npm i --save babel-core babel-polyfill babel-preset-es2016
```

这里我希望代码不要经过频繁的转码，应用可以不考虑兼容性，所以我加入一些垫片使语法糖能够正常工作即可。
在根文件夹下建立一个 `.babelrc` 文件：  

```javascript
{
	"presets": ["es2016"],
	"plugins": ["transform-async-to-generator", "syntax-async-functions", "transform-regenerator"]
}
```

并在根文件夹建立一个 `main.js`，集合这些文件：

```javascript
require('babel-core/register');
require("babel-polyfill");
require("./index");
```

从现在开始我们每次只需运行 `electron main.js` 就能够轻松的启动富含ES7语法糖的应用。当然，你可以引入任何语法，甚至是 Gulp/Webpack 编译代码，
只要你开心。

### 安装数据库

作为一个桌面应用，数据存储是必不可少的一环，但这里并没有使用已携带的浏览器存储：
1. 浏览器的各类存储总是有限的。
2. 它们很难存储复杂结构的数据，你需要为此做很多转换。
3. 最大的局限在于不能够随意的释放窗口对象，这会带来很多的存储丢失问题，这对未来的扩展必然有影响。

除此之外我们还可以选用一些流行的云储存，远程数据库等等，但我希望应用能够在脱机时正常工作，为此我们需要一个安装简单，在本地即时编译的轻量级数据库。

这里我选用的流行的 [nedb](https://github.com/louischatriot/nedb)，它的社区环境足够好，有很多的使用者(保证库能够及时更新并解决各类问题)，
而且与 electron能够很好的结合。
安装 nedb:
```bash
npm i --save nedb
```

在根目录的 `index.js` 中启动数据库：
```javascript
const Datastore = require('nedb')
global.Storage = new Datastore({filename: `${__dirname}/.database/news-feed.db`, autoload: true })
```

> nedb 有多种储存方式，包括内存。这里的 `autoload` 代表每次更新时都会更新数据库的本地文件，将数据写入硬盘。
你也可以选择每次使用 `loadDatabase` 来手动触发写入硬盘的动作。

### 构建爬虫代码
在动手之前我们先尝试分析爬虫代码的逻辑：这里至少需要一个实际工作的爬虫函数，它从 http 请求得到数据并且开始分析 html，最后存储这些数据。
不同的网站结构不同意味着需要不同的解析函数，但其中至少可以将基础的 http 服务抽离出来(它们总是相同的)，
未来我们可以从服务端获取一些解析代码填充在这里。

手动发起 http 请求与处理字符串工作量非常大，我们可以借助一下库来完成这些工作：
```bash
* https://github.com/request/request
npm i --save request

* https://github.com/cheeriojs/cheerio
npm i --save cheerio
```  

**1.新建 http 请求函数**

在 `/browser/task` 下新建 `base.js`：
```javascript
const req = require('request')

module.exports = class Base {
  static makeOptions(url) {
    return {
      url: url,
      port: 8080,
      method: 'GET',
      headers: {
        'User-Agent': 'nodejs',
        'Content-Type': 'application/json',
      },
    }
  }
  
  static request(url) {
    return new Promise((resolve, reject) => {
      req(Base.makeOptions(url), (err, response, body) => {
        if (err) return reject(err)
        resolve(body)
      })
    })
  }
  
}
```  
Base 类有两个静态方法，`makeOptions` 负责根据 url 生成一个 option 对象，为每次请求设置配置项使用，
当未来需要验证 token/cookie 时我们再来扩充此方法，`request` 返回一个 Promise 对象，显然它会发起一个请求，
但更多的作用是在使用时优先返回 body 而非 response。这很重要。

也许你开始注意到，这两个静态函数完全不依赖 `this`，它们仅仅是类的静态方法，无需实例化即可使用，同时也能够被继承。
这样的目的在于暗示这些函数是完全不依赖状态的纯函数，它们总是返回相同的结果，也没有副作用，这样的函数在未来能够被更好的阅读与扩展。  

**2.新建爬虫文件**  
假定这个文件只负责单个网站(例如 ifeng.com)的功能，当然以后这样的文件会越来越多，现在先为这些功能文件创建一个集合文件负责导出：
```javascript
// /browser/task/index.js
module.exports = {
  ifeng: require('./ifeng')
}
```

在task文件夹下再创建一个`ifeng.ts`：
```javascript
const cheerio = require('cheerio')
const Base = require('./base')

module.exports = new class Self extends Base {
  constructor() {
    super()
    this.url = 'http://news.ifeng.com/xijinping/'
  }
  
  start() {
    global.Storage.count({}, (err, c) => {
      if (c || c > 0) return
      this.request()
        .then(res => {
          console.log('全部储存完毕!')
          global.Storage.loadDatabase()
        })
        .catch(err => {
          console.log(err)
        })
    })
    
  }
  
  async request() {
    try {
      const body = await Self.request(this.url)
      let links = await this.parseLink(body)
      for (let index = 1; index < links.length; index++) {
        const content = await Self.request(links[index - 1])
        const article = await this.parseContent(content)
        await this.saveContent(Object.assign({ id: index }, article))
        console.log(`第${index}篇文章:${article && article.title}储存完毕`)
      }
    } catch (err) {
      return Promise.reject(err)
    }
  }
  
  parseLink(html) {
    const $ = cheerio.load(html)
    return $('.con_lis > a')
      .map((i, el) => $(el)
        .attr('href'))
  }
  
  parseContent(html) {
    if (!html) return
    const $ = cheerio.load(html)
    const title = $('title')
      .text()
    const content = $('.yc_con_txt')
      .html()
    return { title: title, content: content }
  }
  
  saveContent(article) {
    if (!article || !article.title) return
    return global.Storage.insert(article)
  }
}()

```

`ifeng.ts` 的主体是 request 函数，它做了以下几件事：
1. `try` 代码块，捕获 await 可能抛出的错误。
2. 利用继承的 request 静态方法获得基础的列表文件，使用 `parseLink` 解析 html 获得一个链接数组。`cheerio` 是一个类似于 JQuery 的库，
可以帮助我们解析这些 html 文件。
3. 循环体内，分别请求文章主体，利用 `parseContent` 分析文章并集合成对象，如果对象获取成功，接下来还会为这篇文章对象合并一个序列号，
便于后面的查询/分类。
4. 每次循环都插入一次数据库。这样做在于单次插入数据较多失败时，neDB 会使所有的数据回滚。当然这其中的量级你可以自己把握。
在更大的应用里你可以抽象出一层类似于 ORM 的服务，专职于有效快速的存储查询，甚至是提供一些语法糖。


这里的 `global.Storage.count` 是一个权宜之计，在未来完全前端代码后再回过头来解决它，目前我们只需要在根目录的 `index.js` 里加入
 `require('./browser/task/index').ifeng.start()` 即可使它工作起来：

![截图](/images/electron/electron-demo-3.png)

OK，这一节的所有目标都已完成，下一节我们开始讨论如何在 Angular 中构建一个合理的展示模块并与数据库通信。

