#koa@2.5.0源代码解读

## koa简介

`koa`是由`Express`原班人马开发的一个nodejs服务器框架。koa使用了ES2017的新标准：`async function`来实现了真正意义上的中间件（`middleware`）。`koa`的源代码极其简单，但是借由其强大的中间件扩展能力，使得`koa`成为了一个极其强大的服务器框架。借助中间件，你可以做任何nodejs能做到的事儿。

## 一些繁琐的交代

本文并不会像其他的代码分析那样贴段代码加注释，**因此需要你自己打开`koa@2.5.0`的源代码一起阅读。**

`koa`目前已经更新到了2.x版本，1.x以前的版本相对于`koa@2.x`已经不再兼容。本文针对的是`koa@2.5.0`进行的代码分析。

此外，`koa`的源代码里面涉及部分`http协议`的内容，这部分内容本文不会过分强调，默认读者已经掌握了基本的知识。


另外，用于`koa2`是用了ES2017新特性编写的，因此你需要了解一些ES2017的新语法才行。

为了使得这篇文章简单，我有意地忽略了错误处理，参数判断之类。

本文你还可以在[这里](https://github.com/doxjs/koa_source_code_analysis)找到。

## $1.查看package.json

对于`nodejs`甚至是`JavaScript`项目，第一件事儿就是看看它的`package.json`。`package.json`里面可以找到不少有用的信息。

我们打开`koa@2.5.0`的目录，发现它依赖了不少的库。其实这些库大多都十分简单，`koa`的编写原则其实就是把功能分割到其他的库中去。我们暂且先不管这些依赖。

我们找到`main`字段，这里就是‘通往新世界的大门’了。顺着`main`打开`lib/application.js`。

## $2.分析application.js

好家伙，一上来就是一大串的引入，这可不是什么好东西，咱么先不看这些东西。先看下面的代码。

首先是定义了一个`application`的类，接着在构造函数中定义了一些变量。我们主要关注以下几个变量，因为他们的用处最大：
```js
    this.middleware = [];
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
```

`Object.create`是用来克隆对象的，这里克隆了三个对象，也是`koa`最重要的三个对象`request`，`response`和`context`。这三个对象几乎就是`koa`的全部内容了。待会儿会逐一分析。

我们接着往下看，`listen`函数大家都很熟悉了，就是用来监听端口的。`koa`的`listen`函数也很简单。
```js
    const server = http.createServer(this.callback());
    return server.listen(...args);
```

短短两行，对于`nodejs`不熟的同学，建议在这里就打住了。其中`this.callback()`是个什么玩意儿呢？它返回一个函数，这个函数接收两个参数`request`和`response`，也就是`createServer`的回调函数，在中间件原理章节会更详细介绍。

接着就是`toJSSON`这个方法。`JSON.stringify`调用的方法，目的是当你`JSON`化一个`application`实例时，返回指定的属性，而并非所有。这个用处不大，几乎用不到。

`inspect`也就是调用了`toJSON`这个方法而已。

接着就是`use`函数了。`use`函数本身不是很复杂，但是`use`函数作为中间件的接口，背后的中间件却有点儿复杂。为此，本文在后面专门解读了中间件相关的源代码，这里暂时跳过。

`callback`，`handleRequest`，`respond`这几个方法涉及中间件的，因此放到中间件的章节讲。

`createContext`这个方法是用来封装`context`的。这个`context`就是你在使用`koa`的`use`方法，你传递的回调函数的第一个`ctx`参数。`createContext`执行的最重要的操作就是把`context.request`设置成了`Request`，把`context.response`设置成了`Response`。以及把`Response.res`h和`Request.req`分别设置成了原生的`response`和`request`。

为什么这样说，这个就得追到`context.js`和`request.js`以及`response.js`的代码里面了，先等等。

值得强调的是，这里的`Request`和`Response`并不是`nodejs`里面的，而是`koa`封装过后的。为了区分原生的和`koa`封装好的，我把`Request`和`Response`称为封装过后的，`request`和`response`称为原生的。

封装的东西看起来并没有什么高大上，无非是把常用的一些方法给简化了。就像`jquery`简化了`js`对`dom`的操作一样。

## $3.分析context.js

打开`context.js`，代码不多，但是含金量挺高的。首先是把`proto`赋值成一个对象，这个对象也是模块的导出值。

`inspect`和`toJSON`功能和`application.js`里面一样，不做过多介绍了。

接着看到个`assert`，这个和nodejs里面的`assert`其实是差不多，它其实是提供了一些断言的操作。比如`equal`，`notEqual`，`strictEqual`之类的。比较有意思的是，`assert`提供了一个深度比较的方法`deepEqual`，这个可是个好东西。`js`里面的深度比较一直是个比较麻烦的问题，有经验的程序员会使用`JSON`来比较，这里提供了一种性能更好的方法。代码其实不复杂，就是引用了`deep-eqaul`这个库而已，有兴趣的可以去看看哦。


跳过两个关于错误处理的函数（本文不讲解错误处理），来到了`context.js`最精华的地方了。
这里使用了`delegate`这个库。这是个啥？`delegate`其实很简单的，你甚至不需要去查看`delegate`的源代码，看我解释就行了。

`delegate`提供了一种类似`Proxy`的手段，也就是代理。代理什么？具体来说`delegate(proto, 'response')`这段代码的意思就是把`proto`上的一些属性代理到`proto.response`上面去。具体是哪些代理呢？就是接下来排列工整的代码做的了。`delegate`区分了`method`，`getter`，`access`等类型。前面两个还好理解，就是方法和只读属性，第三个呢？其实就是可读可写属性罢了，相当于同时代理了`getter`和`setter`。所以其实你访问`ctx.redirect`实际上访问的是`request.redirect`，以此类推。需要注意的是，这里的`request`和`response`不是nodejs原生的，是`koa`封装过后的。

`context.js`就这么简单。

## $4.request.js & response.js

`request.js`和`response.js`分别是对`createServer`回调函数接收的的`request`和`response`进行封装。

先看`request.js`。还记得`createContext`吗？我们说过，他把`Request.req`设置成了原生的`request`。所以你可以看到，很多方法其实本质就是在操作`this.req`，这一点和`response.js`类似，后面就不重复说了。

首先是一些个常用的属性，`header`分别设置了`getter`和`setter`，都是对`this.req.headers`操作。`headers`和`header`一模一样，并不是用来区分单复数的（这有点儿坑，初学以为headers是设置多个的）。接下来还有很多常用的属性，就不一一介绍了，什么`url`，`method`之类的，稍微熟悉点儿nodejs的同学都能够实现出来。

值得注意的是`query`和`querystring`，一个返回的是对象，一个是字符串哦。

你或许会问`search`和`querystring`有啥区别。区别，emmmmn。。。可能是为了完整吧，毕竟express都有个search，`koa`也要提供。。。

另外需要说一下的是，这里的很多属性的操作涉及到了`http`协议的内容了，比如`fresh`。`http`是个很大的内容，不做讲解。如果遇到看不懂的代码，不妨去查看相关的`http`协议哦。

另外在`idempotent`你可以看到`!!~`，这是个啥玩意儿？？？第一次看见都是一脸懵逼。这个其实就是位操作而已。我们一般把`!!`看做一组，它的作用是把任意数据变成`boolean`值。这个操作其实很简单，就是判断是不是-1，如果是-1，那么就是false;如果不是-1，那么都是true。这个操作很巧妙。稍微解释一下吧。

我们假设数字是8位表示的，那么-1的原码就是1000 0001，反码就是1111 1110，补码就是1111 1111。而~操作符是取反的意思，所以取反以后就成了0000 0000。计算机存储负数是用的补码（相关知识可以取google搜索一下），所以最后就是判断是不是-1的。

有几个`accept`打头的函数可以忽略，这几个函数是判断是否符合指定类型、语言、编码的，它内部调用了一个`accepts`的库。这个功能其实用得很少，但涉及编码之类较为复杂的知识了。

在最后的代码里面，`request.js`提供了`get`方法，其实就是获取`header`。

让我们转到`response.js`里面去。劈头盖脸一看，和`request.js`差不多，知识封装的方法和属性不一样而已。

首先是`socket`，这个是套接字，`http`模块的底层，不讲解。

`header`调用的是`getHeaders()`，获取已经设置好的所有的`header`，同`headers`。`status`设置状态码，比如200,404之类的。值得一提的是，通常情况下使用nodejs的statusCode还需要你设置一个statusMessage来提示用户发生了什么错误，`koa`会智能的为你设置好。比如你设置好了`status`为404，会自动把statusMessage设置成`404 not found`。这是因为`koa`使用了`statuses`这个库，这个库会根据你传入的状态码返回指定的状态信息。


接下来是`Response`最重要的一个属性，也就是`body`。对`body`的操作反应在内部的`_body`上面。`body`的setter做了各种处理。比如判断传给body的值是不是空，如果是空就进行一些操作。比较有意思的是，body的setter会在你没有设置`Content-Type`时，判断一下传递给body的数据是个什么类型。

1. 当传递的是字符串时，它使用了一个正则：`/^\s*</`来判断是html还是text。很明显，这个正则很简陋，在很多情况下并不能正确判断，比如`<----`就会被判断成html。所以`body`的类型还是要手动的设置`type`才行。

2. 当传递的是buffer的时候，把类型设置称为`bin`（记住，`type`是koa封装过后的属性，它会根据你设置的`type`自动匹配最佳的`Content-Type`。比如你把`type`设置成'json'，实际上最后的`Content-Type`会是`application/json`。后面会说实现方法的）。

3. 当传递的是个stream（通过判断它是否拥有pipe这个函数）,先绑定回调函数，当res发送完毕的时候，销毁这个stream，避免内存浪费。接着错误处理。接着判断以下现在这个stream和原来`body`的值是否相同，如果不是的话，那就移除`Content-Length`，交给nodejs自己处理。(实际上nodejs也并不会处理，为啥呢？header必须在正文发送之前发送，但是Stream的字节数要在发送完才知道，so，你懂得)。最后把`type`设置成`bin`，因为stream是二进制的数据流。

4. 不满足以上三种，那么就只能是json了呗（别问我为什么不判断boolean，symbol这些，谁会没事儿干发送这些玩意儿？）。移除`Content-Type`（你可能想问，为啥呢？因为你传递的实际上是个Object对象，需要stringify之后才能知道它的字节数，这个其实会在后面处理的）。设置`type`成`json`。

至此，`body`的`setter`分析得差不多了。

接着到了`length`，这个其实就是封装了设置`Content-Length`的方法。反倒是它的getter有点儿复杂来着。我们不妨细看一下。

首先判断设置`Content-Length`没有，有就直接返回，有的话那就分情况读`body`的字节数。当`body`是stream的时候，啥都不返回。

这里有个奇淫巧技，`~~`这个玩意儿可以用来把字符串转换成数字。为什么呢？！我就知道你要问！其实这个东西要对js有比较高的理解才行的，js里面存在隐式类型转换，当遇到一些特殊的操作符，例如位操作符，会把字符串转换成数字来进行计算。其实`+`这个符号也可以进行字符串转数字（str+str这个不算哈，这个不会进行隐式类型转换），那么为什么要用`~~`而不是`+`呢？我思索再三，认为是作者可能不了解。但实际上，`~~`要比'+'安全，`+`在遇到不能转换的式子时，会返回NaN，而`~~`是基于位操作的，返回安全的0。


跳到`type`，这个和`length`类似，是对`Content-Type`实现的封装。这里引用了一个`mime-types`的库，这个库功能很强大，可以根据传入的参数返回指定的`mime`类型。比如我们设置`type`为json，会去调用`mime-types`的`contentType`函数，然后返回`json`类型的`mime`，也就是`application/json`。

同`request.js`一样，`response.js`同样封装了`set`和`get`两个方法，用于设置和读取`header`。

`inspect`和`toJSON`又来了。。。

`response.js`很多的属性和方法并没有提及，这是因为这些属性和方法就是做了简单的封装而已，方便调用，很容易理解。


好了，至此`response.js`也分析完了。

## $5.koa中间件原理分析

`koa`的中间件原理很强大，实现起来其实并不是特别复杂。记得怎么使用`koa`中间件吗？只需要`use`一个函数就行了！这个函数接受两个参数，一个是`context`，我们已经分析过了。另一个是`next`，这个就是中间件的核心了。

让我们回到开头，看看`use`怎么实现的。不看错误处理的那些内容，这里先对`fn`进行了一次判断。判断什么呢？判断`fn`是不是generator function。koa官方建议是不要继续使用Generator function了，换成了async function。如果你使用的是Generator function，那么内部会调用`co`模块来处理。由于处理内容比较晦涩，且与正文关系不大，故不作讲解。我们假设所有的中间件都是async function。

`application`维护了一个`middleware`的队列，`use`方法把中间件推送进这个队列，除此之外什么都没做。

还记得`listen`方法吗？它调用了`callback`这个方法。最终的答案都在这里了！

看到`callback`方法。首先，它对`middleware`队列调用了`compose`方法。我们打开`compose`对应的模块，短短几十行代码。

不看错误处理，那么`compose`只有一个`return`语句，返回一个函数。这个函数有两个参数`context`和`next`，熟悉吗？这不就是中间件函数吗！别慌，接着往下看。

首先声明一个`index`游标，接着定义一个`dispatch`函数，然后默认返回`dispatch(0)`。

`dispatch`函数用来分发中间件（和分发事件很像）。它接收一个数字，这个数字是中间件队列中某个中间件的下标。首先先判断一下有没有越界，也就是index和传入的i进行比较，没有越界把游标移动到当前分发的中间件。接着判断i是否已经遍历完了中间件队列，`i === middleware.length`判断。如果完了，就把fn设置成传入的next。接着使用`Promise.resolve`，并调用当前中间件，注意
```js
return Promise.resolve(fn(context, function next () {
    return dispatch(i + 1)
}))
```
这里传入中间件的第二个参数，也就是next，是一个函数，这个函数正是用来分发下一个事件的！！！中间件最重要的原理就在这里，为什么可以用next转移控制权，逻辑就在这里！

`compose`函数分析完毕了，记住`compose`的返回值，是一个类似中间件的函数。

回到`application`的`callback`方法中。定义了一个`handleRequest`函数并且直接返回，`handleRequest`其实就是`http.createServer`的回调函数。这个回调函数首先封装一下`createContext`，上面已经讲过了。接着调用了`application`上的`handleRequest`方法（别搞混了，这个是下面那个`handleRequest`方法）。

我们看看`handleRequest`方法，它接受两个参数，第一个是`context`，第二个是什么呢？其实就是`compose`处理后的`middleware`中间件队列。抛开一些’多余‘的代码不看，把它精简成这样：
```js
  handleRequest(ctx, fnMiddleware) {
    const handleResponse = () => respond(ctx);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
```
记得`fnMiddleware`的返回值是什么吗？是`dispatch(0)`。那记得`dispatch`的返回值是什么吗？是一个`Promise`。我们再来看看这个`Promise`
```js
return Promise.resolve(fn(context, function next () {
    return dispatch(i + 1)
}))
```

好好想一想，`fn`现在是第一个中间件，它先被调用了。在这个中间件里面调用了`next`函数，也就是相当于调用了`dispatch(i + 1)`，如此下去。这不就相当于依次调用了`dispatch`函数吗？

最后一点，中间件是async function，你明白为什么要使用`Promise`了吗？对了，就是为了await。

最后的最后，就是`respond`这个方法了，这个方法实际上就是对`statusCode`，`header`以及`body`进行处理，最后调用nodejs提供了发送数据的方法，向客户端发送数据。最后调用`ctx.end(body)`，结束本次`http请求`。

那么至此，`koa`中间件也就完了。


## 结语

`koa`的源码并不是十分复杂，有兴趣的同学可以自己再看看。希望这篇文章能给你帮助。
推广一下自己的[GitHub](https://github.com/doxjs),
