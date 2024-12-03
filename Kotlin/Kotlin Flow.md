# Kotlin Flow

## 概述

在Kotlin标准库中虽然也提供了`Sequence`与集合操作符，同样都能进行链式数据流操作。

但依然是**阻塞线程**的**同步数据流**操作，那么有没有与`RxJava`类似的**异步数据流**操作呢？

在Kotlin协程中提供的**响应式编程**方式就是`Kotlin Flow`。

![Flow定义.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75a85cc28a9b4c10870ed94350674f2c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`Flow`的定义很简单，就只是表示这个类能够被订阅收集。

而其中`FlowCollector`则定义发送数据的功能。

![FlowCollector定义.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d992fbdb39f54eea807d77c4c4c4009b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

在基础Kotlin协程中，

- `launch`启动的协程，没有返回值，运行后就结束，类似于`RxJava`的`Completable`。
- `async`启动的协程，能通过`await`函数获取协程返回值，但只能返回单个值，且无法进行数据流操作。

而`Kotlin Flow`的出现，恰好弥补了Kotlin协程对于**多个值异步运算**的不足，并且允许进行复杂的数据流操作。

相当于`RxJava`的`Observable`与`Flowable`类型，`Maybe`类型自然也可以兼容。



## 创建

首先来看看`Flow`数据流是如何创建的。

`Flow`的创建方法有很多种，最简单的是使用顶层函数`flow`。

![flow函数源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/375e5a13c93d4d8bb981c6d95ab81f66~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

其参数是`suspend`修饰，以`FlowCollector`为接收者的**挂起函数**，能直接调用`emit`发送数据。

```kotlin
val flow = flow<String>{
     for(i in 1..5){
         emit("result $i") 
     }
 }
```

其他还有诸如`asFlow`、`flowOf`等创建方式。

![Kotlin Flow其他构造方式](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/265ba7bc991c4c15a77741612f104f08~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

官方提供了很多类型转换到`Flow`的拓展函数，这里就不多列举了，有兴趣可以看源码。

> - 由于`flow`函数的代码块同样是由`suspend`修饰，内部可以调用挂起函数。
> - `flow`代码块的`emit`函数是**线程不安全**的，所以`flow`函数的不能修改协程上下文，无法调用如`withContext`等函数，避免下游`collect`被调度到其他线程。
> - 如果要修改数据流的协程调度，只能调用`flowOn`函数。

## 末端操作符

`Flow`创建的数据流，是**冷流**，也就是必须要在调用**末端操作符**后才会执行数据流生产操作。

> - 数据流的创建与数据流的消费是**成对出现**的
> - 多个数据流订阅消费，也会同样有多个数据源生产创建

在`RxJava`中需要调用`subscribe`订阅消费数据流，在`Flow`中自然就是调用`collect`。

但`Flow`类中声明的`collect`函数是以`@InternalCoroutinesApi`注解的，表示外部不能直接调用该函数。

好在官方提供了同名的`Flow`类型拓展函数供外部调用。

![collect源码.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fa044389f844575bad18a6da5cbe539~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> `collect`是**挂起函数**，限制只能在Kotlin协程（或另一个挂起函数）中调用。
>
> 但`flow`及其他操作符都是普通函数，可以在协程外部创建。
>
> 也就是说可以做到**数据流创建与数据流消费的分离**
>
> ```kotlin
>  fun testFlow(){
>      val scope = CoroutineScope(SupervisorJob())
>      val flow = flow{
>           //创建数据流
>           (0..2).forEach {
>               println("emit $it")
>               emit(it)
>           }
>      }
>      scope.launch{
>           flow.collect {
>               //消费数据流
>               println("last collect result ：$it")
>           }
>      }.join()
>  }
> ```

除此之外，官方还提供了其他末端操作符

- 集合转换类型，如`toList`、`toSet`。
- 获取数据流特定元素，如`first`、`last`。
- 末端的运算符（累加器），如`reduce`、`fold`。

这几种末端操作符都是**挂起函数**，**数据流的消费被限制只能在Kotlin协程（挂起函数）中运行**。

另外，还提供了一个便捷的拓展函数`launchIn`，指定运行在特定的协程作用域，但会忽略末端的数据流，通常和其他操作符配合使用。

![launchIn源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6a0d77bd1e243ba954c205a8862f688~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 中间操作符

在`Flow`中提供了丰富的中间操作符，让数据流能够产生更多玩法。

这些中间操作符会**通过拦截（消费）上游的数据流**，进行处理后，再返回**新的数据流转发给下游数据流**。如此便能使不同中间操作符（消费者），链式调用的串联在一起。

> 中间消费者并不会触发数据流被发射

目前`Flow`的中间操作符数量相对`RxJava`而言还比较少，但也足够满足日常使用了，甚至功能更加强大，可能有小部分较少使用场景的生僻操作符还需要自定义。

### 变换

先从熟悉的老朋友，数据流的变换操作符开始

- **transform**

![transform源码解析.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f744318857c849f8af0f17c33db2fc15~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

接收一个`FlowCollector`作为接收者的函数类型，数据流上游传递的值作为函数参数。

`transform`操作符实现了**拦截收集上游数据并转发**的功能。

> 在订阅拦截了上游数据流后，通过以`FlowCollector`作为接收者的函数类型，也就具备**向下游发送数据**的能力。
>
> - `transform`能多次调用`emit`函数发送数据

作为最基础的实现，其他数据流**变换操作符**都是在`transform`基础上拓展而来。

- **map**

  将上游数据流的每个值，通过指定变换函数的结果传递给下游。

  ```kotlin
  flow{
       (0..2).forEach {
           println("emit $it")
           emit(it) 
       }
   }.map{
       println("running map $it")
       it * 2 + 1
   }.collect{
       println("last collect result $it")
   }
   
   emit 0
   running map 0
   last collect result 1
   emit 1
   running map 1
   last collect result 3
   emit 2
   running map 2
   last collect result 5
  ```

当需要将上游数据流每个值都转化为另一个数据流的使用场景，在`RxJava`会用到`flatMap`。

但在`Flow`并没有同名操作符，不过有类似的存在。

- **flatMapConcat**

  将上游数据流的每个值，转化为另一个数据流。

  将新Flow数据流展开，等待新的数据流将每个值传递给下游，随后才继续从上游获取下一个值，顺序执行。

  ```kotlin
   @FlowPreview
   fun test() = runBlocking{
       flow{
           (0..2).forEach {
               println("emit $it")
               emit(it)
           }
       }.flatMapConcat {index->
          flow<String> {
             println("running flatMapConcat first $index")
             emit("flatMapConcat first $index")
             println("running flatMapConcat second $index")
             emit("flatMapConcat second $index")
             println("running flatMapConcat end $index")
          }
       }.collect {
           println("last collect result ：$it")
       }
   }
   
   emit 0
   running flatMapConcat first 0
   last collect result ：flatMapConcat first 0
   running flatMapConcat second 0
   last collect result ：flatMapConcat second 0
   running flatMapConcat end 0
   emit 1
   running flatMapConcat first 1
   last collect result ：flatMapConcat first 1
   running flatMapConcat second 1
   last collect result ：flatMapConcat second 1
   running flatMapConcat end 1
   emit 2
   running flatMapConcat first 2
   last collect result ：flatMapConcat first 2
   running flatMapConcat second 2
   last collect result ：flatMapConcat second 2
   running flatMapConcat end 2
   after flow
  ```

  这等效于`RxJava`的`flatMap`操作符。

- **flatMapMerge**

  与`flatMapConcat`相似，但其内部是**允许并发操作**的，`flatMapMerge`接收上游数据流后，会尽可能收集上游数据值，直到达到最大并发数，然后一次性将新的数据流**并发**的传递到下游。

  > 由于`flatMapMerge`是并发的，所以不保证发送顺序

  ```kotlin
  @FlowPreview
   fun test() = runBlocking{
       flow{
           (0..2).forEach {
               println("emit $it")
               emit(it)
           }
       }.flatMapMerge {index->
          flow<String> {
             emit("flatMapConcat $index ,1")
             emit("flatMapConcat $index ,2")
          }
       }.collect {
           println("last collect result ：$it")
       }
   }
   
   emit 0
   emit 1
   emit 2
   last collect result ：flatMapConcat 0 ,1
   last collect result ：flatMapConcat 0 ,2
   last collect result ：flatMapConcat 1 ,1
   last collect result ：flatMapConcat 1 ,2
   last collect result ：flatMapConcat 2 ,1
   last collect result ：flatMapConcat 2 ,2
  ```

  > `flatMapMerge`函数接受一个`concurrency`参数，表示允许接收上游数据的并发数，默认为16。
  >
  > 当concurrency = 1时，就相当于`flatMapConcat`。

但官方其实并不太推荐使用这两个操作符。从操作符注释上看，官方似乎更希望我们保持**数据流的线性操作**，提高数据流操作的可读性。

毕竟`flatMap`比起`map`，逻辑相对没有那么清晰，通常情况下使用`map`也就足够了。而且官方也提供了`FlowCollector`拓展函数`emitAll`用于发送数据流。

`flatMapConcat`内部实际就是调用了`emitAll`来发送新的`Flow`数据流。

![flatMapConcat源码.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca7dc3b3c9384a54a7be32ff04e1bbdb~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

并且`flatMapConcat`和`flatMapMerge`都是以`@FlowPreview`注解的，属于预览性质，并不保证后续版本的向下兼容。

> 如果有需要转化为新数据流的需求，在`map`中调用`emitAll`可能是更好的选择。

### 线程调度

还要记得`RxJava`的线程调度操作符`subscribeOn`和`observeOn`吗？

它们分别用于指定**数据流上游**和**数据流下游**的运行线程。

而`Flow`是基于Kotlin协程的，所以线程调度必然是修改`CoroutineContext`调度器。

但对于允许发送数据的操作符来说，会被限制在不允许内部调用诸如`withContext`修改协程上下文的挂起函数，**仅允许在数据流中使用`flowOn`操作符修改协程上下文**。

`flowOn`操作符指定的调度器，**只作用于该操作符上游的数据流**。

> 区别于定位相近的`subscribeOn`，`flowOn`**可以多次调用**。
>
> 指定的`CoroutineContext`作用范围是从当前`flowOn`操作符到前一个`flowOn`（或数据源）。
>
> 下游数据流未指定时，则继承外部协程上下文的调度器。

```kotlin
fun test() = runBlocking{
     val myDispatcher = Executors.newSingleThreadExecutor()
             .asCoroutineDispatcher()
     flow {
         println("emit on ${Thread.currentThread().name}")
         emit("data")
     }.map {
         println("run first map on${Thread.currentThread().name}")
         "$it map"
     }
     //作用于前面flow创建与第一个map
     .flowOn(Dispatchers.IO)
     .map {
         println("run second map on ${Thread.currentThread().name}")
         "${it},${it.length}"
     }
     //作用于第二个map
     .flowOn(myDispatcher)
     .collect {
         println("result $it on ${Thread.currentThread().name}")
     }
 }
 
 emit on DefaultDispatcher-worker-2 @coroutine#3
 run first map on DefaultDispatcher-worker-2 @coroutine#3
 run second map on pool-1-thread-1 @coroutine#2
 result data map,8 on Test worker @coroutine#1
```

相比`subscribeOn`与`observeOn`，`flowOn`在使用上更加灵活。

> 如果想要达到`observeOn`的效果，可以尝试利用外部协程调用`withContext`切换调度器。

### 事件

在`Flow`中还提供了数据流的hook操作符。

- **onStart** 在**上游数据流开始之前**执行操作

  参数代码块是以`FlowCollector`作为接收者，具有**发送数据**功能。

  > 如果`onStart`内发送数据，会先将`onStart`数据发送传递到下游消费完成后，才开始将上游数据流传递到下游。

  ![onStart操作符声明.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72d57d54c84c44479b2f6f4ef2af4b99~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  从功能上看，其相当于`RxJava`中的`startWith`与`doOnSubscribe`操作符的结合替代品，并且功能更加强大。

  其参数代码块是以`suspend`修改的挂起函数，使其内部也能调用挂起函数，如`delay`，能轻松实现延迟发送数据的操作。

- **onEach**

  在**上游数据流的每个值发送前**执行操作。

  ![onEach操作符声明.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98f6edfbbbdf46e6a4d24c415d83d173~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  等效于`RxJava`中的`doOnNext`或`doOnSuccess`，并且同样有`suspend`关键字修饰代码块。

- **onCompletion**

  在**上游数据流**完成、取消、异常时，执行的操作。

  不同于`RxJava`中`onComplete`或`doFinally`的结束时兜底，`onCompletion`接收的参数同样是以`FlowCollector`作为接收者的函数类型，具有**发送数据**功能。

  ![onCompletion操作符声明.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0249b09b5ac448a68f95dd9ec226c28b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  而且函数类型的参数提供了对于异常抛出的收集，如果正常结束则为null。

  > 注意，`onCompletion`**并不能捕获异常**，**出现异常时无法发送数据**。
  >
  > 需要捕获异常并发送数据应使用`catch`操作符。
  >
  > `onCompletion`在数据流中的位置需要注意，详见下文中的重试操作符。

  该操作符可以等价为命令式代码：

  ```kotlin
  try {
       flow.collect { value ->
           ...
       }
   } finally {
       //onCompletion代码块
       ...
   }
  ```

  > 但既然是使用响应式编程的`Flow`，就尽量把所有操作都放到数据流中吧。

- **onEmpty**

  在**上游数据流完成**时，**没有任何值传递**消费，则触发执行操作，**可发送额外的元素**。

  ![onEmpty操作符声明.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e8334a59c1e410e8d54bb379ec6dede~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  > 上游数据流的数据也包括诸如`onStart`、`onCompletion`等操作符发送的数据。

### 异常处理

在`RxJava`时代，对于异常的捕获相信是最好用的功能之一，在`onError`中可以捕获到数据流中所抛出的异常。这在`Flow`里自然也有同样的替代品。

虽说可以直接`try-catch`代码块就把异常捕获了，但在数据流操作中，我们最好还是把**内部异常透明化**，重新抛出到外部来统一处理。

- **catch**

  捕获**上游数据流中所抛出的异常**，并**允许发送新的数据**。

  ![catch操作符声明.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2bfa8a67f489475a9986f3d0a38d3ad8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  > 被`catch`所捕获的异常，不会传递到下游。

  `catch`操作符最好放在数据流的最下游，便于捕获所有上游抛出的异常。

  类似于`RxJava`中的`onErrorRetrun`操作符与`onError`的结合体。

  > 但如果在下游`collect`中抛出异常，是无法捕获到的。
  >
  > 此时可以尝试将`collect`的部分操作，前移到上游的`onEach`中，然后由其下游的`catch`捕获并发送异常时数据。
  >
  > 最后可在外部协程中设置`CoroutineExceptionHandler`兜底捕获异常。

### 重试

与`RxJava`相同，当数据流中**出现异常**，**整个数据流将会结束**，导致**后续数据无法继续发送**。

但有些场景我们并不希望数据流就此结束，此时就需要使用重试机制。

- **retry**

  当**上游数据流出现异常**时，如果满足条件时，会捕获异常，并**重新发出上游数据流**，直到**超出重试次数后，继续向下游抛出异常**。

  ![retry操作符声明.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed989035b056431a867eeb5536c61a95~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **retryWhen**

  `retry`内部就是调用的`retryWhen`，其会在捕获到异常后，如果满足重试条件，将在其上游数据流，从**重新执行一次上游数据流**，直到再次捕获到异常。

  **不满足重试条件，会继续向下游抛出异常**

  ![retryWhen操作符声明.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bced93d0554342798f6b26176fb431d4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> - 如果`catch`操作符在上游数据流，会直接由`catch`捕获， 而不会进入`retryWhen`。通常需要`catch`在下游捕获由`retryWhen`抛出的异常。
>
> 这等效于在`RxJava`中把`onErrorReturn`放到`retryWhen`上游。
>
> - 当`onCompletion`操作符在上游数据流，异常会先进入`onCompletion`执行，但不会发出数据，然后才进入`retryWhen`，重试后也是如此。

如果需要使用`retryWhen`和`retry`操作符，**尽可能靠近需要重试的数据流操作**，避免出现不符合预期的情况。

### 合并

某些场景可能需要我们将数据流合并，在`RxJava`中可以使用`concat`与`zip`等合并操作符，在`Flow`中也提供了类似功能。

- **combine**

  将两个数据源中的**最新值**通过函数合并，并直接传递给下游。

  ![combine操作符源码.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4742c2330ec473499b25f45e028afc4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  类似于`RxJava`的`combineLatest`操作符，但在`flow`中可以是**任意类型的数据流**合并。

  ```kotlin
  //每个值延迟50ms发送
   val flow1 = flowOf("first","second","third").onEach { delay(50) }
   //每个值延迟100ms发送
   val flow2 = flowOf(1,2,3).onEach { delay(100) } 
   
   flow1.combine(flow2){first,second->
     "$first $second" 
   }
   .collect { println("result $it") }
   
   //flow2元素的1元素发送时，flow1数据源的second元素是最新值，
   //所以first会被忽略
   result second 1
   result third 1
   result third 2
   result third 3
  ```

- **combineTransform**

  效果上与`combine`相同。

  唯一区别只是，`combine`在合并后立即发送数据，而`combineTransform`会经过转换由**用户控制是否发送合并后的值**。

  ![combineTransform操作符源码.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/430aaeeab73a415eb4974dd24f26ae3e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **zip**

  将**两个数据源中的每个值都合并**，发送给下游，只要其中一个数据源结束，则数据流完成结束。

  ![zip操作符声明.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39f3c45bac72423687a782006a1d3e37~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  `zip`操作符与在`RxJava`中的表现一致，但函数类型支持挂起函数。

  ```kotlin
  //每个值延迟50ms发送
   val flow1 = flowOf("first","second","third").onEach { delay(50) }
   //每个值延迟100ms发送
   val flow2 = flowOf(1,2,3,4).onEach { delay(100) } 
   
   flow1.combine(flow2){first,second->
     "$first $second" 
   }
   .collect { println("result $it") }
   
   //由于flow1只有3个元素，所以flow2的第4个元素被抛弃了。
   result first 1
   result second 2
   result third 3
  ```

> `combine`与`combineTransform`操作符支持多个数据源合并，
>
> 而`zip`目前仅支持两个数据源合并。

### 常用操作符

除此之外，还有一些很多的操作符，这里只列举一些常用的，基本都是等效于`RxJava`对应的同名操作符。

- **first**

  **末端操作符**，只取上游数据流的第一个元素，或满足条件的第一个元素，后续元素抛弃。

  ![first操作符声明.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0be7164b9dfd4204bbb94b0afc10dc08~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **last**

  **末端操作符**，只取上游数据流的最后一个元素，**如果元素为null则抛出异常**。

  可使用`lastOrNull`获取最后一个可空元素。

- **filter**

  中间操作符，过滤上游数据流的值，仅允许**满足条件的值**继续传递。基于`transform`

- **take**

  中间操作符，只从**上游数据流获取指定个数的元素**，传递到下游，后续元素抛弃

- **debounce**

  中间操作符，在**指定时间内，只允许最新的值传递**到下游，多用于过滤上游高频率的生产者数据。

  ![debounce操作符声明.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47209ea1bcde4116851ab5c87fbb4d42~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  在`RxJava`时代，利用[RxBinding](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FJakeWharton%2FRxBinding)库可以将View事件转化为数据流，然后利用`decode`操作符来实现对于`EditText`的输入防抖过滤，以及`View`的点击防抖操作。

  > `decode`操作符在`Flow`中目前是不保证后续版本兼容的预览版本。

- **distinctUntilChanged**

  中间操作符，过滤上游数据流中的重复元素，等价于`RxJava`内的`distinct`操作符。

  ![distinctUntilChanged操作符声明.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4666cf71d2e14c139033866e10b4db1c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 操作符复用

在`RxJava`中，对于一系列通用的数据流操作，我们可能会想在多个数据流中复用，此时会先将这一系列操作利用`Transformer`封装起来，然后利用`compose`操作符进行统一调用。

比如对于网络请求返回数据的一些通用处理。

```kotlin
class MyTransformer<T>(...) : MaybeTransformer<T,T>{
     override fun apply(upstream: Maybe<T>): MaybeSource<T> {
      return upstream
             .subscribeOn(Schedulers.io())    
             .map { ... } //预处理请求返回
             .observeOn(AndroidSchedulers.mainThread())
             .onErrorReturn{...} //在异常时发送表示错误的数据 
     }
 }
 
 api.compose(MyTransformer(...)).subscribe(...)
```

在`Flow`的设计中，看得出来设计者在早期其实也有相同的想法，确实是有一个`compose`操作符。不过已经是废弃状态，在Kotlin里也确实不需要再整这么麻烦的事情了。

回想一下`Flow`里操作符是如何定义的呢？利用Kotlin的拓展函数，我们完全可以自定义一个操作符，内部调用这一系列操作。

```kotlin
fun <T> Flow<T>.preHandleHttpResponse(
     dispatcher: CoroutineDispatcher = Dispatchers.IO,
 ){
     map {response->
         if (it.code == 0) it
         else throw MyHttpException(...) //抛出自定义异常，表示业务执行失败
     }
         .flowOn(dispatcher)
         .catch{error->
             //全局处理网络请求异常
             emit(...) //在异常时发送表示错误的数据 
         }
 }
 
 flow{...}.preHandleHttpResponse().collect{...}
```

## 取消

在`RxJava`中`subscribe`会返回`Disposable`用于控制数据流取消，或者利用[AutoDispose](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fuber%2FAutoDispose)绑定`Lifecycle`来自动管理数据流关闭。

而`Flow`本身并没有提供取消功能，不过`Flow`的消费是**挂起函数**，必须运行在Kotlin协程，直接利用外部协程启动时的`Job`对象就能管理`Flow`的取消。

## 背压

在响应式编程中，数据背压是必然会存在的问题。

> 在生产者-消费者模型中，如果**生产者的生产速率远大于消费者的消费速率**，也就产生了**背压问题**。
>
> 仅出现在生产者与消费者分别在不同线程的异步场景。

### 限制生产端

在`RxJava`中，从RxJava2开始就专门推出了`Flowable`来解决数据背压问题。

并为其设置有个默认固定为128的异步缓存池以及`Backpressure`缓存策略：

> - **MISSING** ：不做缓存处理
> - **ERROR** ：缓存队列满后，抛出`MissingBackpressureException`异常
> - **BUFFER** ：缓存队列满后，继续扩大缓存容量，知道OOM
> - **DROP** ：缓存队列满后，直接抛弃后续新的值。
> - **LAEST** : 永远会将最新的一个值放入缓存队列，不论缓存池状态如何。

`Flow`同样也提供了类似的背压缓存策略

- **buffer**

  基于`Channel`开启一个缓存区。

  > `Channel`可以理解为Kotlin协程的`BlockingQueue`，接收和发送都是挂起函数。

  ```kotlin
  public fun <T> Flow<T>.buffer(
       capacity: Int = BUFFERED,
       onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
   ): Flow<T>
  ```

  参数`capacity`，表示缓存区容量，默认为`BUFFERED`，是`Channel`的常量。

  > - **RENDEZVOUS** ，无缓存区，生产一个消费一个。
  > - **BUFFERED**，创建个默认缓存容量为64的缓存区
  > - **CONFLATED** ，相当于把缓存容量设置为1，且缓存策略强制为`DROP_OLDEST`
  > - 自定义缓存区容量大小

  参数`onBufferOverflow`，缓存策略方式，默认使用`SUSPEND`，

  > - **SUSPEND** ：当缓存区满时，将生产者挂起，不继续发送后续值，直到缓冲区有空缺位置。
  > - **DROP_OLDEST** ：缓存区满时，移除缓存区中最旧的值，并插入最新的值
  > - **DROP_LATEST** ：缓存区满时，抛弃最新的值。

- **conflate**

  专门提供的快捷使用`buffer(CONFLATED)`的操作符

- **flowOn**

  实际上`flowOn`在**修改了协程上下文**后，内部也会开启默认缓冲区。

  相当于默认的`buffer()`，天然支持背压。

  ```kotlin
  flow{
       (0..100).forEach {
           delay(50)
           val currTime = System.currentTimeMillis()
           println("emit $it on ${currTime - startTime} ms")
           emit(it)
       }
   }
   .flowOn(Dispatchers.IO) //等价于buffer()，并且指定调度器
   .onEach { delay(300) } //消费端延迟
   .collect {
       val endTime = System.currentTimeMillis()
       println("result : $it on ${endTime - startTime} ms")
   }
   
   emit 0 on 93 ms
   emit 1 on 156 ms
   emit 2 on 220 ms
   emit 3 on 282 ms
   emit 4 on 345 ms
   emit 5 on 409 ms
   result : 0 on 409 ms
   emit 6 on 471 ms
   ...
   emit 9 on 657 ms
   result : 1 on 719 ms
   emit 10 on 719 ms
   ...
   //超出默认64位缓存区，生产端就被挂起暂停了，速度减缓
   //消费一个，从缓存队列中移除，然后生产一个填充队列。
   emit 79 on 5050 ms
   result : 15 on 5081 ms
   emit 80 on 5112 ms
   emit 81 on 5174 ms
   result : 16 on 5390 ms
   emit 82 on 5451 ms
   result : 17 on 5700 ms
   ...
   //直到生产端完成任务
   result : 34 on 11026 ms
   emit 100 on 11090 ms
   result : 35 on 11338 ms
   result : 36 on 11651 ms
   result : 37 on 11965 ms
   ... 
   //直到消费结束
  ```

  > 注意上面emit运行的时间，`buffer`、`flowOn`操作符还有**将上游数据快速并发**的附带功能。
  >
  > - 其实可以利用这一特性快速并发运行多个任务，但`buffer`无法指定调度器

  如果将上面`flowOn`替换为`conflate`，程序运行结果：

  ```yaml
   emit 0 on 99 ms
   emit 1 on 162 ms
   emit 2 on 224 ms
   emit 3 on 287 ms
   emit 4 on 351 ms
   emit 5 on 413 ms
   result : 0 on 413 ms
   emit 6 on 474 ms
   ...
   emit 9 on 660 ms
   result : 5 on 720 ms
   emit 10 on 720 ms
   ...
   emit 14 on 970 ms
   result : 9 on 1034 ms
   emit 15 on 1034 ms
   ...
   emit 19 on 1286 ms
   result : 14 on 1350 ms
   emit 20 on 1350 ms
   ...
  ```

  此时，都只能接收上一个最新的值，毕竟缓存区容量只有一个，其他新值都被丢弃。

  类似于`RxJava`的`LAEST`策略。

### 限制消费端

在消费端也提供了背压数据取舍的功能

- **collectLatest**

  按顺序收集上游数据流传递的每一个元素，如果当前元素还未处理完，又新元素传递过来后，取消当前元素的收集操作。

  ![collectLast操作符声明.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f412a980305047cc95c4ba5c96126c8e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  > `collectLatest`操作符对于上游数据流的耗时操作是不在意的，只有在`collectLatest`函数**内部的处理时间过长**才会被算作还未完成处理。

  将上面的程序稍作修改：

  ```kotlin
  //生产端不变
   flow.collectLatest{
       delay(300)  //延迟移到消费端
       ...
   }
   
   emit 0 on DefaultDispatcher-worker-1 @coroutine#3 (115 ms)
   emit 1 on DefaultDispatcher-worker-1 @coroutine#3 (177 ms)
   emit 2 on DefaultDispatcher-worker-1 @coroutine#3 (240 ms)
   emit 3 on DefaultDispatcher-worker-1 @coroutine#3 (304 ms)
   emit 4 on DefaultDispatcher-worker-1 @coroutine#3 (369 ms)
   emit 5 on DefaultDispatcher-worker-1 @coroutine#3 (433 ms)
   emit 6 on DefaultDispatcher-worker-1 @coroutine#3 (496 ms)
   ...
   //由于消费端一直在取消收集，只获取到最后一个值。
   result : 100 on Test worker @coroutine#104 (6721 ms)
  ```





## Channel

`Channel`在概念上类似于`BlockQueue`，**并发安全**的缓冲队列（**先进先出**），**实现多个协程间的通信**。

> 目前版本内部基于**无锁双向链表**结构，存在一个**永远不会删除的节点**作为将整个链表进行首尾相连的**根节点**。
>
> - 添加新元素时，就会在根节点的左侧进行添加，即添加到整个队列的末尾。
> - 取出元素时，会在先从根节点右侧开始移除并取出元素，即从队列顶部取出。
> - 每个元素节点的左侧（前）、右侧（后）节点都为CAS引用类型。
>
> `Channel`内的发送数据和接收数据默认都是挂起函数。

对于同一个`Channel`对象，允许多个协程发送数据，也允许多个协程接收数据。

区别于`Flow` ，`Channel`是一个**热流**，但其并不支持数据流操作。

> 即使没有订阅消费，生产端同样也会开始发送数据，并且始终处于运行状态。 ![Channel源码.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54c366517e8c4f53bab46e8e4cfed4a9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`Channel`接口只是定义了一些常量，实际功能定义还在其继承的两个接口`SendChannel`与`ReceiveChannel`。

### 创建

`Channel`本身是个接口不能直接创建，但其有个同名函数用于创建`Channel`，相当于一个工厂方法。

![Channel工厂方法.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4429c9751ad14ae5a830da6a3ff0c6e5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

其中`capacity`参数为缓冲区容量，通常是以`Channel`中定义的常量为值。

- **RENDEZVOUS**

  默认无锁、无缓冲区，只有消费端调用时，才会发送数据，否则挂起发送操作。

  > 当缓存策略不为`BufferOverflow.SUSPEND`时，会创建缓冲区容量为1的`ArrayChannel`。

- **CONFLATED**

  队列容量仅为一，且`onBufferOverflow`参数**只能**为`BufferOverflow.SUSPEND`。

  缓冲区满时，**永远用最新元素替代**，之前的元素将被废弃。

  > 创建实现类`ConflatedChannel`
  >
  > 内部会使用`ReentrantLock`对发送与接收元素操作进行加锁，**线程安全**。

- **UNLIMITED**

  无限制容量，缓冲队列满后，会直接扩容，直到OOM。

  > 内部无锁，**永远不会挂起**

- **BUFFERED**

  默认创建64位容量的缓冲队列，当缓存队列满后，会挂起发送数据，直到队列有空余。

  > 创建实现类`ArrayChannel`，内部会使用`ReentrantLock`对发送与接收元素操作进行加锁，**线程安全**。

- 自定义容量

  > 当`capacity`容量为1，且`onBufferOverflow`为`BufferOverflow.DROP_OLDEST`时，由于与`CONFLATED`工作原理相同，会直接创建为实现类`ConflatedChannel`。
  >
  > 其他情况都会创建为实现类`ArrayChannel`。

### 生产者

`SendChannel`是发送数据的生产者。

- **send**

  挂起函数，向队列中添加元素，**在缓冲队列满时，会挂起协程**，暂停存入元素，直到队列容量满足存入需求，恢复协程。

  ```kotlin
   public suspend fun send(element: E)
  ```

- **trySend**

  **尝试**向队列中添加元素，返回`ChannelResult`表示操作结果。

  ```kotlin
   public fun trySend(element: E): ChannelResult<Unit>
  ```

- **close** 关闭队列，**幂等操作**，后续操作都无效，只允许存在一个

  ```kotlin
   public fun close(cause: Throwable? = null): Boolean
  ```

- **isClosedForSend**

  实验性质API，为ture时表示`Channel`已经关闭，停止发送。

#### ProducerScope

`SendChannel`还有个子接口`ProducerScope`，表示允许发送数据的协程作用域。

> 还是处于实验性质的api，不推荐外部直接使用

![ProducerScope接口定义.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5490729eff6d4e27868d868f3642cbab~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

同时官方还提供了一个`CoroutineScope`的拓展函数`produce`，用于快速启动生产者协程，并返回`ReceiveChannel`。

其内部实际是在协程构建中创建`Channel`，用以发送数据。

![CoroutineScope的produce源码解析.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d23271d9987249709ab91e721bca410f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **awaitClose**

  **挂起函数**，`ProducerScope`中的拓展函数，会挂起等待协程关闭，在关闭前执行操作，通常用于资源回收。

  > 调用`awaitClose`后，需要外部手动调用`SendChannel`的`close`进行关闭，否则协程会一直**挂起**等待关闭，直到协程作用域被关闭。

  ![awaitClose源码.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd9cd1ed0ca34cf785c9f1556ba627b2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 消费者

`ReceiveChannel`表示接收数据的消费者，其只有一个收集数据的作用

- **receive**

  挂起函数，从缓冲队列中**接收并移除元素**，如果**缓冲队列为空，则挂起协程**。

  > 如果在`Channel`被关闭后，调用`receive`去取值，会抛出`ClosedReceiveChannelException`异常。

  ```kotlin
   public suspend fun receive(): E
  ```

- **receiveCatching**

  挂起函数，功能与`receive`相同，只是防止在缓冲队列关闭时突然抛出异常导致程序崩溃，会返回`ChannelResult`包裹取出的元素值，同时表示当前操作的状态

  ```kotlin
   public suspend fun receiveCatching(): ChannelResult<E>
  ```

- **tryReceive**

  **尝试**从缓冲队列中拉取元素，返回`ChannelResult`包裹取出的元素，并表示操作结果。

  ```kotlin
   public fun tryReceive(): ChannelResult<E>
  ```

- **cancel**

  缓冲队列的接收端停止接收数据，会移除缓冲队列的所有元素，并停止`SendChannel`发送数据，内部会调用`SendChannel`的`close`函数。

  > **谨慎调用该函数**，通常`Channel`应该由发送端`SendChannel`来主导通道是否关闭。
  >
  > 毕竟很少会有老师还在【台上发言】，下面学生就已经说【我听完了】的场景。

- **iterator**

  挂起函数，接收`Channel`时，允许使用`for`循环进行迭代

  ```kotlin
   public operator fun iterator(): ChannelIterator<E>
  ```

  `ReceiverChannel`的迭代器是挂起函数，只能在协程中使用。

  ```kotlin
   public interface ChannelIterator<out E> {
       public suspend operator fun hasNext(): Boolean
       ...
   }
  ```

  可以将缓冲队列中的元素依次取出。

- **actor**与**ActorScope**

  以`@ObsoleteCoroutinesApi`注解的废弃api，与`produce`相对应的**消费者协程作用域**。

  > 这两个API据说要重新设计，目前不要使用。[issues](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FKotlin%2Fkotlinx.coroutines%2Fissues%2F87)

  ```kotlin
   public interface ActorScope<E> : CoroutineScope, ReceiveChannel<E> 
   
   public fun <E> CoroutineScope.actor(...) : SendChannel<E>
  ```

- **consume**

  `ReceiveChannel`的拓展函数，在`Channel`出现异常或结束后，调用`cancel`关闭`Channel`接收端。

  ![ReceiveChannel的consume源码.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b0ea1df91cab4202988fb9e3a8ea085c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### ChannelResult

`ChannelResult`是一个[内联类](https://link.juejin.cn?target=https%3A%2F%2Fwww.bennyhuo.com%2F2021%2F01%2F18%2Fkotlin-inline-class-improvements%2F)，仅用于表示`Channel`操作的结果，并携带元素。

![ChannelResult源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81333049c85c437697047e7e2f93ac9b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 小结

`Channel`目前版本仅作为**生产者-消费者模型缓冲队列**，多协程间通信的基础设施而存在。

在`Flow`中只要涉及到切换协程调度器与背压缓冲都少不了`Channel`参与的身影。

## 选择表达式

说到`Channel`就不得不提`Kotlin Coroutine`中的特殊机制——**选择表达式**。

在`Kotlin Coroutine`中存在一个特殊的挂起函数——`select`，其**允许同时等待多个挂起的结果**，并且只取用其中**最快完成**的作为函数恢复的值。

![Kotlin协程select.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/15b82df6b6a145d497af6db68316b0e9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

初看这个函数实现，与Kotlin协程中回调API转协程的`suspendCoroutine`函数非常相似，同样是尝试获取结果，否则就挂起等待结果。

不过其参数`builder`却是以`SelectBuilder`作为接收者的函数类型。

![SelectBuilder源码解析.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e099e8e127dd414498f0a50c86f48ed8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`select`函数更像是一种`Kotlin DSL`的语法，在`select`的代码块并不是挂起函数，只能调用`SelectBuilder`中定义的**表达子句**。

允许从上述**子表达式**中，**选择其中一项执行**，同时`select`会将子表达式中最后语句的类型，作为自身的返回值类型。

> - `select`会**优先执行第一个表达式**，如果第一项无法执行，才会选择下一项，优先级依次类推。
> - 如果需要**完全公平的选择表达式**，则使用`selectUnbiased`。

而要使用选择表达式，就要使用返回值为`SelectClause`系列类型的函数作为子语句。

### Deferred选择

在使用`async`启动协程的返回类型`Deferred`中，就定义了`onAwait`函数作为选择表达式的**子表达式**，以`SelectClause1`作为返回值类型。

![Deferred的onAwait定义.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcac3769e90e408391b9b71eb8f46e06~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

其作用和`await`是一致的，只是当其在`select`语句中作为子语句时，就能**同时等待多个协程返回值**，并且选择其中**最快执行完成**的一个作为实际返回值。

```kotlin
 fun testSelect() = runBlocking {
     val d1 = async {
         delay(60)
         1
     }
     val d2 = async {
         delay(50)
         2
     }
     val d3 = async {
         delay(70)
         3
     }
 
     val data = select<Int> {
         d3.onAwait{data->
           println("d3 first result $data")
           data
         }
         d1.onAwait{data->
           println("d1 first result :$data")
           data
         }
         d2.onAwait{i->
           println("d2 first result : $data")
           data
         }
     }
     println("result : $data")
 }
 
 d2 first result : 2
 result ： 2
```

由于第2项`Deferred`是**最先通过`await`获取到值**的，所以`select`也是以其作为返回值。

### Channel选择

同样的，在`Channel`中也定义了以`SelectClause`类型为返回值的函数。

![Channel的SelectClause定义.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fdab50aa1bbc42829af5fb444a0713a7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **onSend** ：等效于`send`，**参数作为需要发送的数据**，并在被选择后，回调当前执行发送的`Channel`实例。
- **onReceive** ：等效于`receive`，回调从缓存队列中取出值的结果。
- **onReceiveCatching** ：等效于`receiveCatching`，回调从缓存队列中取出值的操作状态`ChannelResult`。

`Channel`使用选择表达式，通常用于多个`Channel`备份切换的场景。

```kotlin
 fun testSelectChannel() = runBlocking {
     val slowChannel = Channel<Int>(
         capacity = 1,onBufferOverflow = BufferOverflow.SUSPEND
     )
     val fastChannel = Channel<Int>(
         capacity = 3,onBufferOverflow = BufferOverflow.SUSPEND
     )
     //生产者协程
     launch(Dispatchers.IO){
         for (i in 1..5){
             if (!isActive) break
             //选择表达式不需要返回值
             select<Unit> {
                 //需要发送的数据
                 slowChannel.onSend(i){channel->
                     //lamda的参数是当前选中的channel
                     println("slow channel selected $channel send $i")
                     delay(50)
                 }
                 fastChannel.onSend(i){channel->
                     println("fast channel selected $channel send $i")
                 }
             }
         }
         delay(300)
         //注意要关闭通道
         slowChannel.close()
         fastChannel.close()
     }
     
     //消费者协程
     launch {
         while (isActive && !slowChannel.isClosedForSend && !fastChannel.isClosedForSend){
             //以ChannelResult携带的值作为选择表达式的值
             val result = select<Int> {
                 slowChannel.onReceiveCatching{
                     println("slowChannel is receive ${it.getOrNull()}")
                     delay(100)
                     it.getOrNull()?:-1
                 }
                 fastChannel.onReceiveCatching{
                     println("fastChannel is receive ${it.getOrNull()}")
                     it.getOrNull()?:-1
                 }
             }
             println("receive result : $result")
         }
     }
     delay(500)
 }
```

上述代码中，将`fastChannel`作为备份，在`slowChannel`无法发送数据时，选择使用`fastChannel`发送数据，接收端亦是同样的逻辑。程序运行结果：

```sql
 slowChannel receive 1
 slow channel selected ArrayChannel@1cc4b438{EmptyQueue}(buffer:capacity=1,size=1) send 1
 slow channel selected ArrayChannel@1cc4b438{EmptyQueue}(buffer:capacity=1,size=1) send 2
 receive result : 1
 slowChannel receive 2
 slow channel selected ArrayChannel@1cc4b438{EmptyQueue}(buffer:capacity=1,size=1) send 3
 fast channel selected ArrayChannel@580f2a18{EmptyQueue}(buffer:capacity=2,size=1) send 4
 fast channel selected ArrayChannel@580f2a18{EmptyQueue}(buffer:capacity=2,size=2) send 5
 receive result : 2
 slowChannel receive 3
 receive result : 3
 fastChannel receive 4
 receive result : 4
 fastChannel receive 5
 receive result : 5
 //Channel关闭后，取出元素时ChannelResult携带的元素为null
 slowChannel receive null
 receive result : -1
```

## ChannelFlow

虽然`Channel`能够在多个协程中线程安全的通信，但做不到复杂的数据流操作，使用上又比较繁琐。

而`Kotlin Flow`拥有灵活的数据流操作能力，却是线程不安全的，甚至不允许在发送时切换协程上下文。

那么将这两个组合在一起不就实现互补了吗？于是`ChannelFlow`就应运而生了。

![ChannelFlow源码定义.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/839d1b8ab33b41a089f10f7e0b15d853~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

不过`ChannelFlow`类本身是内部API，在外部是无法直接调用的。

### flowOn原理解析

在[上一篇](https://link.juejin.cn?target=)介绍的`buffer`和`flowOn`操作符，其内部实现`ChannelFlowOperator`正是继承于`ChannelFlow`。

这里就以`flowOn`为例，看看内部是如何实现切换`CoroutineContext`的。

![flowOn操作符源码解析.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4760308ad54d4a1fb35a47f44f1b3b79~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`ChannelFlowOperatorImpl`是`ChannelFlowOperator`的实现类，本身只是实现了父类的`flowCollect`，用于接收上游的数据。

而`flowCollect`又是在父类`ChannelFlowOperator`中的`collectTo`内调用。

同时其还重写了`collect`，做些快速检测，避免不需要的`Channel`创建造成资源浪费，在**需要修改协程调度器**时，还是使用父类的`collect`实现。

> 其实`buffer`操作符内部也是同样创建`ChannelFlowOperatorImpl`，只是`capacity`是允许外部设定的，而`flowOn`是固定为`Channel.OPTIONAL_CHANNEL`。

![ChannelFlowOperator源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/513c8a44756444a7b89a6afd2b274256~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

那么当前实现的`collectTo`函数中的`ProducerScope`类型参数又是从何而来呢？继续向父类`ChannelFlow`追溯。

![ChannelFlow源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbb1d96bb1974b74aa8dadd3b901cc4f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

其实在父类`ChannelFlow`的`collect`中也就只是在**当前收集器所在协程上下文**创建了一个新协程，通过`emitAll`发送数据。

很明显，子类实现的`collectTo`中的其实就是前面介绍过的`produce`里的lamda代码块。在**新的协程上下文**中创建一个`Channel`生产者协程，用于将上游数据添加到`Channel`缓冲队列。

不过这里的`emitAll`居然这么神奇，还能发送`ReceiveChannel`的？

当然这也只是`FlowCollector`的拓展函数而已，内部会一直循环取出`Channel`内的值，持续发送给下游。

![FlowCollector的emitAll源码.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5af85c643e9486c90e0af30db47cd0b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 小结

所以每次调用`flowOn`操作符会在内部创建一个新的`Channel`，并在**新设置的协程调度器**，创建新协程，由`Channel`发送所有上游数据，添加到缓存队列中。

由默认的背压策略`BufferOverflow.SUSPEND`，决定缓冲区队列的背压规则，**缓冲区满后挂起发送操作**。

而下游则是在**收集器所在协程调度器**内，新创建一个协程，作为消费者，循环接收`Channel`缓冲区队列的值，并发送数据给下游。

> 这也就是`flowOn`的线程调度**只对上游数据流生效**的原因。

### 回调API转数据流

而`ChannelFlow`的另一个实现`ChannelFlowBuilder`，则提供了将回调API转化为`Flow`数据流（**冷流**）的功能。

官方这里提供了两个对外公开的函数。

- **channelFlow**

![channelFlow源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0f2fcbe66104a31b3e552d82af7f3e7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`block`参数是以`ProducerScope`为接收者的函数类型。

这也就是前面提到的`Channel`的生产者协程作用域，能够利用`send`或者`trySend`来发送数据。

- **callbackFlow**

![callbackFlow源码定义.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8cfd2d6b1a6545549163eaff907209ef~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

实际上`CallbackFlowBuilder`就是`ChannelFlowBuilder`的子类，其唯一的区别就是在协程代码块结束时，**强制要求**调用`close`或挂起函数`awaitClose`，用以处理协程结束时的资源回收操作。

关闭`Channel`通道，不允许`Channel`继续发送元素。

> 准确的说是`callbackFlow`的lamda函数体执行完之前，必须确保调用`close`，停止`Channel`通道发送元素，否则会抛出异常。
>
> 而调用`awaitClose`后，会**一直挂起**，不执行后续逻辑，一直等待`Channel`通道关闭，或者收集器所在协程被关闭。

更多是用于**反注册回调API**，等待注册的回调传递数据，避免内存泄漏，否则抛出异常。

> 使用`channelFlow`与`callbackFlow`创建数据流时，允许在生产端的使用`withContext`切换协程上下文，默认使用`collect`收集器所在协程的协程调度器。

```kotlin
 fun testChannelFlow() = runBlocking {
     val flow = channelFlow<String> {
         send("11")
         println("send first on ${Thread.currentThread()}")
         withContext(Dispatchers.IO){
             send("22")
             println("send second on ${Thread.currentThread()}")
         }
         send("33")
         println("send third on ${Thread.currentThread()}")
         awaitClose {
             println("awaitClose")
         }
     }
     val job = launch {
         flow.collect {
             println("result : $it")
         }
     }
     delay(200)
     job.cancel() //交由外部协程控制channel通道关闭
 }
 
 send first on Thread[Test worker @coroutine#3,5,main]
 result : 11
 send second on Thread[DefaultDispatcher-worker-1 @coroutine#3,5,main]
 result : 22
 send third on Thread[Test worker @coroutine#3,5,main]
 result : 33
 awaitClose
```

#### 拓展应用

假设有这样一段回调函数，要把它变成`Flow`数据流。

```kotlin
 fun registerCallBack(callBack : (String)->Unit){
     for (i in 0..5){
         //do something
         callBack("data $i")
     }
 }
 
 suspend fun createCallBackFlow() = callbackFlow<String>{
     registerCallBack{result->
        send(result)    //这里回调是普通函数，是无法调用send的
        trySendBlocking(result)              
     }
     //可由外部collect的协程控制关闭
     awaitClose{ //一直挂起等待数据回调
         unRegisterCallBack() //反注册回调，回收资源
     }
 }
```

但在这个回调的代码块中，由于并不是挂起函数，所以不能在这里调用`send`来发送数据。

此时除了`trySend`尝试发送数据，其实还有一种`SendChannel`的拓展函数`trySendBlocking`，一直**阻塞线程**，等待发送数据结果，返回`ChannelResult`，表示元素入队操作结果。

`Channel`通道已经被关闭时，也会返回失败结果。

![trySendBlocking源码.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8130088462814706a2f76a501c84102a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> 不要在挂起函数或协程中调用该函数，仅推荐在普通回调函数内调用。
>
> - 在线程阻塞时，如果线程被结束会抛出`InterruptedException`异常。

#### 原理解析

那么`ChannelFlowBuilder`内部究竟是如何将回调转成`Flow`数据流的呢？

![ChannelFlowBuilder源码.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/328716184e0e4d01b2d4406b235e3cf3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`ChannelFlowBuilder`并没有做什么，只是将挂起函数`collectTo`实现为了外部传入的函数类型。

与前面解析的`flowOn`操作符一样，只是将原本由接收上游数据流的数据，变为由外部**手动控制`Channel`添加数据进入缓冲队列的逻辑**。

那么`callbackFlow`创建的`CallBackFlowBuilder`又是如何对`Channel`的关闭进行强制检测呢？

![CallbackFlowBuilder源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/145b32d6ffd84164bd9c93ba65238bad~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 小结

相比`flow`构建数据流，`channelFlow`是基于`Channel`来发送数据的，是**线程安全**的，允许在发送数据时切换协程上下文，同时还能使用`Flow`的操作符。

如果回调API是通过类似注册的方式进行添加，需要在最后调用`awaitClose`函数中进行反注册，避免内存泄露，同时一直等待回调返回数据。

所以为了避免发送完后忘记`close`，造成内存泄漏，更推荐使用`callBackFlow`。

> 虽然`channelFlow`与`callBackFlow`这两个函数已经转正，但其内部还是有实验性质api，直接使用还是会有警告。
>
> 需要添加`@OptIn(ExperimentalCoroutinesApi::class)`注解标记。

## 使用实验阶段API

使用Kotlin协程中的那些还处于**实验阶段**或者**预览阶段**的API时，IDE都会有警告，提示这是个不稳定API，后续可能会被修改。

让开发者在每个调用API的函数上都要添加`@ExperimentalCoroutinesApi`或者`@FlowPreview`注解。但这样难免有些麻烦，有没有什么一劳永逸的偷懒办法呢？

其实可以在需要调用这些API的模块（module）内的`build.gradle`文件中添加

```kotlin
//module build.gradle
 ...
 android {
     ...
     kotlinOptions {
         jvmTarget = '1.8'
         freeCompilerArgs += [
             "-Xuse-experimental=kotlinx.coroutines.ExperimentalCoroutinesApi",
             "-Xuse-experimental=kotlinx.coroutines.FlowPreview"
             "-Xopt-in=kotlin.RequiresOptIn"
         ]
     }
 }
```

重新编译后，就能直接使用实验阶段与预览阶段API了。

> 如果后续版本升级后，API转为了`@ObsoleteCoroutinesApi`注解的废弃函数或类，依然会有提示，利用IDE的提示替换为等价操作即可。

## 总结

虽然`Channel`的出现，为协程间通信提供了相当方便的工具。随着`Kotlin Flow`问世之后，`Channel`就迅速转为幕后，连旧版本中的操作符以及`BroadcastChannel`也都被废弃。

在`Flow`中的许多功能内部实现有不少`Channel`的身影，其本身的职责越发单一，仅作为**协程间通信的并发安全的缓冲队列**而存在。

对于大部分场景而言更多还是推荐使用`Flow` ，并**不推荐**直接使用`Channel`。



## SharedFlow

熟悉`RxJava`的人应该对其中的`Subject`有所印象，与于`Observale`这类需要调用`subscribe`订阅后才发送数据的数据流不同。

`Subject`是**热流**，并且同时具有发送数据与接收数据的功能，既是生产者也是消费者。

而在`Flow`中，同样也有对应功能实现——`SharedFlow`

相比普通`Flow`，`SharedFlow`意为可共享的数据流。

- 对于同一个数据流，可以允许有**多个订阅者共享**。
- 不调用`collect`收集数据，也会开始发送数据。
- 允许缓存历史数据
- 发送数据函数都是**线程安全**的。

![SharedFlow定义.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab73a9cb17b9456dbf6eadee70ae39ad~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`SharedFlow`本身的定义很简单，只比`Flow`多个**历史数据缓存**的集合，**只允许订阅数据**。

就如同Kotlin中`List`与`MutableList`，一个仅表示可读，一个表示可读可写的类型。

`ShardFlow`同样也有一个可变类型的版本`MutableShardFlow`，**定义了发送数据功能**。

![MutableSharedFlow定义.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47c502bc1c3c4f3e80a27f7879cd9fc9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 创建

首先来看`MutableSharedFlow`是如何创建的。

![MutableSharedFlow(2).png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/882322979acd4c8782d0ae87f71c6e0c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **replay**

  表示**历史元素缓存区容量**。

  > - 能够**将最新的数据缓存到集合内**，当**历史缓存区满**后，**会移除最早的元素**。
  > - 当在新消费者订阅了该数据流，会**先将历史缓存区元素依次发送给新的消费者**，然后才发送新元素。

- **extraBufferCapacity**

  表示除历史缓存区外的额外缓存区容量，用于扩充内部整体缓存容量。

- **onBufferOverflow**

  缓存区背压策略，默认是熟悉的`BufferOverflow.SUSPEND`，当**额外缓冲区满**后，挂起`emit`函数，暂停发送数据。

  只有在`replay`和`extraBufferCapacity`均不为0时才支持其他背压策略。

### 简单使用

不同于`Flow`与`ChannelFlow`中需要利用`FlowCollector`或`ProducerScope`来发送数据，由于`MutableSharedFlow`本身就拥有发送数据功能，使其用法更接近于日常使用`MutableList`。

```kotlin
 fun test() = runBlocking{
     val sharedFlow = MutableSharedFlow<String>()
     
     //假设处于另一个类，异步发送数据
     val produce = launch(Dispatchers.IO){
         for (i in 0..50){
             sharedFlow.emit("data$i")
             delay(50)
         }
     }
 }
```

虽然此时并没有消费者订阅，但**依旧会执行发送数据操作**，只是目前没有设置历史缓存，所有数据都被"抛弃"了。

通常我们并不希望在消费者订阅端能够发送数据，只允许外部进行数据流订阅，此时就需要调用`asSharedFlow`函数，将可变的`MutableSharedFlow`转化为只读的`SharedFlow`。

```kotlin
 fun test() = runBlocking{
     ...
      //模拟在外部调用
     val readOnlySharedFlow = sharedFlow.asSharedFlow()
     val scope = CoroutineScope(SupervisorJob())
     
     delay(100)
     val job1 = scope.launch { //消费者单独一个协程
         readOnlySharedFlow.map {
             delay(100)
             "$it receive 1"
         }
         .collect {
             println("collect1 result : $it")
         }
     }
     
     delay(1000)
     job1.cancel() //注意要关闭消费者所在的协程
 }
 
 collect1 result : data3 receive 1
 collect1 result : data4 receive 1
 collect1 result : data5 receive 1
 collect1 result : data6 receive 1
 collect1 result : data7 receive 1
 collect1 result : data8 receive 1
 collect1 result : data9 receive 1
 collect1 result : data10 receive 1
 collect1 result : data11 receive 1
```

我们模拟一个外部消费者，这里延迟100ms再订阅数据，就只接收到从订阅后开始发送的所有后续数据。

> `SharedFlow`作为`Flow`的子类，自然也能够使用`Flow`的中间操作符。

这等效于`RxJava`中的`PublishSubject`。

如果在`SharedFlow`创建时设置`replay`属性，比如设置为2，就会缓存最新的两个值，此时运行结果就变成了：

```kotlin
val sharedFlow = MutableSharedFlow<String>(replay = 2)
 
 collect1 result : data1 receive 1
 collect1 result : data2 receive 1
 collect1 result : data3 receive 1
 collect1 result : data4 receive 1
 collect1 result : data5 receive 1
 collect1 result : data6 receive 1
 collect1 result : data7 receive 1
 collect1 result : data8 receive 1
```

原本在订阅前发送的两个值也被消费者收集到了，等效于`RxJava`的`ReplayRelay.createWithSize<String>(2)`。

> `SharedFlow`是**热流**，而`collect`是个**挂起函数**，会**一直等待上游数据**，不论上游是否发送数据。
>
> 所以对于`SharedFlow`需要注意**消费者所在的协程内，后续任务是不会执行的**。

```kotlin
fun test() = runBlocking{
     ...
     delay(200)
     val job2 = scope.launch { //消费者单独一个协程
         readOnlySharedFlow.map {"$it receive 2"}
         .collect{println("collect2 result : $it")}
     }
     
     delay(1000)
     job1.cancel()
     job2.cancel()
 }
```

如果再新增一个消费者，其就会继续接收上游新发送的数据，直到消费者所在协程被关闭。

```yaml
 collect1 result : data3 receive 1
 collect2 result : data5 receive 2
 collect1 result : data4 receive 1
 collect2 result : data6 receive 2
 ...
 collect2 result : data13 receive 2
 collect1 result : data12 receive 1
 collect2 result : data14 receive 2
 collect1 result : data13 receive 1
```

但如果在`job2`的消费者中主动抛出异常：

```kotlin
 readOnlySharedFlow.map {
     if (it == "data6") throw Exception("test Exception")
     "$it receive 2"
 }
 
 collect1 result : data3 receive 1
 collect2 result : data5 receive 2
 collect1 result : data4 receive 1
 
 test Exception
 java.lang.Exception: test Exception
 ...
```

在消费者中出现了**未捕获异常**，此时根据消费者运行所在协程的`Job`类型有两种情况

> - 如果协程作用域（父协程）的context是`Job`，则抛出异常无法捕获，如果像是测试程序中使用`runBlocking`会直接抛出异常，程序崩溃。
> - 如果协程作用域（父协程）的context是`SupervisorJob`，则只会影响到消费者所在的协程，其他消费者接收数据不受影响。

所有消费者抛出的异常并不会影响上游共享数据流，当然所有`SharedFlow`的订阅者最好都利用`catch`操作符捕获住异常，也可在协程内设置`CoroutineExceptionHandler`进行捕获。

### 冷流转热流

在`RxJava`中，允许将`Subject`的热流通过`toFlowable`函数转化为`Flowable`类型的冷流，~~但反过来将**冷流**转化为**热流**的功能，却似乎并没有提供。~~

> 2021-11-27更新：
>
> 感谢评论区大佬的指正补充，`RxJava`中提供了操作符`publish`将**冷流**转化为**热流**。
>
> 同时提供`refCount`将其转化为自动管理结束的热流，以及将`publish`与`refCount`合并的`share`操作符。
>
> 详情参见：[关于 Observable 的冷热，常见的封装方式以及误区](https://juejin.cn/post/6844903476883881998)

而在`ShardFlow`中也同样提供了**冷流转热流**的函数——`shareIn`。

![shareIn代码块.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/539f0e7163884c4598db3d273f229e58~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这里`started`表示新创建的共享数据流的**启动与停止策略**。

- **Eagerly**

  立即开始发送数据源。并且**消费端永远收集数据**，只会收到历史缓存和后续新数据，**直到所在协程取消**。

- **Lazily**

  等待出现第一个消费者订阅后，才开始发送数据源。保证第一个消费者能收到所有数据，但后续消费者只能收到历史缓存和后续数据。

  消费者会永远等待收集数据，直到所在协程取消

- **WhileSubscribed**

  可以说是`Lazily`策略的进阶版，同样是等待第一个消费者订阅后，才开始发送数据源。

  但其可以配置在最后一个订阅者关闭后，**共享数据流上游停止的时间**（默认为立即停止），与**历史数据缓存清空时间**（默认为永远保留）。

  ```kotlin
  public fun WhileSubscribed(
       stopTimeoutMillis: Long = 0, //上游数据流延迟结束，ms
       replayExpirationMillis: Long = Long.MAX_VALUE //缓冲数据清空延迟,ms
   ): SharingStarted
  ```

利用`shareIn`以及后文介绍的`stateIn`，即可将消耗一次资源从数据源获取数据的`Flow`数据流，转化为`SharedFlow`或`StateFlow`，实现一对多的**事件分发**，并减少多次调用资源的损耗。

> 需要注意，在使用`shareIn`**每次都会创建一个新`SharedFlow`实例**，并且**该实例会一直保留在内存中**，直到被垃圾回收。
>
> 所以最好减少转换流的执行次数，不要在函数内每次都调用这类函数。

更多关于`SharingStarted`的使用场景，可以参见[shareIn 和 stateIn 使用须知](https://juejin.cn/post/6998066384290709518)

### SharedFlow原理分析

在[上一篇](https://juejin.cn/post/7034398812789538852)提到的`ChannelFlow`内部是通过`Channel`实现线程安全的多协程通信，那么`SharedFlow`的内部实现又是怎么样的呢？通过`MutableSharedFlow`工厂函数创建的`SharedFlow`，内部实际是创建了`SharedFlowImpl`对象，是**使用数组缓存所有数据**。

![SharedFlowImpl源码解析(1).png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd3b4a993ac44d35871f3c721ad03336~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 发送数据

`SharedFlowImpl`这个类第一次看起来有点多，这里先从`emit`与`tryEmit`作为切入点，看看其是如何实现发送数据。

![SharedFlowImpl源码解析emit.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32e02e257c4d44fda0e599df07a89f05~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

在`tryEmit`内部通过`synchronized`加锁，是**线程安全**的。

而当快速通道`tryEmit`方法，返回`false`，也即是校验当前缓存数组已经满了，就会调用`emitSuspend`方法**创建挂起函数**。

1. 再一次**加锁**检测是否能够发送数据（校验缓存池是否已满），如果此时已经能够发送数据，则直接恢复挂起函数运行
2. 如果不能发送数据，则将数据包装为`Emitter`类，并调用`enqueueLocked`来将`Emitter`类实例保存到数组内部。
3. 如果已能够发送数据，或者额外缓冲区，会调用`findSlotsToResumeLocked`获取所有正在挂起等待的接收器挂起函数，然后遍历进行挂起恢复。

其中`findSlotsToResumeLocked`的逻辑这里暂且先放一放，先来看看这个包装的`Emitter`类对象。

`Emitter`是一个能够在挂起函数被取消时执行回收函数的数据发射器包装类，在挂起函数被取消时执行发射器缓存的清理工作。

![Emitter内部原理.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c20357d8e75f4d8c8c922b2bcb263e13~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 历史缓存

在`SharedFlow`发射数据逻辑的分析中，会两次到调用`tryEmitLocked`进行尝试进行直接“发送数据”。

但这里似乎还并没有看到缓存的身影，而所谓的**历史数据缓存**又是什么呢？继续深入到`tryEmitLocked`看看。

![tryEmitLocked内部实现.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/948087c08e4d4d8c885ad7cfa776bcb3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> 其中`minCollectorIndex`变量表示`StateFlow`数据流的所有订阅收集器，已接收到的数据在缓存数组内的最小索引，默认等于`replayIndex`。

在`bufferSize`表示的**已缓存数量**已经达或超到设置的`bufferCapacity`最大缓存容量时，并且订阅收集器还有值需要分发，就会**执行背压策略**；

否则添加新元素，也即是“发送数据”，实际上添加新元素到缓存数组的逻辑在`enqueueLocked`函数内。

![SharedFlowImpl源码enqueueLocked.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ce012f164ad46819eae2b9d0881be2b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

其中的`totalSize`变量，表示当前缓存数组长度 + 待发送的发射器数量

```java
private val totalSize: Int get() = bufferSize + queueSize
```

每次在数组添加新元素时，会检查缓存数组容量，默认会先创建**容量为2的数组**。

如果**数组容量满后**，**以当前容量的2的倍数进行扩容**。

而**历史缓存数据**则是每次调用都在这个缓存数组`buffer`中取出最新的`replay`个数元素的组成一个`List`。

![SharedFlowImpl源码replayCache(2).png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92a8a867f5de4493bd8ab243b48eded7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> 如果缓存策略为`BufferOverflow.DROP_OLDEST`:
>
> 在`tryEmitLocked`中，已缓存的数量`bufferSize`超出允许保留的总缓存容量`bufferCapacity`，就会调用`dropOldestLocked`函数抛弃最早发送的数据缓存。
>
> `dropOldestLocked`内部抛弃数据只是将**数组索引的元素置空**，**缓存数组总长度不变**。

#### 收集数据

有了生产数据到缓存的功能，那么自然需要有消费数据，从缓存取出数据的地方，而在`Flow`中进行数据订阅，自然是从`collect`函数切入。

![SharedFlowImpl源码collect.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/49f3db324fa64875a0aca8d6e276e67a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

内部逻辑实现很清晰，通过分配绑定一个`Slot`，在轮询中尝试从缓存数组中取值并通过`emit`函数发送到下游。

> `SharedFlow`的`collect`内是个无限循环，会一直尝试从缓存中取值，所以`collect`会一直处于挂起状态，直到所在协程关闭。

而这个`Slot`则是种**收集器状态工具**，与在数据流消费者进行绑定，并记录当前分发数据在缓存数组的索引。

![SharedFlowSlot源码.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04034f6066ea41baa21dcbc7890af9e5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

在`SharedFlow`的父类`AbstractSharedFlow`内，利用数组缓存了这些收集器状态记录工具，并实现了`allocateSlot`函数，用于分配绑定到收集器。

> 重新绑定收集器时，会将`Solt`内记录的下游重置为最新值索引的前`replay`个元素索引，用于**确保下游会先由历史数据开始接收数据**。

#### 分发数据

继续回到在轮询分发数据的逻辑中，从缓存数组中取出元素的`tryTakeValue`方法，依旧是`synchronized`**加锁取值**，并将待发送元素在缓存数组的索引+1。

![tryTakeValue内部实现.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4992ab88c8a74638804dfa29b684da52~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 其中，当前订阅收集器中**需要分发数据所在的缓存索引**，核心逻辑都在`tryPeekLocked`方法内，存在4种情况：

1. 当前分发索引在小于缓存池的索引，则正常返回最新的索引值
2. 超出有效缓存，并且缓存池容量大于0，则表示当前订阅收集器已经分发完了所有缓存数数据
3. **缓存池容量为0**，则在第一个订阅收集器内立即分发新的索引，后续所有订阅器都会处于空闲状态
4. **缓存池容量为0**，如果没有挂起等待添加缓存的数据，自然也就处于空闲状态

![tryPeekLocked内部实现.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd758eb445fe44bf8c703bfdc921e798~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

如果此时没有需要发送的新值，会返回-1，即处于**空闲状态**，进而调用挂起函数`awaitValue`，创建一个新的挂起函数，内部是一段加锁的线程安全逻辑。

- 先再次检测是否存在需要发送的值，有值则**直接恢复挂起函数执行**
- 还是为空闲状态，则**进入挂起状态**，**等待缓存数组中添加数据后，再恢复挂起执行**，此时外部的无限循环自然也会进入挂起等待阶段。

![SharedFlowImpl源码awaitValue.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5243819d82045468025c3871cac65fd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

反之，则调用`getPeekedValueLockedAt`函数，会从缓存中取出指定索引的值，并继续在下一次循环时取出数据。

```kotlin
private fun getPeekedValueLockedAt(index: Long): Any? =
         when (val item = buffer!!.getBufferAt(index)) {
             is Emitter -> item.value //等待添加到缓存的值
             else -> item
         }
```

但光是取出缓存数据自然还是不够的，在空闲状态时还存在挂起等待的协程体在**等待恢复**，所以这里还调用了`updateCollectorIndexLocked`函数，去遍历检索所有需要恢复的挂起函数。

`updateCollectorIndexLocked`函数比较长，大体上分为3个步骤：

1. 计算收集器需要分发的最小索引

2. 计算需要恢复的最大挂起函数的数量

3. 获取所有需要恢复的挂起函数协程体，并添加到待恢复的协程体数组

   > - 如果存在前面发射的挂起等待发送`Emitter`，则此时也需要添加在需要恢复的协程体数组内，并将其中的等待添加的数据添加到缓存数组内，移除对应索引的`Emitter`。
   > - 否则协程体数组内只有正在挂起等待数据的收集器协程体

![updateCollectorIndexLocked内部实现.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b7118591f6e4b798c9e25ba625ad1ef~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 

### 小结

`SharedFlow`的出现，意味着**一对多数据流**传递成为可能，而且还能享受到`Flow`操作符带来的便利。

不过`SharedFlow`内部并不是基于`Channel`，而是基于数组+`synchronized`。

相比内部基于`Channel`的`ChannelFlow`：

- 两者都是**线程安全**的，`Channel`使用`ReentrantLock`+CAS，而`SharedFlow`使用`synchronized`。

- `Channel`内部是**无锁双向链表**结构，每个节点都是CAS引用类型，`SharedFlow`则是**数组**结构，允许存在一个从数组中截取的**历史缓存**集合。

- 由于`SharedFlow`在订阅前就允许添加数据到缓存数组，根据`replay`参数的设置，可能存在部分数据遗漏的问题；而`ChannelFlow`只在的下游订阅后才开始发送数据，默认就能接收到所有数据。

- 两者的订阅逻辑都通过轮询进行分发数据。

  > - `ChannelFlow`**轮询**取出`Channel`内缓存的数据逐个分发到下游，而且这个过程是在新创建的协程中执行的，允许切换`CoroutineContext`。
  > - `SharedFlow`**轮询**取出内部缓存数组的数据，没有数据则将订阅挂起，直到上游再次发送数据。

- `SharedFlow`是**热流**，`ChannelFlow`是**冷流**

  > - `SharedFlow`允许**多个消费者订阅同一个数据流**，且订阅消费者所在协程不会自动关闭；
  > - `ChannelFlow`只允许**单个消费者订阅**，数据分发完后自动关闭订阅消费，结束数据流。

虽说目前**事件总线**的概念由于过渡滥用令人相当厌恶，不过`SharedFlow`对于这类场景就和以前的`RxBus`、`EventBus`那样好用，并且还是线程安全的。

严格规范限制其使用作用范围，进行读写分离，限定为特定场景的事件分发机制，比如从定位数据只取出一次数据，分发给所有订阅者的场景，还是不错的选择。

## StateFlow

当需要**多个消费者**都只订阅到**最新的一个值**，并接收后续传递的所有值，同时还能**不需要订阅也能直接获取最新值**时，在`RxJava`时代我们还有`BehaviorSubject`的选项，亦或是后来职责更加单一的`LiveData`。

而在有了`Flow`之后，官方提供了另一个更便捷的类——`StateFlow`，来处理这种场景。

`StateFlow`实际上是`SharedFlow`的子类，同样也拥有**只读**与**可读可写**的两种类型，`StateFlow`与`MutableStateFlow`。

![StateFlow定义.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fe0c9c1e0a6436fa4388905ec09937e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

从接口定义上，`StateFlow`与`LiveData`的定义可谓是非常相似。

相同点：

- 都允许多个消费者
- 都有只读与可变类型
- 永远只保存一个状态值
- 同样支持`DataBinding`（`StateFlow`需要新版本才支持）

与`LiveData`不同的是：

- **强制要求**初始默认值
- 支持CAS模式赋值
- 默认支持**防抖过滤**
- value的空安全校验
- `Flow`丰富的异步数据流操作
- 默认没有`Lifecycle`支持，`flow`的`collect`是挂起函数，会一直等待数据流传递数据
- **线程安全**，`LiveData`的`postValue`虽然也可在异步使用，但会导致数据丢失。

`LiveData`除了对于`Lifecycle`的支持，`StateFlow`基本都是处于全面碾压的态势。

### 使用

作为`SharedFlow`的子类，`StateFlow`在使用上与其父类基本相同。

同样是利用同名工厂函数的进行创建，只是相比`SharedFlow`，`StateFlow`**必须设置默认初始值**。

![MutableStateFlow工厂方法.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d633005a35c34f2b98417a3069ccb431~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

而且`MutableStateFlow`是无法配置缓冲区的，或者说**固定永远只有一个，只会缓存最新的值**。

同时我们需要屏蔽外部发送污染数据，只对外部提供只读属性的`StateFlow`，此时就需要`asStateFlow`。

```kotlin
 fun test() = runBlocking{
     val stateFlow = MutableStateFlow(1)
     val readOnlyStateFlow = stateFlow.asStateFlow()
     
     //模拟外部立即订阅数据
     val job0 = launch {
         readOnlyStateFlow.collect { println("collect0 : $it") }
     }
     delay(50)
     //模拟在另一个类发送数据
     launch {
         for (i in 1..3){
             println("wait emit $i")
             stateFlow.emit(i)
             delay(50)
         }
     }
     //模拟启动页面，在新页面订阅
     delay(200)
     val job1 = launch {
         readOnlyStateFlow.collect{ println("collect1 : $it") }
     }
     val job2 = launch {
         readOnlyStateFlow.collect{ println("collect2 : $it") }
     }
     println("get value : ${readOnlyStateFlow.value}")
     
     delay(200)
     job0.cancel()
     job1.cancel()
     job2.cancel()
 }
 
 collect0 : 1
 wait emit 1
 wait emit 2
 collect0 : 2
 wait emit 3
 collect0 : 3
 get value : 3
 collect1 : 3
 collect2 : 3
 
```

可以看到，在**没有发送数据时订阅**，**会先接收默认值**。

而新发送的数据后，由于第一个值与原有值相同，直接被过滤掉了。

后续新添加的订阅者能够接收到的就只有**最新的值**。

> `StateFlow`订阅者所在的协程，**最好使用独立协程**，`collect`会一直挂起，**协程内的后续操作不会执行**

### 冷流转换热流

`StateFlow`同样也有由`Flow`冷流转化为热流的操作符函数`stateIn`。

![stateIn源码.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0360b2d808044c48dbec117df2b2c4d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

与`shareIn`函数的区别只是**必须设置默认值**，`stateIn`转化的共享数据流只缓存一个最新值。

### StateFlow原理分析

`StateFlow`内部并没有`SharedFlow`的缓存数组，只是用`atomic`引用类型的状态值，永远只保留一个最新的值。

![StateFlowImpl源码构造函数.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/41bf0194f7f24c23bff12bbe7ac2ac71~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

由于只保存了一个值，可以通过对`value`对象进行取值与赋值操作。

#### 发送数据

所有发送数据操作`tryEmit`与`emit`都是调用`setValue`操作，并最终调用`updateState`函数进行CAS状态赋值。

![StateFlowImpl源码updateState.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9161178acfb9474a9624dafa903f8d7c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

函数看着比较长，其内部很巧妙的根据一个`sequence`更新序号与`synchronized`加锁，配合无限循环，只允许在**更新序号为偶数才正常进行更新流程**，并最终更新序号为奇数。

如果本身更新序号就为奇数，则表示已经执行过更新流程，直接跳过后续流程。

#### 收集器状态

`StateFlowImpl`的父类同样也是`AbstractSharedFlow`，不同于`SharedFlow`，这里的消费者状态工具是`AbstractSharedFlowSlot`的另一个实现——`StateFlowSlot`。

![StateFlowSlot源码.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/498ce9a3445747f9b39de5387df130de~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这个`Slot`内部同样是`atomic`CAS引用类型，其允许有四种状态

- **null** - 表示已经空闲释放，可以分配给消费者收集器
- **NONE** - 表示已经分配给消费者接收器
- **PENDING** - 表示上游已更新新值，待发送给收集器
- **CancellableContinuationImpl** - 表示收集器已挂起在等待上游数据

在`StateFlow`更新状态值的流程中，会遍历所有已分配的`Solt`，调用`makePending`尝试将所有已分配的`Solt`状态CAS更新为`PENDING`，**使消费端准备好接收数据**。

![StateFlowSlot源码makePending（new）.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/247ba41fa7ca422b947cfeb9344e023a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 收集数据

`Flow`数据流消费端收集数据，自然还是使用`collect`。

![StateFlowImpl源码collect.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64a0909cff7b4cb28a12e64bb200e63a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

在消费者订阅函数，如`SharedFlow`相同，会**一直循环等待上游数据**。

> 仅在**刚订阅**时，`StateFlow`会**立即发送最新数据**。随后每次发送数据前都会进行**重复过滤**，并进行**空安全检查**。

随后也是同样的调用`awaitPending`函数，创建新协程并进入**挂起状态**，直到上游重新传递数据将该协程恢复。

![StateFlowSlot源码awaitPending.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70fc23c2ed7448b5922ce21abd676e26~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 关联生命周期

早在`RxJava`时代，如果直接在视图中调用`subscribe`订阅数据流，若是视图生命周期处于后台状态时，接收数据更新就可能会出现不符合预期的情况。

所以在`LiveData`出现后，在视图层的数据订阅都逐渐移交给`LiveData`了，而`RxJava`逐渐退居到数据层进行数据逻辑处理。

现在的`Flow`在视图内调用`collect`订阅数据流自然也会存在与`RxJava`相同的问题。

因此我们需要让`Flow`在视图生命周期处于后台时，不对数据流进行订阅处理、不传递数据到消费端。

### LifecycleCoroutineScope

在早期，`Lifecycle`库提供了拓展属性`coroutineScope`。

作为视图专用的`LifecycleCoroutineScope`协程作用域，会在**视图销毁时取消协程作用域**，其中拥有多个`launchWith`系列函数。

```kotlin
public val Lifecycle.coroutineScope: LifecycleCoroutineScope
     
 public abstract class LifecycleCoroutineScope internal constructor() : CoroutineScope {
     ...
     public fun launchWhenCreated(block: suspend CoroutineScope.() -> Unit): Job = launch {
         lifecycle.whenCreated(block)
     }
     
     public fun launchWhenStarted(block: suspend CoroutineScope.() -> Unit): Job = launch {
         lifecycle.whenStarted(block)
     }
     
     public fun launchWhenResumed(block: suspend CoroutineScope.() -> Unit): Job = launch {
         lifecycle.whenResumed(block)
     }
     ...
 }
```

比如`launchWhenStarted`函数，会将`Flow`数据流消费端所在的协程，函数执行限定在小于`Lifecycle.State.STARTED`状态下。（图片来自网络）

![Lifecycle-State.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0a07daba8324c8cb13b9d25e100ca44~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

也即是消费者端所在协程内的`collect`函数内部逻辑，只会在视图处于`onStart`生命周期后才**恢复执行**，而处于后台的生命周期会被**暂时挂起**。

但对于`Flow`数据流来说，即便消费端的处理逻辑挂起了，**生产端数据源依然还在执行**，尤其是某些数据源一直运行的场景可能造成不必要的资源损耗，**官方目前也并不推荐使用这些函数**。

> 在`Android Studio 2021.1.1 Patch 1`中，如果在编译器自带的`DataBinding`版本内中使用`StateFlow`，自动生成的`Binding`文件内，依然还是会使用`launchWhenCreated`函数进行订阅收集数据流变化。
>
> ```kotlin
> //ViewDataBindingKtx.kt
>  internal class StateFlowListener(
>           binder: ViewDataBinding?,
>           localFieldId: Int,
>           referenceQueue: ReferenceQueue<ViewDataBinding>
>   ) : ObservableReference<Flow<Any?>> {
>   ...
>   override fun addListener(target: Flow<Any?>?) {
>       val owner = _lifecycleOwnerRef?.get() ?: return
>       if (target != null) {
>           startCollection(owner, target)
>       }
>   }
>   ...
>   private fun startCollection(owner: LifecycleOwner, flow: Flow<Any?>) {
>       observerJob?.cancel()
>       //在onCreate -> onDestroy之间执行该协程代码块
>       observerJob = owner.lifecycleScope.launchWhenCreated {
>           flow.collect {
>               //更新databinding数据，执行requestBinding
>               listener.binder?.handleFieldChange(listener.mLocalFieldId, listener.target, 0)
>           }
>       }
>   }
>  }
>  
> ```

### repeatOnLifecycle

随着`Lifecycle-runtime-ktx`库更新至`2.4.0`版本，`Lifecycle`提供了一个新的拓展函数`repeatOnLifecycle`。

![repeatOnLifecycle源码解析.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c11ee2d2c914ea6aa188f21e5abe56c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

看着代码实现很多，内部逻辑其实很简单，切换`CoroutineContext`到UI主线程，在进入允许的生命周期状态时，**启动协程，订阅数据流**。在超出设定的生命周期状态后，**关闭协程，取消订阅**

与其他方式的比较：（图片来自官方）

![repeatOnLifecycle原理.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38c1e08a56944c819888db338e83078b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 

对于共享数据流的`StateFlow`来说，每次订阅都只会获取最新的值，这也更接近`LiveData`的使用逻辑，也即所谓的**粘性数据**。

> 在`repeatOnLifecycle`函数出现后，官方也开始计划删除`launchWith`系列函数。
>
> - 关于`repeatOnLifecycle`的设计原因可以参见[设计 repeatOnLifecycle API 背后的故事](https://juejin.cn/post/7001371050202103838)。

除此之外，`Lifecycle`库还提供了一个`Flow`的中间操作符`flowWithLifecycle`，利用`callbackFlow`来内部调用`repeatOnLifecycle`。

而`callbackFlow`内部实际上是基于`Channel`实现的，**对于上游数据流具有缓冲区的作用**。

![flowWithLifecycle源码解析.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3af67c4524b417e9ae97d52271eaa28~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> - 对于单个`Flow`数据流的生命周期控制，`flowWithLifecycle`操作符可以很好解决样板代码。
> - 如果需要同时控制多个`Flow`数据流的生命周期，还是推荐使用`repeatOnLifecycle`避免重复创建`Channel`。
> - 冷流`Flow`不推荐直接使用`flowWithLifecycle`，避免多次创建新的数据源。

于是`Flow`数据流就可以在`Activity`或`Fragment`中很方便的绑定视图生命周期

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
      super.onCreate(savedInstanceState)
      //模拟从viewModel开放出来的状态更新
      viewModel.readOnlyStateFlow
             //在onStart开启订阅上游数据流，onPause取消订阅
             .flowWithLifecycle(lifecycle,Lifecycle.State.STARTED) 
             .onEach { //do something }
             .launchIn(lifecycleScope) //运行在主线程的协程作用域，在视图销毁时自动取消作用域
 }
```

## 总结

至此，作为`Kotlin Flow`的最后一部分拼图——**共享数据流**也就此集齐了。

- `SharedFlow`作为允许保留历史缓存并且只能收到新数据的存在，对于**一对多事件分发**的场景是个很好的选择。
- `StateFlow`则是与原本`LiveData`的定位重合，永远只持有最新数据，更适用于处理**状态更新**。

配合`repeatOnLifecycle`限制视图生命周期订阅，`StateFlow`可以完全替代`LiveData`，更新视图的状态显示，同时支持**粘性数据**。

而原本需要封装`LiveData`才能处理的不需要**粘性**的**单次执行事件**，只需要将`replay`设置为0，`SharedFlow`就能很好的承担这个职责。

当然，如果不需要`Flow`数据流操作与线程安全的需求，像`LiveData`这样职责单一的类，承担视图状态更新也还是不错的选择，简单也意味着不容易出错，便于维护。

毕竟`StateFlow`到底还是要依靠Kotlin协程来实现，`LiveData`利用`observer`能直接订阅状态还是比较方便的。

总的来说，随着`Kotlin Flow`的出现，从数据源的逻辑处理到视图层的状态与事件订阅，都有了很好的新选择。

也唯有熟悉与理解其背后运作机制，才能更好在合适的场景中灵活运用。







## 相关引用原文

[Kotlin Flow上手指南](https://juejin.cn/post/7034379406730592269) 作者：yuPFeG1819