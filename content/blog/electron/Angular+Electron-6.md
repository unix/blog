---
title: 使用 Angular 构建 Electron 应用 - 6
date: 2016-03-06
slug: electron/ng-electron-6
---

这一小节我们只做几件小事，它们在项目整体中显得微不足道。特别是 new-feed 中，我们并没有设计出完善的业务逻辑，现在我们甚至可以结束教程，
但我希望能够借助这几个项目中细节实现来传达整体的构建思想与编程思维方式，如果你认为学习它有些困难，可以跳过这些方法转而使用一些不太优雅的解决方案。

news-feed 如今还不能正确的浏览文章，至少缺少返回列表与列表翻页。Angular 中它们有很多完全不同的解决方案，你需要争对不同场景选择合理的实现方式，
下文里我们尝试几种不同的方案来解决这些问题。

### 实现返回按钮
#### 组件方式
文章在详情浏览时需要一个返回至列表页的按钮，它仅仅只做路由上的回退，换言之，这个按钮无需接受任何的参数，也不会受环境变化影响。
返回按钮始终是一个固定的功能组件，我们先使用组件的方式完成它：  

1. 在 `src/app/shared` 里新建 component 文件夹并创建 Back 组件。
2. 为组件添加 html 模板与样式。
3. 为组件添加逻辑。
4. 将 back 组件链接至 `shared.module.ts` 中，并导出它。

```javascript 
import { Component, OnInit } from '@angular/core'
import { Location } from '@angular/common'

@Component({
  selector: 'app-back',
  templateUrl: './back.component.html',
  styleUrls: ['./back.component.scss'],
})
export class BackComponent implements OnInit {
  
  constructor(private location: Location) {
  }
  
  goBack(): void {
    this.location.back()
  }
  
  ngOnInit() {
  }
  
}

```
现在 Back 组件已经能够正常工作，但还需要注意一个小问题，在快速点击按钮时可能触发多次的 `goBack ()` 导致路由返回到上一层或登录页，
我们需要用一些小技巧来解决它：  

1. 命名一个具备阀门功能的布尔变量 (`private returnOnlyOnce: boolean = true`)，每次执行函数时设置一次变量。这是一个不错的实现方式。
2. 或者你可以尝试在 `goBack ()` 函数中传入一个事件对象，然后利用 *RxJS* 的 `fromEvent` 来将事件转化为 *Observable*，
订阅 *Observable* 时我们仅仅需要使用动态操作符即可屏蔽连续的点击事件。  

#### 指令方式
> Back 组件是共享的，它能够在任何需要的地方被引入使用，但这又引申出一个值得注意的问题：每当我们需要改动 Back 组件的样式时，
就不得不为组件嵌套一层模板或加入一些新的接口，随着业务越来越复杂，Back 组件会俞加臃肿，直到有一天，它看起来和 *React 组件*一样浑身被打满属性疮口。
很明显，这不是我们期望的结果。
简单的说，你仅仅只需要做一些逻辑/属性上的改变而模板不会多次复用时，你需要尽量避免组件，转而使用*属性型指令*，这是 Angular 与 React 的不同之处。
在介绍属性型指令之前，我们先看看 React 最明显的问题：在构建一个模板与样式会变化的组件时，React 总是需要传达属性或 style，
这样的代码很多时候会显得臃肿而且富含黏性。简单的说，它是基于 View 思考问题的。在 Angular 中遇到类似问题时我们首先要做的并非创建组件，
转而考虑改变 DOM 的行为，从逻辑上来说，这是行得通而且非常具有形象意义的。

开始创建back指令：
```javascript
import { Directive, HostListener } from '@angular/core'
import { Location } from '@angular/common'

@Directive({
  selector: '[routeBack]',
})
export class BackDirective {
  
  constructor(private location: Location) {
  }
  
  @HostListener('click')
  goBack() {
    this.location.back()
  }
  
}

```
`@HostListener` 装饰器可以标注 DOM 宿主的动作，它避免我们直接操作DOM元素(如果你想，当然也可以)，在编写 Angular 代码的大部分时间，
我们都不必考虑 DOM 的问题，也很少直接参与 DOM 事件的注销。
现在，只需要将指令文件导入至 `src/app/shared` 中并导出，我们就可以在任何的模板中轻松使用它：
```html
<div routeBack>返回</div>
```

这是一个不错的开始，让我们体会到 Angular 的不同寻常之处，现在开始编写相对复杂一些的加载列表按钮。

### 实现加载列表
#### 使用变量控制

在类似于计数器、时间控制、购物车等等业务逻辑之处，你都可以通过暂存一个变量来解决数量的缓存与显示，所有的操作逻辑被看作 Action，用来触发缓存的变化。
它们实现起来非常简单，特别是只需要加载更多的单一翻页功能时：  

1. 创建一个接口用来继承或声明属性：`interface Pagination {page: number}`
2. 创建变量 pagination，初始值为1
3. 为点击事件绑定函数 `loadMore ()`，并在其中发起一次 service 文件中的 `getList`，并使变量 pagination + 1。

这里的 Pagination 接口非常简陋，但实际业务中肯定远远不止这些。翻页需要考虑到一共有多少页码数量，在最后一页时需要对下一页或加载更多隐藏，
返回上一页时也需要请求接口，由于列表使用的服务是公共的 `getList`，你可能还要集齐每页数量、排序方式、筛选条件、搜索条件等等参数，
每一个参数发声变化时都需要重复计算整个逻辑，并重新请求一次接口，当然还需要对这些参数进行保存，以便于下一页继续使用。
它们太复杂了，特别像电商网站这样的复杂的筛选参数时，你需要为此付出巨大精力，而且代码也未必有足够的扩展性，甚至于其他开发人员也很难理解。
面对这类功能时，我们可以开始考虑 RxJS。

#### 通过可观察对象

在使用 RxJS之前，有一点值得我们关注：在实现翻页功能时，我们需要订阅的并非是来自于翻页按钮的事件，而使基于页码本身的 Observable。
页面中可能有多个位置会触发翻页函数，但操作的始终只是随着时间推进而变化的单个值。即便是在未来，我们需要关注也只是一个稍稍复杂的对象而已。
```javascript
private pagination:Subject<number> = new Subject<number>()
private paginationSub: Subscription

this.paginationSub = this.pagination
  .filter(page => page > 0)
  .switchMap(page => this.listService.getList(page))
  .subscribe(
    list => this.list.push(...list),
    err => Observable.of<any>([])
  )
```
`Subject` 是 RxJS 中一种特殊的 Observable，它允许值被多播到多个观察者。我们可以通过调用 `this.pagination.next(1)` 为观察者发射一个新的值，
那么它的观察者会再次执行 filter 与 switch 直到处理订阅函数。代码中的 filter 用来帮助验证和过滤一些不合理的值，它在未来会有其他用处，
通过 `switchMap` 切换至新的流并取消原来流的订阅。最后我们订阅的是 `listService` 返回的流，并将数据更新至 list 中。

在未来业务逻辑变化的更复杂时，我们可以为这些列表筛选与排序产生的值创建多个可观察对象，再使用 `combineLatest` 将它们合并起来：
```javascript
this.paginationSub = Observable.combineLatest(
  this.pagination,
  this.sort,
  this.filter,
  // ...
)
```
无论逻辑怎样复杂，我们始终仅仅只维护这些可观察对象，在合理的时间为它们发射新的值即可。理所应当的，任何过滤，验证操作都可以使用 RxJS 的操作符完成，
甚至你可以自己创建一些操作符来过滤、合并这些流。相比于前一种实现方式，RxJS 使代码具备了高度可读性与可扩展性，这是难能可贵的。

不知道你是否注意到，这段代码还存在一个问题，如果我们需要对 `this.pagination` 进行维护，
如果你不能够从 http 服务返回的流中获取这个值就需要自己维护所谓的**当前值**。这类情况下我们更需要 Subject 的一个变种 *BehaviorSubject*。
*BehaviorSubject* 储存着最新发射的值，可以通过 `getValue()` 获取：
```javascript
private pagination: BehaviorSubject<number> = new BehaviorSubject<number>(1)

// ...

loadMore (nextNumber: number):void{
  this.pagination.next(this.pagination.getValue() + nextNumber)
}
```  

对于排序、筛选或其他任何逻辑都是相同的，未来我们永远只把注意力放在可观察对象上，通过少量的高可读性的代码来解决逻辑问题。
对于刚刚解除 Angular 或 RxJS 的开发者来说，这需要一些学习时间，
可参考 [github记录](https://github.com/unix/news-feed/tree/2b8039abd28a94192083678c5f7beeb04dede025) 理解这一节。
在下一小节中，我将会默认大家已经掌握这些技能，开始着手完成剩余的用户模块。























