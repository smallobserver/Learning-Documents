# Kotlin协程



优点：

- 轻量：可以在单个线程上运行多个协程，因为协程支持挂起，不会使正在运行协程的线程阻塞。挂起比阻塞节省内存，且支持多个并行操作
- 内存泄露更少：使用结构化并发机制在一个作用域内执行多个操作
- 内置取消支持：取消功能会自动通过正在运行的协程层次结构传播
- Jetpack 集成：许多 Jetpack 库都包含提供全面协程支持的扩展。某些库还提供自己的协程作用域，可供你用于结构化并发



协程的四个基础概念：

- suspend function。即挂起函数，delay() 就是协程库提供的一个用于实现非阻塞式延时的挂起函数
- CoroutineScope。即协程作用域，GlobalScope 是 CoroutineScope 的一个实现类，用于指定协程的作用范围，可用于管理多个协程的生命周期，所有协程都需要通过 CoroutineScope 来启动
- CoroutineContext。即协程上下文，包含多种类型的配置参数。`Dispatchers.IO` 就是 CoroutineContext 这个抽象概念的一种实现，用于指定协程的运行载体，即用于指定协程要运行在哪类线程上
- CoroutineBuilder。即协程构建器，协程在 CoroutineScope 的上下文中通过 launch、async 等协程构建器来进行声明并启动。launch、async 均被声明为 CoroutineScope 的扩展方法



## 概念

- **协程**

  - 挂起恢复
  - 程序自己处理挂起恢复
  - 程序自己处理挂起恢复来实现协程的协作运行

  核心就是一段程序能够被**挂起**，并在稍后再**在挂起的位置恢复**。

  Kotlin协程是依赖于线程池API的，在一个线程可以创建多个协程，并且协程运行时并**不会阻塞当前线程**。

  > 其最大的特点就是能够**以阻塞（同步）方式写出非阻塞（异步）的代码**。
  >
  > **Kotlin 协程的核心竞争力在于：它能简化`异步并发`任务。**

- **挂起恢复**

  表示**从当前运行线程中暂时离开**，等待任务完成。 当**任务执行完成**后，再从**当前运行线程**继续执行后续任务。



## suspend挂起函数

### 概念

使用`suspend`关键字修饰的函数被称为**挂起函数**。

`suspend`关键字的作用就只是告诉编译器，这是一个**挂起函数**，仅作一个**标记**，提醒会**挂起**执行。

> - 挂起函数被限制在**只能**在协程作用域`CoroutineScope`、协程、另一个`suspend`挂起函数内被调用。
> - **挂起并不等于切线程**，同一个线程内依然可以**挂起**。

比如官方API中的`delay`函数就是个顶层的`suspend`函数，**不会阻塞线程**，将协程延迟指定时间，然后**恢复**执行后续任务。

![delay源码解析.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81ec5aa6fa854868a88cd627e0ff1495~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> **挂起函数在挂起时，不会阻塞协程所运行的线程**

在编译器内，`suspend`修饰的函数，以及调用`suspend`函数的地方都还会在左侧有个**挂起点**的标识

但`suspend`修饰的函数**不一定会有挂起操作**，协程的挂起是由内部框架执行的，`suspend`关键字本身只是个提示作用，并限制挂起函数调用位置。

> 如果`suspend`函数内部没有挂起逻辑，编译器也会提示`redundant suspend modifier`警告，表示这个`suspend`关键字是多余的。

### suspend挂起函数的原理

`suspend` 的本质，就是 `CallBack`，其实就是一个`Continuation `的回调。

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
//      相当于 onSuccess     结果   
//                 ↓         ↓
    public fun resumeWith(result: Result<T>)
}
```

以上这个从`挂起函数`转换成`Continuation 函数`的过程，被称为：CPS 转换(Continuation-Passing-Style Transformation)。

调用挂起函数返回 `CoroutineSingletons.COROUTINE_SUSPENDED` 表示函数被挂起了。此时框架会将执行挂起，让出执行线程，等到返回正常值后，从挂起点恢复。

CPS工作过程：

```kotlin
//代码示例
  suspend fun testCoroutine() {
//        log("start")
        val user = getUserInfo()
//        log(user)
        val friendList = getFriendList(user)
//        log(friendList)
        val feedList = getFeedList(friendList)
//        log(feedList)
    }

    // delay(1000L)用于模拟网络请求

    //挂起函数
// ↓
    suspend fun getUserInfo(): String {
        withContext(Dispatchers.IO) {
            delay(1000L)
        }
        return "BoyCoder"
    }
    //挂起函数
// ↓
    suspend fun getFriendList(user: String): String {
        withContext(Dispatchers.IO) {
            delay(1000L)
        }
        return "Tom, Jack"
    }
    //挂起函数
// ↓
    suspend fun getFeedList(list: String): String {
        withContext(Dispatchers.IO) {
            delay(1000L)
        }
        return "{FeedList..}"
    }

//反编译源码
@Nullable
   public final Object testCoroutine(@NotNull Continuation $completion) {
      Object $continuation;
      label37: {
         if ($completion instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)$completion;
            if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
               ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
               break label37;
            }
         }

         $continuation = new ContinuationImpl($completion) {
            Object L$0;
            // $FF: synthetic field
            Object result;
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
               this.result = $result;
               this.label |= Integer.MIN_VALUE;
               return NetUtil.this.testCoroutine((Continuation)this);
            }
         };
      }

      Object var10000;
      label31: {
         Object var7;
         label30: {
            Object $result = ((<undefinedtype>)$continuation).result;
            var7 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
            switch (((<undefinedtype>)$continuation).label) {
               case 0:
                  ResultKt.throwOnFailure($result);
                  ((<undefinedtype>)$continuation).L$0 = this;
                  ((<undefinedtype>)$continuation).label = 1;
                  var10000 = this.getUserInfo((Continuation)$continuation);
                  if (var10000 == var7) {
                     return var7;
                  }
                  break;
               case 1:
                  this = (NetUtil)((<undefinedtype>)$continuation).L$0;
                  ResultKt.throwOnFailure($result);
                  var10000 = $result;
                  break;
               case 2:
                  this = (NetUtil)((<undefinedtype>)$continuation).L$0;
                  ResultKt.throwOnFailure($result);
                  var10000 = $result;
                  break label30;
               case 3:
                  ResultKt.throwOnFailure($result);
                  var10000 = $result;
                  break label31;
               default:
                  throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
            }

            String user = (String)var10000;
            ((<undefinedtype>)$continuation).L$0 = this;
            ((<undefinedtype>)$continuation).label = 2;
            var10000 = this.getFriendList(user, (Continuation)$continuation);
            if (var10000 == var7) {
               return var7;
            }
         }

         String friendList = (String)var10000;
         ((<undefinedtype>)$continuation).L$0 = null;
         ((<undefinedtype>)$continuation).label = 3;
         var10000 = this.getFeedList(friendList, (Continuation)$continuation);
         if (var10000 == var7) {
            return var7;
         }
      }

      String var4 = (String)var10000;
      return Unit.INSTANCE;
   }

   @Nullable
   public final Object getUserInfo(@NotNull Continuation $completion) {
      Object $continuation;
      label20: {
         if ($completion instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)$completion;
            if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
               ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
               break label20;
            }
         }

         $continuation = new ContinuationImpl($completion) {
            // $FF: synthetic field
            Object result;
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
               this.result = $result;
               this.label |= Integer.MIN_VALUE;
               return NetUtil.this.getUserInfo((Continuation)this);
            }
         };
      }

      Object $result = ((<undefinedtype>)$continuation).result;
      Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch (((<undefinedtype>)$continuation).label) {
         case 0:
            ResultKt.throwOnFailure($result);
            CoroutineContext var10000 = (CoroutineContext)Dispatchers.getIO();
            Function2 var10001 = (Function2)(new Function2((Continuation)null) {
               int label;

               public final Object invokeSuspend(Object $result) {
                  Object var2 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
                  switch (this.label) {
                     case 0:
                        ResultKt.throwOnFailure($result);
                        Continuation var10001 = (Continuation)this;
                        this.label = 1;
                        if (DelayKt.delay(1000L, var10001) == var2) {
                           return var2;
                        }
                        break;
                     case 1:
                        ResultKt.throwOnFailure($result);
                        break;
                     default:
                        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                  }

                  return Unit.INSTANCE;
               }

               public final Continuation create(Object value, Continuation $completion) {
                  return (Continuation)(new <anonymous constructor>($completion));
               }

               public final Object invoke(CoroutineScope p1, Continuation p2) {
                  return ((<undefinedtype>)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
               }

               // $FF: synthetic method
               // $FF: bridge method
               public Object invoke(Object p1, Object p2) {
                  return this.invoke((CoroutineScope)p1, (Continuation)p2);
               }
            });
            ((<undefinedtype>)$continuation).label = 1;
            if (BuildersKt.withContext(var10000, var10001, (Continuation)$continuation) == var4) {
               return var4;
            }
            break;
         case 1:
            ResultKt.throwOnFailure($result);
            break;
         default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }

      return "BoyCoder";
   }

   @Nullable
   public final Object getFriendList(@NotNull String var1, @NotNull Continuation $completion) {
      Object $continuation;
      label20: {
         if ($completion instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)$completion;
            if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
               ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
               break label20;
            }
         }

         $continuation = new ContinuationImpl($completion) {
            // $FF: synthetic field
            Object result;
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
               this.result = $result;
               this.label |= Integer.MIN_VALUE;
               return NetUtil.this.getFriendList((String)null, (Continuation)this);
            }
         };
      }

      Object $result = ((<undefinedtype>)$continuation).result;
      Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch (((<undefinedtype>)$continuation).label) {
         case 0:
            ResultKt.throwOnFailure($result);
            CoroutineContext var10000 = (CoroutineContext)Dispatchers.getIO();
            Function2 var10001 = (Function2)(new Function2((Continuation)null) {
               int label;

               public final Object invokeSuspend(Object $result) {
                  Object var2 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
                  switch (this.label) {
                     case 0:
                        ResultKt.throwOnFailure($result);
                        Continuation var10001 = (Continuation)this;
                        this.label = 1;
                        if (DelayKt.delay(1000L, var10001) == var2) {
                           return var2;
                        }
                        break;
                     case 1:
                        ResultKt.throwOnFailure($result);
                        break;
                     default:
                        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                  }

                  return Unit.INSTANCE;
               }

               public final Continuation create(Object value, Continuation $completion) {
                  return (Continuation)(new <anonymous constructor>($completion));
               }

               public final Object invoke(CoroutineScope p1, Continuation p2) {
                  return ((<undefinedtype>)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
               }

               // $FF: synthetic method
               // $FF: bridge method
               public Object invoke(Object p1, Object p2) {
                  return this.invoke((CoroutineScope)p1, (Continuation)p2);
               }
            });
            ((<undefinedtype>)$continuation).label = 1;
            if (BuildersKt.withContext(var10000, var10001, (Continuation)$continuation) == var5) {
               return var5;
            }
            break;
         case 1:
            ResultKt.throwOnFailure($result);
            break;
         default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }

      return "Tom, Jack";
   }

   @Nullable
   public final Object getFeedList(@NotNull String var1, @NotNull Continuation $completion) {
      Object $continuation;
      label20: {
         if ($completion instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)$completion;
            if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
               ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
               break label20;
            }
         }

         $continuation = new ContinuationImpl($completion) {
            // $FF: synthetic field
            Object result;
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
               this.result = $result;
               this.label |= Integer.MIN_VALUE;
               return NetUtil.this.getFeedList((String)null, (Continuation)this);
            }
         };
      }

      Object $result = ((<undefinedtype>)$continuation).result;
      Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch (((<undefinedtype>)$continuation).label) {
         case 0:
            ResultKt.throwOnFailure($result);
            CoroutineContext var10000 = (CoroutineContext)Dispatchers.getIO();
            Function2 var10001 = (Function2)(new Function2((Continuation)null) {
               int label;

               public final Object invokeSuspend(Object $result) {
                  Object var2 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
                  switch (this.label) {
                     case 0:
                        ResultKt.throwOnFailure($result);
                        Continuation var10001 = (Continuation)this;
                        this.label = 1;
                        if (DelayKt.delay(1000L, var10001) == var2) {
                           return var2;
                        }
                        break;
                     case 1:
                        ResultKt.throwOnFailure($result);
                        break;
                     default:
                        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                  }

                  return Unit.INSTANCE;
               }

               public final Continuation create(Object value, Continuation $completion) {
                  return (Continuation)(new <anonymous constructor>($completion));
               }

               public final Object invoke(CoroutineScope p1, Continuation p2) {
                  return ((<undefinedtype>)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
               }

               // $FF: synthetic method
               // $FF: bridge method
               public Object invoke(Object p1, Object p2) {
                  return this.invoke((CoroutineScope)p1, (Continuation)p2);
               }
            });
            ((<undefinedtype>)$continuation).label = 1;
            if (BuildersKt.withContext(var10000, var10001, (Continuation)$continuation) == var5) {
               return var5;
            }
            break;
         case 1:
            ResultKt.throwOnFailure($result);
            break;
         default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }

      return "{FeedList..}";
   }


```

when 表达式实现了协程状态机

`continuation.label` 是状态流转的关键

`continuation.label` 改变一次，就代表协程切换了一次

每次协程切换后，都会检查是否发生异常

testCoroutine 里的原本的代码，被`拆分`到状态机里各个状态中，`分开执行`

getUserInfo(continuation)，getFriendList(user, continuation)，getFeedList(friendList, continuation) 三个函数调用传的同一个 `continuation` 实例。

一个函数如果被挂起了，它的返回值会是：`CoroutineSingletons.COROUTINE_SUSPENDED`

切换协程之前，状态机会把之前的结果以成员变量的方式保存在 `continuation` 中。



再看挂起具体流程：

ContinuationImpl继承BaseContinuationImpl

```kotlin
BaseContinuationImpl:
 public final override fun resumeWith(result: Result<Any?>) {
        // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
        var current = this
        var param = result
        while (true) {
            // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure
            // can precisely track what part of suspended callstack was already resumed
            probeCoroutineResumed(current)
            with(current) { // 循环，主要是循环执行invokeSuspend，根据该方法里的lable状态执行不同的挂起函数
                val completion = completion!! // fail fast when trying to resume continuation without completion
                val outcome: Result<Any?> =
                    try {
                        // 执行协程体
                        val outcome = invokeSuspend(param)
                        // 重点关注这里如果返回值是COROUTINE_SUSPENDED，则跳出循环，也就是不再执行invokeSuspend方法，其实这就是挂起点了
                        // 那就看看什么时候invokeSuspend会返回COROUTINE_SUSPENDED呢，这就需要往invokeSuspend追溯
                        if (outcome === COROUTINE_SUSPENDED) return
                        Result.success(outcome)
                    } catch (exception: Throwable) {
                        Result.failure(exception)
                    }
                releaseIntercepted() // this state machine instance is terminating
                if (completion is BaseContinuationImpl) {
                    // unrolling recursion via loop
                    current = completion
                    param = outcome
                } else {
                    // top-level completion reached -- invoke and return
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }
```

父协程体执行的时候遇到了挂起函数withContext，withContext里开启自身的协程执行自身的协程体任务；
同时withContext返回COROUTINE_SUSPENDED
父协程体resumeWith方法里判断到了返回值是COROUTINE_SUSPENDED，于是结束了resumeWith的执行，父协程体里面withContext后面的代码暂停执行，但其实没有阻塞线程
父协程的协程体SuspendLambda保存了协程状态，记录了当前挂起的地方，此时withContext的协程体在执行，父协程处于挂起状态，等待被通知恢复





## 创建协程

### 标准库创建（不推荐）

协程创建的方式有很多种，首先来看看在kotlin协程标准库中，是如何创建协程的。

```kotlin
val coroutine = suspend {
     //模拟异步任务
     ...
     println("create coroutine")
     "result"
 }.createCoroutine(object : Continuation<String>{
     override val context: CoroutineContext = Job()
 
     override fun resumeWith(result: Result<String>) {
         println("Coroutine Result $result")
     }
 })
 //执行协程
 coroutine.resumeWith(Result.success(Unit))
```

`suspend`修饰函数作为接收者的拓展函数`createCoroutine`，创建协程，调用`resultWith`启动协程。

![createContinuation源码解析](.\res\createContinuation源码解析.webp)

而通常我们创建协程都是要让协程直接启动的，所有官方还提供了另一种拓展函数`startCoroutine`。

![startCoroutine源码](.\res\startCoroutine源码.webp)

官方不推荐直接使用这种方式创建协程，这就如同直接`new Thread()`把线程跑完再回调回来一样，中间几乎是个黑盒无法干预。

所以官方要求使用`CoroutineScope`管理内部协程的生命周期。

最常用的方法还是使用`CoroutineScope`的拓展函数`launch`与`async`来创建、启动协程。



### launch

`launch`函数会创建新协程，**立即执行**，**不会阻塞当前线程**

> 通常是用于启动不需要返回值的新协程任务。

![Kotlin协程的launch源码解析.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f0605ff46964fca952c3866cd07d8a3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

该函数会返回一个`Job`对象是协程的句柄，可用于管理协程的开启与关闭。

```kotlin
fun testCoroutineLaunch() {
     val scope = CoroutineScope(Job())
     val job = scope.launch{
         println("running launch")
         ...
     }
     //控制新创建协程任务的关闭
     job.cancel()
 }
```

参数`block`就是以`CoroutineScope`为接收者的函数类型。

### async

而`async`函数同样会创建新协程并**立即执行**，**不会阻塞当前线程**。

![Kotlin协程async源码解析.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37be23583e8a4a4e9f3debe33e436c3e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

与`launch`的唯一区别是，该函数会返回一个`Deferred`对象（`Job`子类），可以通过调用`await`函数挂起协程，直到协程执行完成，**返回执行结果**。（**不阻塞当前线程**）

> - `async`方法构建的协程，支持并发
> - `async`协程不调用`await`函数也会开始执行协程

```kotlin
fun testAsync() = runBlocking {
     val startTime = System.currentTimeMillis()
     val task1 = async {
         delay(100)
         println("task1 running print")
         1
     }
     val task2 = async {
         delay(150)
         println("task2 running print")
         2
     }
     task1.await()
     task2.await()
     val endTime = System.currentTimeMillis()
     println("async task over ${endTime - startTime}")
 }
 
 task1 running print
 task2 running print
 async task over 164 //总计时间表明两个协程为并发关系
```

此外官方还提供了`awaitAll`函数，可以将相同类型返回值的协程同步挂起并返回List

```kotlin
val taskList = listOf(task1,task2)
 taskList.awaitAll()
 
 awaitAll(task1,task2)
```

> `launch` 和 `async` 仅能够在 `CouroutineScope` 中使用，所以任何创建的协程都会被该`CoroutineScope`追踪。
>
> Kotlin禁止创建不能够被追踪的协程，从而避免协程泄漏。

### runBlocking

还有一种仅在**单元测试**时使用的协程创建方式`runBlocking`。

`runBlock`以**阻塞当前线程**的方式创建一个新协程作用域，直到协程体执行完成。

![runBlocking源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4433347b9f464a6e91e8e377bdda6e3c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 回调转协程

回想一下在Android日常开发中是如何进行异步任务？

```kotlin
val handler = Handler()
 
 getTestData(object : CallBack{
      override fun onSuccess(result : String){
          ...
          handler.post(...)
      }
 })
```

在线程池开启一个线程，然后在异步任务结束后，要有个**恢复**的动作，将线程切换回UI主线程更新UI。而这种切换线程恢复的动作，在以前都会演变成**回调**的方式。

如果此时需要顺序触发下一个异步任务，叠加一层回调，依次类推，也就演变成了俗称的"**回调地狱**"。

那么协程要如何解决**回调地狱**呢？

官方提供了`suspendCoroutine`和`suspendCancellableCoroutine`来将回调API转化为`suspend`的函数。

- **suspendCoroutine**

  ![suspendCoroutine源码解析.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b0c0e2d38c3419cb5a1ee308bb2cd1c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **suspendCancellableCoroutine**

  ![suspendCancellableCoroutine源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/358b22a3eacd481180ed6990ee0ea6d8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

其中`suspendCancellableCoroutine`是最为常用的，也是官方推荐使用转换回调API的方式。

相比`suspendCoroutine`新增了**可取消**的能力，能调用`cancel`方法取消函数挂起。

> 内部调用的`suspendCoroutineUninterceptedOrReturn`函数是编译器内部字节码的函数，没有源码，从注释上来看，作用就是拿到`Continuation`实例

这两个函数的参数`block`都是以`Continuation`（`CancellableContinuation`是其子类）作为参数的高阶函数。

![Continuation源码.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef6d739cbe1b42d088ca565a90f727b4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

在函数体内调用`resumeWith`函数，接收`Result`类型参数，即**恢复**挂起，返回函数处理结果。

此外，官方还提供了拓展函数`resume`和`resumeWithException`来更方便的恢复协程。

![resume与resumeWithException源码.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e2299c9453a4b05b72d8c3db2cf62b8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`Retrofit`自2.6.0版本后就默认支持协程，他们又是如何做到的呢？

其实同样也是利用`suspendCancellableCoroutine`，调用`resume`方法进行网络请求回调。

![Retrofit对协程的支持源码.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8774a00595dd4750a120f88f3b1f14bc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

如此，利用`suspendCancellableCoroutine`，我们便能优雅的写出串行的异步（回调）逻辑

```kotlin
//模拟的回调API封装
 fun queryData1() = suspendCancellableCoroutine<String>{continuation->
     getTestData1(object : CallBack){
         override fun onSuccess(result : String){
             continuation.resume("result")
         }
         
         override fun onFail(throwable : Throwable){
             continuation.resumeWithException(throwable)
         }
     }
 }
 //模拟的回调API封装
 fun queryData2(value : String) 
 = suspendCancellableCoroutine<String>{continuation->
     getTestData2(object : CallBack){
         override fun onSuccess(result : String){
             continuation.resume("$value new query")
         }
         
         override fun onFail(throwable : Throwable){
             continuation.resumeWithException(throwable)
         }
     }
 }
 
 fun test() = runBlocking{
     val result = queryData1()
     val result2 = queryData2(result)
 }
```

> 注意，这两种方式内部并不是处于协程作用域，不能调用挂起函数。
>
> 仅用于将回调API转化为挂起函数。

### 小结

观察`launch`、`async`、`suspendCoroutine`、`suspendCancellableCoroutine`函数，其实都可以算作是对标准协程构建的封装，最终会调用`Continuation`的`resumeWith`方法来恢复协程。

> 在kotlin协程中，有很多操作都围绕`Continuation`展开的，而这怎么看都像是个回调嘛

**kotlin协程的挂起后恢复本质上还是回调**，只是把这部分代码隐藏在内部，实现了一个**有限状态机**，让外部的异步代码看起来像是同步一样。

- 如果一个协程内调用的挂起函数（如`delay`、`yeild`等），如果没有主动调用`resumeWith`进行回调值，就会返回`COROUTINE_SUSPENDED`表示进入**挂起状态**，而不会执行协程内部的后续内容，相当于当前这个函数已经执行完毕，直到这个挂起函数调用`resumeWith`才会继续执行一次。
- **协程的挂起恢复逻辑**都集中在原始协程体的包装类`ContinuationImpl`内部，该包装类是在创建协程过程中通过内部调用的`createCoroutineUnintercepted`方法对原始协程体包装创建的。
- **协程在返回`COROUTINE_SUSPENDED`挂起时相当于已经执行完包装类的`resumeWith`函数的逻辑，当前线程可以继续执行其他之后的逻辑**，所以并不会阻塞线程



## CoroutineContext

在Kotlin协程中，`CorroutineContext`是相当重要的组成部分。

`launch`、`async`、`runBlocking`等函数都接收`CorroutineContext`类型的参数。

![CoroutineContext源码解析.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af57c6a17d1c4c4e8e6435626eadfcef~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`CorroutineContext`其实是一个包含了用户定义的一些各种不同元素的`Element`对象**集合**

> 内部实现为**单链表**，每一种`Element`都有一个唯一key。

作为协程的持久上下文， 其允许定义协程的行为：

- [`Job`](https://link.juejin.cn?target=https%3A%2F%2Fkotlin.github.io%2Fkotlinx.coroutines%2Fkotlinx-coroutines-core%2Fkotlinx.coroutines%2F-job%2Findex.html)：控制协程的生命周期。
- [`CoroutineDispatcher`](https://link.juejin.cn?target=https%3A%2F%2Fkotlin.github.io%2Fkotlinx.coroutines%2Fkotlinx-coroutines-core%2Fkotlinx.coroutines%2F-coroutine-dispatcher%2Findex.html)：将工作分派到适当的线程。
- [`CoroutineName`](https://link.juejin.cn?target=https%3A%2F%2Fkotlin.github.io%2Fkotlinx.coroutines%2Fkotlinx-coroutines-core%2Fkotlinx.coroutines%2F-coroutine-name%2Findex.html)：协程的名称，可用于调试。
- [`CoroutineExceptionHandler`](https://link.juejin.cn?target=https%3A%2F%2Fkotlin.github.io%2Fkotlinx.coroutines%2Fkotlinx-coroutines-core%2Fkotlinx.coroutines%2F-coroutine-exception-handler%2Findex.html)：处理未捕获的异常。

### Job

**Job** 作为协程的句柄，能够管理协程的生命周期

> 对于每一个您所创建的协程 (通过`launch`或者 `async`)，它会返回一个 Job 实例

![Job源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b68fa9cc318c49319e1e54b95ec0284c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 生命周期

每个Job任务都包含一系列生命周期状态机制

- **New** ：新创建，未开始执行
- **Active** ：调用start开启协程，使协程处于活跃状态
- **Completing** ：当前协程已完成，等待子协程的执行完成
- **Completed** : 协程已完成已结束
- **Cancelling** : 处于取消中状态，出现异常或者调用`cancel`时，等待子协程结束
- **Canceled**：已取消

这些生命周期状态并不能直接访问，但`Job`提供了间接访问属性

```kotlin
 //处于活跃状态（Active）
 public val isActive: Boolean
 //是否已完成
 public val isCompleted: Boolean
 //是否已取消
 public val isCancelled: Boolean
```

Job的生命周期状态流转图（来自官网）

![job生命周期.webp](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52b6af73b28c44f6b529e867d869572d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### Deferred

`async`方法创建协程返回的`Deferre`继承自`Job`。拥有相同的状态机制。除此之外，`Deferre`能够调用`await`函数，等待协程执行完后获取返回值。

![Deferred await源码解析.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36da2b90b6e446b4b5163a6c8c8ba6a4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### CoroutineName

![CoroutineName源码解析.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d12514ae9cc84b9894118e8895b13647~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`CoroutineName`是用户用来指定的协程名称的，方便调试和定位问题



### 调度器

**协程调度器**顾名思义就是调度协程在哪个线程上执行，在JVM底层就是基于**线程池API**，而Android主线程就是基于主线程`Handler`。

> 在启动协程时，如果没有指定`CoroutineContext`的调度器，且没有拦截器时，会默认就会添加一个`Dispatchers.default`调度器，也即是**协程作用域的默认调度器**。
>
> ```kotlin
> public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
>      val combined = coroutineContext + context  //继承自当前作用域的上下文
>      val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined
>      return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
>          debug + Dispatchers.Default else debug  //默认添加default作为拦截器
>  }
> ```

![CoroutineDispatcher源码解析.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f633e6230ea64481bbe45ba0186ad5fb~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

官方API默认提供了四种调度器

- **Dispatchers**

> - **Default** ：默认的后台计算线程池
> - **IO** : 适合 IO 密集型的任务线程池，比如：读写文件，操作数据库以及网络请求
> - **Main** : UI主线程
> - **Unconfined** ：当前默认的协程中运行。（但是在遇到第一个挂起点之后，恢复的线程是不确定的。所以对于 Unconfined 其实是无法保证全都在当前线程中调用的。）

其中Default和IO在内部会优化共享同一个线程池,避免频繁切换线程造成的资源损耗。

同时还提供了个顶层`suspend`函数`withContext`用于切换协程内部运行的协程上下文（包括调度器）。

![withContext源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc63270ba35b4a868fad8057c937c20b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> 可以只在外部调用 `withContext` 只切换一次协程上下文，这样可以在多次调用的情况下，以尽可能避免了线程切换所带来的性能损失。

- 自定义调度器

  官方提供了`ExecutorService`的拓展函数，可以很方便的将线程池转化为`Dispatcher`。 ![asCoroutineDispatcher源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c62ea8248a804502b90ea0d8b2985b09~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 拦截器

`CoroutineDispatcher`是`ContinuationInterceptor`的实现类

![ContinuationInterceptor源码解析.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a4583588f29488e8849bb99c3e8ef9b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

前面通过`launch`或`async`创建协程时，内部会调用`intercepted`方法，从而都是经过拦截器拦截包装后的`Coroutine`对象。

这就类似于`okHttp`中的拦截器的作用。所不同的是，**拦截器在`CoroutineContext`中只能存在一个**。

> - 所有拦截器实现类，如果将`Element`的Key自定义，内部依旧会将自定义的Key重新替换成`ContinuationInterceptor`。
> - 拦截器在`CoroutineContext`中**永远处于队列末尾**，`CoroutineContext`取值时是从最后入队的开始取的，**永远只会取到最先添加的拦截器**。

#### 调度器原理

引用官方的一张图，清晰表述了协程的线程调度流程

![协程调度器流程.webp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e6d17c171a424e35b2b8721301597a6c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

前面提到，由协程调度器`CoroutineDispatcher`实现的`interceptContinuation`函数，在通过`intercept`进行拦截时，将上一层的`ContinuationImpl`对象，再次包装一层，创建`DispatchedContinuation`。

当通过`launch`或`async`创建协程时，最后会调用这个包装`DispatchedContinuation`的`resumeCancellableWith`方法，进而实现线程调度功能

![协程调度器的调度入口.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84eb583e106f46d0940682b1f2d76770~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> `DispatchedContinuation`类继承自`DispatchedTask`，而其父类`SchedulerTask`是`Task`类的类型别名，`Task`又实际是实现了`Runnable`接口。
>
> 所以实际在调度器内部执行的任务就是对`DispatchedContinuation`的**拆包装过程**，操作内部的上一层`Continuation`包装类`ContinuationImpl`的`resumeWith`方法。
>
> ```kotlin
> //SchedulerTask.kt
>  internal actual typealias SchedulerTask = Task //实现了Runnable接口
>  //DispatchedTask.kt
>  internal abstract class DispatchedTask<in T>(
>      @JvmField public var resumeMode: Int
>  ) : SchedulerTask() {
>      public final override fun run() {
>          ...
>          val delegate = delegate as DispatchedContinuation<T>
>          val continuation = delegate.continuation //上一层的协程体
>          ...
>          continuation.resume(getSuccessfulResult(state)) //执行恢复上一层协程体
>          ...     
>      }
>  }
> ```

以`DEFAULT`默认调度器来说，最终会创建`ExperimentalCoroutineDispatcher`，其内部维持着一个`CoroutineScheduler`，由Kotlin协程自己实现`Executor`接口的线程池。

而`ExperimentalCoroutineDispatcher`内部调用`dispatch`也是分发到这个线程池内部进行执行调度。

> - `IO`调度器创建的`LimitingDispatcher`内部也还是会调用`ExperimentalCoroutineDispatcher`的`disptach`方法。只是在`LimitingDispatcher`中对线程池调度的并发数量进行了限制，默认被限制在最大并发数64个的程度上。
> - 实际进行调度的功能还是利用其父类`ExperimentalCoroutineDispatcher`中创建的线程池来实现。
> - 所以`Default`与`IO`调度器实际上是公用同一个线程池。

![协程默认调度器原理.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50e7227a5029408f99b78869d89f8881~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

至于前面提到的外部线程池转为协程调度器的拓展函数`asCoroutineDispatcher`方法，创建的`ExecutorCoroutineDispatcherImpl`类则是继承自其父类的`ExecutorCoroutineDispatcher`。

**将协程任务转交给外部线程池进行调度**。

```kotlin
internal class ExecutorCoroutineDispatcherImpl(override val executor: Executor) : ExecutorCoroutineDispatcher(), Delay {
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        try {
            executor.execute(wrapTask(block)) //交由外部线程池调度
        } catch (e: RejectedExecutionException) {
            unTrackTask()
            cancelJobOnRejection(context, e)
            Dispatchers.IO.dispatch(context, block) //出现线程异常则交由默认IO线程池重新调度
        }
    }
}
```

在Android平台上自然要使用主线程的`Handler`才能在主线程运行，那么`Dispatchers.Main`又是如何的进行调度的呢？

```kotlin
//Dispatchers.kt
 @JvmStatic
 public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher
```

> 按注释上的说明 :
>
> 在其他平台上，如 JS 和 Native 上，其相当于`Default`调度器。
>
> 在JVM平台上，也有`JavaFx`、`Swing EDT` 等调度器需要添加对应依赖才存在实现

在`MainCoroutineDispatcher`内部由`ServiceLoader`来寻找`MainDispatcherFactory`的实现类中关于`createDispatcher`的实现。

而Android平台上的`MainDispatcherFactory`实现类就是`AndroidDispatcherFactory`。

内部创建`HandlerContext`的协程调度器，利用主线程的`Handler.post`方法将协程任务添加到消息队列进行执行。

![协程的主线程调度器.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60a5f62f765245ab9a51fc35d6594243~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

##### Dispatchers.DEFAULT

Dispatchers.DEFAULT它持有的线程池是CoroutineScheduler：



```kotlin
  【SchedulerCoroutineDispatcher】
    internal open class SchedulerCoroutineDispatcher(
    private val corePoolSize: Int = CORE_POOL_SIZE,
    private val maxPoolSize: Int = MAX_POOL_SIZE,
    private val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS,
    private val schedulerName: String = "CoroutineScheduler",
) : ExecutorCoroutineDispatcher() {
   private var coroutineScheduler = createScheduler()

    private fun createScheduler() =
        CoroutineScheduler(corePoolSize, maxPoolSize, idleWorkerKeepAliveNs, schedulerName)
    
    override fun dispatch(context: CoroutineContext, block: Runnable): Unit = coroutineScheduler.dispatch(block) // 线程池分发

    ...略
}

  【CoroutineScheduler】
   internal class CoroutineScheduler(
    @JvmField val corePoolSize: Int, // 核心线程数
    @JvmField val maxPoolSize: Int, // 最大线程数
    @JvmField val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS, // 空闲线程存活时间
    @JvmField val schedulerName: String = DEFAULT_SCHEDULER_NAME // 线程池名称"CoroutineScheduler"
) : Executor, Closeable {
    // 用于存储全局的纯CPU(不阻塞)任务
    @JvmField
    val globalCpuQueue = GlobalQueue()

    // 用于存储全局的执行非纯CPU(可能阻塞)任务
    @JvmField
    val globalBlockingQueue = GlobalQueue()

    // 用于记录当前处于Parked状态(一段时间后自动终止)的线程的数量
    private val parkedWorkersStack = atomic(0L)

    // 用于保存当前线程池中的线程
    // workers[0]永远为null，作为哨兵位
    // index从1到maxPoolSize为有效线程
    @JvmField
    val workers = AtomicReferenceArray<Worker?>(maxPoolSize + 1)

    // 控制状态
    private val controlState = atomic(corePoolSize.toLong() shl CPU_PERMITS_SHIFT)
    // 表示已经创建的线程的数量
    private val createdWorkers: Int inline get() = (controlState.value and CREATED_MASK).toInt()
    // 表示可以获取的CPU令牌数量，初始值为线程池核心线程数量
    private val availableCpuPermits: Int inline get() = availableCpuPermits(controlState.value)

    // 获取指定的状态的已经创建的线程的数量
    private inline fun createdWorkers(state: Long): Int = (state and CREATED_MASK).toInt()
    // 获取指定的状态的执行阻塞任务的数量
    private inline fun blockingTasks(state: Long): Int = (state and BLOCKING_MASK shr BLOCKING_SHIFT).toInt()
    // 获取指定的状态的CPU令牌数量
    public inline fun availableCpuPermits(state: Long): Int = (state and CPU_PERMITS_MASK shr CPU_PERMITS_SHIFT).toInt()

    // 当前已经创建的线程数量加1
    private inline fun incrementCreatedWorkers(): Int = createdWorkers(controlState.incrementAndGet())
    // 当前已经创建的线程数量减1
    private inline fun decrementCreatedWorkers(): Int = createdWorkers(controlState.getAndDecrement())
    // 当前执行阻塞任务的线程数量加1
    private inline fun incrementBlockingTasks() = controlState.addAndGet(1L shl BLOCKING_SHIFT)
    // 当前执行阻塞任务的线程数量减1
    private inline fun decrementBlockingTasks() {
        controlState.addAndGet(-(1L shl BLOCKING_SHIFT))
    }

    // 尝试获取CPU令牌
    private inline fun tryAcquireCpuPermit(): Boolean = controlState.loop { state ->
        val available = availableCpuPermits(state)
        if (available == 0) return false
        val update = state - (1L shl CPU_PERMITS_SHIFT)
        if (controlState.compareAndSet(state, update)) return true
    }
    // 释放CPU令牌
    private inline fun releaseCpuPermit() = controlState.addAndGet(1L shl CPU_PERMITS_SHIFT)

    // 表示当前线程池是否关闭
    private val _isTerminated = atomic(false)
    val isTerminated: Boolean get() = _isTerminated.value

companion object {
    // 用于标记一个线程是否在parkedWorkersStack中(处于Parked状态)
    @JvmField
    val NOT_IN_STACK = Symbol("NOT_IN_STACK")

    // 线程的三个状态
    // CLAIMED表示线程可以执行任务
    // PARKED表示线程暂停执行任务，一段时间后会自动进入终止状态
    // TERMINATED表示线程处于终止状态
    private const val PARKED = -1
    private const val CLAIMED = 0
    private const val TERMINATED = 1

    // 以下五个常量为掩码
    private const val BLOCKING_SHIFT = 21 // 2x1024x1024
    // 1-21位
    private const val CREATED_MASK: Long = (1L shl BLOCKING_SHIFT) - 1
    // 22-42位
    private const val BLOCKING_MASK: Long = CREATED_MASK shl BLOCKING_SHIFT
    // 42
    private const val CPU_PERMITS_SHIFT = BLOCKING_SHIFT * 2
    // 43-63位
    private const val CPU_PERMITS_MASK = CREATED_MASK shl CPU_PERMITS_SHIFT

    // 以下两个常量用于require中参数判断
    internal const val MIN_SUPPORTED_POOL_SIZE = 1
    // 2x1024x1024-2
    internal const val MAX_SUPPORTED_POOL_SIZE = (1 shl BLOCKING_SHIFT) - 2

    // parkedWorkersStack的掩码
    private const val PARKED_INDEX_MASK = CREATED_MASK
    // inv表示01反转
    private const val PARKED_VERSION_MASK = CREATED_MASK.inv()
    private const val PARKED_VERSION_INC = 1L shl BLOCKING_SHIFT
}

       
    

    fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, tailDispatch: Boolean = false) {
        trackTask() // this is needed for virtual time support
        // 传入ruanable和taskContext构建一个Task实例，taskContext决定线程是cpu密集型还是阻塞型，通过上面Dispatchers.DEFAULT的代码可以知
        // 道，它分发任务的时候是创建的NonBlockingContext，也就是非阻塞型的
        val task = createTask(block, taskContext) 
        // 判断当前线程是否运行在当前线程池
        val currentWorker = currentWorker()
        // 尝试加入本地队列，注意这个方法是Woker的扩展方法，这个本地队列是Woker的变量，
        // Woker是对Thread的一层封装，专门用于协程线程池里用的
        val notAdded = currentWorker.submitToLocalQueue(task, tailDispatch)
        // notAdded不为null，代表加入本地任务队列失败，也代表此时的线程不是运行在线程池
        if (notAdded != null) {
            // 尝试加入全局任务队列，这里的全局指的是线程池里维护的两个变量
            if (!addToGlobalQueue(notAdded)) {
                // Global queue is closed in the last step of close/shutdown -- no more tasks should be accepted
                throw RejectedExecutionException("$schedulerName was terminated")
            }
        }
        val skipUnpark = tailDispatch && currentWorker != null
        // Checking 'task' instead of 'notAdded' is completely okay
        // 如果任务是非阻塞任务，则唤醒cpu线程
        if (task.mode == TASK_NON_BLOCKING) {
            if (skipUnpark) return
            signalCpuWork()
        } else {
            // 否则就唤醒阻塞线程
            signalBlockingWork(skipUnpark = skipUnpark)
        }
    }
    // currentWorker方法
    private fun currentWorker(): Worker? = (Thread.currentThread() as? Worker)?.takeIf { it.scheduler == this }

    // Worker的扩展函数submitToLocalQueue
    private fun CoroutineScheduler.Worker?.submitToLocalQueue(task: Task, tailDispatch: Boolean): Task? {
      // Worker 为空，直接返回任务本身
      if (this == null) return task
      // 非阻塞的任务且此时Worker处于阻塞状态，则直接返回
      if (task.mode == TASK_NON_BLOCKING && state === CoroutineScheduler.WorkerState.BLOCKING) {
        return task
      }
      //表示本地队列里存有任务了
      mayHaveLocalTasks = true
      //加入到本地队列里
      //localQueue 为Worker的成员变量
      return localQueue.add(task, fair = tailDispatch)
    }
    // addToGlobalQueue方法
    private fun addToGlobalQueue(task: Task): Boolean {
      return if (task.isBlocking) {
        //加入到全局阻塞队列
        globalBlockingQueue.addLast(task)
    } else {
        //加入到全局cpu队列
        globalCpuQueue.addLast(task)
      }
    }
    fun signalCpuWork() {
     //尝试去唤醒正在挂起的线程，若是有线程可以被唤醒，则无需创建新线程
      if (tryUnpark()) return
      //若唤醒不成功，则需要尝试创建线程
      if (tryCreateWorker()) return
      //再试一次，边界条件
      tryUnpark()
    }
    // 尝试创建线程的方法
    private fun tryCreateWorker(state: Long = controlState.value): Boolean {
     //获取当前已经创建的线程数
      val created = createdWorkers(state)
      //获取当前阻塞的任务数
      val blocking = blockingTasks(state)
      //已创建的线程数-阻塞的任务数=非阻塞的线程数
      //coerceAtLeast(0) 表示结果至少是0
      val cpuWorkers = (created - blocking).coerceAtLeast(0)
      //如果非阻塞数小于核心线程数
    // 现在若是已经创建了5个线程，而这几个线程都在执行IO任务，此时就需要再创建新的线程来执行任务，因为此时CPU是空闲的。
     //只要非阻塞任务的个数小于核心线程数，那么就需要创建新的线程，目的是为了充分利用CPU。
      if (cpuWorkers < corePoolSize) {
          //创建线程
          val newCpuWorkers = createNewWorker()
          //如果当前只有一个非阻塞线程并且核心线程数>1，那么再创建一个线程
          //目的是为了方便"偷"任务...
          if (newCpuWorkers == 1 && corePoolSize > 1) createNewWorker()
          //创建成功
          if (newCpuWorkers > 0) return true
      }
      return false
    }
    // 创建线程的方法
    //workers 为Worker 数组，因为需要对数组进行add 操作，因此需要同步访问
    private fun createNewWorker(): Int {
      synchronized(workers) {
        if (isTerminated) return -1
        val state = controlState.value
        //获取已创建的线程数
        val created = createdWorkers(state)
        //阻塞的任务数
        val blocking = blockingTasks(state)
        //非阻塞的线程数
        val cpuWorkers = (created - blocking).coerceAtLeast(0)
        //非阻塞的线程数不能超过核心线程数
        if (cpuWorkers >= corePoolSize) return 0
        //已创建的线程数不能大于最大线程数
        if (created >= maxPoolSize) return 0
        val newIndex = createdWorkers + 1
        require(newIndex > 0 && workers[newIndex] == null)
        //构造线程
        val worker = Worker(newIndex)
        //记录到数组里
        workers[newIndex] = worker
        //记录创建的线程数
        require(newIndex == incrementCreatedWorkers())
        //开启线程
        worker.start()
        //当前非阻塞线程数
        return cpuWorkers + 1
    }
}

    ...略
}
```

通过以上分析，知道有三个任务队列，这里要理清一下这些任务队列的关系：

![img](https:////upload-images.jianshu.io/upload_images/4045300-5df3f1bf2dc9391e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

**线程池dispatch的操作可以大概做个总结：
 1.先把分发下来的runable任务封装成Task，并标记它是阻塞型的还是非阻塞型的
 2.判断当前线程是否是运行在当前线程池里的线程，是的话就把传进来的Task直接加入当前线程的任务队列中
 3.否则根据task类型加入到线程池的相应任务队列中
 4.尝试唤醒相应类型的线程，没有的话就创建线程来执行工作**



##### 线程池里的Worker

看看真正执行任务的地方，重点看Worker里的runWorker方法



```kotlin
// Worker内部类
    internal inner class Worker private constructor() : Thread() {
        init {
            isDaemon = true
        }

        // guarded by scheduler lock, index in workers array, 0 when not in array (terminated)
        @Volatile // volatile for push/pop operation into parkedWorkersStack
        var indexInArray = 0
            set(index) {
                name = "$schedulerName-worker-${if (index == 0) "TERMINATED" else index.toString()}"
                field = index
            }

        constructor(index: Int) : this() {
            indexInArray = index
        }

        inline val scheduler get() = this@CoroutineScheduler

        @JvmField
        val localQueue: WorkQueue = WorkQueue()
        // 线程状态
        var state = WorkerState.DORMANT
        

        override fun run() = runWorker()

        private fun runWorker() {
            var rescanned = false
          //一直查找任务去执行，除非worker终止了
          while (!isTerminated && state != CoroutineScheduler.WorkerState.TERMINATED) {
            //从队列里寻找任务
            //mayHaveLocalTasks：本地队列里是否有任务
            val task = findTask(mayHaveLocalTasks)
            if (task != null) {
                rescanned = false
                minDelayUntilStealableTaskNs = 0L
                //任务获取到后，执行任务
                executeTask(task)
                //任务执行完毕，继续循环查找任务
                continue
            } else {
                mayHaveLocalTasks = false
            }
            if (minDelayUntilStealableTaskNs != 0L) {
                // 这个rescanned控制下面的分支代码的执行
                if (!rescanned) {
                    rescanned = true
                } else {
                    //挂起一段时间再去偷
                        rescanned = false
                        tryReleaseCpu(WorkerState.PARKING)
                        interrupted()
                        LockSupport.parkNanos(minDelayUntilStealableTaskNs)
                        minDelayUntilStealableTaskNs = 0L   // 只执行一次，下次循环不会再命中              
                }  
                continue
            }
          //尝试挂起
          tryPark()
        }
        //释放token
        tryReleaseCpu(CoroutineScheduler.WorkerState.TERMINATED)
      }

      fun findTask(scanLocalQueue: Boolean): Task? {
          //尝试获取cpu 许可
          //若是拿到cpu 许可，则可以执行任何任务
           // 它和核心线程数相关，假设我们是8核CPU，那么同一时间最多只能有8个线程在CPU上执行。因此，若是其它线程想
          // 要执行非阻塞任务(占用CPU)，需要申请许可(token)，申请成功说明有CPU空闲，此时该线程可以执行非阻塞任务。否则，只能执行阻塞任务。
          if (tryAcquireCpuPermit())
               return findAnyTask(scanLocalQueue)
           //拿不到，若是本地队列有任务，则从本地取，否则从全局阻塞队列取
           val task = if (scanLocalQueue) {
               localQueue.poll() ?: globalBlockingQueue.removeFirstOrNull()
           } else {
               globalBlockingQueue.removeFirstOrNull()
           }
           //都拿不到，则偷别人的
            // 当从本地队列、全局队列里都没找出任务时，当前的Worker打起了别个Woker的主意。我们知道全局队列是所有Worker共
          // 享，而本地队列是每个Worker私有的。因此，当前Worker发现自己没任务可以执行的时候会去看看其它Worker的本地队列里
          // 是否有可以执行的任务，若是有就可以偷过来用。
           return task ?: trySteal(blockingOnly = true)
       }

       private fun findAnyTask(scanLocalQueue: Boolean): Task? {
           if (scanLocalQueue) {
               //可以从本地队列找
               val globalFirst = nextInt(2 * corePoolSize) == 0
               if (globalFirst) pollGlobalQueues()?.let { return it }
               localQueue.poll()?.let { return it }
               if (!globalFirst) pollGlobalQueues()?.let { return it }
           } else {
               //从全局队列找
               pollGlobalQueues()?.let { return it }
           }
           //偷别人的
           return trySteal(blockingOnly = false)
       }
// 从全局队列获取任务
private fun pollGlobalQueues(): Task? {
    // 随机获取CPU任务或者非CPU任务
    if (nextInt(2) == 0) {
        // 优先获取CPU任务
        globalCpuQueue.removeFirstOrNull()?.let { return it }
        return globalBlockingQueue.removeFirstOrNull()
    } else {
        // 优先获取非CPU任务
        globalBlockingQueue.removeFirstOrNull()?.let { return it }
        return globalCpuQueue.removeFirstOrNull()
    }
}

      // 挂起函数，这是针对woker的状态
      private fun tryPark() {
          //没有在挂起栈里
        if (!inStack()) {
            //将worker放入挂起栈里
            parkedWorkersStackPush(this)
            return
        }
        while (inStack() && workerCtl.value == CoroutineScheduler.PARKED) { // Prevent spurious wakeups
            if (isTerminated || state == CoroutineScheduler.WorkerState.TERMINATED) break
            //真正挂起(不是实时，会暂时挂起一段时间idleWorkerKeepAliveNs，线程空闲时间)，并标记worker state 状态，会修改state = WorkerState.TERMINATED，runWorker循环里会判断该标记，若是终止了，则循环停止，整个线程执行结束。
            park()
        }

        ...略
    }
```

**做个小总结：
 1.线程执行的时候从全局队列、本地队列里查找任务。
 2.若是没找到，则尝试从别的Worker 本地队列里偷取任务。
 3.能够找到任务则最终会执行协程体里的代码。
 4.若是没有任务，则根据策略挂起一段时间或是最终退出线程的执行。**

##### Dispatchers.IO:



```kotlin
internal object DefaultIoScheduler : ExecutorCoroutineDispatcher(), Executor {

    private val default = UnlimitedIoScheduler.limitedParallelism(
        systemProp(
            IO_PARALLELISM_PROPERTY_NAME,
            64.coerceAtLeast(AVAILABLE_PROCESSORS)
        )
    )
    override fun dispatch(context: CoroutineContext, block: Runnable) {
            default.dispatch(context, block)
    }
    ...略
}

// The unlimited instance of Dispatchers.IO that utilizes all the threads CoroutineScheduler provides
private object UnlimitedIoScheduler : CoroutineDispatcher() {

    @InternalCoroutinesApi
    override fun dispatchYield(context: CoroutineContext, block: Runnable) {
        DefaultScheduler.dispatchWithContext(block, BlockingContext, true)
    }
     // 最终调用了DefaultScheduler的分分发方法
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        DefaultScheduler.dispatchWithContext(block, BlockingContext, false)
    }
}
// UnlimitedIoScheduler父类CoroutineDispatcher的方法
    public open fun limitedParallelism(parallelism: Int): CoroutineDispatcher {
        parallelism.checkParallelism()
        // 返回一个UnlimitedIoScheduler的代理Dispatcher
        return LimitedDispatcher(this, parallelism)
    }
```

通过以上代码可以看到Dispatchers.IO最终调用到的线程池分发方法是DEFAULT里的，而DEFAULT是个单例，所以两者其实**共享了线程池CoroutineScheduler**.
 但是随着对代理类LimitedDispatcher的深入研究发现Dispatchers.IO策略上有所不同。



```kotlin
// 因为Dispatchers.IO是单例的，所以内部的这个LimitedDispatcher也是单例的，先看名字，这是一个受限制的分发器，
// 限制啥？限制的是最大并行数量，由系统属性设定的值或 CPU 核心数的最大值决定，系统属性值一般设置的是 64，也就是说，一般来说，该调度器可能会创建 64 个线程来执行任务
internal class LimitedDispatcher(
    private val dispatcher: CoroutineDispatcher,
    private val parallelism: Int
) : CoroutineDispatcher(), Runnable, Delay by (dispatcher as? Delay ?: DefaultDelay) {

    @Volatile
    private var runningWorkers = 0

    private val queue = LockFreeTaskQueue<Runnable>(singleConsumer = false)

    // A separate object that we can synchronize on for K/N
    private val workerAllocationLock = SynchronizedObject()

    @ExperimentalCoroutinesApi
    override fun limitedParallelism(parallelism: Int): CoroutineDispatcher {
        parallelism.checkParallelism()
        if (parallelism >= this.parallelism) return this
        return super.limitedParallelism(parallelism)
    }

    // 核心方法，分发任务
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        // 先执行dispatchInternal
        dispatchInternal(block) {
            // 再把自己作为runnable参数给到代理的dispatcher
            dispatcher.dispatch(this, this)
        }
    }
  
    private inline fun dispatchInternal(block: Runnable, dispatch: () -> Unit) {
        // 添加任务到队列，如果此时最大并行任务超过了限制就直接return，暂时不执行dispatch方法
        if (addAndTryDispatching(block)) return
        // 统计当前的并行任务，超过了就直接return，暂时不执行dispatch方法
        if (!tryAllocateWorker()) return
        // 做完了添加操作执行dispatch()，此时才会真正分发到协程线程池CoroutineSchduler
        dispatch()
    }
    // 添加任务到队列并尝试分发
     private fun addAndTryDispatching(block: Runnable): Boolean {
         // 添加本地任务队列，添加不需要条件 来了就添加
        queue.addLast(block)
        // 判断正在执行的任务的数量是否大于最大并行线程数量，正是在这里做到了限制最大并行IO线程的作用
        return runningWorkers >= parallelism 
    }
    // 尝试分配线程，其实不会新建线程，只是统计一下当前的并行线程数量，在这里仍然会先判断是否大于了最大并行限制
    private fun tryAllocateWorker(): Boolean {
        synchronized(workerAllocationLock) {
            if (runningWorkers >= parallelism) return false
            ++runningWorkers
            return true
        }
    }

       // 注意他自己实现了Runnable，最终他是把自己送到CoroutineSchduler去执行的，协程传进来的任务在这里被包装了执行
    override fun run() {
        var fairnessCounter = 0
        // 别被这个循环体迷惑，就以为线程都是串行执行的，实际每次新协程创建任务都会执行run方法，然后最终包装到协程线程池去并发执行
        // 这个循环是为了执行超过最大并发数的时候，那些只添加到了队列但是没有立马执行的任务
        while (true) {
            val task = queue.removeFirstOrNull()
            if (task != null) {
                try {
                    task.run()
                } catch (e: Throwable) {
                    handleCoroutineException(EmptyCoroutineContext, e)
                }
                //  这里比较有意思，当有大量的并发任务（比如短时间添加了200个任务），那么这个方法有可能会长时间的执行下去，所以
              // 这里为了公平起见不长期霸占资源，当超过执行了16个任务后就重新分发一次，这样就能短暂的让出cpu让别的线程执行（不知道理解的对不对）
                if (++fairnessCounter >= 16 && dispatcher.isDispatchNeeded(this)) {
                    // Do "yield" to let other views to execute their runnable as well
                    // Note that we do not decrement 'runningWorkers' as we still committed to do our part of work
                    dispatcher.dispatch(this, this)
                    return
                }
                continue
            }

            synchronized(workerAllocationLock) {
                --runningWorkers
                if (queue.size == 0) return
                ++runningWorkers
                fairnessCounter = 0
            }
        }
    }
}
```

**由上可以看出Dispatchers.IO 任务分发是借助于DefaultScheduler，也就是Dispatchers.Default的能力，因此两者是共用一个线程池。
 只是Dispatchers.IO 比较特殊，它有个队列，该队列作用：
 当IO 任务分派个数超过设定的并行数时，不会直接进行分发，而是先存放在队列里。**



### 组合上下文元素

如果需要为协程定义多个元素，则可以使用`+`运算符进行合并。比如同时设置Job、调度器、协程名称

```kotlin
val context = Job() + Dispatchers.Main + CoroutineName("name")
```

> 内部重写了`plus`操作符
>
> 如果有相同类型(Key相同)，则会替代旧元素



## CoroutineScope

kotlin协程的另一个重要组件就是`CoroutineScope`。

前面也提到过，`CoroutineScope`定义了协程所运行在的作用域，**所有协程都必须在作用域内启动**。

而可管理作用域内运行的任务，则通过调用 `scope.cancel()`来取消正在进行的任务

![CoroutineScope源码.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c821a899ab45478cb5e155dba8396301~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

官方提供了 [KTX 库](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzAwODY4OTk2Mg%3D%3D%26mid%3D2652045796%26idx%3D1%26sn%3D5c80872cd652a0ffd35e4891ea0984d9%26scene%3D21%23wechat_redirect)在一些类的生命周期里提供了`CoroutineScope`，比如 在ViewModel内的`viewModelScope` 和 LifecycleOwner的`lifecycleScope`，都会在各自生命周期结束时取消作用域内的协程。

此外还有个全局协程作用域`GlobalScope`，但**不推荐使用**，不会继承外部作用域（即永远是顶级协程作用域）且无法自定义`CoroutineContext`

> 如果需要全局的驻留任务，最好还是在`Application`内构建自定义的`CoroutineScope`。
>
> 参考：[协程中的取消和异常 | 驻留任务详解](https://juejin.cn/post/6883730571848056845)



### 作用域层级

在`CoroutineScope`中可以创建协程，而在**协程体**内也隐含了当前协程所处的`CorroutinScope`。

```kotlin
val scope = CoroutineScope(Job() + Dispatchers.Main)
 
 val job = scope.launch {
     // 新的协程会将 CoroutineScope 作为父级
     ...
     val result = launch {
         // launch创建的新协程会将当前协程作为父级
         ...
     }
 }
```

以最开始的`CoroutineScope`为根节点层级，在哪个`CoroutineScope`中创建新协程，就是新的协程的父级。(图片来自官方) ![CoroutineContext图解.webp](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c5b1cff902f47c2bc945190c7eb45a4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

但使用最根节点`CoroutinesScop`的创建的新协程的`CoroutineContext`实际是

> 新的 CoroutineContext = 父级 CoroutineContext + 新context （默认只创建Job）
>
> 父级 `CoroutineContext` 里的 **Job** 是 scope 对象的 **Job (红色)** ，
>
> 而新的 **Job 实例 (绿色)** 会赋值给新的协程的 `CoroutineContext`。
>
> **在新协程的范围内，会覆盖父级CoroutineContext的Job对象。**

父协程只能在**全部子协程执行完成后**才会进入完成状态，即使父协程本身的任务已经执行完成。



### 作用域分类

- **顶级作用域** ：

  > 没有父协程的协程所在的作用域称之为顶级作用域。

- **协同作用域** ：

  > 在协程中启动一个协程，新协程为所在协程的子协程。子协程所在的作用域默认为协同作用域。此时子协程抛出未捕获的异常时，会将异常传递给父协程处理，如果父协程被取消，则所有子协程同时也会被取消。

- **主从作用域** ：

  > 与协同作用域在协程的父子关系上一致，区别在于，处于该作用域下的协程出现未捕获的异常时，不会将异常向上传递给父协程。

官方还提供了两种在协程内部创建作用域的API：

- **coroutineScope**

  `coroutineScope`是顶层suspend函数，创建一个新的**协程作用域**，并调用指定的协程代码块，等待内部协程结束后再结束作用域，属于**协同作用域**。

  ![顶层函数coroutineScope源码.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be5d8f019ae549da830244a99c5c13cb~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **supervisorScope**

  `supervisorScope`是顶层suspend函数，与`coroutineScope`的区别就是协程作用域在取消\异常不会自动传递到父协程层级，属于**主从作用域**。

  ![顶层函数supervisorScope源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b934cf7efc04c399211fc6db295fe9c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 协程启动模式

在`launch`和`async`函数的start参数中，允许接收枚举类`CoroutineStart`。

- **DEFAULT**

  创建协程后 立即调度执行，调度前如果被取消，直接进入取消响应的状态，**有可能在执行前被取消**。

- **ATOMIC**

  创建协程后，立即调度执行，协程执行到第一个挂起点之前，不响应取消，**协程一定会被执行（执行途中可能会被取消）** 。

- **LAZY**

  如果调度前被取消了，直接进入异常结束状态，**且不调用start、await等方法是不会执行的**。

- **UNDISPATCHED**

  协程在这种模式下会直接开始在当前线程下执行，直到运行到第一个挂起点。这听起来有点像 `ATOMIC`，不同之处在于`UNDISPATCHED`是不经过任何调度器就开始执行的。当然遇到挂起点之后的执行，将取决于挂起点本身的逻辑和协程上下文中的调度器。

还记得`launch`和`async`内部调用的`Coroutine.start`方法吗？

其内部最终会调用到`CoroutineStart`的`invoke`方法

![Coroutine的start源码解析.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed29f18781874166be9173550590f57b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

以默认的`DEFAULT`模式为例，调用`startCoroutineCancellable`方法来启动协程

![startCoroutineCancellable源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/177cf2e9b97e4af4a14abf99dcfd7faf~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

是不是很眼熟？其实就是对于标准协程的拓展封装，其内部依然是围绕`Continuation`来启动协程。

> 更多关于启动模式，参见[破解 Kotlin协程(2) - 协程启动篇](https://link.juejin.cn?target=https%3A%2F%2Fwww.bennyhuo.com%2F2019%2F04%2F08%2Fcoroutines-start-mode%2F)

## 协程取消

前面提到`CoroutineScope`能统一管理作用域内的协程，最终是协程上下文的`job`对象，调用`cancel`方法来取消协程。

```kotlin
//Job.kt
 public fun cancel(cause: CancellationException? = null)
```

该方法会使协程抛出一个特殊的`CancellationException`异常来结束协程操作。

> 同时作为参数`cause`可以传递指定结束原因，默认为null，使用默认的异常`defaultCancellationException`
>
> ```kotlin
> //JobSupport.kt
>  internal inline fun defaultCancellationException(
>   message: String? = null,
>   cause: Throwable? = null
>  ) = JobCancellationException(
>   message ?: cancellationExceptionMessage(),
>   cause, this
>  )
>  protected open fun cancellationExceptionMessage(): String 
>      = "Job was cancelled"
>  
> ```

协程在取消时会轮询，**所有子协程都将被取消**

**而被取消的协程作用域是不能再创建新协程的**

> 被取消的协程并不会影响到**相同层级**的其他协程

```kotlin
val scopeJob = CoroutineScope().launch{
     val job1 = launch{
         ...
     }
     val job2 = launch{
         ...
     }
     //只会取消job1协程
     job1.cancel()
 }
 ...
 //会取消作用域内的所有协程
 scopeJob.cancel()
```



### 协程不一定取消

虽说调用`cancel`方法会使协程进入取消流程。

但就与线程中执行`Runnable`类似，在协程开始运行后，取消协程并不意味着协程执行的任务也会随之停止。

```kotlin
fun test() = runBlocking{
     val startTime = System.currentTimeMillis()
     val job = launch(Dispatchers.Default) {
         var nextPrintTime = startTime
         var i = 0
         while (i <= 5){
             if (System.currentTimeMillis() >= nextPrintTime){
                 println("job running print ${i++}")
                 nextPrintTime += 500
             }
         }
     }
     //程序执行1s
     delay(1000)
     println("coroutine scope delay done,wait cancel job")
     job.cancel()
     println("job canceled")
 }
 
 job running print 0
 job running print 1
 job running print 2
 coroutine scope delay done,wait cancel job
 job canceled
 job running print 3
 job running print 4
 job running print 5
```

就像上面的代码，`cancel`执行完成后，协程依然在运行。

如果需要让上面的代码按预期执行，则需要在协程体中，定时检查协程是否已被取消。

- **isActive**

  还记得`Job`提供的`isActive`属性吗？可以通过该属性检查协程是否处于活跃状态。

  同时官方还提供了`CoroutineScope`的拓展函数可以检查协程活跃状态

  ![isActive源码.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d1a4b6bb16e4f5e9c97900d4b6ab7ff~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

  另外`Job`还提供了更方便的拓展方法。

  ![ensureActive源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d54d522e86149fbb58d1d91bb3e8df8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **yield**

  这个函数会挂起当前协程，让出线程资源去执行其他协程任务。

  其内部首先会调用`ensureActive`方法检查协程的活跃状态

  ![yield源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3d6c007e5c04dd7afa6b642242fb20c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

- **join**

  `Jop.join`方法会**挂起当前协程**，等待协程执行完成，才执行后续任务

  > - 如果在调用`cancel`方法后再调用`join`方法，协程会处于**挂起**直到协程执行完成
  > - 如果先调用`join`再调用`cancel`，则不会产生影响，因为`join`执行后，协程就已经结束了。

- **await**

  `Deferred.await`方法**挂起当前协程**，等待协程执行完，并返回协程结果内容。

  > - 如果在调用`cancel`方法后，再调用`await`方法，会抛出`CancellationException`异常（表示正常结束），结束协程。
  > - 如果先调用`await`再调用`cancel`，则不会产生影响，因为`await`执行后，协程就已经结束了。

**协程的cancel能否成功，仅仅取决于是否在协程体中加入了检查点，比如 `isActive` 、`yield`、`delay`等， 如果协程没有加入检查点，那么`cancel` 一定是无效的**。



### 释放资源

在协程被结束后，可以在`finally`代码块中，执行清理资源操作

```kotlin
fun test() = runBlocking{
     val job = launch{
         try{
             //执行任务
             doSomeWork()
         }finally{
             //在协程结束后，执行的操作
             doCleanWork()
         }
     }
     println("协程已结束")
 }
```

但由于`finally`代码块是在协程结束后才会执行的，此时不能继续挂起协程。

此时如果还需要挂起，则可以指定context 为`NonCancellable`。

```kotlin
finally{
     withContext(NonCancellable){
         println("job NonCancellable suspend finally")
         doCleanWork()
     }    
 }    
```

> 注意不能滥用`NonCancellable`，这会导致协程无法被取消，造成内存泄漏。
>
> 最好只用于处理一些资源回收操作



### 拓展

还记得在前面提到转换回调API的`suspendCancellableCoroutine`函数吗？该函数的`block`参数类型是`(CancellableContinuation<T>) -> Unit`。

作为`Continuation`子类的`CancellableContinuation`，其内部提供了`invokeOnCancellation`函数，可以在**协程被取消（或出现异常）** 时，执行一些**资源回收**操作。

![CancellableContinuation源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae6fa20b19794dfbae25cbe9bbe2530f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

比如在`Retrofit`中，就是在`invokeOnCancellation`中，协程结束时尝试结束`Call`。

![Call拓展函数await源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/30e4981c8717404b9f07179280570b77~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)



## 协程异常传递

默认情况下，调用`launch`、或者`async`方法，会默认使用`Job`来创建协程。

而在协程内抛出异常结束时，且协程内部未捕获，会抛出异常给父协程，同时**父协程内的所有子协程都将被取消**，并且进一步向上一层级传递，并最终将**异常传递给顶层的作用域**

官方的示例图很形象的展示了这一过程

![协程异常传递](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a359c318b264bf7af59f20da70b37fe~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> 1. 取消它自己的子级；
> 2. 取消它自己；
> 3. 将异常传递给它的父级。

```kotlin
CoroutineScope().launch{
     ...
     lauch(CoroutineName("coroutine_1")){
         //协程1出现异常
         throw Exception("...")
     }
     
     launch(CoroutineName("coroutine_2")){
         //协程2在异常出现后，会被取消
         ...
     }  
 }
```

而传递给顶层作用域`CoroutineScope`，将会把作用域内所有协程全部取消。

![JobSupport源码解析.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/76073f048aba4dffb4f0a0b476df80f1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> 协程作用域被取消后，无法继续开启新协程

### SupervisorJob

如果我们不想因为一个任务的失败而影响其他任务，子协程运行失败不影响其他子协程和父协程，在协程创建时可以使用`SupervisorJob`

![SupervisorJob源码解析.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1399f025117c4e62bd9537ebace797ea~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

作为`Job`的子类，其重写了`chileCanceled`方法为false，表示**不会传递异常到父类协程**，只会在协程内部处理（结束当前协程），并结束内部子协程。

```kotlin
lifecycleScope.launch(SupervisorJob()){
     ...
 }
```

> 当子协程任务出错或失败时，`SupervisorJob` 只会取消它和它自己的子级，也不会传播异常给它的父级，它会让子协程自己处理异常

而使用挂起函数`supervisorScope`创建的协程作用域，同样有`SupervisorJob`的作用。

![supervisorScope源码解析.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ead0547502004dcd8d46743939ad6a73~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)



### 异常捕获处理

但不论是`Job`还是`SupervisorJob`，只要**内部没有捕获的异常都会被抛出**，最终导致程序崩溃。

唯有**协程取消时抛出的`CancellationException`异常是会被忽略的**，这对于协程而言就意味着协程是**正常结束或者退出**。

- **launch**

  对于`launch`创建的协程，异常会在第一时间被抛出，可以直接使用`try catch`来捕获。

  ```kotlin
  fun test() = runBlocking{
       launch(CoroutineName("coroutine_1")){
           try{
               throw NullPointException("...")
           }catch(e : Exception){
               //直接捕获异常
           }
       }
   }
  ```

  但对于父协程内部开启的子协程，在父协程如果使用默认`Job`创建的子协程，会在子协程抛出异常后，**直接将异常抛出到最顶层作用域**，内部无法拦截。

  ```javascript
  fun test() = runBlocking{
       launch(CoroutineName("coroutine_1")){
           try{
               launch(CoroutineName("inner_coroutine_1")){
                   throw NullPointerException("...")
               }
           }catch(e : Exception){
               //无法捕获内部协程异常，会直接越过当前协程范围传播到父级作用域
           }
       }
   }
  ```

  > 如果**coroutine_1**使用`SupervisorJob`或者`supervisorScope`，则**异常会被拦截在coroutine_1范围内**，不会延伸传播到`runBlocking`，只会在协程内部处理。

- **async**

  对于`async`创建的协程，若协程内没有捕获异常

  如果没有调用`await`，外部`try-catch`是无法捕获到异常。

  ```kotlin
  fun test() = runBlocking{
       try{
           val task = async{
               throw NullPointerException("...")
           }
       }catch(e : Exception){
           //无法捕获到异常
       }
   }
  ```

  而在调用了`await`后，**async代码块本身是不会抛出异常**，**`await`本身会抛出异常**。

  ```kotlin
  fun test() = runBlocking{
       val task = async{
           throw NullPointerException("...")
       }
       try{
           task.await()
       }catch(e : Exception){
           //能够捕获到异常
       }
   }
  ```

### CoroutineExceptionHandler

在线程中有`Thread.setDefaultUncaughtExceptionHandler` 方法来处理未知异常。Kotlin协程中同样有相同的机制。

`CoroutineExceptionHandler`作为`CoroutineContext`中的一种，可以添加到`CoroutineContext`中，处理协程中未捕获的异常。

- 仅针对`launch`创建协程**自动抛出的异常生效**。
- 对于`async`创建协程是不会有任何效果，无法捕获。

> `CoroutineExceptionHandler`是兜底机制，无法从异常中恢复协程！
>
> 通常用作记录异常、资源回收等操作。

对于默认的`Job`而言，在子协程内设置`CoroutineExceptionHandler`是无意义的，异常会被向上传递到父协程，由设置的`CoroutineExceptionHandler`处理，如果没有设置，则继续向上传递，根作用域。

```kotlin
fun test() = runBlocking {
     val scope = CoroutineScope(Job())
     
     val handler = CoroutineExceptionHandler { coroutineContext, throwable ->
         println("handler cast $throwable")
     }
     
     scope.launch(handler){  //只有作用域的根协程，能拦截处理异常
         
         launch(handler){    //子协程无法拦截
             throw NullPointerException("...")
         }
         
         async(handler){    //async抛出的异常无法被handler捕获
             throw NullPointerException("...")
         }
     }
 }
```

如果内部子协程使用`SupervisorJob`，则异常就不会向上传递，直接在内部协程的`CoroutineExceptionHandler`捕获处理

```kotlin
scope.launch(handler){
     launch(SupervisorJob()+handler){    //拦截在子协程内部的handler
         throw NullPointerException("...")
     }
 }
```

而使用`supervisorScope`创建的作用域，也有相同的作用。作用域根协程不会将异常向上传递，而是在协程内部的`CoroutineExceptionHandler`捕获处理

```kotlin
supervisorScope{
     launch(handler){ //在协程内部handler捕获
         throw NullPointerException("...")
     }
 }
```

# 总结

Kotlin协程并没有什么神奇的“魔法”。

> 在基于JVM的Android平台上，本质上还是通过**协程调度器**把协程任务**挂起**，将**阻塞**转移到另一个线程，然后在协程任务执行完成后**恢复**，**回调**到原来的线程。

只是Kotlin协程将实现细节隐藏在框架内部与编译器字节码中，把它”拍平“了，这才显得能直接把异步任务写成同步的方式。

使用`CoroutineScope`的`launch`和`async`来启动协程，便于在`CoroutineScope`统一管理内部协程。

- `launch` 适合启动外部不需要返回值的协程
- `async` 适合启动外部需要返回值的协程

在`CoroutineContext`中设置`Dispatcher`来让协程运行在指定协程调度器（线程）。

可通过`withContext`来切换协程内代码块所运行的协程调度器（线程）。

在Android中可以使用拓展库`lifecycleScope`和`viewModelScope`作用域管理协程。

- 作用域（父级协程）取消（异常）时，会取消所有子协程
- 作用域取消后无法创建新协程
- 父级协程需等待所有子协程执行完才能完成
- 默认情况下，子协程未捕获的异常会传递到父协程。

对于异步任务，内部可以使用`isActive`、`yield`检查协程状态，使其能够被取消。

对于回调API转化的协程，最好使用`suspendCancellableCoroutine`来创建可取消的协程（内部会检查协程状态）。

而当我们不希望协程出现异常时，自动传递到父级协程（无法拦截），造成同层级的其他协程被取消，可以在`CoroutineContext`中设置`SupervisorJob`，或者使用`supervisorScope`创建子协程作用域，将异常拦截在协程体内部。

最后，可以给`CoroutineContext`设置`CoroutineExceptionHandler`来作为最后的异常拦截器，处理一些出现异常后的资源回收操作。





## 相关引用原文

[扒一扒Kotlin协程](https://juejin.cn/post/7027331505336614948) 作者：yuPFeG1819 

[Kotlin Jetpack 实战 | 09. 图解协程原理](https://juejin.cn/post/6883652600462327821) 作者：朱涛的自习室

[庖丁解牛，一文搞懂Kotlin协程的挂起和恢复](https://www.jianshu.com/p/925f558a0d30) 作者：Android程序员老鸦