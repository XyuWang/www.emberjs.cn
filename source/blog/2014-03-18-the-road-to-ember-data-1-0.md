---
title: Ember Data 1.0 之路
author: Tom Dale & Yehuda Katz
tags: Recent Posts
---

**TL;DR** Ember Data即将迎来1.0时代。在能自信的保证不再破坏API之前，还需要实现最后几个特性。特别是：

1. 保证在一个双向的关联中得一端发生改变时，另一端保持同步，即使其并未加载。

2. 所有关联将使用异步模式，`DataBoundPromises`将确保其能再观察器、计算属性和模板中都能正常工作。

就像之前路由设计的尝试一样（已被证明是Javascript中得最好的实现），为了让Ember
Data更好的工作，花费了比设想要多很多的事件，不过现在其已经非常接近1.0正式版了。

* * *

在Ember 1.0发布后，基于Ember.js开发应用的开发者，非常感激在[语义版本](http://semver.org)的保证下，获得升级的动力及升级后的稳定性。

常常被问及“Ember.js非常棒，那么Ember Data怎么样呢？”现在，将告知Ember Data目前处于一个什么状态，还有就是不久的将来会变成什么样。

首先，好消息是：在Ember Data 1.0发布之前只会做一个主要的破坏性改变，目前正在尽最大的努力希望做到本次修改尽可能少的影响到现有应用。

在此之外，期望当前的特性集和API将会成为可见未来的稳定的基础。换而言之，不在希望为了容纳未来架构的改变，进行面向用户的破坏性修改。

那么为什么还不发布1.0呢？主要是因为正在完成两个特性：一个是改进关联架构，另一个是使处理关联关系的API更加一致。

## 唯一真理

记录间关联关系建模无疑是Ember Data中最难的特性。找到一个通用的方案非常的复杂，因此每个JSON服务器可能返回一种格式。当引入通过Websocket实现的流式改变就变得更加有意思了。

最简单的办法就是宣告这个问题与领域相关度太高，每个开发人员应该自己选择一个简单的抽象方法手动的实现关联关系（例如使用计算属性）。

然而却发现自己尝试为自己的领域模型关系构建最小化框架的开发者，不久都会放弃。就如在Ember中一样，都希望能梳理出一种抽象，能解决应用中得这些问题。

因此希望能构建一种强大的可以为高级开发者节约开发时间，并且也能让Web客户端应用初学者能快速上手的方案。

例如，有时一个`has-many`关联关系可能保存在父记录的JSON中：

```js
{
  "id": 1,
  "name": "Lord Grantham",
  "children": [2, 3, 4]
}
```

但是也可能被保存在子的一个外键：

```js
{
  "id": 2,
  "name": "Lady Mary",
  "parent_id": 1
}
```

同一个应用根据记录是否被加载或者保存，而同时提供两种格式的关联关系。

当引入通过与服务器端的一个`socket`链接来进行流式改变时，问题变得更加复杂，这里不进行详细的描述。

强调这个复杂性德关键原因是表示不可以通过硬编码的方式来支持每种方式；每种表示都会与其他多种表示同时存在并相互协作。

单向的关联关系问题并不大，在碰壁之前可以通过手动的方案很好的解决问题。真实的问题来源于双向的关联关系，例如：一篇文章有很多评论，而每一条评论又都属于这篇文章。

由于关联关系的两端并不是总是同时加载的，而且每边都可能使用不同的方式表示关联关系（一个在评论中得主键，或者在文章中的一个数组），维护这个关系一直以来都非常的困难。

特别在需要维护从服务端更新过来的数据的时候，这个问题变得尤为复杂。假设刷新文章时，提供了一个新的评论集合，而这时用户又正在创建新的评论。那么此时如何确保`comments`数组能够考虑这些所有的因素？

在上层，解决方法是这样的：作为在每条需要被同步的记录中保存关联关系的状态的一种替代方案，在内存中维护了一个实例，用来标识记录间的逻辑关联关系。无论应用加载了关联关系的那一端，这个实例便被创建。

例如，如果在一条评论上修改了`belongs-to`，那意味着改变了`has-many`一端，即便文章并没有加载到应用中。当应用最终加载这篇文章时，应该可以将`belongs-to`的改变应用到这篇文章。

The long and short of it is that the two sides of a two-way relationship will
remain in sync in Ember Data 1.0, regardless of the order the records were
loaded, the way they were loaded, or how the relationships were represented in
the payloads.

简单地说就是Ember Data 1.0维护关联关系的两端，使其一直保持同步，不论记录加载的顺序是如何的，也不理会加载的方式是什么，或者载荷中关联关系的表示是什么形式的。

[single-source-of-truth](https://github.com/emberjs/data/tree/single-source-of-truth)分支正在完成这项工作，这里要特别感谢[Igor Terzic](https://github.com/igort)，期望能很快的合并这个分支。

## 异步关联关系

目前在Ember Data中，需要事先指定关联关系是异步的还是同步的。决定一个关联关系是否是异步的，需要知道服务器在发送数据给应用时是如何表达这些数据的。这并不太大的问题，但是却使得应用的语义与服务器端的语义紧密耦合。

最严重的问题是开始重构服务端的API。例如，在一个`has-many`关联关系中，可能会用一个客户端可以用来获取记录的URL来取代一个内联的记录ID的数组。

一下子，应用就会以`Zalgo-esque`的方式无法工作（如果没有阅读过Issaac的[异步接口的优秀随笔](http://blog.izs.me/post/59142742143/designing-apis-for-asynchrony)，可以去看一看）。

当设计API的时候，认真的考虑了重构的方便性，但在这个问题上犯错误了。对服务器端API的一些小得修改不应该影响到使用其的应用。

它的价值在于不是所有的重构都是对等的。在这种情况下，发现所有的真实的应用程序，当其成长时，几乎总是围绕载荷来完成的，这样总是会遇到问题。（其他情况下，如果发现应用很好按照一个特定的方式改变，那么可能会采用惯例优于重构。）

解决的方法是将所有的关联关系视作异步的，并使用承诺表示其值。通过依赖`Promises/A+`来保证异步性，这样可以避免应用中出现`Zalgo`。今后，如果关联关系的表示发生改变，或者关联关系加载的顺序发生改变，应用不需要做任何改变就可以如同之前一样工作。

在JavaScript中使用承诺非常的简单：只需要调用`.then`方法，并在回调中处理返回值即可。但是如果希望在模板或者一个计算属性中使用Ember Data的关联关系应该怎么办呢？它们又会以怎么样的形式工作呢？

Ember Data 1.0引入了一个`Promise`的子类`DataBoundPromise`。这个对象允许如同观察一个不同对象的属性一样观察承诺的属性。当承诺被履行时，这些属性就会被更新来匹配基础对象。如果从一个`DataBoundPromise`对象中`get`一个属性，如果这个承诺并没有被履行，此时会返回`undefined`。

```js
var article = comment.get('article');

// If the promise has not yet resolved
article.get('title'); //=> undefined

// If the promise has resolved
article.get('title'); //=> "Ember Data Roadmap"
```

基于这个思想，在使用Ember Data绑定的上下文中（模板、计算属性和观察器），可以如同使用同步值一样使用关联关系，让数据绑定系统来处理承诺。这就意味着在大多数情形下，这个改变不会影响到现有的代码，并的确会提高在模板和计算属性中使用异步关联关系的技能（对于现今这是最主要的难点）。

当在一个命令式的Javascript中（计算属性之外）使用关联对象时，则不应该依赖这种行为；在那种情况下，应该使用承诺的`then()`方法来确保值的可用性，以免导致应用中出现`Zalgo`。只有当承诺履行时，依赖Ember的数据绑定功能来更新模板或者计算属性，这时才应该使用`.get`来获取`DataBoundPromise`的属性。

**TL;DR** 应该仅在观察器或者计算属性中使用`DataBoundPromise`的`get`方法。其余情况应该将其视为普通的承诺，并使用其`.then`方法。

## 其他改进

在发布1.0之前，还将修复一系列的Bug。其中一些是目前还没有定义的行为，一些是希望确保应用不再依赖的Bug。

最突出的是，目前`RESTAdapter`在`pushPayload`中处理嵌套的或者旁路载入的数据时错误的调用`normalize`钩子。[修复 #1804](https://github.com/emberjs/data/issues/1804)目前优先级很高，是阻碍发布1.0的一个关键问题之一。

此外还需要保证1.0版本的文档应该具有一个高的质量标准，如同Ember一般。

## 接下来

Ember Data将按照[语义版本化](http://semver.org)来发布，因此需要为未来几年的发布提供一个稳定的基础。

Ember.js未锁定API之前的迭代方案为以后几年的迭代提供了一个基础，其可以保证良好的向后兼容性。希望这一切也被用于到Ember Data中。

一旦以上列出的问题一解决，就将发布Ember Data 1.0，此后一段时间将不在做破坏性的修改。另外将保持Ember.js和Ember Data的发布同步，因此Ember Data的第一个版本1.x版本将与那时的Ember的版本相同。

例如，当Ember Data发布第一个稳定版本时，Ember.js的版本是1.7，那么Ember Data的第一个版本号也将是1.7。

如同Ember.js，Ember
Data也将按照被Chrome影响的[六周发布方法](http://emberjs.com/blog/2013/09/06/new-ember-release-process.html)。这种发布方法已经在可预见性和动力方面得到了证明，并且反馈都是积极正面的。重要的是，有这种分离的`stable`，`beta`和`canary`发布也可以清晰的知道[哪些特性是稳定的](http://emberjs.com/builds/#/beta)可以在生产环境中使用的，哪些还需要继续改进。

## 数据框架

如[上周在Fluent上的基调](http://www.youtube.com/watch?v=jScLjUlLTLI)中所述，框架存在是为了使得做正确的事情感觉比做错误的事情要好。通过在代码中编写最佳实践，框架使得社区可以构建未来的抽象，创建随时间变得越来越强大的良性循环。

**Ember Data就是一个用来管理模型和关联关系的框架。**

要使关联关系正确这一点非常难，但是通过向诱惑屈服，让每个人写自己的方案，那么社区就不能构建一个在关联关系概念之上的进一步的抽象。

在早期，开始构建Ember Data时，尝试去编写好的实践，但是没有提供一种足够灵活的基本的底层作为逃逸值。因为人们总是与没有掌控能力的服务端进行交互，所以意识到有一个较好的逃逸值对Ember Data来说比惯例更加重要。

为了确保确保社区依然能够基于Ember Data抽象进行构建，通过努力将应用之间不同的代码独立到适配器中。这就意味着，如果某人写了一个Ember Data插件，那么可以假设模型和关联关系在所有使用其的应用中都是一样的，尽管应用的后端可能不一样。

在早期的Ember Data版本中，太过强调分离了，迫使每个应用在适配器层需要承受太大的代价。当六个月前重新启动Ember Data时，认真分析了如何在这些竞争问题间需求一种更好的平衡。基于此后获得的反馈，现在相信Ember Data已经非常适合每种有特殊后端的应用，就如同按照Ember Data的方式来构建一个后端的应用一般。

现在需要保证1.0发布前所有的都能正确工作。Ember.js中得路由也经过了几个类似的迭代，就如同现在Ember Data遇到的苦恼一样，不过相信最终结果会为自己发声的。真实的使用对于设计过程非常的重要和有用，在发布之前仔细的思考问题也非常的重要。

鉴于此，希望当宣布Ember Data符合1.0的质量要求时能得到大家的信任。如果现在开始使用Ember Data来构建应用，那么会因为将带来的有一个能分享理解的社区的向前的动力而高兴。

