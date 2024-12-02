# Flutter异步编程

## EventLoop

### 1. 先来梳理几个概念

在开始正式异步编程讲解之前还是先来梳理回顾几个异步编程中的概念。

#### 1.1 同步(Synchronous)和异步(Asynchronous)

同步和异步一直是异步并发编程中比较重要概念，需要深刻理解它们之间的区别，它们更**「侧重于描述不同的流程设计方式」**。

- 同步: 存在依赖顺序关系的操作之间是同步的，比如说B操作是依赖A操作的返回，B操作的执行必须要等到A操作执行之后才能执行。 
- 异步: 操作之间并没有依赖关系的换句话说操作执行的顺序并没有一定严格。比如操作A，B，C并没有依赖联系，哪个先开始执行，那么先执行结束都没有关系。

比如异步情况下，A，B，C三个没有依赖关系它们执行顺序可能有多种，下面只列出两种用于理解。



#### 1.2 阻塞和非阻塞

阻塞和非阻塞更**「侧重于描述操作程序是否会立即返回」**，这个概念也容易和同步异步搞混。 同步不一定会阻塞主要取决于在同步执行的操作中是否存在有阻塞的操作，而基于单线程的异步API实现里的操作一般都会要求为非阻塞。

#### 1.3 多线程和单线程

我们经常会把同步异步和单线程多线程概念搞混了，认为多线程一定就是异步的，还有就是单线程就一定是同步的。实际上这些理解都是有问题的，比如很多基于EventLoop的单线程异步模型。**「单线程和多线程主要侧重的是运行的环境」**。其实多线程既可以实现同步也能实现异步，单线程也是如此既可以实现同步也可以实现异步。

 ![单线程和多线程.png](.\res\单线程和多线程.png) 

从上图就能明显分析出**「单线程和多线程只是侧重于程序执行环境不一样，而同步和异步和它没有本质必然的联系。只是说一般用单线程实现同步，多线程实现异步会更简单和更易于理解，实际上单线程和多线程都能实现同步和异步」**。

### 2. Dart中的EventLoop(事件循环)

首先，我们必须要清楚**「Dart是单线程模型」**的，这也就意味着**「Dart一次只能执行一个操作，然后其他操作一个接着一个，只要有一个操作正在执行，它就不能被其他任何操作给中断」**。那么是不是就意味着Dart代码只能是同步的，答案肯定不是。实际上Dart是支持单线程非阻塞式异步，支持的背后是因为Dart的事件循环(EventLoop)的驱动。我们都知道在前端开发框架大多数都是事件驱动的，事件驱动也意味着程序中存在事件循环和事件队列, 事件循环会不停的从事件队列中获取和处理各种事件。其实类似事件循环这种机制，在不同技术领域都有体现，比如Android中的 `Looper/MessageQueue/Handler` 、iOS中的 `RunLoop` 以及js中的 `EventLoop` 等等。所以Dart一样也是属于事件驱动，它也有自己的 `EventLoop` . ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4eaf609fa254bf282f11b70b6e6ec6a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 3. EventLoop的组成

Dart的EventLoop事件循环和Javascript很类似，同样循环中拥有两个FIFO队列: **「一个是事件队列(Event Queue),另一个就是微任务队列(MicroTask Queue)」**

#### 3.1 事件队列(Event Queue)

主要包含一些外部事件和 `Future` ，比如IO，绘制，手势以及 `Timer` 等事件。

- 外部事件: IO、手势、绘制、定时器(Timer)、Stream流等
- Future

实际上，**「每一次触发外部事件时，需要执行的相应的代码就会被引用到事件队列中」**。**「一旦如果没有任何微任务执行的话，EventLoop就会取出事件队列中的第一个事件然后执行它」**。值得一提的是** `Future` 也是通过事件队列进行处理**的，也就意味在Dart一次网络请求触发返回包装的 `Future` ，最后是否拿到正常数据或异常这个事件最终也是交由事件队列来处理的。

#### 3.2 微任务队列(MicroTask Queue)

主要用于Dart内部的微任务(内部非常短暂操作)，一般是通过 `scheduleMicroTask` 方法实现调度，这些任务**「需要在其他内部微任务执行完之后且在把事件移交给事件队列之前」**立即异步执行。下面可以通过一个简单的例子(假如需要在资源关闭之后执行某个操作，但是这个资源关闭又需要一定的时间)来看下:

```javascript
import 'dart:async';

void main() {
  release();
}

void release() {
  scheduleMicrotask(_dispose);//在资源关闭后执行dispose操作
  _close();//关闭资源
}

void _close() {
  print('close resource');
}

void _dispose() {
  print('dispose run');
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c37d8d40cf4b48d195c82647ab272924~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 4. EventLoop的执行流程

当启动一个Dart应用程序或者Flutter应用程序时，将创建并启动一个新的Thread线程(在Dart中不叫Thread，而是isolate), 这个线程将是整个应用的主线程。 所以当这个线程被创建， Dart就会自动执行以下几步:

- 1、初始化两个队列，分别是事件队列和微任务队列
- 2、执行 `main` 方法
- 3、启动事件循环EventLoop 

 ![EventLooper流程](.\res\EventLooper流程.awebp)



#### 4.1 两种队列异步执行

在Dart要实现异步代码，只需要把代码加入到微任务队列或者事件队列中就可以了。

##### 微任务队列实现异步

MicroTask Queue实现异步主要就是通过上述所提到的 `scheduleMicroTask` 方法来实现，参数就是一个异步执行的函数。

```javascript
import 'dart:async';

void main() {
  scheduleMicrotask(() => print('async code in microtask queue'));
  print('execute main');
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03bb41e24503442a92ddfd90fb10d0df~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

##### 事件队列实现异步

Event Queue实现异步任务主要是借助 `Timer` 类来实现，上述也说了事件队列主要包括事件其中就有 `Timer` .

```javascript
import 'dart:async';

void main() {
  Timer.run(() => print('async code in event queue'));
  print('execute main');
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99b07392b4b64f90a05449e29b60fc78~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

#### 4.2 用一个例子来加深对EventLoop理解

有下面这段代码，分析输出的结果.

```javascript
import 'dart:async';

void main() {
  Timer.run(() => print('async code in event queue'));//async code in event queue这句话是否能输出???
  release();
  print('execute main');
}

void release() {
  scheduleMicrotask(release);
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc490bc7a4e14732bf4ed7b3757c1369~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

分析结果: 为什么 `async code in event queue` 这句输出，永远不会输出呢？其实分析一下就很清楚了，首先 `Timer` 是会被添加到事件队列中的，而 `release` 方法中的 `scheduleMicroTask` 是会被添加到微任务队列中的，通过上面那种流程图就可以得到**「事件队列中的事件执行必须等到微任务队列中所有微任务执行完后，才会取出事件队列事件进行处理」**，然而 `scheduleMicrotask(release)` 类似递归调用 `release` 方法，始终保持了微任务队列中不为空，那么事件队列中的事件永远得不到处理。所以不会输出`async code in event queue`

### 5. 用一个Flutter例子来分析EventLoop的工作流程

比如通过一个Flutter按钮的点击，发出一次网络请求，并拿到网络请求的结果, 具体代码如下：

```dart
RaisedButton( 
  child: Text('Click me'),
  onPressed: () { 
    final myFuture = http.get('https://example.com');
    myFuture.then((response) { 
      if (response.statusCode == 200) {
        print('Success!');
      }
    });
  },
)
```

上述是一个Flutter的简单例子，就是点击一个按钮发出一个网络请求，然后拿到网络请求结果。

- 1、首先，当运行这个Flutter应用程序的时候，Flutter会把这个 `RaisedButton` 按钮绘制在屏幕上，此时的EventLoop可能处于空闲期或者在处理其他的事件。

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad2bd48958cd49f0a02bac6c470d6000~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 2、然后，创建好的按钮等待用户点击触发，当用户此时点击了这个 `RaisedButton` 时，此时会有一个 `Tap` 事件进入事件队列，直到这个事件被EventLoop处理，当被EventLoop处理的时候Flutter的渲染系统会识别出手势点击坐标正好落入了 `RaisedButton` 坐标范围了，这时候Flutter就会执行 `onPressed` 方法，此时 `Tap` 处理完后也就会被丢弃了。

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3e3381f5f8c4a59bafaee975c1b8736~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 3、然后， `onPressed` 方法中是启动了一个网络请求并返回一个 `Future` ，并且使用 `then` 方法来注册当 `Future` 完成后的回调监听, 我们上面也说过 `Future` 最终也会被加入事件队列(Event Queue)中，可以这样理解 `Future` 正在等待网络数据也是属于一个特定的事件，直到网络数据到了，此时就会有个特定事件进入事件队列(Event Queue), 最后也会被EventLoop处理。

 ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f86e8efa7471477f9029be4a94328540~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 4、最后， `Future` 完成后，相应回调方法就会被触发，就会返回正常的 `response` 了，这就是大概分析了下Flutter中一个简单交互操作怎么和EventLoop联系起来的，这个地方需要好好深入理解下。



## Future

### 1. 为什么需要Future

我们都知道Dart是单线程模型语言，如果需要执行一些延迟的操作或者IO操作，默认采用单线程同步的方式就有可能会造成主isolate阻塞，从而导致渲染卡顿。基于这一点我们需要实现单线程的异步方式，基于Dart内置的非阻塞API实现，那么其中**Future就是异步中非常重要的角色。Future表示异步返回的结果，当执行一个异步延迟的计算时候，首先会返回一个Future结果，后续代码可以继续执行不会阻塞主isolate，当Future中的计算结果到达时，如果注册了 `then` 函数回调，对于返回成功的回调就会拿到最终计算的值，对于返回失败的回调就会拿到一个异常信息**。

可能很多人有点疑惑，实现异步使用isolate就可以了，为什么还需要Future？ 关于这点前面文章也有说过，**Future相对于isolate来说更轻量级**，而且创建过多的isolate对系统资源也是非常大的开销。熟悉isolate源码小伙伴就知道，实际上isolate映射对应操作系统的OSThread. 所以对于一些延迟性不高的还是建议使用Future来替代isolate.

可能很多人还是有疑惑，关于实现异步Dart中的async和await也能实现异步，为什么还需要Future？ **Future相对于async, await的最大优势在于它提供了强大的链式调用，链式调用优势在于可以明确代码执行前后依赖关系以及实现异常的捕获**。 一起来看个常见的场景例子，比如需要请求book详情时，需要先请求拿到对应book id, 也就是两个请求有前后依赖关系的，这时候如果使用async,await可能就不如future来的灵活。

- 使用async和await实现:

```dart
_fetchBookId() async {
  //request book id
}

_fetchBookDetail() async {
  //需要在_fetchBookDetail内部去await取到bookId,所以要想先执行_fetchBookId再执行_fetchBookDetail，
  //必须在_fetchBookDetail中await _fetchBookId()，这样就会存在一个问题，_fetchBookDetail内部耦合_fetchBookId
  //一旦_fetchBookId有所改动，在_fetchBookDetail内部也要做相应的修改
  var bookId = await _fetchBookId();
  //get bookId then request book detail
}

void main() async {
   var bookDetail = await _fetchBookDetail();// 最后在main函数中请求_fetchBookDetail
}

//还有就是异常捕获
_fetchDataA() async {
  try {
    //request data
  } on Exception{
    // do sth
  } finally {
    // do sth
  }
}

_fetchDataB() async {
  try {
    //request data
  } on Exception{
    // do sth
  } finally {
    // do sth
  }
}
void main() async {
  //防止异常崩溃需要在每个方法内部都要添加try-catch捕获
  var resultA = await _fetchDataA();
  var resultB = await _fetchDataB();
}
```

- Future的实现

```dart
dart 代码解读复制代码_fetchBookId() {
  //request book id
}

_fetchBookDetail() {
  //get bookId then request book detail
}

void main() {
   Future(_fetchBookId()).then((bookId) => _fetchBookDetail());
  //或者下面这种方式
   Future(_fetchBookId()).then((bookId) => Future(_fetchBookDetail()));
}

//捕获异常的实现
_fetchBookId() {
  //request book id
}

_fetchBookDetail() {
  //get bookId then request book detail
}

void main() {
  Future(_fetchBookId())
      .catchError((e) => '_fetchBookId is error $e')
      .then((bookId) => _fetchBookDetail())
      .catchError((e) => '_fetchBookDetail is error $e');
}
```

总结一下，为什么需要Future的三个点:

- 在Dart单线程模型，Future作为异步的返回结果是Dart异步中不可或缺的一部分。
- 在大部分Dart或者Flutter业务场景下，Future相比isolate实现异步更加轻量级，更加高效。
- 在一些特殊场景下，Future相比async, await在链式调用上更有优势。

### 2. 什么是Future

#### 2.1 官方描述

用专业术语来说**future 是 *[Future<T>](https://link.juejin.cn?target=https%3A%2F%2Fapi.dart.dev%2Fstable%2Fdart-async%2FFuture-class.html)* 类的对象，其表示一个 `T` 类型的异步操作结果。如果异步操作不需要结果，则 future 的类型可为 `Future<void>`**。当一个返回 future 对象的函数被调用时，会发生两件事：

- 将函数操作列入队列等待执行并返回一个未完成的 `Future` 对象。
- 不久后当函数操作执行完成，`Future` 对象变为完成并携带一个值或一个错误。

#### 2.2 个人理解

其实我们可以把**Future理解为一个装有数据的“盒子”**，一开始执行一个异步请求会返回一个Future“盒子”，然后就接着继续下面的其他代码。等到过了一会异步请求结果返回时，这个Future“盒子”就会打开，里面就装着请求结果的值或者是请求异常。那么这个Future就会有三种状态分别是: 未完成的状态(**Uncompleted**), “盒子”处于关闭状态；完成带有值的状态(**Completed with a value**), “盒子”打开并且正常返回结果状态；完成带有异常的状态(**Completed with a error**), “盒子”打开并且失败返回异常状态；

 ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/14252ef25cea489186b7331e0c3ed9f2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

下面用一个Flutter的例子来结合EventLoop理解下Future的过程，这里有个按钮点击后会去请求一张网络图片，然后把这张图片显示出来。

```dart
dart 代码解读复制代码RaisedButton(
  onPressed: () {
    final myFuture = http.get('https://my.image.url');//返回一个Future对象
    myFuture.then((resp) {//注册then事件
      setImage(resp);
    });
  },
  child: Text('Click me!'),
)
```

- 首先，当点击 `RaisedButton` 的时候，会传递一个 `Tap` 事件到事件队列中(因为系统手势属于Event Queue)，然后 `Tap` 事件交由给事件循环EventLoop处理。

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9fb59e241274468eaf0f4941b3fc9b00~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 然后，事件循环会处理 `Tap` 事件最终会触发执行 `onPressed` 方法，随即就会利用http库发出一个网络请求，并且就会返回一个Future, 这时候你就拿到这个数据“盒子”，但是这时候它的状态是关闭的也就是uncompleted状态。然后再用 `then` 注册当数据“盒子”打开时候的回调。这时候 `onPressed` 方法就执行完毕了，然后就是一直等着，等着HTTP请求返回的图片数据，这时候整个事件循环不断运行在处理其他的事件。

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ece95bb868b7462f8c5b0b443c707a4e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 最终，HTTP请求的图片数据到达了，这时候Future就会把真正的图片数据装入到“盒子”中并且把盒子打开，然后注册的 `then` 回调方法就会被触发，拿到图片数据并把图片显示出来。

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cac0f534fb3244d7a84889bf53bf296e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 4. Future的状态

通过上面理解Future总共有3种状态分别是: 未完成的状态(**Uncompleted**), “盒子”处于关闭状态；完成带有值的状态(**Completed with a value**), “盒子”打开并且正常返回结果状态；完成带有异常的状态(**Completed with a error**), “盒子”打开并且失败返回异常状态. 

实际上从Future源码角度分析，总共有5种状态:

- __stateIncomplete : _初始未完成状态，等待一个结果
- __statePendingComplete: _Pending等待完成状态, 表示Future对象的计算过程仍在执行中，这个时候还没有可以用的result.
- __stateChained: _链接状态(一般出现于当前Future与其他Future链接在一起时，其他Future的result就变成当前Future的result)
- __stateValue: _完成带有值的状态
- __stateError: _完成带有异常的状态

```dart
class _Future<T> implements Future<T> {
  /// Initial state, waiting for a result. In this state, the
  /// [resultOrListeners] field holds a single-linked list of
  /// [_FutureListener] listeners.
  static const int _stateIncomplete = 0;

  /// Pending completion. Set when completed using [_asyncComplete] or
  /// [_asyncCompleteError]. It is an error to try to complete it again.
  /// [resultOrListeners] holds listeners.
  static const int _statePendingComplete = 1;

  /// The future has been chained to another future. The result of that
  /// other future becomes the result of this future as well.
  /// [resultOrListeners] contains the source future.
  static const int _stateChained = 2;

  /// The future has been completed with a value result.
  static const int _stateValue = 4;

  /// The future has been completed with an error result.
  static const int _stateError = 8;

  /** Whether the future is complete, and as what. */
  int _state = _stateIncomplete;
  ...
}
```

### 5. 如何使用Future

#### 5.1 Future的基本使用

- **1. factory Future(FutureOr computation())**

Future的简单创建就可以通过它的构造函数来创建，通过传入一个异步执行Function.

```dart
//Future的factory构造函数
factory Future(FutureOr<T> computation()) {
  _Future<T> result = new _Future<T>();
  Timer.run(() {//内部创建了一个Timer
    try {
      result._complete(computation());
    } catch (e, s) {
      _completeWithErrorCallback(result, e, s);
    }
  });
  return result;
}
dart 代码解读复制代码void main() {
  print('main is executed start');
  var function = () {
    print('future is executed');
  };
  Future(function);
  print('main is executed end');
}
//或者直接传入一个匿名函数
void main() {
  print('main is executed start');
  var future = Future(() {
    print('future is executed');
  });
  print('main is executed end');
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ebc26a29ab741aab96e2d65ce1a1fc0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

从输出结果可以发现Future输出是一个异步的过程，所以 `future is executed` 输出在 `main is executed end` 输出之后。这是因为main方法中的普通代码都是同步执行的，所以是先把main方法中 `main is executed start` 和 `main is executed end` 输出，等到main方法执行结束后就会开始检查 `MicroTask` 队列中是否存在task, 如果有就去执行，直到检查到 `MicroTask Queue` 为空，那么就会去检查 `Event Queue` ，由于Future的本质是在内部开启了一个 `Timer` 实现异步，最终这个异步事件是会放入到 `Event Queue` 中的，此时正好检查到当前 `Future` ，这时候的Event Loop就会去处理执行这个 `Future` 。所以最后输出了 `future is executed` .

- **2. Future.value()**

创建一个返回指定value值的Future对象, **注意在value内部实际上实现异步是通过 `scheduleMicrotask`** . 之前文章也说过实现Future异步方式无非只有两种，一种是使用 `Timer` 另一种就是使用`scheduleMicrotask`

```dart
void main() {
  var commonFuture = Future((){
    print('future is executed');
  });
  var valueFuture = Future.value(100.0);//
  valueFuture.then((value) => print(value));
  print(valueFuture is Future<double>);
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74ef21acf148457fa621af6aaa67a791~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

通过上述输出结果可以发现，true先输出来是因为它是同步执行的，也说明最开始同步执行拿到了 `valueFuture` . 但是为什么 `commonFuture` 执行在 `valueFuture` 执行之后呢，这是因为 `Future.value` 内部实际上是通过 `scheduleMicrotask` 实现异步的，那么就不难理解了等到main方法执行结束后就会开始检查 `MicroTask` 队列中是否存在task，正好此时的 `valueFuture` 就是那么就先执行它，直到检查到 `MicroTask Queue` 为空，那么就会去检查 `Event Queue` ，由于Future的本质是在内部开启了一个 `Timer` 实现异步，最终这个异步事件是会放入到 `Event Queue` 中的，此时正好检查到当前 `Future` ，这时候的Event Loop就会去处理执行这个 `Future` 。

不妨来看下 `Future.value` 源码实现:

```dart
factory Future.value([FutureOr<T> value]) {
    return new _Future<T>.immediate(value);//实际上是调用了_Future的immediate方法
  }

//进入immediate方法
 _Future.immediate(FutureOr<T> result) : _zone = Zone.current {
    _asyncComplete(result);
  }
//然后再执行_asyncComplete方法
 void _asyncComplete(FutureOr<T> value) {
    assert(!_isComplete);
    if (value is Future<T>) {//如果value是一个Future就把它和当前Future链接起来，很明显100这个值不是
      _chainFuture(value);
      return;
    }
    _setPendingComplete();//设置PendingComplete状态
    _zone.scheduleMicrotask(() {//注意了：这里就是调用了scheduleMicrotask方法
      _completeWithValue(value);//最后通过_completeWithValue方法回调传入value值
    });
  }
```

- **3. Future.delayed()**

创建一个延迟执行的future。实际上内部就是通过创建一个延迟的 `Timer` 来实现延迟异步操作。 `Future.delayed` 主要传入两个参数一个是 `Duration` 延迟时长，另一个就是异步执行的Function。

```dart
dart 代码解读复制代码void main() {
  var delayedFuture = Future.delayed(Duration(seconds: 3), (){//延迟3s
    print('this is delayed future');
  });
  print('main is executed, waiting a delayed output....');
}
```

`Future.delayed` 方法就是使用了一个延迟的 `Timer` 实现的，具体可以看看源码实现:

```dart
factory Future.delayed(Duration duration, [FutureOr<T> computation()]) {
    _Future<T> result = new _Future<T>();
    new Timer(duration, () {//创建一个延迟的Timer，传递到Event Queue中，最终被EventLoop处理
      if (computation == null) {
        result._complete(null);
      } else {
        try {
          result._complete(computation());
        } catch (e, s) {
          _completeWithErrorCallback(result, e, s);
        }
      }
    });
    return result;
  }
```

#### 5.2 Future的进阶使用

- **1. Future的forEach方法**

forEach方法就是根据某个集合，创建一系列的Future，然后再按照创建的顺序执行这些Future. Future.forEach有两个参数:一个是 `Iterable` 集合对象，另一个就是带有迭代元素参数的Function方法

```dart
void main() {
  var futureList = Future.forEach([1, 2, 3, 4, 5], (int element){
    return Future.delayed(Duration(seconds: element), () => print('this is $element'));//每隔1s输出this is 1, 隔2s输出this is 2, 隔3s输出this is 3, ...
  });
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae456434fc2c4947ae3639e95cac40a8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- **2. Future的any方法**

Future的any方法返回的是第一个执行完成的future的结果，不管是否正常返回还是返回一个error.

```dart
dart 代码解读复制代码void main() {
  var futureList = Future.any([3, 4, 1, 2, 5].map((delay) =>
          new Future.delayed(new Duration(seconds: delay), () => delay)))
      .then(print)
      .catchError(print);
}
```

输出结果: 

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4b7aa45124a4553bc7f631f74afefa9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- **3. Future的doWhile方法**

Future.doWhile方法就是重复性地执行某一个动作，直到返回false或者Future，退出循环。特别适合于一些递归请求子类目的数据。

```dart
void main() {
  var totalDelay = 0;
  var delay = 0;

  Future.doWhile(() {
    if (totalDelay > 10) {//超过10s跳出循环
      print('total delay: $totalDelay s');
      return false;
    }
    delay += 1;
    totalDelay = totalDelay + delay;
    return new Future.delayed(new Duration(seconds: delay), () {
      print('wait $delay s');
      return true;
    });
  });
}
```

输出结果:

- **4. Future的wait方法**

用来等待多个future完成，并整合它们的结果，有点类似于RxJava中的zip操作。那这样的结果就有两种：

- **若所有future都有正常结果返回：则future的返回结果是所有指定future的结果的集合**
- **若其中一个future有error返回：则future的返回结果是第一个error的值**

```dart
void main() {
  var requestApi1 = Future.delayed(Duration(seconds: 1), () => 15650);
  var requestApi2 = Future.delayed(Duration(seconds: 2), () => 2340);
  var requestApi3 = Future.delayed(Duration(seconds: 1), () => 130);

  Future.wait({requestApi1, requestApi2, requestApi3})
      .then((List<int> value) => {
        //最后将拿到的结果累加求和
        print('${value.reduce((value, element) => value + element)}')
      });
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/baa8b12e29e249b6be837dfb5cee57fe~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

```dart
//异常处理
void main() {
  var requestApi1 = Future.delayed(Duration(seconds: 1), () => 15650);
  var requestApi2 = Future.delayed(Duration(seconds: 2), () => throw Exception('api2 is error'));
  var requestApi3 = Future.delayed(Duration(seconds: 1), () => throw Exception('api3 is error'));//输出结果是api3 is error这是因为api3先执行，因为api2延迟2s

  Future.wait({requestApi1, requestApi2, requestApi3})
      .then((List<int> value) => {
        //最后将拿到的结果累加求和
        print('${value.reduce((value, element) => value + element)}')
      });
}
```

输出结果: ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89869566f25946f3a7ac4c91089b4e9e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- **5. Future的microtask方法**

我们都知道Future一般都是会把事件加入到Event Queue中，但是Future.microtask方法提供一种方式将事件加入到Microtask Queue中，创建一个在microtask队列运行的future。 在上面讲过，microtask队列的优先级是比event队列高的，而一般future是在event队列执行的，所以Future.microtask创建的future会优先于其他future进行执行。

```dart
void main() {
  var commonFuture = Future(() {
    print('common future is executed');
  });
  var microtaskFuture = Future.microtask(() => print('microtask future is executed'));
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b5917cac2a9458cad2297712e9d434f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- **6. Future的sync方法**

Future.sync方法返回的是一个同步的Future, 但是需要注意的是，如果这个Future使用 `then` 注册Future的结果就是一个异步，它会把这个Future加入到MicroTask Queue中。

```dart
void main () {
  Future.sync(() => print('sync is executed!'));
  print('main is executed!');
}
```

输出结果: ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/976b1dc006ba4379b4539b7ccf5649ab~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 如果使用then来注册监听Future的结果，那么就是异步的就会把这个Future加入到MicroTask Queue中。

```dart
void main() {
  Future.delayed(Duration(seconds: 1), () => print('this is delayed future'));//普通Future会加入Event Queue中
  Future.sync(() => 100).then(print);//sync的Future需要加入microtask Queue中
  print('main is executed!');
}
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aca2303c43564bc2bcb7e36e5104190d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

#### 5.3 处理Future返回的结果

- **1. Future.then方法**

Future一般使用 `then` 方法来注册Future回调，需要注意的Future返回的也是一个Future对象，所以可以使用链式调用使用Future。这样就可以将前一个Future的输出结果作为后一个Future的输入，可以写成链式调用。

```dart
void main() {
  Future.delayed(Duration(seconds: 1), () => 100)
      .then((value) => Future.delayed(Duration(seconds: 1), () => 100 + value))
      .then((value) => Future.delayed(Duration(seconds: 1), () => 100 + value))
      .then((value) => Future.delayed(Duration(seconds: 1), () => 100 + value))
      .then(print);//最后输出累加结果就是400
}
```

输出结果: ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/082aa84d68f54c8dbe0faf30e70447a8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- **2. Future.catchError方法**

注册一个回调，来处理有异常的Future

```dart
void main() {
  Future.delayed(Duration(seconds: 1), () => throw Exception('this is custom error'))
      .catchError(print);//catchError回调返回异常的Future
}
```

输出结果: ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb119c917bf7427ab007c879d9de6a9f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- **3. Future.whenComplete方法**

Future.whenComplete方法有点类似异常捕获中的try-catch-finally中的finally, 一个Future不管是正常回调了结果还是抛出了异常最终都会回调 `whenComplete` 方法。

```dart
//with value
void main() {
  Future.value(100)
      .then((value) => print(value))
      .whenComplete(() => print('future is completed!'));
  print('main is executed');
}

//with error
void main() {
  Future.delayed(
          Duration(seconds: 1), () => throw Exception('this is custom error'))
      .catchError(print)
      .whenComplete(() => print('future is completed!'));
  print('main is executed');
}
```

输出结果: ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0329ddd02fd548d38eaedd3b561cfb76~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f7497efa946436491673f6b4a7767eb~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 6. Future使用的场景

由上述有关Future的介绍，我相信大家对于Future使用场景应该心中有个底，这里就简单总结一下：

- 1、对于Dart中一般实现异步的场景都可以使用Future，特别是处理多个Future的问题，包括前后依赖关系的Future之间处理，以及聚合多个Future的处理。
- 2、对于Dart中一般实现异步的场景单个Future的处理，可以使用async和await
- 3、对于Dart中比较耗时的任务，不建议使用Future这时候还是使用isolate.



### 7. 总结

到这里有关异步编程中Future就介绍完毕了，Future相比isolate更加轻量级，很容易轻松地实现异步。而且相比aysnc,await方法在链式调用方面更具有优势。但是需要注意的是如果遇到耗时任务比较重的，还是建议使用isolate，因为毕竟Future还是跑在主线程中的。还有需要注意一点需要深刻理解EventLoop的概念，比如Event Queue, MicroTask Queue的优先级。因为Dart后面很多高级异步API都是建立事件循环基础上的。





## Isolate

### 1. 为什么需要isolate

在Dart/Flutter应用程序启动时，会启动一个主线程其实也就是Root Isolate, 在Root Isolate内部运行一个EventLoop事件循环。所以所有的Dart代码都是运行在Isolate之中的，它就像是机器上的一个小空间，具有自己的私有内存块和一个运行事件循环的单个线程。isolate是提供了Dart/Flutter程序运行环境，包括所需的内存以及事件循环EventLoop对事件队列和微任务队列的处理。来张图理解下Root Isolate在Flutter应用程序中所处作用: ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/464a65fbf6684ea9870c2c7eee11a3b4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 2. 什么是isolate

用官方文档中定义一句话来概括: `An isolated Dart execution context` .大概的意思就是**isolate实际就是一个隔离的Dart执行的上下文环境(或者容器)**。isolate是Dart对**Actor并发模型**的实现。运行中的Dart程序由一个或多个**Actor**组成，这些**Actor**其实也就是Dart中的isolate。**isolate是有自己的内存和单线程控制的事件循环**。isolate本身的意思是“隔离”，因为**isolate之间的内存在逻辑上是隔离的，不像Java一样是共享内存的**。isolate中的代码是按顺序执行的，**任何Dart程序的并发都是运行多个isolate的结果**。因为**Dart没有共享内存的并发**，没有竞争的可能性所以不需要锁，也就不存在死锁的问题。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b234f600c3f542f3bd775e8dbfc8dc88~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) )

#### 2.1 什么是Actor并发模型

Actor类似于面向对象（OOP）编程中的对象——其封装了状态，并通过消息与其他Actor通信。 在面向对象中，我们使用方法调用的方式去传递信息，而在Actor中，则使用发送消息去传递信息。一个object接收消息(对应为OOP中的方法调用)，然后基于该消息来做一些事情。但是Actor并发模型的不同之处在于：**每个Actor是完全隔离(和isolate是一致的)**，他们不会共享内存；同时，Actor也会维护自身的私有状态，并且不会直接被其他的Actor修改。**每个Actor之间都彼此隔离所以它们是通过发消息来进行通信的, 在Actor模型中每个工作者被称为actor。Actor之间可以直接异步地发送和处理消息**。 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7991abd9181746f9ac4332fc903c3f78~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

#### 2.2 Actor并发通信消息模型

其实在Actor更具体并发消息模型中，还隐藏着另一个概念那就是 **mailbox(信箱)** 的概念，也就是说Actor之间不能直接发送和接收消息通信的, 而是发送到一个 **信箱（mailbox）。其实一般在Actor实现中，一个Actor内部都会有一个对应的mailbox(信箱)实现，用于对消息发送和接收处理。**![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf32eaf4cbb5401c9ef207462e1541ce~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 比如ActorA想和ActorB通信，必须通过发消息方式给ActorB发送一个“邮件(message)”,地址就填ActorB的mailbox地址，至于ActorB是否接收这份“邮件(message)”交由ActorB自己决定。
- 每个Actor都有一个自己的mailbox, 任意的Actor都可以对应的Actor的mailbox地址发送“邮件(message)”，“邮件(message)”投递和读取是两个过程这样一来就把Actor之间交互通信给完全解耦了.

#### 2.2 Actor并发模型的特征

- 在Actor内部可以进行计算，且不会占用调用方CPU的时间片, 甚至包括并发策略都是自己决定的
- 在Actor之间可以通过发送异步消息进行通信
- 在Actor内部可以保存状态以及修改状态

#### 2.3 Actor并发模型的规则

- 所有的计算都是在Actor中执行的
- Actor之间只能通过消息进行通信
- 当一个Actor接收到消息，可以做3个操作: **发送消息给其他的Actor、创建其他的Actor、接受并处理消息，修改自己的状态**

#### 2.4 Actor并发模型的优点

- Actor可以独立更新和升级，因为每个Actor都是相对独立的个体，彼此独立，互不影响
- Actor可以支持本地调用和远程调用，因为Actor本质上还是使用基于消息的通讯机制，无论是和本地的Actor，还是远程Actor交互，都是通过消息通信
- Actor拥有很好错误处理机制，不需要当心消息超时或者等待的问题，因为Actor之间消息都是基于异步方式发送的
- Actor高效、扩展性也很强，因为支持本地调用和远程调用，当本地Actor处理能力有限可以使用远程Actor来协同处理

#### 2.5 isolate并发模型特点

isolate可以理解为是概念上Thread线程，但是它和Thread线程唯一不一样的就是**多个isolate之间彼此隔离且不共享内存空间，每个isolate都有自己独立内存空间，从而避免了锁竞争**。由于每个isolate都是隔离，它们之间的通信就是**基于上面Actor并发模型中发送异步消息来实现通信的**，所以更直观理解把一个isolate当作Actor并发模型中一个Actor即可。在isolate中还有**Port**的概念，分为send port和receive port可以把它理解为Actor模型中每个Actor内部都有对mailbox(信箱)的实现，可以很好地管理Message。 

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8098c6c215e44afaacdfad26a716184f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 3. 如何使用isolate

#### 3.1 isolate包介绍

使用isolate类进行并发操作，需要导入 `isolate` 

```dart
import 'dart:isolate';
```

该Library主要包含下面:

- `Isolate` 类: Dart代码执行的隔离的上下文环境
- `ReceivePort` 类: 它是一个接收消息的 `Stream` ， `ReceivePort` 可以生成 `SendPort` ,可以由上面的Actor模型规则就知道 `ReceivePort` 接收消息，可以把消息发送给其他的 `isolate` 所以要发送消息就需要生成 `SendPort` , 然后再由 `SendPort` 发送给对应isolate的 `ReceivePort` .
- `SendPort` 类: 将消息发送给isolate, 准确的来说是将消息发送到isolate中的 `ReceivePort` 

此外可以使用 `spawn` 方法生成一个新的 `isolate` 对象， `spawn` 是一个静态方法返回的是一个 `Future<Isolate>` , 必传参数有两个，函数 `entryPoint` 和参数 `message` ,其中 **`entryPoint`函数必须是顶层函数(这个概念之前文章有提到过)或静态方法，参数message需要包含SendPort. **

```dart
external static Future<Isolate> spawn<T>(
      void entryPoint(T message), T message,
      {bool paused: false,
      bool errorsAreFatal,
      SendPort onExit,
      SendPort onError,
      @Since("2.3") String debugName});
```

#### 3.2 创建和启动isolate

有时候需要根据特定场景采用不同方式创建和启动isolate.

##### 创建一个新的Isolate以及建立RootIsolate和NewIsolate握手连接

正如前面所描述的， `isolate` 不会共享任何的内存空间且它们之间的通信是通过异步消息实现的。因此需要找到一种在 `rootIsolate` 和新创建的 `newIsolate` 之间建立的通信方式。 每个 `isolate` 都会向外暴露一个端口(Port), 用于向该 `isolate` 发送消息的端口称为 `SendPort` . 这就意味要实现 `rootIsolate` 和新创建的 `newIsolate` 通信，必须知道彼此的端口(Port).

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc29fcc94f84454181cd6af7b2ee20d8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

```dart
//实现newIsolate与rootIsolate(默认)建立连接
import 'dart:isolate';

//定义一个newIsolate
late Isolate newIsolate;
//定义一个newIsolateSendPort, 该newIsolateSendPort需要让rootIsolate持有，
//这样在rootIsolate中就能利用newIsolateSendPort向newIsolate发送消息
late SendPort newIsolateSendPort;

void main() {
  establishConn(); //建立连接
}

//特别需要注意:establishConn执行环境是rootIsolate
void establishConn() async {
  //第1步: 默认执行环境下是rootIsolate，所以创建的是一个rootIsolateReceivePort
  ReceivePort rootIsolateReceivePort = ReceivePort();
  //第2步: 获取rootIsolateSendPort
  SendPort rootIsolateSendPort = rootIsolateReceivePort.sendPort;
  //第3步: 创建一个newIsolate实例，并把rootIsolateSendPort作为参数传入到newIsolate中，为的是让newIsolate中持有rootIsolateSendPort, 这样在newIsolate中就能向rootIsolate发送消息了
  newIsolate = await Isolate.spawn(createNewIsolateContext, rootIsolateSendPort); //注意createNewIsolateContext这个函数执行环境就会变为newIsolate, rootIsolateSendPort就是createNewIsolateContext回调函数的参数
  //第7步: 通过rootIsolateReceivePort接收到来自newIsolate的消息，所以可以注意到这里是await 因为是异步消息
  //只不过这个接收到的消息是newIsolateSendPort, 最后赋值给全局newIsolateSendPort，这样rootIsolate就持有newIsolate的SendPort
  var messageList = await rootIsolateReceivePort.first;
  //第8步，建立连接成功
  print(messageList[0] as String);
  newIsolateSendPort = messageList[1] as SendPort;
}

//特别需要注意:createNewIsolateContext执行环境是newIsolate
void createNewIsolateContext(SendPort rootIsolateSendPort) async {
  //第4步: 注意callback这个函数执行环境就会变为newIsolate, 所以创建的是一个newIsolateReceivePort
  ReceivePort newIsolateReceivePort = ReceivePort();
  //第5步: 获取newIsolateSendPort, 有人可能疑问这里为啥不是直接让全局newIsolateSendPort赋值，注意这里执行环境不是rootIsolate
  SendPort newIsolateSendPort = newIsolateReceivePort.sendPort;
  //第6步: 特别需要注意这里，这里是利用rootIsolateSendPort向rootIsolate发送消息，只不过发送消息是newIsolate的SendPort, 这样rootIsolate就能拿到newIsolate的SendPort
  rootIsolateSendPort.send(['connect success from new isolate', newIsolateSendPort]);
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d703c1d460694886885f167aca0eac4e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

#### 3.2 isolate另一个封装方法compute

compute方法是一个Flutter SDK中已经封装好的isolate的实现，注意: 只有Flutter SDK中才有定义，在Dart SDK是没有此方法定义的。由于dart中的Isolate比较重量级，UI线程和Isolate中的数据的传输比较复杂，因此flutter为了简化用户代码，在foundation库中封装了一个轻量级compute操作。**该方法一般用于只需要运行一段耗时代码，然后完成以后不需要再和isolate有交互，可以直接使用这个封装好的方法**。源码位于flutter/foundation/_isolates_io.dart.

```dart
Future<R> compute<Q, R>(isolates.ComputeCallback<Q, R> callback, Q message, { String debugLabel }) async {
  if (!kReleaseMode) {
    debugLabel ??= callback.toString();
  }
  final Flow flow = Flow.begin();
  Timeline.startSync('$debugLabel: start', flow: flow);
  final ReceivePort resultPort = ReceivePort();
  final ReceivePort errorPort = ReceivePort();
  Timeline.finishSync();
  final Isolate isolate = await Isolate.spawn<_IsolateConfiguration<Q, FutureOr<R>>>(
    _spawn,
    _IsolateConfiguration<Q, FutureOr<R>>(
      callback,
      message,
      resultPort.sendPort,
      debugLabel,
      flow.id,
    ),
    errorsAreFatal: true,
    onExit: resultPort.sendPort,
    onError: errorPort.sendPort,
  );
  final Completer<R> result = Completer<R>();
  errorPort.listen((dynamic errorData) {
    assert(errorData is List<dynamic>);
    assert(errorData.length == 2);
    final Exception exception = Exception(errorData[0]);
    final StackTrace stack = StackTrace.fromString(errorData[1] as String);
    if (result.isCompleted) {
      Zone.current.handleUncaughtError(exception, stack);
    } else {
      result.completeError(exception, stack);
    }
  });
  resultPort.listen((dynamic resultData) {
    assert(resultData == null || resultData is R);
    if (!result.isCompleted)
      result.complete(resultData as R);
  });
  await result.future;
  Timeline.startSync('$debugLabel: end', flow: Flow.end(flow.id));
  resultPort.close();
  errorPort.close();
  isolate.kill();
  Timeline.finishSync();
  return result.future;
}
```

#### 3.3 isolate存在的限制

需要注意的是: **Platform-Channel的通信仅仅在主Isolate中支持**，这个主Isolate是在Application被启动时候创建的。换句话说Platform-Channel的通信不能在我们自定义创建的Isolate中运行。

### 4. isolate之间互相通信

**两个isolate需要通信首先要在各自作用域内拿到对方的 `SendPort` 这样就完成第一步双方握手过程，然后就是注意发送消息时需要再次把当前sendPort带上, 最后才能实现向对方发送消息实现互相通信**。继续接着用上面已经成功建立连接例子分析: ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cdc3df913ff04a86b24b48177b1acd7f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

```dart
//实现newIsolate与rootIsolate互相通信

import 'dart:io';
import 'dart:isolate';

//定义一个newIsolate
late Isolate newIsolate;
//定义一个newIsolateSendPort, 该newIsolateSendPort需要让rootIsolate持有，
//这样在rootIsolate中就能利用newIsolateSendPort向newIsolate发送消息
late SendPort newIsolateSendPort;

void main() {
  establishConn(); //建立连接
}

//特别需要注意:establishConn执行环境是rootIsolate
void establishConn() async {
  //第1步: 默认执行环境下是rootIsolate，所以创建的是一个rootIsolateReceivePort
  ReceivePort rootIsolateReceivePort = ReceivePort();
  //第2步: 获取rootIsolateSendPort
  SendPort rootIsolateSendPort = rootIsolateReceivePort.sendPort;
  //第3步: 创建一个newIsolate实例，并把rootIsolateSendPort作为参数传入到newIsolate中，为的是让newIsolate中持有rootIsolateSendPort, 这样在newIsolate中就能向rootIsolate发送消息了
  newIsolate = await Isolate.spawn(createNewIsolateContext, rootIsolateSendPort); //注意createNewIsolateContext这个函数执行环境就会变为newIsolate, rootIsolateSendPort就是createNewIsolateContext回调函数的参数
  //第7步: 通过rootIsolateReceivePort接收到来自newIsolate的消息，所以可以注意到这里是await 因为是异步消息
  //只不过这个接收到的消息是newIsolateSendPort, 最后赋值给全局newIsolateSendPort，这样rootIsolate就持有newIsolate的SendPort
  newIsolateSendPort = await rootIsolateReceivePort.first;
  //第8步: 建立连接后，在rootIsolate环境下就能向newIsolate发送消息了
  sendMessageToNewIsolate(newIsolateSendPort);
}

//特别需要注意:sendMessageToNewIsolate执行环境是rootIsolate
void sendMessageToNewIsolate(SendPort newIsolateSendPort) async {
  ReceivePort rootIsolateReceivePort = ReceivePort(); //创建专门应答消息rootIsolateReceivePort
  SendPort rootIsolateSendPort = rootIsolateReceivePort.sendPort;
  newIsolateSendPort.send(['this is from root isolate: hello new isolate!', rootIsolateSendPort]);//注意: 为了能接收到newIsolate回复消息需要带上rootIsolateSendPort
  //第11步: 监听来自newIsolate的消息
  print(await rootIsolateReceivePort.first);
}

//特别需要注意:createNewIsolateContext执行环境是newIsolate
void createNewIsolateContext(SendPort rootIsolateSendPort) async {
  //第4步: 注意callback这个函数执行环境就会变为newIsolate, 所以创建的是一个newIsolateReceivePort
  ReceivePort newIsolateReceivePort = ReceivePort();
  //第5步: 获取newIsolateSendPort, 有人可能疑问这里为啥不是直接让全局newIsolateSendPort赋值，注意这里执行环境不是rootIsolate
  SendPort newIsolateSendPort = newIsolateReceivePort.sendPort;
  //第6步: 特别需要注意这里，这里是利用rootIsolateSendPort向rootIsolate发送消息，只不过发送消息是newIsolate的SendPort, 这样rootIsolate就能拿到newIsolate的SendPort
  rootIsolateSendPort.send(newIsolateSendPort);
  //第9步: newIsolateReceivePort监听接收来自rootIsolate的消息
  receiveMsgFromRootIsolate(newIsolateReceivePort);
}

//特别需要注意:receiveMsgFromRootIsolate执行环境是newIsolate
void receiveMsgFromRootIsolate(ReceivePort newIsolateReceivePort) async {
  var messageList = (await newIsolateReceivePort.first) as List;
  print('${messageList[0] as String}');
  final messageSendPort = messageList[1] as SendPort;
  //第10步: 收到消息后，立即向rootIsolate 发送一个回复消息
  messageSendPort.send('this is reply from new isolate: hello root isolate!');
}
```

运行结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ce62e18fff7422aa0cd68a7488c28a2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 5. isolate和普通Thread的区别

isolate和普通Thread的区别需要从不同的维度来区分:

- 1、从底层操作系统维度

在isolate和Thread操作系统层面是一样的，都是会去创建一个OSThread，也就是说最终都是委托创建操作系统层面的线程。

- 2、从所起的作用维度

都是为了应用程序提供一个运行时环境。

- 3、从实现机制的维度

isolate和Thread有着明显区别就是大部分情况下的Thread都是共享内存的，存在资源竞争等问题，但是isolate彼此之间是不共享内存的。

### 6. 什么场景该使用Future还是isolate

其实这个问题，更值得去注意，因为这是和实际的开发直接相关，有时候确实需要知道什么时候应该是 `Future` ，什么时候应该使用 `isolate` . 有的人说使用 `isolate` 比较重，一般不建议采用，其实不能这样一概而论。 `isolate` 也是有使用场景的，可能有的小伙伴会疑惑网络请求应该算耗时吧，平时一般使用 `Future` 就够了，为什么不使用 `isolate` 呢。这里将一一给出答案。

- 如果一段代码不会被中断，那么就直接使用正常的同步执行就行。
- 如果代码段可以独立运行而不会影响应用程序的流畅性，建议使用 `Future` 
- 如果繁重的处理可能要花一些时间才能完成，而且会影响应用程序的流畅性，建议使用 `isolate` 

换句话说，建议尽可能多地使用 `Future` （直接或间接通过异步方法），因为一旦 `EventLoop` 有空闲期，这些 `Future` 的代码就会运行。这么说可能还是有点抽象，下面给出一个代码运行时间指标衡量选择:

- 如果一个方法要花费几毫秒时间那么建议使用 `Future` .
- 如果一个处理可能需要几百毫秒那么建议使用 `isolate` .

下面列出一些使用 `isolate` 的具体场景:

- 1、JSON解析: 解码JSON，这是HttpRequest的结果，可能需要一些时间，可以使用封装好的 `isolate` 的 `compute` 顶层方法。
- 2、加解密: 加解密过程比较耗时
- 3、图片处理: 比如裁剪图片比较耗时
- 4、从网络中加载大图

### 7. 总结

到这里有关Dart异步编程中的isolate相关内容就介绍完毕，这篇文章对isolate分析得还是很全面的，包括从为什么需要isolate到怎么使用isolate到isolate的使用场景到最后的root isolate的启动源码分析。





## async和await

### 1. 为什么需要async-await

通过学习之前异步编程中的Future我们知道，Future一般使用 `then` 和 `catchError` 可以很好地处理数据回调和异常回调。这实际上还是一种基于异步回调的方式，如果异步操作依赖关系比较复杂需要编写回调代码比较繁杂，为了简化这些步骤 `async-await` 关键字通过同步代码结构来实现异步操作，从而使得代码更加简洁和具有可读性，此外在异常处理方式也会变得更加简单。

#### 1.1 对比实现代码

- Future实现方式

```dart
void main() {
  _loadUserFromSQL().then((userInfo) {
    return _fetchSessionToken(userInfo);
  }).then((token) {
    return _fetchData(token);
  }).then((data){
    print('$data');
  });
  print('main is executed!');
}

class UserInfo {
  final String userName;
  final String pwd;
  bool isValid;

  UserInfo(this.userName, this.pwd);
}

//从本地SQL读取用户信息
Future<UserInfo> _loadUserFromSQL() {
  return Future.delayed(
      Duration(seconds: 2), () => UserInfo('gitchat', '123456'));
}

//获取用户token
Future<String> _fetchSessionToken(UserInfo userInfo) {
  return Future.delayed(Duration(seconds: 2), '3424324sfdsfsdf24324234');
}

//请求数据
Future<String> _fetchData(String token) {
  return Future.delayed(
      Duration(seconds: 2),
      () => token.isNotEmpty
          ? Future.value('this is data')
          : Future.error('this is error'));
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b780bfdf3cc74cb99e802c99da9a5c96~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- async-await实现方式

```dart
void main() async {//注意：需要添加async，因为await必须在async方法内才有效
  var userInfo = await _loadUserFromSQL();
  var token = await _fetchSessionToken(userInfo);
  var data = await _fetchData(token);
  print('$data');
  print('main is executed!');
}

class UserInfo {
  final String userName;
  final String pwd;
  bool isValid;

  UserInfo(this.userName, this.pwd);
}

//从本地SQL读取用户信息
Future<UserInfo> _loadUserFromSQL() {
  return Future.delayed(
      Duration(seconds: 2), () => UserInfo('gitchat', '123456'));
}

//获取用户token
Future<String> _fetchSessionToken(UserInfo userInfo) {
  return Future.delayed(Duration(seconds: 2), () => '3424324sfdsfsdf24324234');
}

//请求数据
Future<String> _fetchData(String token) {
  return Future.delayed(
      Duration(seconds: 2),
      () => token.isNotEmpty
          ? Future.value('this is data')
          : Future.error('this is error'));
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f218f507df749ca84cf13de691c4a55~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

#### 1.2 对比异常下处理

- Future的实现

```dart
void main() {
  _loadUserFromSQL().then((userInfo) {
    return _fetchSessionToken(userInfo);
  }).then((token) {
    return _fetchData(token);
  }).catchError((e) {
    print('fetch data is error: $e');
  }).whenComplete(() => 'all is done');
  print('main is executed!');
}
```

- async-await的实现

```dart
void main() async {
  //注意：需要添加async，因为await必须在async方法内才有效
  try {
    var userInfo = await _loadUserFromSQL();
    var token = await _fetchSessionToken(userInfo);
    var data = await _fetchData(token);
    print('$data');
  } on Exception catch (e) {
    print('this is error: $e');
  } finally {
    print('all is done');
  }
  print('main is executed!');
}
```

通过对比发现使用Future.then相比async-await调用链路更清晰，基于异步回调的方式调用相比比较清晰。而在代码实现以及同步结构分析async-await显得更加简单且处理异常也更加方面，更加符合同步代码的逻辑。

### 2. 什么是async-await

#### 2.1 async-await基本介绍

async-await本质上是对Future API的简化形式，将异步回调代码写成同步代码结构形式。**async关键字修饰的函数总是返回一个Future对象，所以async并不会阻塞当前线程**，由前面的EventLoop和Future我们都知道Future的最终会加入EventQueue中，而EventQueue执行是当main函数执行完毕后，才会检查Microtask Queue和Event Queue并处理它们。**await关键字意味着中断当前代码执行流程直到当前的async方法执行完毕，如果没有执行完毕下面的代码将处于等待的状态**。但是需要遵循下面两个规则:

- **要定义异步函数，请在函数主体之前添加async关键字**
- **await关键字只有在async关键字修饰的函数才会有效 **

```dart
void main() {
  print("executedFuture return ${executedFuture() is Future}");//所以输出true
  print('main is executed!');
}

executedFuture() async {//async函数默认返回的是Future对象
  print('executedFuture start');
  await Future.delayed(Duration(seconds: 1), () => print('future is finished'));//await等待Future执行完毕
  print('Future is executed end');//executedFuture end 输出必须等待await的Future结束后才会执行
}
```

输出结果：

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c94deef07d0c4dfd9f465aa8f3712bce~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

**PS：指正原文中这里的执行结果漏了第一个输出语句，实际结果应该如下**

```cmd
executedFuture return true
executedFuture start
main is executed!
future is finished
Future is executed end
```



分析下输出结果，首先main函数同步执行executedFuture函数和print函数，所以马上就会同步输出 **“executedFuture start”**，但是由于executedFuture是一个async函数，await等待一个Future, 在executedFuture函数作用域内，所以并且在await后面执行，所以需要等待Future数据到了才会执行后面语句，但是此时的executedFuture执行完毕，马上就执行了main函数中的**“main is executed!”**, main执行结束后，就会去检查MicroTask Queue是否存在需要执行的微任务，如果没有就会继续检查Event Queue是否存在需要处理的Event(其中Future也是一种Event), 所以检查到executedFuture函数中的Future,就会把它交给EventLoop处理，Future执行完毕后，输出**“future is finished”**， 最后执行输出**“Future is executed end”。** ** 如果把上述例子修改一下:

```dart
void main() async {//main函数变成一个async函数
  await executedFuture(); //executedFuture加入await关键字
  print('main is executed!');
}

executedFuture() async {
  print('executedFuture start');
  await Future.delayed(Duration(seconds: 1), () => print('future is finished'));
  print('Future is executed end');//executedFuture end 输出必须等待await的Future结束后才会执行
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/962b754be29c408bbc94b77ead895ed3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

分析输出结果，可能很多人都很疑惑为什么**“main is executed”**会输出在最后，其实很容易理解，因为这时候main函数变成了一个async函数，所以必须等待 `await executedFuture()` 执行完毕后才会执行后面的 `print('main is executed!')` . 如果executedFuture没有执行完毕那么整个main函数后面代码只能等待中，所以一般没有直接给整个main函数加async关键字，这样会使得整个main函数强行变成了同步执行。

#### 2.2 结合EventLoop理解async-await

通过上面介绍我们知道async-await本质实际上还是Future，本质还是通过向Event Queue中添加Event，然后EventLoop来处理它。但是它们在代码结构形式上完全不一样它是怎么做到的呢？一起来看下。先给出一个例子：

- **Future.then的理解**

```dart
class DataWrapper {
  final String data;

  DataWrapper(this.data);
}

Future<DataWrapper> createData() {
  return _loadDataFromDisk().then((id) {
    return _requestNetworkData(id);
  }).then((data) {
    return DataWrapper(data);
  });
}

Future<String> _loadDataFromDisk() {
  return Future.delayed(Duration(seconds: 2), () => '1001');
}

Future<String> _requestNetworkData(String id) {
  return Future.delayed(
      Duration(seconds: 2), () => 'this is id:$id data from network');
}
```

其实通过Future的链式调用执行逻辑可以把 `createData` 代码按事件进行拆分成下面形式, 可以看到createData函数分为1、2、3： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ccc24f4b04d49ed9abf565aaaf7601e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

首先第1块是当 `createData` 函数被调用后，就同步调用执行 `_loadDataFromDisk` 函数，它返回的是一个Future对象，然后就是等待数据的到达EventLoop, 这样 `_loadDataFromDisk` 函数就执行结束了。

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5423fe52c0c44c5e8fcdbf86387b79d5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

然后，当`_loadDataFromDisk` 函数Future的数据到来时，会触发第2块中的 `then` 函数回调，拿到id后就立即执行了 `_requestNetworkData` 函数发出一个HTTP请求，并且先返回一个Future对象，然后就是等待HTTP数据到达EventLoop被处理即可，这样`_requestNetworkData` 函数就结束了。 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca7f17f3c88044e7ac7cd041a7c78374~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

最后，当 `_requestNetworkData` 函数Future的数据到来时, 第二个 `then` 函数就被回调触发了，然后就是创建 `DataWrapper` 返回最终数据对象即可。那么整个createData函数返回的Future最终就返回真实数据，如果外部调用者调用了这个函数，那么在它的 `then` 函数中就能拿到最终的`DataWrapper`

- **async-await的理解**

```dart
class DataWrapper {
  final String data;

  DataWrapper(this.data);
}

Future<DataWrapper> createData() async {//createData添加async关键字，表示这是一个异步函数
  var id = await _loadDataFromDisk(); //await执行_loadDataFromDisk从磁盘中获取到id
  var data = await _requestNetworkData(id); //通过传入_loadDataFromDisk的id执行_requestNetworkData返回data
  return DataWrapper(data); //最后返回DataWrapper对象
}

Future<String> _loadDataFromDisk() {
  return Future.delayed(Duration(seconds: 2), () => '1001');
}

Future<String> _requestNetworkData(String id) {
  return Future.delayed(
      Duration(seconds: 2), () => 'this is id:$id data from network');
}
```

其实通过async-await的类似同步调用执行顺序逻辑也可以把 `createData` 代码按事件进行拆分成下面形式，只不过之前是通过 `then` 函数来断开，这里通过await关键字来断开分析, 可以看到createData函数分为1、2、3：

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9588f8465e5842289e170de2fba86363~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

首先，当createData函数开始执行就会触发第一个等待，此时createData就会将自己Future对象返回给调用函数，注意:**这里遇到第1个await等待并调用_loadDataFromDisk函数的时候，createData函数就会把自己Future对象返回给调用函数, 此时的createData函数就已经执行完毕**，可能大家比较疑惑没有显式看到返回了一个Future对象，这是因为async关键字语法糖帮你做了。有的人又会疑惑下面第3步不是返回了吗？请注意:下面返回的是 `DataWrapper` 对象不是一个 `Future<DataWrapper>` ，所以当执行createData函数的时候碰到第1个await等待，就会马上return一个Future对象，然后createData函数就执行完毕了。触发第一个等待并且就执行 `_loadDataFromDisk` 函数，它返回的是一个Future对象, 然后就会一直等待者数据 `id` 到来。 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0271d7df8734ad5848159a9441fa621~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

然后，执行第2块代码，当`_loadDataFromDisk` 函数，它返回的是一个Future对象中的 `id` 数据到来时，就会触发第2个await等待并且调用`_requestNetworkData` 函数，发出HTTP网络请求并且先返回一个Future,然后使用await等待这个HTTP Future的到来。 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6baddcb5951a46948d20cfba0391a88d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

最后，`_requestNetworkData` 函数返回Future中的数据到来后，就能拿到HTTP的data数据，最后返回一个DataWrapper, 那么整个createData函数的Future就拿到最终的数据DataWrapper。 

### 3. 如何使用async-await

#### 3.1 基本使用

通过对比一般同步实现、异步Future.then的实现、异步async-await实现，可以更好地理解async-await用法，使用async-await实际上就是相当于把它当作同步代码结构形式来写即可。下面也是以上面例子为例

- 同步实现

假设_loadDataFromDisk和_requestNetworkData函数都是同步执行的，那么就能很容易写成它们执行代码:

```dart
DataWrapper createData() {
  var id = _loadDataFromDisk();//同步执行直接return id
  var data = _requestNetworkData(id);//同步执行直接return data
  return DataWrapper(data);
}
```

- 异步Future.then实现

```dart
Future<DataWrapper> createData() {//由于是异步执行，所以注意返回的对象是一个Future
  return _loadDataFromDisk().then((id) {//需要在异步回调then函数中拿到id
    return _requestNetworkData(id);
  }).then((data) {//需要在异步回调then函数中拿到data
    return DataWrapper(data);//最后返回最终的DataWrapper
  });
}
```

- 异步async-await实现

整体代码结构形式和同步代码结构形式一模一样只是加了async-await关键字以及最后返回的是一个Future对象

```dart
Future<DataWrapper> createData() async {//createData添加async关键字，表示这是一个异步函数
  //注意: createData遇到第1个await _loadDataFromDisk，就会先返回一个Future<DataWrapper>对象，结束了createData函数。
  var id = await _loadDataFromDisk(); //await执行_loadDataFromDisk从磁盘中获取到id
  var data = await _requestNetworkData(id); //通过传入_loadDataFromDisk的id执行_requestNetworkData返回data
  return DataWrapper(data); //最后返回DataWrapper对象
}
```

#### 3.2 异常处理

- 同步实现

同步执行代码实现异常处理也只能是我们常用的try-catch-finally。

```dart
DataWrapper createData() {
  try {
    var id = _loadDataFromDisk();
    var data = _requestNetworkData(id);
    return DataWrapper(data);
  } on Exception catch (e) {
    print('this is error');
  } finally {
    print('executed done');
  }
}
```

- 异步Future.then实现

Future.then实现异步异常的捕获一般是借助 `catchError` 来实现的。

```dart
Future<DataWrapper> createData() {
  return _loadDataFromDisk().then((id) {
    return _requestNetworkData(id);
  }).then((data) {
    return DataWrapper(data);
  }).catchError((e) {//catchError捕获异常
    print('this is error: $e');
  }).whenComplete((){
    print('executed is done');
  });
}
```

- 异步async-await实现

async-await实现异步异常的捕获和同步实现一模一样。

```dart
Future<DataWrapper> createData() async {
  try {
    var id = await _loadDataFromDisk();
    var data = await _requestNetworkData(id);
    return DataWrapper(data);
  } on Exception catch (e) {
    print('this is error: $e');
  } finally {
    print('executed is done');
  }
}
```

### 4. async-await使用场景

其实关于async-await的使用场景, 学完上面的内容基本上都可以总结分析出来。

- 大部分Future使用的场景都可以使用async-await来替代，也建议使用async-await，毕竟人家这俩关键字就是为了简化Future API的调用，而且能将异步代码写成同步代码形式。代码简洁度上也能得到提升。
- 其实通过上面例子也能明显发现，对于一些依赖关系比较明显的Future，建议还是使用Future,毕竟链式调用功能非常强大，可以一眼就能看到每个Future之间的前后依赖关系。因为通过async-await虽然简化了回调形式，但是在某种程度上降低了Future之间的依赖关系。
- 对于依赖关系不明显且彼此独立Future，可以使用async-await。

### 5. 总结

到这里有关Dart异步编程中的async-await使用到这里就结束了，实际上async-await就是一个语法糖的使用，本质还是对Future API的使用，它的好处和优势就是可以将异步执行写成同步代码结构形式，我们也对它的代码结构进行拆分分析，清晰地分析了其语法糖背后的原理。这样对使用async-await非常有帮助。







## Stream

### 1. 为什么需要Stream

首先，在Dart单线程模型中，要实现异步就需要借助类似Stream、Future之类的API实现。所以**Stream可以很好地实现Dart的异步编程**。 此外，在Dart中一些异步场景中，比如磁盘文件、数据库读取等类似需要读取一系列的数据时，这种场景Future是不太合适的，所以在一些需要实现一系列异步事件时Stream就是不错的选择，**Stream提供一系列异步的数据序列**。换个角度理解**Stream就是一系列的Future组合，Future只能有一个异步响应，而Stream就是一系列的异步响应**。

```dart
//Futures实现
void main() {
  Future.delayed(Duration(seconds: 1), () => print('future value is: 1'));
}
```

输出结果:

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/18c254e4bd7c4e17a3621b72640ea303~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

```dart
//Stream实现
void main() async {
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), (int value) {
    return value + 1;
  });
  await stream.forEach((element) => print('stream value is: $element'));
}
```

输出结果: (输出结果是一直在运行的)

 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37183cc798724d12a4c02d191fcb2db0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### 2. 什么是Stream

用官方的术语来说: **Stream 是一系列异步事件的序列。其类似于一个异步的 Iterable，不同的是当你向 Iterable 获取下一个事件时它会立即给你，但是 Stream 则不会立即给你而是在它准备好时告诉你。** Streams是异步数据的源，Stream提供了一种接收事件序列的方式。每个事件要么是数据事件(或称为流的元素)，要么就是用于通知异常信息error事件。当Stream所有的事件发出以后，一个"done"结束事件将作为最后一个事件发出。实际上类似RX响应式流的概念。

#### 2.1 单一订阅模型(Single-subscription)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37720d74455a4cf891488a27c02a1e29~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 2.2 广播订阅模型(Broadcast-subscription)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f343b904d22f4ea6ba9a1e4c005b487e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 2.3 模型分析

- `StreamController` 是创建 `Stream` 对象主要方式之一
- 每个 `StreamController` 都会有一个槽口(Sink), 也就是Stream事件的入口，通过Sink的 `add` 将事件序列加入到 `StreamController` 中。
- `StreamController` 类似一个生产者和消费者模型，它不知道什么时候会有事件从Sink槽口加进来，而对于外部订阅者也不知道何时有事件出来，所以对于外部订阅者只需要添加监听就好了。
- 当有事件通过sink槽口加入到StreamController后，StreamController就开始工作，然后直到它输出数据。
- 需要注意的是从sink槽口加入的事件序列是有序的，监听器得到序列是和加入序列一致，也就是说StreamController处理并不会打乱事件序列顺序。
- 单一订阅者顾明思意就是只能有一个订阅者监听整个事件流，而对于广播订阅可以有若干个订阅者监听整个事件流，类似于广播通知的机制。

### 3. 如何使用Stream

#### 3.1 创建Stream的方法

##### 3.1.1 通过Stream构造器创建

- **Stream.fromFuture**: 通过Future创建一个新的 **single-subscription(单一订阅)Stream** , 当Future完成时触发 `then` 回调，然后就会把返回的value加入到 `StreamController` 中, 并且还会添加一个 `Done` 事件表示结束。若Future完成时触发 `onError` 回调，则会把error加入到`StreamController` 中, 并且还会添加一个 `Done` 事件表示结束。

```dart
factory Stream.fromFuture(Future<T> future) {
    _StreamController<T> controller =
        new _SyncStreamController<T>(null, null, null, null);
    future.then((value) {//future完成时，then回调
      controller._add(value);//将value加入到_StreamController中
      controller._closeUnchecked();//最后发送一个done事件
    }, onError: (error, stackTrace) {//future完成时，error回调
      controller._addError(error, stackTrace);//将error加入到_StreamController中
      controller._closeUnchecked();//最后发送一个done事件
    });
    return controller.stream;//最后返回stream
  }
void main() {
  Stream.fromFuture(Future.delayed(Duration(seconds: 1), () => 100)).listen(
      (event) => print(event),
      onDone: () => print('is done'),
      onError: (error, stacktrace) => print('is error, errMsg: $error'),
      cancelOnError: true);//cancelOnError: true(表示出现error就取消订阅，之后事件将无法接收；false表示出现error后，后面事件可以继续接收)
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33b4a8ca7b47485c83217df039c5dd5f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

- **Stream.fromIterable**: 通过从一个集合中获取其数据来创建一个新的**single-subscription(单一订阅)Stream**

```dart
void main() {
  Stream.fromIterable([1, 2, 3, 4, 5, 6, 7, 8])
      .map((event) => "this is $event")//还可以借助map,fold,reduce之类操作符，可以变换事件流
      .listen((event) => print(event),
          onDone: () => print('is done'),
          onError: (error, stacktrace) => print('is error, errMsg: $error'),
          cancelOnError: true);
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9702a5f888ac429e9e9324505c1a7f8b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 



- **Stream.fromFutures**:从一系列的Future中创建一个新的 **single-subscription(单一订阅)Stream**，每个future都有自己的data或者error事件，当整个Futures完成后，流将会关闭。如果Futures为空，流将会立刻关闭。

```dart
void main() {
  var future1 = Future.value(100);
  var future2 = Future.delayed(Duration(seconds: 1), () => 200);
  var future3 = Future.delayed(Duration(seconds: 2), () => 300);
  Stream.fromFutures([future1, future2, future3])
      .reduce((previous, element) => previous + element)//累加所有future中的值
      .asStream()
      .listen((event) => print(event),
          onDone: () => print('is done'),
          onError: (error, stacktrace) => print('is error, errMsg: $error'),
          cancelOnError: true);
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7e906064fae4f7681a234044b121ad6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

- **Stream.periodic:** 可以创建一个新的重复发射事件而且可以指定间隔时间的Stream,通过再StreamController的onResume方法中创建一个Timer对象，最后调用Timer的 **periodic方法。**

```dart
factory Stream.periodic(Duration period,
      [T computation(int computationCount)]) {
    Timer timer;
    int computationCount = 0;
    StreamController<T> controller;
    // Counts the time that the Stream was running (and not paused).
    Stopwatch watch = new Stopwatch();//创建Stopwatch用于计算Stream运行时间, 会一直运行不会停止

    void sendEvent() {
      watch.reset();
      T data;
      if (computation != null) {
        try {
          data = computation(computationCount++);
        } catch (e, s) {
          controller.addError(e, s);
          return;
        }
      }
      controller.add(data);
    }

    void startPeriodicTimer() {
      assert(timer == null);
      //创建Timer对象
      timer = new Timer.periodic(period, (Timer timer) {
        sendEvent();
      });
    }

    controller = new StreamController<T>(
        sync: true,
        onListen: () {
          watch.start();
          startPeriodicTimer();
        },
        onPause: () {
          timer.cancel();
          timer = null;
          watch.stop();
        },
        onResume: () {
          assert(timer == null);
          Duration elapsed = watch.elapsed;
          watch.start();
          timer = new Timer(period - elapsed, () {
            timer = null;
            startPeriodicTimer();
            sendEvent();
          });
        },
        onCancel: () {
          if (timer != null) timer.cancel();
          timer = null;
          return Future._nullFuture;
        });
    return controller.stream;
  }
void main() {
  Stream.periodic(Duration(seconds: 1), (value) => value + 100)
        .listen((event) => print(event),
          onDone: () => print('is done'),
          onError: (error, stacktrace) => print('is error, errMsg: $error'),
          cancelOnError: true);
}
```

输出结果: ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56b83cef2f4d4b089755750aa1ab0e91~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

##### 3.1.2 通过StreamController创建

- 创建任意类型StremController，也就是sink槽口可以加入任何类型的事件数据

```dart
import 'dart:async';

void main() {
  //1.创建一个任意类型StreamController对象
  StreamController streamController = StreamController(
      onListen: () => print('listen'),
      onCancel: () => print('cancel'),
      onPause: () => print('pause'),
      onResume: () => print('resumr'));
  //2.通过sink槽口添加任意类型事件数据
  streamController.sink.add(100);
  streamController.sink.add(100.121212);
  streamController.sink.add('THIS IS STRING');
  streamController.sink.close();//只有手动调用close方法发送一个done事件，onDone才会被回调
  //3.注册监听
  streamController.stream.listen((event) => print(event),
      onDone: () => print('is done'),
      onError: (error, stacktrace) => print('is error, errMsg: $error'),
      cancelOnError: true);
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6eae7ecbbc9e4da181a0684407bd2396~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

- 创建指定类型的StreamController, 也就是sink槽口可以加入对应指定类型的事件数据

```dart
import 'dart:async';

void main() {
  //1.创建一个int类型StreamController对象
  StreamController<int> streamController = StreamController(
      onListen: () => print('listen'),
      onCancel: () => print('cancel'),
      onPause: () => print('pause'),
      onResume: () => print('resumr'));
  //2.通过sink槽口添加int类型事件数据
  streamController.sink.add(100);
  streamController.sink.add(200);
  streamController.sink.add(300);
  streamController.sink.add(400);
  streamController.sink.add(500);
  streamController.sink.close(); //只有手动调用close方法发送一个done事件，onDone才会被回调
  //3.注册监听
  streamController.stream.listen((event) => print(event),
          onDone: () => print('is done'),
          onError: (error, stacktrace) => print('is error, errMsg: $error'),
          cancelOnError: true);
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1acaacc73334656b31ebe1c0e5fc1b1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

##### 3.1.3 通过async*创建

如果有一系列事件需要处理，也许会需要把它转化为 stream。这时候可以使用 **async*** 和 **yield** 来生成一个 Stream。

```dart
void main() {
  generateStream(10).listen((event) => print(event),
      onDone: () => print('is done'),
      onError: (error, stacktrace) => print('is error, errMsg: $error'),
      cancelOnError: true);
}

Stream<int> generateStream(int dest) async* {
  for (int i = 1; i <= dest; i++) {
    yield i;
  }
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/411721a4d8d44eee8b0b60f85f6acf0b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 3.2 监听Stream

##### 3.2.1 listen方法监听

监听Stream流主要就是使用 `listen` 这个方法，它有 `onData(必填参数)` , `onError(可选参数)` , `onDone(可选参数)` , `cancelOnError(可选参数)` 

- onData: 接收到数据时触发回调
- onError: 接收到异常时触发回调
- onDone: 数据接收完毕触发回调
- cancelOnError: 表示true(出现第一个error就取消订阅，之后事件将无法接收；false表示出现error后，后面事件可以继续接收)

```dart
StreamSubscription<T> listen(void onData(T data),
      {Function onError, void onDone(), bool cancelOnError}) {
    cancelOnError = identical(true, cancelOnError);
    StreamSubscription<T> subscription =
        _createSubscription(onData, onError, onDone, cancelOnError);
    _onListen(subscription);
    return subscription;
  }
```

##### 3.2.2 async-await配合for或forEach循环处理

通过async-await配合for或forEach可以实现当Stream中每个事件到来的时候处理它，由于Stream接收事件时机是不确定，所以for或forEach循环退出的时候一般是Stream关闭或者完成结束的时候

```dart
void main() async {
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), (int value) {
    return value + 1;
  });
  await stream.forEach((element) => print('stream value is: $element'));
}
```

输出结果:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56dd6e4ac5ed46e79ecba9e273e3d58a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 3.3 Stream流的转换

Stream流的转换实际上是通过类似 `map` 、 `take` 、 `where` 、 `reduce` 、 `expand` 之类的操作符函数实现流的变换。实际上他们作用和集合中变化操作符意思是类似，所以这里由于篇幅问题就不一一展开，有了前面集合操作符函数基础，这里也是类似，只不过这边返回的是 `Stream<T>` 而已。不过这里需要特别说下 `transform` 操作符函数.

`transform` 操作符函数它能实现更多自定义的流变化规则, 它通过传入一个 `StreamTransformer<T, S>` 参数，最后返回一个 `Stream<T>` . 也就是输入的流类型是 `T` . 输出的是 `S` ,通过`StreamTransformer` 输出一个新的Stream流。

```dart
Stream<S> transform<S>(StreamTransformer<T, S> streamTransformer) {
    return streamTransformer.bind(this);
  }

import 'dart:async';

void main() {
  //1.创建一个int类型StreamController对象
  StreamController<int> streamController = StreamController();
  //2.通过sink槽口添加int类型事件数据
  streamController.sink.add(100);
  streamController.sink.add(200);
  streamController.sink.add(300);
  streamController.sink.add(400);
  streamController.sink.add(500);
  streamController.sink.close(); //只有手动调用close方法发送一个done事件，onDone才会被回调

  //自定义StreamTransformer
  final transformer = StreamTransformer<int, String>.fromHandlers(handleData: (value, sink) {
    sink.add("this number is: $value");
  });
  //3.注册监听
  streamController.stream
      .transform(transformer)
      .listen((event) => print("second listener: $event"), onDone: () => print('second listener: is done'));
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/45f911a2790f4a6ea452a65bffb759ef~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 3.4 Stream流的种类

其实这里Stream流的种类划分是基于Stream流的订阅模型来划分，所以那么这里Stream流的种类只有两种: **Single-subscription(单一订阅)Stream、Broadcast-subscription(广播订阅)Stream**

#### 3.4.1 Single-subscription(单一订阅)Stream

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6b0ff05e70c45a4b03ea52d73ea101d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

单一订阅Stream在整个流的生命周期中**只会有一个订阅者监听，\**也就是listen方法只能调用一次，而且\**第一次listen取消(cancel)后，不能重复监听，否则会抛出异常**。

```dart
import 'dart:async';

void main() {
  //1.创建一个int类型StreamController对象
  StreamController<int> streamController = StreamController(
      onListen: () => print('listen'),
      onCancel: () => print('cancel'),
      onPause: () => print('pause'),
      onResume: () => print('resumr'));
  //2.通过sink槽口添加int类型事件数据
  streamController.sink.add(100);
  streamController.sink.add(200);
  streamController.sink.add(300);
  streamController.sink.add(400);
  streamController.sink.add(500);
  streamController.sink.close(); //只有手动调用close方法发送一个done事件，onDone才会被回调
  //3.注册监听
  streamController.stream.listen((event) => print(event), onDone: () => print('is done'));
  streamController.stream.listen((event) => print(event), onDone: () => print('is done')); //不允许两次监听
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74a8e40c454542d8b7fb9708a1925a3d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 3.4.2 **Broadcast-subscription**(**广播订阅**)Stream

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f6593388024d4f32bb19e1728323ec0f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

Broadcast广播订阅模型Stream, 可以同时存在任意多个订阅者监听，无论是否有订阅者，它都会产生事件。所以中途进来的收听者将**不会收到**之前的消息。 如果多个收听者要监听单一订阅Stream，需要使用 `asBroadcastStream` 转化成Broadcast广播订阅Stream. 或者创建BroadcastStream流可以通过继承Stream然后重写isBroadcast为true即可。

```dart
import 'dart:async';

void main() {
  //1.创建一个int类型StreamController对象
  StreamController<int> streamController = StreamController(
      onListen: () => print('listen'),
      onCancel: () => print('cancel'),
      onPause: () => print('pause'),
      onResume: () => print('resumr'));
  //2.通过sink槽口添加int类型事件数据
  streamController.sink.add(100);
  streamController.sink.add(200);
  streamController.sink.add(300);
  streamController.sink.add(400);
  streamController.sink.add(500);
  streamController.sink.close(); //只有手动调用close方法发送一个done事件，onDone才会被回调
  //3.注册监听
  Stream stream = streamController.stream.asBroadcastStream();//转换成BroadcastStream
  stream.listen((event) => print("first listener: $event"), onDone: () => print('first listener: is done'));
  stream.listen((event) => print("second listener: $event"), onDone: () => print('second listener: is done'));
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d98913d6aa8b4f9886cfc8e58d4c0a1d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

### 4. Stream使用的场景

Stream的使用场景有很多比如数据库的读写、文件IO的读写、基于多个网络请求转化处理都可以使用流来处理。下面会给出一个具体的文件复制例子实现IO文件读写使用Stream的场景。

```dart
import 'dart:async';
import 'dart:io';

void main() {
  copyFile(File('/Users/mikyou/Desktop/gitchat/test.zip'),
      File('/Users/mikyou/Desktop/gitchat/copy_dir/test.copy.zip'));
}

void copyFile(File sourceFile, File targetFile) async {
  assert(await sourceFile.exists() == true);
  print('source file path: ${sourceFile.path}');

  print('target file path: ${targetFile.path}');
  //以WRITE方式打开文件，创建缓存IOSink
  IOSink sink = targetFile.openWrite();

  //文件大小
  int fileLength = await sourceFile.length();
  //已读取文件大小
  int count = 0;
  //模拟进度条
  String progress = "-";

  //以只读方式打开源文件数据流
  Stream<List<int>> inputStream = sourceFile.openRead();
  inputStream.listen((List<int> data) {
    count += data.length;
    //进度百分比
    double num = (count * 100) / fileLength;
    print("${progress * (num ~/ 2)}[${num.toStringAsFixed(2)}%]");
    //将数据添加到缓存sink中
    sink.add(data);
  }, onDone: () {
    //数据流传输结束时，触发onDone事件
    print("复制文件结束！");
    //关闭缓存释放系统资源
    sink.close();
  });
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4c3f40fc5bb74af295c4d5dd19804bb9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

### 5. Stream与Future的区别

实际上有了上面对Stream的介绍，相信很多人基本上都能分析出Stream和Future的区别了。 先用官方专业术语做下对比: **Future 表示一个不会立即完成的计算过程**。与普通函数直接返回结果不同的是异步函数返回一个将会包含结果的 Future。该 Future 会在结果准备好时通知调用者 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7fb6503a5a0417d8c215f1b1251079e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

 **Stream 是一系列异步事件的序列**。其类似于一个异步的 Iterable，不同的是当你向 Iterable 获取下一个事件时它会立即给你，但是 Stream 则不会立即给你而是在它准备好时告诉你 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6916488f99a54337b20b473681341bf2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)  

可以使用一个餐厅吃饭场景来理解Future和Stream的区别:

**Future**就好比你去一家餐厅吃饭，在前台点好你想吃的菜后，付完钱后服务员会给你一个等待的号码牌(相当于先拿到一个Future)，后厨就开始根据你下的订单开始做菜，等到你的菜好了后，就可以通过号码牌拿到指定的菜了(返回的数据或异常信息)。 **Stream**就好比去一家餐厅吃饭，在前台点好A,B,C,D4种你想吃的菜后(订阅数据流过程)，然后你就去桌子等着，至于菜什么时候好，你也不知道所以就一直等着(类似于一直监听listen着)，后厨就开始根据你下的订单开始做菜, 等着你的第一盘A种菜好了后，服务员就会主动传送A到你的桌子上(基于一种类似订阅-推送机制)，没有特殊意外，服务员推送菜的顺序应该也是A,B,C,D。

### 6. 总结

到这里有关Dart异步编程中Stream就介绍完毕了，Stream在数据库读写，文件IO读写方面是非常实用。此外它支持响应式流式编程，可以利用它一些转换操作符实现对流的变换，可以实现一些类似Rx中的操作。



### 7. Flutter 中的 `Stream` 是冷流还是热流？(补充)

在 Flutter 中，**`Stream`** 可以是 **冷流（Cold Stream）** 也可以是 **热流（Hot Stream）**，具体取决于 `Stream` 的实现和数据源的行为。

------

#### **7.1 冷流（Cold Stream）**

- **特性：**
  - 数据流的生产是 **懒加载的**，只有当有订阅者时才开始生产数据。
  - 每个订阅者都会从头开始接收数据流（类似于一次快照）。
  - 常用于一次性事件流，如 HTTP 请求、文件读取、延迟加载数据等。
- **适用场景：**
  - 冷流适合用于处理那些订阅后才开始产生的事件流，比如异步任务结果、延迟加载内容。
- **冷流的例子：**

**示例 1：`Stream.fromIterable` 是冷流**

```
dart复制代码void main() async {
  final stream = Stream.fromIterable([1, 2, 3]);
  stream.listen((event) {
    print('Subscriber 1: $event');
  });

  // 延迟订阅第二个监听器
  await Future.delayed(Duration(seconds: 1));
  stream.listen((event) {
    print('Subscriber 2: $event');
  });
}
```

**输出：**

```
plaintext复制代码Subscriber 1: 1
Subscriber 1: 2
Subscriber 1: 3
Subscriber 2: 1
Subscriber 2: 2
Subscriber 2: 3
```

**解释：**

- 每个订阅者都会从头开始接收数据，即使第二个订阅者在第一组数据发送完成后才订阅。

------

#### **7.2 热流（Hot Stream）**

- **特性：**
  - 数据流的生产是 **独立于订阅者** 的，数据源始终在生产数据，无论是否有订阅者。
  - 订阅者只能接收到数据流中 **当前或之后产生的数据**。
  - 常用于实时数据流，如用户交互事件、传感器数据、股票价格等。
- **适用场景：**
  - 热流适合用于实时更新的场景，比如 UI 状态更新、实时通知。
- **热流的例子：**

**示例 2：`StreamController.broadcast` 是热流**

```
dart复制代码void main() async {
  final controller = StreamController<int>.broadcast();

  // 定期添加数据
  Timer.periodic(Duration(seconds: 1), (timer) {
    controller.add(timer.tick);
    if (timer.tick == 5) {
      timer.cancel();
      controller.close();
    }
  });

  // 第一个订阅者
  controller.stream.listen((event) {
    print('Subscriber 1: $event');
  });

  // 延迟订阅第二个监听器
  await Future.delayed(Duration(seconds: 3));
  controller.stream.listen((event) {
    print('Subscriber 2: $event');
  });
}
```

**输出：**

```
plaintext复制代码Subscriber 1: 1
Subscriber 1: 2
Subscriber 1: 3
Subscriber 1: 4
Subscriber 2: 4
Subscriber 1: 5
Subscriber 2: 5
```

**解释：**

- 数据流是实时的，第二个订阅者只能接收到 **它订阅时及之后** 的数据（从 `4` 开始）。
- 由于是广播流（`broadcast`），多个订阅者可以同时接收数据。





## sync*和async*生成器函数

生成器函数可能比较陌生，在平时开发中使用的不是特别频繁，但是因为它也是Dart异步编程的不可缺少的一部分，所以这里还是展开讲解分析。**生成器函数是一种用于延迟生成值序列的函数**，并且在Dart中生成器函数主要分为两种: **同步生成器函数和异步生成器函数**。我们知道比如 `int` 类型变量(同步)或 `Future<int>` (异步)类型变量都只能产生单一的值，而 `Iterable<int>` 类型变量(同步)或 `Stream<int>` 类型变量(异步)具有产生多个值的能力。**其中同步生成器函数是立即按需生成值，并不会像Future,Stream那样等待，而异步生成器函数则是异步生成值，也就是它有足够时间去生成值**。

|             | Single value(单一的值) | Zero or more value(零个值或者更多的值) |
| ----------- | ---------------------- | -------------------------------------- |
| Sync(同步)  | int                    | Iterable                               |
| Async(异步) | Future                 | Stream                                 |

### 1. 为什么需要生成器函数

其实在平时Flutter或者Dart开发中生成器函数使用并不多，但是遇到使用它的场景，有了生成器函数就会非常简单，因为手动去实现值的生成函数还是比较繁杂的。比如说要实现一个同步生成器，需要自定义一个可迭代的类去继承 `IterableBase` 并且需要重写 `iterator` 方法用于返回一个新的Iterator对象。为此还得需要声明自己的迭代器类。此外还得实现成员方法 `moveNext` 和成员属性 `current` 以此来判断是否迭代到末尾。这还是写一个同步生成器步骤，整个过程还是比较繁杂的。所以Dart给你提供一个方法可以直接生成一个同步生成器函数，简化整个实现的步骤。但是如果要去手动实现一个异步生成器远比同步生成器更复杂，所以直接提供一个简单异步生成器函数会使得整个开发更加高效，可以把精力更加专注于业务。这里就以同步生成器函数举例：

说到同步生成器函数先来回顾下 `Iterable` 和 `Iterator` 

```dart
//Iterator可以将一系列的值依次迭代
abstract class Iterator<E> {
  bool moveNext();//表示是否迭代器有下一个值，迭代器把下一个值加载为当前值，直到下一个值为空返回false

  E get current; //返回当前迭代的值，也就是最近迭代一次的值
}
```

按照上面介绍步骤一起来手动实现一个 `Iterable` ，实际上也很简单只是简单地包了个壳

```dart
class StringIterable extends Iterable<String> {
  final List<String> stringList; //实际上List就是一个Iterable

  StringIterable(this.stringList);

  @override
  Iterator<String> get iterator =>
      stringList.iterator; //通过将List的iterator，赋值给iterator

}

//这样StringIterable就是一个特定类型String的迭代器，我们就可以使用for-in循环进行迭代
void main() {
  var stringIterable = StringIterable([
    "Dart",
    "Java",
    "Kotlin",
    "Swift"
  ]);
  for (var value in stringIterable) {
    print('$value');
  }
}

//甚至你还可以将StringIterable结合map、where、reduce之类操作符函数之类对迭代器值进行变换
void main() {
  var stringIterable = StringIterable([
    "Dart",
    "Java",
    "Kotlin",
    "Swift"
  ]);
  stringIterable
      .map((language) => language.length)//可以结合map一起使用，实际上本质就是Iterable对象转换，将StringIterable转换成MappedIterable
      .forEach(print);
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/163044e63f194ce3afc1f8e581db7ca8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)  、



![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b88597abf55a4f26842e84fd5c61f317~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

可以看到上面其实还不是一个真正严格意义手动实现一个 `Iterable` , 那么问题来了如何手动实现一个同步生成器函数，注意：**同步生成器函数必须返回一个 `Iterable` 类型，然后需要使用 `sync\*` 修饰该函数，以此来标记该函数是一个同步生成器函数。**

```dart
void main() {
  final numbers = getRange(1, 10);
  for (var value in numbers) {
    print('$value');
  }
}

//用于生成一个同步序列
Iterable<int> getRange(int start, int end) sync* { //sync*告诉Dart这个函数是一个按需生产值的同步生成器函数
  for (int i = start; i <= end; i++) {
    yield i;//yield关键字有点像return，但是它是单次返回值，并不会像return直接结束整个函数
  }
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/feb7b79a25364999910cfd7e8c06a365~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

通过对比发现了生成器函数是真的简单方便，只需要通过 `sync*` 和 `yield` 关键字就能实现一个任意类型迭代器，比手动实现一个同步生成器函数更加简单，所以应该知道为什么需要生成器函数。其实异步生成器函数也是类似。

### 2. 什么是生成器(Generator)

生成器函数是**一种可以很方便延迟生成值的序列的函数**，生成器主要分为两种：**同步生成器函数和异步生成器函数。其中同步生成函数需要使用 `sync*` 关键字修饰，返回一个 `Iterable` 对象(表示可以顺序访问值的集合)；异步生成器函数需要使用 `async*` 关键字修饰，返回的是一个 `Stream` 对象(表示异步数据事件)。**此外同步生成器函数是立即按需生成值，并不会像Future,Stream那样等待，而异步生成器函数则是异步生成值，也就是它有足够时间去生成值。

#### 2.1 同步生成器(**Synchronous** Generator)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dfe651517ed8447db60fd876afd7dc8d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

同步生成器函数需要配合 `sync*` 关键字和 `yield` 关键字，最终返回的是一个 `Iterable` 对象，其中`yield` 关键字用于每次返回单次的值，并且需要注意它的返回并不是结束整个函数。

```dart
import 'dart:io';

void main() {
  final numbers = countValue(10);
  for (var value in numbers) {
    print('$value');
  }
}

Iterable<int> countValue(int max) sync* {//sync*告诉Dart这个函数是一个按需生产值的同步生成器函数
  for (int i = 0; i < max; i++) {
    yield i; //yield关键字每次返回单次的值
    sleep(Duration(seconds: 1));
  }
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06597ca4e85249a5be879c9be4e0234a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 2.2 异步生成器(**Asynchronous** Generator)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d45f99349094b3bbe1b925d65e2dbf7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

异步生成器函数需要配合 `async*` 关键字和 `yield` 关键字,最终返回的是一个 `Stream` 对象，需要注意的是**生成Stream也是一个单一订阅模型的Stream,** 也就是说不能同时存在多个订阅者监听否则会出现异常，如果要实现支持多个监听者通过 `asBroadcastStream` 转换成一个广播订阅模型的Stream。

```dart
import 'dart:io';

void main() {
  Stream<int> stream = countStream(10);
  stream.listen((event) {
    print('event value: $event');
  }).onDone(() {
    print('is done');
  });
}

Stream<int> countStream(int max) async* {
  //async*告诉Dart这个函数是生成异步事件流的异步生成器函数
  for (int i = 0; i < max; i++) {
    yield i;
    sleep(Duration(seconds: 1));
  }
}
```

输出结果： 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bda569da0eab494fbdba1a036cc658c7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 2.3 yield关键字

**yield关键字**在生成器函数中是用于依序生成每一个值，它有点类似return语句，但是和return语句不一样的是执行一次yield并不会结束整个函数。相反它每次只提供单个值，并挂起(注意不是阻塞)等待调用者请求下一个值，然后它就会再执行一遍，比如上述例子for循环中，**每一次循环执行都会触发yield执行一次返回每次的单值并且进入下一次循环的等待**。

```dart
Iterable<int> countValue(int max) sync* {//sync*告诉Dart这个函数是一个按需生产值的同步生成器函数
  for (int i = 0; i < max; i++) {
    yield i; //每执行一次循环就会触发当前次yield生成值，然后进入下一次的等待
    sleep(Duration(seconds: 1));
  }
}
```

#### 2.4 yield* 关键字

yield关键字是用于循环中生产值后跟着一个具体对象或者值，但是yield*后面则是跟着一个函数，一般会跟着递归函数，通过它就能实现类似二次递归函数功能。虽然yield关键字也能实现一个二次递归函数但是比较复杂，但是如果使用yield*关键字就会更加简单。

```dart
//这是使用yield实现二次递归函数
Iterable naturals(n) sync* { 
  if (n > 0) { 
     yield n; 
     for (int i in naturals(n-1)) { 
       yield i; 
     } 
  }
} 

//这是使用yield*实现的二次递归函数
Iterable naturals(n) sync* { 
  if ( n > 0) { 
    yield n; 
    yield* naturals(n-1); 
 } 
} 
```

#### 2.5 await for

关于await for在Stream那一篇文章中已经有说到，对于一个同步的迭代器Iterable而言我们可以使用for-in循环来遍历每个元素，而对于一个异步的Stream可以很好地使用await for来遍历每个事件元素。

```dart
void main() async {
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), (int value) {
    return value + 1;
  });
  await stream.forEach((element) => print('stream value is: $element'));
}
```

### 3. 如何使用生成器函数(Generator)

#### 3.1 sync* + yield实现同步生成器

```dart
void main() {
  final Iterable<int> sequence = countDownBySync(10);
  print('start');
  sequence.forEach((element) => print(element));
  print('end');
}

Iterable<int> countDownBySync(int num) sync* {//sync*
  while (num > 0) {
    yield num--;//yield返回值
  }
}
```

输出结果： 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7740b6e64444c099df4d97cd0533d7c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 3.2  async* + yield实现异步生成器

```dart
void main() {
  final Stream<int> sequence = countDownByAsync(10);
  print('start');
  sequence.listen((event) => print(event)).onDone(() => print('is done'));
  print('end');
}

Stream<int> countDownByAsync(int num) async* {//async*表示异步返回一个Stream
  while (num > 0) {
    yield num--;
  }
}
```

输出结果： 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73c96469aea941ada8684b8542b953d5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 3.3 sync* + yield*实现递归同步生成器

```dart
void main() {
  final Iterable<int> sequence = countDownBySyncRecursive(10);
  print('start');
  sequence.forEach((element) => print(element));
  print('end');
}

Iterable<int> countDownBySyncRecursive(int num) sync* {
  if (num > 0) {
    yield num;
    yield* countDownBySyncRecursive(num - 1);//yield*后跟一个递归函数
  }
}
```

输出结果: 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65d3ca575b7f4faba7ade52a208a3d9b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

#### 3.4 async* + yield*实现递归异步生成器

```dart
void main() {
  final Stream<int> sequence = countDownByAsync(10);
  print('start');
  sequence.listen((event) => print(event)).onDone(() => print('is done'));
  print('end');
}

Stream<int> countDownByAsyncRecursive(int num) async* {
  if (num > 0) {
    yield num;
    yield* countDownByAsyncRecursive(num - 1);//yield*后跟一个递归函数
  }
}
```

输出结果： 

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7671b48be6d480db5012c8a9ba4d188~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 

### 4. 生成器函数(Generator)使用场景

关于生成器函数使用在开发过程其实并不多，但是也不是说关于生成器函数使用场景就没有了，否则这篇文章就没啥意义了。实际上如果有小伙伴接触Flutter开发中其中有一个**Bloc的状态管理框架**，可以发现它里面大量地使用了异步生成器函数，一起来看下:

```dart
class CountBloc extends Bloc<CounterEvent, int> {
  @override
  int get initialState => 0;

  @override
  Stream<int> mapEventToState(CounterEvent event) async* {
    switch (event) {
      case CounterEvent.decrement:
        yield state - 1;
        break;
      case CounterEvent.increment:
        yield state + 1;
        break;
    }
  }
}
```

其实除了上面所说Bloc状态管理框架中使用到的场景，可能还有一种场景那就是异步二次函数递归场景，比如实现某个动画轨迹计算，实际上都是通过一些二次函数计算模拟出来，所以这时候生成器递归函数是不是就可以派上用场了。虽然生成器函数使用场景不是很频繁，但是需要做到某个特定场景第一时间想到用它可以更加简单的实现就可以了。

### 5. 总结

到这里有关Flutter异步编程系列就结束了，下一个系列将进入Dart注解+APT生成代码技术专题。尽管生成器函数使用在开发过程其实并不多，但是它也作为Flutter异步中一部分，有一些特定场景如果能想到用它来解决，一定会事半功倍的。







作者：熊喵先生
链接：https://juejin.cn/post/6931742162195644430、https://juejin.cn/post/6934286884050829320、https://juejin.cn/post/6936899202987327518、https://juejin.cn/post/6939510009654935588、https://juejin.cn/post/6942103045320769550、https://juejin.cn/post/6944697869563609119
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。