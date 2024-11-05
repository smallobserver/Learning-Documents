# Android 复杂项目崩溃率收敛至0.01%实践

引用原文地址：https://juejin.cn/post/7377200392059617295 作者：半山居士

看到写的很好，做一个收藏学习，方便日后回看

## 一、崩溃收敛机制

### 1、创建修BUG分支

在我们的项目中，每个版本发布之后，我们会创建一个opt分支，用于修复线上崩溃以及业务逻辑BUG。

开发过程中，一个APP可能同时并行开发多个需求，每个需求上线的预期时间可能会有不同。但是这个opt分支我们会保证在下个版本一定上线，QA同学也会在每个版本发布前预留测试opt分支的时间。

### 2、每天早晨查看Dump后台

每天上班第一件事就是查看DUMP后台，收集昨天线上发生的DUMP崩溃，具体的堆栈分配给对应的业务负责人。

业务负责人收到崩溃之后，会优先跟进排查。排查下来如果相对好修复，会第一时间直接修复掉，并提交到opt分支。如果排查下来发现，较难定位或者耗时较久，则需要给出修复预期。也可以将Bug转为技术优化，作为专项推进。因为确实有一些Bug需要通盘考虑，所有业务配合。

## 二、崩溃容灾机制

### 1、背景

我们为什么要开发一套崩溃容灾逻辑？

在对线上崩溃进行收敛时，我们发现线上有几类崩溃是我们在应用无法修复的。

#### 例如：案例一

```csharp
java.lang.NullPointerException: Attempt to invoke virtual method 'boolean android.content.ClipDescription.hasMimeType(java.lang.String)' on a null object reference
    at android.widget.TextView.canPasteAsPlainText(TextView.java:15065)
    at android.widget.Editor$TextActionModeCallback.populateMenuWithItems(Editor.java:4692)
    at android.widget.Editor$TextActionModeCallback.onCreateActionMode(Editor.java:4627)
```

#### 案例二：小米手机上出现

```csharp
java.lang.NullPointerException:Attempt to invoke virtual method 'int android.text.Layout.getLineForOffset(int)' on a null object reference
```

#### 案例三：集成华为推送SDK后，偶现

```css
java.lang.RuntimeException:Unable to start activity ComponentInfo{com.netease.popo/com.huawei.hms.activity.BridgeActivity}:
android.util.AndroidRuntimeException: requestFeature() must be called before adding content"
```

#### 案例四：BadTokenException

```csh
android.view.WindowManager$BadTokenException:Unable to add window -- token android.os.BinderProxy
```

我们大致将以上问题划分为四类：

- 我们认为是系统异常，应用层仅能在使用的位置try cache，有些崩溃甚至无处try cache；
- 排查下来发现仅在某个厂商的手机上出现；
- 集成的一些第三方SDK所引入，依赖对方修复，时间上不好掌控；
- 由于Android系统的一些机制引发的崩溃，如弹出弹框时，恰好依赖的Actity正在销毁。业务层希望弹框可以不弹出但不要崩溃，可是系统最终是抛出来一个BadTokenException。我们可以在使用DiaLog时做判断，但是总会用有同学忘记。

基于以上我们思考是否可以开发一个框架，将这些崩溃统计进行拦截，使其不影响用户的使用。

### 2、技术方案

作为Android开发应该都比较清楚Handler机制。我们的崩溃容灾主要是利用了Handler机制。 具体的逻辑图如下：

![崩溃容灾技术方案](.\res\崩溃容灾技术方案.png)

- 应用启动后，初始化崩溃白名单（应用内置，也支持服务端动态下发）
- 通过Handler#post()方法,向主线程中发送一条消息。
- 在Runnable#run()方法中，执行一个死循环逻辑
- 死循环中逻辑中使用try cache将Looper.loop()防护
- 这样只要应用的进程不结束，相当于任务一直执行在我们前面post的消息中
- 只是我们在这个消息中，再次执行了Looper.loop()方法，执行后续消息队列中所有的消息
- 一旦后续所有消息遇到崩溃，会先被try cache捕获。
- 然后判断崩溃信息是否在我们的白名单中，一旦在白名单中直接捕获掉，不向外抛异常，逻辑会回到外部的死循环中，继续执行Looper.loop()方法获取后续的消息。这样就保证了逻辑的连贯，后续的事件可以继续处理。
- 不再我们的白名单中则继续将这个异常throw出去。

### 3、现状

崩溃拦截框架上线至今几年的时间，积累的崩溃种类目前已经达到81种。

比较典型的除了上面介绍的几类崩溃以外，还有如我们在适配Android 12的SplasScree时，遇到的TransferSplashScreenViewStateItem 相关的错误

```php
php 代码解读复制代码java.lang.IllegalArgumentException: Activity client record must not be null to execute transaction item: android.app.servertransaction.TransferSplashScreenViewStateItem@de845fa
at android.app.servertransaction.ActivityTransactionItem.getActivityClientRecord(ActivityTransactionItem.java:85)
at android.app.servertransaction.ActivityTransactionItem.getActivityClientRecord(ActivityTransactionItem.java:58)
at android.app.servertransaction.ActivityTransactionItem.execute(ActivityTransactionItem.java:43)
at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:149)
at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:103)
at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2708)
at android.os.Handler.dispatchMessage(Handler.java:114)
at android.os.Looper.loopOnce(Looper.java:206)
at android.os.Looper.loop(Looper.java:296)
```

按照过往的手段，我们仅能等待Google官方来修复这个错误，或者先下线掉这个错误。本框架可以直接将其进行拦截，同时用户无感知。

## 三、其他崩溃收敛

我们针对自身业务特点，大致梳理以下几种业务中非常常见的崩溃场景，提醒每一位同学注意。同时内部维护了一个研发质量表，每个需求提测时我们会过一遍研发质量表格，提醒同学注意相关代码质量与性能。

### 1、空指针问题

NPE应该是最常见的问题了。针对NPE的问题，我们的解决方式有：

- 推荐组内所有同学习惯使用注解@NonNull和@Nullable
- 推荐大家使用Kotlin；并且在Java调用kotlin方法时，一定要注意Kotlin方法是否要求入参不为空
- 从List、Map中取到的对象，使用前必须判空
- 业务代码需要将Context传给第三方工具类，传入之前必须判空
- 对象建议声明为final或者val，防止后续其他位置置空引起NPE
- 外接传入的对象使用前必须判空
- 基于AS插件进行检测

### 2、IndexOutBoundsException

角标越界异常在平时开发中也特别常见，在我们的业务中常见于集合以及Span操作

- 集合传入index时需要判断是否在[0, size]内
- 操作Spannable接口setSpan方法时，需要start与end的数值不会超过长度，同时不能够为负数

### 3、ConcurrentModificationException 并发修改异常

并发修改异常在复杂的业务中，是非常容易遇到的。通常有两个场景容易触发，分别是foreach循环中直接调用remove方法移除元素，以及线程不安全环境下使用线程不安全集合。

针对并发修改异常：

- 我们推荐在遍历集合时，可以new一个新的List集合，将要遍历的List集合作为参数传入，然后遍历新的集合。这样原集合在遍历时改变也不会抛异常
- 使用线程安全的集合，如CopyOnWriteArraylist、ConcurrentHashMap等

### 4、系统服务（FrameWork API）

- 调用系统服务通常需要跨进程通信，其内部很可能会抛异常，所有调用系统服务的地方都必须使用try cache。cache异常必须写入日志文件，根据业务重要性判断是否需要上报埋点数据；
- 系统服务频繁调用时可能会引发ANR，这点也需要特别注意；

### 5、数据库类问题

由于我们的业务重度依赖数据库，所以数据库相关的问题占比也比较高。 主要有以下几类问题：

#### CursorWindowAllocationException 2048问题：

```sql
com.tencent.wcdb.CursorWindowAllocationException: 
Cursor window allocation of 2048 kb failed. total:8159,active:49
at com.tencent.wcdb.CursorWindow.<init>(SourceFile:127)
```

针对CursorWindowAllocationException，在我们的工程中主要是短时间内大量的内存申请。 解决方案是基于SQL监控，统计工程中SQL执行的数量，基于SQL语句针对性的优化相关逻辑，将SQL语句执行数量降低了90%以上，这个问题线上不再复现。

#### 存储空间不足

```css
Caused by: 
com.tencent.wcdb.database.SQLiteFullException:database or disk is full (code 13,errno 0): 
    at com.tencent.wcdb.database.SQLiteConnection.nativeExecute(Native Method)
    at com.tencent.wcdb.database.SQLiteConnection.execute(SourceFile:728)
    at com.tencent.wcdb.database.SQLiteSession.endTransactionUnchecked(SourceFile:436)
    at com.tencent.wcdb.database.SQLiteSession.endTransaction(SourceFile:400)
    at com.tencent.wcdb.database.SQLiteDatabase.endTransaction(SourceFile:533)
    at com.tencent.wcdb.room.db.WCDBDatabase.endTransaction(SourceFile:100)
```

针对存储空间不足，在我们的APP中主要是添加手机存储空间检测，当空间不足时引导用户清理。

### 数据库损坏

```css
com tencent wcdb database.SQLiteDatabaseCorruptException: database disk image is malformed (code 11, errno 0): 
at com.tencent.wcdb.database.SQLiteConnection.nativePrepareStatement(Native Method)
at com.tencent.wcdb.database.SQLiteConnection.acquirePreparedStatement(SQLiteConnection.java:1004)
at com,tencent.wcdb.database.SQLiteConnection.executeForString(SQLiteConnection.java:807)
at com.tencent.wcdb.database.SQLiteConnection.setJournalMode(SQLiteConnection.java:424)
at com.tencent.wcdb.database.SQLiteConnection.setWalModeFromConfiguration(SQLiteConnection.java:414)
at com.tencent.wcdb.database.SQLiteConnection.open(SQLiteConnection.java:289)
at com.tencent.wcdb.database.SQLiteConnection.open(SQLiteConnection.java:254)
at com,tencent.wcdb.database.SQLiteConnectionPool.openConnectionLocked(SQLiteConnectionPool.java:603)
at com.tencent.wcdb.database.SQLiteConnectionPool.open(SQLiteConnectionPool.java:225)
at com.tencent.wcdb.database.SQLiteConnectionPool.open(SQLiteConnectionPool.java:217)
at com.tencent.wcdb.database.SQLiteDatabase.openInner(SQLiteDatabase.java:1002)
```

数据库损坏我们是引入了修复工具进行修复，同时对数据库损坏崩溃进行拦截。修复完成后退出到登录页面引导用户重新登录。

数据库的崩溃问题一度在我们的工程中占比超过50%，所以我们有启动数据库优化专项投入大量时间。针对数据库优化的具体介绍可以查看：[Android 数据库系列三：复杂项目SQL治理与数据库的优化总结 ](https://juejin.cn/post/7338306790173704192/)，内部有更加详细的介绍。

## 四、OOM问题收敛

### 1、OOM介绍

OOM的问题在Android中也是非常常见了，所以这里单独拎出来说说。 OOM产生的条件：待申请的内存大于系统分配给应用的剩余内存。 OOM原因大致可以归为以下几类

- 堆内存分配失败
  - 堆内存溢出
  - 没有足够的连续内存空间
- 创建线程失败（pthread_create (1040KB stack) failed: Try again）
- FD数量超出限制
- Native虚拟内存OOM

### 2、内存泄露监控

线上内存泄露的监控我们是使用的快手的KOOM。 KOOM原理这里笔者就不详解了，社区内也有专门分析的文章，大家可以找找看，不过还是建议去读读源码，写的挺不错的。

> 指路地址：[github.com/KwaiAppTeam…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FKwaiAppTeam%2FKOOM)

将KOOM分析的报告上报到我们的后台中，有专门的同学每周会排时间跟进。

### 3、全局浮窗实时显示APP当前总体内存

除了线上的监控，我们也有一个自研的开发者工具。工具有一个浮窗功能，我们会在浮窗上实时显示当前应用的内存信息（每秒采集一次）。数据主要是通过获取Debug.MemoryInfo#getTotalPss()。与Android Studio Profile中Memory数据基本一致。同时在UI层面我们还会设置一个阈值，超过阈值就会将浮窗中内存的数值颜色改为红色，旨在提醒开发同学关注内存变化。

通过实时显示内存我们在开发过程中，就可以发现一些问题。如我们再进入某一个业务时，发现内存会固定涨50M+，基于此开发同学去排查，发现了很多优化点。线下发现这个问题的意义就是可以质量左移，避免到线上影响到用户。

### 4、线程数量监控与收敛

获取线程数量，我们可以读取文件/proc/[pid]/status 中的线程数量，代码大致如下：

```java
public static String readThreadStatus(String pid){
    RandomAccessFile reader2= null;
    try {
        reader2 = new RandomAccessFile("/proc/" + pid + "/status", "r");
        String str;
        while ((str = reader2.readLine()) != null) {
            if (str.contains("Threads")) {
                return str;
            }
        }
        reader2.close();
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            if (reader2 != null) {
                reader2.close();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    return "";
}
```

在前面提到的开发者工具的浮窗中，我们也有一行用来实时显示线程的数量。不过在我们的工具中，我们没有使用上面的方法，而是使用：Thread.getAllStackTraces();

使用Thread.getAllStackTraces();获取的数量比 /proc/[pid]/status 获取的少，但是在我们的工程中，我们主要关注Java线程而且通过Java线程的数量的波动也能观察到App当中线程的变化。而且Thread.getAllStackTraces()会返回Thread对象，以及堆栈数据这个对我们更加有用。

在开发者工具我们有一个单独的页面可以实时查看线程的ID、名称以及对应堆栈。在线上我们会间隔一段收集一次线程数据上报到我们的后台中。

在笔者过往的开发经历中，遇到过一次由于线程数量较多直接导致应用崩溃的情况，即某个独立业务使用OkHttp没有创建OkHttpClient单例对象，而是每次接口回调都创建一个新的client...

定位过程比较简单，在特定的场景下，可以看到浮窗中的线程数量基本处于线性增长，通过开发者工具查看线程列表可以直接看到非常多的OkHttp相关的线程。

### 5、FD 数量监控

获取FD数量大致可以通过以下代码

```java
public static int getCurrentFdSize() {
    int size = 0;
    File dir = new File("/proc/self/fd");
    try {
        File[] fds = dir.listFiles();
        if (fds != null) {
            size = fds.length;
            for (File fd : fds) {
                if (Build.VERSION.SDK_INT >= 21) {
                    MLog.d("message", Os.readlink(fd.getAbsolutePath()));
                }
            }
        }

    } catch (Exception e) {
        e.printStackTrace();
    }
    return size;
}
```

同时在KOOM中每次分析的结果中会携带所有FD句柄信息，所以我们没有单独做额外的监控了，直接查看KOOM的解析数据。

笔者仅遇到过一次由于FD句柄超限导致的异常。异常信息如下

```less
java.lang.RuntimeException: Could not read input channel file descriptors from parcel.
at android.view.InputChannel.nativeReadFromParcel(Native Method)
at android.view.InputChannel.readFromParcel(InputChannel.java:148)
at android.view.InputChannel$1.createFromParcel(InputChannel.java:39)at android.view.InputChannel$1.createFromParcel(InputChannel.java:37)
at com.android.internal.view.InputBindResult.<init>(InputBindResult.java:68)
at com.android.internal.view.InputBindResult$1.createFromParcel(InputBindResult.java:112)at com.android.internal.view.InputBindResult$1.createFromParcel(InputBindResult.java:110)at com.android.internal.view.IInputMethodManager$Stub$Proxy.startInputOrWindowGainedFocus(IInp
at android.view.inputmethod.InputMethodManager.startInputInner(InputMethodManager.java:1361)
at android.view.inputmethod.InputMethodManager.onPostWindowFocus (InputMethodManager.java:1631)
at android.view.ViewRootImpl$ViewRootHandler.handleMessage(ViewRootImpl.java:4259)
at android.os.Handler.dispatchMessage(Handler.java:109)
at android.os.Looper.loop(Looper.java:166)
```

后面经过排查进入内置浏览器查看某网页，FD句柄的数量会瞬间飙升。通过遍历 /proc/pid/fd 文件发现大多是都是Socket。后面排查下来是应用内某个前端页面存在Bug，疯狂new Socket...

## 五、总结

以上简单介绍了一下我们在工程中如何针对各类崩溃信息进行收敛，值得欣喜的是经过几年的努力基本可以将崩溃控制在万一。回过头来看很多问题还是遇到问题-解决问题的思路，这就依赖我们的开发同学本身所写的代码质量要高，否则就会陷入到写Bug-改Bug这样的循环中。

所以我们也在积极探索如何通过前期的review，工具扫描的方式尽量降低线上问题的发生概率。尽可能将问题提前暴露。不过这方面目前还没有建设的特别好，人工review实验了一段时间发现在一个业务较多的团队的实施起来很难，没有时间不说盯着代码review也不见得就能发现一些逻辑上的异常。好在公司内其他团队再研究基于AI的代码扫描，后续计划接入到当项目中看看。

作者：半山居士
链接：https://juejin.cn/post/7377200392059617295
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

