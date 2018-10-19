---
layout: post
keywords: blog
description: blog
title: "[译文] Angular 应用中的状态管理"
categories: [frontend]
tags: [Angular]
icon: code
---

>  原文: [Managing State in Angular Applications](https://blog.nrwl.io/managing-state-in-angular-applications-22b75ef5625f#.sijb4abxh)
>
>  已经过作者[Victor Savkin](https://blog.nrwl.io/@vsavkin?source=post_header_lockup)授权翻译

译者注：

Angular 开发组最近将 SemVer 版本命名方式引入了 Angular 2 中，参见[这篇文章](http://angularjs.blogspot.jp/2017/01/branding-guidelines-for-angular-and.html)。这篇文章中出现的 Angular 指的是 Angular 2 框架。

本文原作者 Victor Savkin 为前谷歌 Angular 项目的核心成员，负责了 Angular 的依赖注入、状态监测、列表和路由模块的实现。

译文将根据作者的原意，以中文文法稍加润色。

# Angular 应用中的状态管理

![img](https://cdn-images-1.medium.com/max/2000/1*P3b-lB3BUd0T8ktHSoCW5g.jpeg)

状态管理是个大学问。我们总是需要在不同的地方以并发的形式更新状态，比如协同多个后端服务、Web Worker 脚本、或是 UI 组件。借助像是 Redux 这样已有的解决方案，我们可以显式的去实现状态的协同更新，但这种模式还不能算是完全让我们高枕无忧。实际上，状态管理的需求还要广的多。

应选择把什么东西储存在内存还是 URL 中；如何处理本地的 UI 组件状态；如何在持久化的数据、URL 还有后端服务之间同步状态：以上这些状态相关的问题，都需要我们在应用设计中去解答。

### 状态类型的种类 

一个典型的网页应用有以下六种状态类型：

- 后端服务状态
- （储存在客户端的）持久化状态
- URL 和路由状态
- 客户端状态
- 瞬时客户端状态 
- 本地 UI 状态

接下来我们来深入探讨每种类型。

**显然，后端服务状态是存储在后端服务上的状态，我们可以通过比如访问 REST  端点这样的方式去取值。持久化状态是存储在客户端内存中的服务器状态的一个子集。** 简单来说，我们可以把持久化状态看做后端服务状态的一个缓存副本。不过在实际的应用中，我们通常会优化缓存的更新过程，以增强用户的体验。

**客户端状态不会储存在服务器上。**举一个典型的例子：我们会根据用户提供的过滤条件，创建并展示给用户被过滤的项目列表；项目本身会被存储在服务端的某个数据库上，但被过滤的值不会被存储在服务端。

> ##### 建议：把客户端状态和持久化状态同时反映在 URL 中是个不错的实践。

应用的状态通常被存储在客户端上，但不会在 URL 中被展示出来。*比如，YouTube 会记住我在哪里停止观看的视频，并在下次观看时从上次停止观看的位置开始播放。由于这个信息没有存储在 URL 中，如果我把视频连接发给另外某个人，他就只能从头开始观看这个视频。**这种场景中的状态就是瞬时状态。**

**最后，每个独立的 UI 组件都可能含有决定各自行为的内部状态。**比方说，要不要把一个可折叠项目展开，或者这个按钮的颜色是什么。**这就是本地 UI 状态。**

> ##### 建议：在鉴定一个状态的种类时，可以问一下自己两个问题：这个状态可否被共享？它的生命周期是什么？

### 状态同步

**持久化状态和服务器务状态存储着相同的信息；对于客户端状态和 URL 也需要存储相同的信息。因此，我们必须去解决状态同步的问题。**选择状态的同步策略通常也就成为了设计应用的状态管理最重要的决定之一。

我们是否可以选择将某些同步做成严格同步的？哪些状态是可以做成异步的？或者用分布式系统的术语描述: 我们应该选择使用严格（strict）一致性还是最终（eventual）一致性？

这篇文章，我们将要去研究以上所提到的各个问题。 

![img](https://cdn-images-1.medium.com/max/800/0*I3jAfhOZPcF1Mjil.)

---

### 案例

我们先从一个看起来构建的还算合理的系统看起。这个应用展示了一个演讲列表，用户可以过滤列表，或是选择观看、评价某个演讲。

![img](https://cdn-images-1.medium.com/max/600/0*2RSbSLzFhSBjNVyZ.)

![img](https://cdn-images-1.medium.com/max/600/0*RMmxMvaBGacVWkKA.)



这个应用中有两大块路由：一个用来展示演讲列表，另外一个用来显示一个演讲的详细信息。

```typescript
// routes.ts
RouterModule.forRoot([
  { path: 'talks',  component: TalksAndFiltersCmp },
  { path: 'talk/:id', component: TalkDetailsCmp }
])
```

下面是应用的简略架构图：

![img](https://cdn-images-1.medium.com/max/800/0*zPz__yd_tEC427D8.)

下面是应用的数据模型信息：

```typescript
// model.ts
export interface Talk {
  id: number;
  title: string;
  speaker: string;
  description: string;
  yourRating: number;
  rating: number;
}

export interface Filters {
  speaker: string;
  title: string;
  minRating: number;
}
```

以及两个主要组件：

```typescript
// talks-and-filters.ts
@Component({
  selector: 'app-cmp',
  templateUrl: './talks-and-filters.html',
  styleUrls: ['./talks-and-filters.css']
})
export class TalksAndFiltersCmp {
  constructor(public backend: Backend) {}

  handleFiltersChange(filters: Filters): void {
    this.backend.changeFilters(filters);
  }
}
```

```typescript
// talk-detail.ts
@Component({
  selector: 'talk-details-cmp',
  templateUrl: './talk-details.html',
  styleUrls: ['./talk-details.css']
})
export class TalkDetailsCmp {
  talk: Talk;

  constructor(private backend: Backend, 
              public watchService: WatchService, 
              private route: ActivatedRoute) {
    route.params
      .mergeMap(p => this.backend.findTalk(+p['id']))
      .subscribe(t => this.talk = t);
  }

  handleRate(newRating: number): void {
    this.backend.rateTalk(this.talk.id, newRating);
  }

  handleWatch(): void {
    this.watchService.watch(this.talk);
  }
}
```

两个组件本身不会处理任何实际的业务逻辑。这一部分处理被委托给了 `Backend` 和 `WatchService` 服务。

```typescript
// backend.ts
@Injectable()
export class Backend {
  _talks: {[id:number]: Talk} = {};
  _list: number[] = [];

  filters: Filters = {speaker: null, title: null, minRating: 0};

  constructor(private http: Http) {}

  get talks(): Talk[] {
    return this._list.map(n => this._talks[n]);
  }

  findTalk(id: number): Observable<Talk> {
    return of(this._talks[id]);
  }

  rateTalk(id: number, rating: number): void {
    const talk = this._talks[id];
    talk.yourRating = rating;
    this.http.post(`/rate`, {id: talk.id, yourRating: rating}).forEach(() => {});
  }

  changeFilters(filters: Filters): void {
    this.filters = filters;
    this.refetch();
  }

  private refetch(): void {
    const params = new URLSearchParams();
    params.set("speaker", this.filters.speaker);
    params.set("title", this.filters.title);
    params.set("minRating", this.filters.minRating.toString());
    this.http.get(`/talks`, {search: params}).forEach((r) => {
      const data = r.json();
      this._talks = data.talks;
      this._list = data.list;
    });
  }
}
```

每当过滤条件改变时，`Backend`服务将会重新获取演讲数组。因此每当用户浏览某项单独的演讲时，`Backend`将会从内存中获取所需的信息。*译者注：也就是说查看某个演讲的详情时不会重新获取数据。*

`WatchService`的实现十分简单：

```typescript
// watch-service.ts
export class WatchService {
  watched: {[k:number]:boolean} = {};

  watch(talk: Talk): void {
    console.log("watch", talk.id);
    this.watched[talk.id] = true;
  }

  isWatched(talk: Talk): boolean {
    return this.watched[talk.id];
  }
}
```

#### 源码

你可以在[这里](https://github.com/vsavkin/state-app-examples/tree/ad_hoc)获取以上应用的源码。

#### 案例中的状态类型

我们来研究一下有哪些东西管理着应用的各项状态。

- `Backend`管理持久化状态（演讲数组）和客户端状态（过滤条件）
- 路由管理 URL 和路由状态
- `WatchService`管理瞬时状态（已经观看过的演讲）
- 每个组件管理自己的本地 UI 状态

### 问题

乍看之下，这个应用的实现是合理的：应用逻辑由依赖注入的服务处理，每个方法短小精悍，代码风格不错。但只要往细里研究，我们就能在其中发现不少问题。

#### 客户端持久化状态与服务器状态的同步

首先，当载入一个演讲的详情时，我们会调用 `Backend.findTalk` 函数，把储存在内存中的集合数据中取出相应数据。当用户从列表页开始浏览然后访问不同页面时，这样的做法倒还行得通；但如果用户直接从演讲的详情 URL 载入应用时，`Backend`中的数据集合还是空的，应用因此就无法正确的进行工作。我们可以在此做个变通，在获取演讲详情前先检查一下演讲集中是否有正确的演讲数据，如果没有就从服务器端把数据拉回来。

```typescript
// backend.ts
export class Backend {
  //...
  findTalk(id: number): Observable<Talk> {
    if (this._talks[id]) return of(this._talks[id]);

    const params = new URLSearchParams();
    params.set("id", id.toString());
    return this.http.get(`${this.url}/talk/`, {search: params})
                    .map(r => this._talks[id] = r.json()['talk']);
  }
  //...
}
```

其次，为了增强用户体验，演讲的评分方法的实现过于乐观的直接修改了传入的演讲对象。问题在于我们没有处理好这里可能出现的错误：如果服务端更新演讲数据失败，客户端将会显示错误的信息（指的是前端显示的数据与后端不一致）。

我们可以这样修复上面的问题，当出现数据更新异常时，把评价值重设为`null`。

```typescript
// backend.ts
export class Backend {
  //...
  rateTalk(talk: Talk, rating: number): void {
    talk.yourRating = rating;
    this.http.post(`${this.url}/rate`, {id: talk.id, yourRating: rating}).catch((e:any) => {
      talk.yourRating = null;
      throw e;
    }).forEach(() => {});
  }
  //...
}
```

经过以上修改，持久化状态就可以与服务端数据正确的进行同步了。

#### URL 和客户端状态的同步

更改过滤条件时，我们也会发现 URL 并没有同步得到更新。我们可以用手动同步的方式解决这个问题。

![img](https://cdn-images-1.medium.com/max/800/0*dOJxcL5N7cjxztZz.)



```typescript
// talks-and-filters.ts
Import {paramsToFilters, filtersToParams} from './utils';

@Component({
  selector: 'app-cmp',
  templateUrl: './talks-and-filters.html',
  styleUrls: ['./talks-and-filters.css']
})
export class TalksAndFiltersCmp {
  constructor(public backend: Backend, 
              private router: Router, 
              private route: ActivatedRoute) {
    route.params.subscribe((p:any) => {
      // the url changed => update the backend
      this.backend.changeFilters(paramsToFilters(filters));
    });
  }

  handleFiltersChange(filters: Filters): void {
    this.backend.changeFilters(filters);
    // the backend chagned => update the URL
    this.router.navigate(["/talks", filtersToParams(filters)]);
  }
}
```

技术上讲，这段代码是可以解决问题的（URL 和客户端状态做到了同步），但这个解决方案本身也是有问题的：

- `Backend.refetch`被调用了两次。过滤条件改变时将会调用一次`refetch`，但它也会触发一次导航，而每次导航的触发也会最终调用一次`refetch`。
- 路由和`Backend`状态的同步是以异步的方式实现的。这就意味着，路由信息将无法从`Backend`中获取到任何可靠的信息；反之，`Backend`也无法从路由状态或是 URL 中获取到可靠的信息 - 这些信息也许还没在那里。
- 我们还没处理过路由守卫被阻挡时的情况（注：Angular 原生路由的功能，在路由触发前可以先检查是否当前状态可否跳转到新的状态）。这样就会导致，无论客户端状态如何进行更新，导航的变化就像是永远都会成功一样。
- 我们的解决方案只能处理一个特定路由和客户端状态的同步的问题。但如果我们需要新加入一个路由的话，这块逻辑就得重新实现一遍。
- 最后，我们的模型是可变的。这就意味着，我们可以在不去更新 URL 的情况下去更新模型数据。这正是导致错误产生的普遍原因之一。

### 错在哪里？

对于这样一个小应用我们就能找到好多潜在的问题。为什么状态管理这么难解决？我们在解决问题的过程中都犯了哪些错误呢？

- **没有把状态管理从业务逻辑和后端交互服务中抽离出来。** `Backend` 在与服务器进行交互的同时，也在维护着状态。这对 `WatchService`来说也有同样的问题。
- **客户端持久化状态和后端服务器状态之间的同步策略不够清晰。**就算我们之后修复了这个问题，但解决方案只能针对某个路由起作用，没有全局角度考虑复用性。
- **客户端状态和 URL 之间的同步策略不够清晰。**由于目前没有设计路由守卫，同时`refetch`的实现是幂等性的，我们的方案暂时避开了某些问题，但它不是一个能长久发展的解决方案。
- **数据模型是可变的，**这就意味着保持应用状态的可依赖性变得困难。

---

### 重构步骤一： 分离状态管理

![img](https://cdn-images-1.medium.com/max/1250/1*GYkkazzYVEhf6tIM2u5ZVQ.jpeg)

**我们需要解决的最大也是首先需要去解决的问题就是，如何把状态管理逻辑从应用的其他部分抽离出来**。管理状态的难度令人生畏，因此需要把它跟“与服务器交互”、“正在观看视频”或是其他任何复杂逻辑混在一起时，将使得我们痛不欲生。这里，我们来通过引入 “Redux 式” 的状态管理策略到我们的应用中。



> #### 规则一：将后端交互和业务逻辑从状态管理中抽离出来



#### Redux 介绍参考链接

考虑到目前在网上已经有很多关于 Redux 的介绍了，在这篇文章中我就略过不表。你可以去参考以下文章了解更多信息：

- [肢解状态](https://vsavkin.com/managing-state-in-angular-2-applications-caf78d123d02#.tnhbe6f16)
- [使用Redux 和 Typescript 进行开发Angular 2应用](http://blog.ng-book.com/introduction-to-redux-with-typescript-and-angular-2/)
- [@ngrx/store 库的综合性介绍](https://gist.github.com/btroncone/a6e4347326749f938510)



首先，我们从定义应用可以执行的每种动作开始：

```typescript
// model.ts
export type ShowDetail = { type: 'SHOW_DETAIL', talkId: number };
export type Filter = { type: 'FILTER', filters: Filters };
export type Watch = { type: 'WATCH', talkId: number };
export type Rate = { type: 'RATE', talkId: number, rating: number };
export type Action = Filter | ShowDetail | Watch | Rate | Unrate;
```

然后是状态：

```typescript
// model.ts

// all non-local state of the application
export type State = { 
  talks: { [id: number]: Talk }, 
  list: number[], 
  filters: Filters, 
  watched: { [id: number]: boolean } 
};

// init state
export const initState: State = {
  talks: {}, 
  list: [], 
  filters: {speaker: null, title: null, minRating: 0}, 
  watched: {}
};
```

最后是 reducer：

```typescript
// model.ts

// a factory to create reducer
export function reducer(backend: Backend, watch: WatchService) {
  return (store: Store<State, Action>, state: State, action: Action) => {
    switch (action.type) {
      case 'FILTER':
        return backend.findTalks(action.filters).
            map(r => ({...state, ...r, filters: action.filters}));

      case 'SHOW_DETAIL':
        if (state.talks[action.talkId]) return state;
        return backend.findTalk(action.talkId).
            map(t => ({...state, talks: {...state.talks, [t.id]: t}}));

      //...
      default:
        return state;
    }
  }
}
```

操作非本地状态的的任务**有且只有**如上 reducer 进行处理。这样`Backend`和`WatchService`就可以变得无状态。

```typescript
// watch-service.ts
export class WatchService {
  watch(talk: Talk): void {
    console.log("watch", talk.id);
  }
}
```

#### 理想状态的更新

之前，我们有个临时的理想状态的更新策略。我们还能做的更好。

我们引入另外一个叫做`UNRATE`的动作，用以处理服务器拒绝更新的情况。

```typescript
// model.ts
export function reducer(backend: Backend, watch: WatchService) {
  return (store: Store<State, Action>, state: State, action: Action) => {
    switch (action.type) {
      //...
      case 'RATE':
        backend.rateTalk(action.talkId, action.rating).catch(e =>
          store.dispatch({type: 'UNRATE', talkId: action.talkId, error: e})
        ).forEach(() => {});

        const talkToRate = state.talks[action.talkId];
        const ratedTalk = {...talkToRate, yourRating: action.rating};
        const updatedTalks = {...state.talks, [action.talkId]: ratedTalk};
        return {...state, talks: updatedTalks};

      case 'UNRATE':
        const talkToUnrate = state.talks[action.talkId];
        const unratedTalk = {...talkToUnrate, yourRating: null};
        const updatedTalksAfterUnrating = {...state.talks, [action.talkId]: unratedTalk };
        return {...state, talks: updatedTalksAfterUnrating};

      default:
        return state;
    }
  }
}
```

这个更新很有必要。这将保证所有操作将按顺序进行执行、避免了交错的可能性。



> #### 规则二： 理想的状态更新过程需要引入额外的动作来处理错误。



#### 不可变数据

最后，我们把数据模型改为不可变的类型。这将会带来很多有益的后果，稍后我会聊一聊这个话题。



> #### 规则三：为持久性状态和客户端状态使用不可变数据



#### 更新后的组件

重构后，我们的组件变得简单了。现在他们就只需负责状态的查询和动作的派送。

```typescript
@Component({
  selector: 'talk-details-cmp',
  templateUrl: './talk-details.html',
  styleUrls: ['./talk-details.css']
})
export class TalkDetailsCmp {
  constructor(private store: Store<State, Action>, private route: ActivatedRoute) {
    route.params.forEach(p => {
      this.store.dispatch({
        type: 'SHOW_DETAIL',
        talkId: p['id']
      });
    });
  }

  get talk(): Talk {
    return this.store.state.talks[+this.route.snapshot.params['id']];
  }

  get watched(): boolean {
    return this.store.state.watched[+this.route.snapshot.params['id']];
  }

  handleRate(newRating: number): void {
    this.store.dispatch({
      type: 'RATE',
      talkId: this.talk.id,
      rating: newRating
    });
  }

  handleWatch(): void {
    this.store.dispatch({
      type: 'WATCH',
      talkId: this.talk.id
    });
  }
}
```

### 分析

- **状态管理与逻辑处理/服务交互分离开来。使用 reducer 成为了我们去改变本地客户端状态的惟一方式。**前后端交互、观看视频等，现在被无状态服务进行处理。
- **不再为持久化状态和客户端状态使用可变对象数据类型。**
- 客户端持久化数据与服务器状态的同步有了新策略。现在我们有了`UNRATE`动作来处理错误，这使得我们可以按顺序去处理动作。

不过这里需要说明的是，我们的目的不是为了在系统中用上 Redux。就算使用了 Redux，我们仍然有可能出现将逻辑处理与状态管理混杂在一起、没有处理错误的情况下进行了乐观性更新、或是使用了可变状态的问题。Redux 可以帮助我们去解决以上问题，但他不是一种万能药，也不是解决问题的唯一途径。

> #### 规则四：使用 Redux ，是为了解决问题，而不是目的本身。

另外你可能会注意到，在重构过程中我们完全没有触碰任何本地 UI 状态。这是因为，本地 UI 的状态几乎从来都不会成为我们的问题。 **组件可以有别人访问不到的可变属性，而这是我们不太需要关注的东西。**

#### 使用 GraphQL 和 Apollo

就算是做了以上重构，我们依然是以手动的方式管理客户端-服务端的状态同步，而这可能会导致我们犯错。我们有可能会忘了处理异常，或是将缓存失效化。

GraphQL 和 Apollo 在全局的高度解决了上面的问题，但这也意味着我们需要投入更多精力在后端的基础架构上。这两个库也可以做到和 Redux 一同协作，可以参考一下这个项目: [Hongbo-Miao/apollo-chat]([https://github.com/Hongbo-Miao/apollo-chat](https://github.com/Hongbo-Miao/apollo-chat))。

如果你能平衡好研发成本，我很推荐你去调研一下 Apollo。



#### 源码

经过这次重构后的代码可以在此处找到：[点我](https://github.com/vsavkin/state-app-examples/tree/redux_no_router)



### 重构二：路由和数据仓库

![img](https://cdn-images-1.medium.com/max/1250/1*sl9VGPO4MKCijyd0LKKVag.jpeg)

#### 剩下的问题

我们的设计还遗留了以下问题：

- 路由无法可靠的从`Backend`获取信息
- `Backend`也无法可靠的从路由或是 URL 获得信息
- 如果路由守卫拒绝了导航行为，客户端状态依然以导航成功的情况进行更新
- reducer 不能阻止导航行为
- 路由和后端状态的同步是临时的。如果我们增加了个新的路由，我们还得把同步的逻辑在那里重新实现一遍。

![img](https://cdn-images-1.medium.com/max/1000/0*90kjqGaPRcWpVJth.)

#### 将路由作为状态的事实来源（Source of Truth）

一种解决问题的方式是，构建个通用的数据和路由同步更新的库。这没法解决所有的问题，不过至少解决方案不再是临时的了。另外一种方式是，将导航行为作为更新状态的一部分。再或者，我们可以把状态更新作为导航变化的一部分。那么，我们应该选择哪种方式呢？



> #### 规则五：永远将路由器作为状态的事实来源



由于用户总是会通过 URL 跟应用进行交互，那么我们就应该把路由作为状态处理的事实来源和动作的发起者。换个角度说，应该是通过路由器调用 reducer，而不是 reducer 去掉用路由的更新。

![img](https://cdn-images-1.medium.com/max/1000/0*ZQAo7NKZDhmyqy-N.)



在这种架构下，路由器首先解析 URL，创建新的路由器状态快照，再把快照交给 reducer 进行处理，在 reducer 在处理完后才进行真正的导航行为。

实现这种模式并不困难。可以引入`RouterConnectedToStoreModule` ：

```typescript
// app.ts
@NgModule({
  declarations: [
    //...
  ],
  imports: [
    //...
    RouterConnectedToStoreModule.forRoot(
      "reducer",
      [
        { path: '', pathMatch: 'full', redirectTo: 'talks' },
        { path: 'talks',  component: TalksAndFiltersCmp },
        { path: 'talk/:id', component: TalkDetailsCmp }
      ]
    )

  ],
  providers: [
    Backend,
    WatchService,
    { provide: "reducer", useFactory: reducer, deps: [Backend, WatchService]}
  ]
})
export class AppModule { }
```

`RouterConnectedToStoreModule` 会帮我们设置路由器：在 URL 解析完，生成新的路由状态后，路由器将会派发 `ROUTER_NAVIGATION`动作。

```typescript
// reducers.ts
export function reducer(backend: Backend, watch: WatchService) {
  return (store: Store<State, Action>, state: State, action: Action) => {
    switch (action.type) {
      case 'ROUTER_NAVIGATION':
        const route = action.state.root.firstChild.firstChild;

        if (route.routeConfig.path === "talks") {
          const filters = createFilters(route.params);
          return backend.findTalks(filters).
            map(r => ({...state, ...r, filters}));

        } else if (route.routeConfig.path  === "talk/:id") {
          const id = +route.params['id'];
          if (state.talks[id]) return state;
          return backend.findTalk(id).
            map(t => ({...state, talks: {...state.talks, [t.id]: t}}));

        } else {
          return state;
        }

      //...
      default:
        return state;
    }
  }
}
```

就像你所看到的，这个 reducer 可能会返回 observable，在这种情况下会使得路由器等待 observable 完成后才进行页面跳转。如果 reducer 抛出异常的时候，路由将会取消掉这次导航变化。

借助以上方法我们就不需要 `Filter` 动作了。现在，我们可以通过路由的导航变化来触发正确的动作。

```typescript
// talks-and-filters.ts
@Component({
  selector: 'app-cmp',
  templateUrl: './talks-and-filters.html',
  styleUrls: ['./talks-and-filters.css']
})
export class TalksAndFiltersCmp {
  constructor(private router: Router, private store: Store<State, Action>) {}

  get filters(): Filters {
    return this.store.state.filters;
  }

  get talks(): Talk[] {
    return this.store.state.list.map(n => this.store.state.talks[n]);
  }

  handleFiltersChange(filters: Filters): void {
    this.router.navigate(["/talks", filtersToParams(filters)]);
  }
}
```

#### 分析

**这次重构使得客户端状态与 URL 紧密相联。**路由导航行为将会调用 reducer；一旦 reducer 执行完毕后，导航将会根据新的状态继续进行。

以下是我们重构带来的结果：

- reducer 可靠的利用新的 URL 和新的路由状态进行状态计算
- 路由守卫或是路由解析器（resolver，用以在跳转到某个新的路由之前准备惰性状态的解析）也可以可靠的利用 reducer 创建的新状态
- reducer 可以中止导航行为
- 没有任何状态是并发更新的。我们总是知道每项数据的来源是什么。
- 这个解决方案也是全局性的。如果这个做好了，我们就不需要在加入新的路由的时候去操心如何同步两个状态（指的路由和 `Backend`）

也就是说，我们已经把之前列举的所有问题都解决了。



#### 源码

经过这次重构后的代码可以在此处找到：[点我](https://github.com/vsavkin/state-app-examples/tree/redux_with_router)



### 使用 @ngrx/store

在这篇文章中，我有意的没有使用任何已有的 Redux 库来构建我们的应用。我实现了自己的数据仓库，并把他连接到了路由器上（加起来也就是几百行的代码）。

这样做的目的是为了向你展示：认真地思考问题是非常重要的，而不是盲目使用最新的某个第三方库。

话虽如此，我认为 @ngrx/store 是个 Angular 的非常好的 Redux 实现，如果没有特别的理由，你应该使用它。如果你正在使用了，可以来试试 [vsavkin/router-store](https://github.com/vsavkin/router-store)，这个库为 @ngrx/store 实现了路由链接器。这个库应该很快就会成为 ngrx 的一部分。


### 总结

我们从一个简单应用开始。初看实现是合理的：函数短小精悍，代码优雅；随后我们深入研究后发现了很多的潜在问题。如果没有一双训练有素的眼睛，这些问题很可能就被忽视了。我们尝试用一些变通方案解决了问题，但仍然很多没有解决，并且解决方案都不能很好的拓展到全局。

这个应用有如此多的问题，是因为我们没有从头到尾仔细想清楚应用状态管理的策略。每种解决方案都是临时的。当我们需要去处理一个并发的分布式系统时，临时性的解决方案会迅速击垮整个系统。

之后我们开始着手重构系统，在应用引入了类似 Redux 的数据仓库和不可变数据类型。但这并不是我们的目的，而是我们实现目标的手段。为了解决剩余的问题，我们实现了将 reducer 连接到路由的策略。

在整个重构过程中，我们还随之总结了几个有用的规则。

这篇文章一个主要的目的是：你应该有意的思考如何进行状态管理，因为这是个很需要仔细琢磨的难题。不要相信任何人说“有一个简单的模式/第三方库”可以解决它，因为它根本不存在。

鉴定应用中的状态类型，管理好不同类型的状态，保证好状态的一致性，以用心的态度设计你的应用。
