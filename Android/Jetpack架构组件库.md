# Jetpack架构组件库



## 简单介绍

概括：Jetpack是众多优秀组件的集合。是谷歌推出一套引领Android开发者逐渐统一开发规范的架构。



#### 组成

##### （1）Architecture（架构） 

​        Data Binding（数据绑定）：属于支持库可使用声明式将布局中的界面组件绑定到应用中的数据源。
​        Lifecycles：管理和感知Activity和Fragment生命周期
​        LiveData：是一个可观察的数据持有者类。与常规observable不同，LiveData是有生命周期感知的。
​        Navigation：处理应用内导航所需的一切。
​        Paging：一次加载or按需加载&显示小块数据。
​        Room：帮助开发者更友好、流畅的访问SQLite数据库。
​        ViewModel：以生命周期感知的方式存储和管理与UI相关的数据。
​        WorkManager：调度预期将要运行的可延迟异步任务。（即便应用程序推出or重启）。



##### （2）Foundation（基础）

​        Android KTX：一组Kotlin扩展程序；优化了供Kotlin使用的Jetpack和Android API。以更简洁的方式使用Kotlin进行Android开发。
​        Appcompat：提供向后兼容的API。
​        Security：按照最佳的安全做法读取加密文件和共享偏好设置。
​        Multidex：当方法数超过64K时启用多dex文件。
​        Test：测试框架。



##### （3）Behavior（行为）

​        相机-CameraX：简化相机应用的开发工作，可向后兼容至Android5.0(API Level21）
​        下载-DownloadManager：可处理长时间运行的HTTP下载&超时重连。
​        多媒体-Media&playback：用于媒体播放&路由的向后兼容API。
​        通知-Notifications：提供向后兼容的通知API，支持Wear和Auto。
​        权限-Permissions：用于检查和请求应用权限的兼容性API。



##### （4）UI

​        动画-Animation&Transitions：提供各类内置动画，也可以自定义动画效果。
​        表情-Emoji：使用户在未更新系统版本的情况下也可以使用表情符号。
​        Fragment：组件化界面的基本单位。
​        布局-Layout：xml书写的界面布局或者使用Compose完成的界面。
​        调色板-Palette：从调色板中提取出有用的信息。





## DataBinding原理

android底层只识别xml，<layout><data>标签都是databinding使用的，在APT注解处理器在编译期处理，会讲其剔除，拆分成两个布局文件，还给android底层干净的xml。会给对应布局增加tag，binding_1,binding_2，activity的布局文件也会增加tag，如activity_main_0

@={}双向绑定

java 用注解成员和方法可以实现定义model

kotlin使用observerFiled

databinding绑定是，现通过activity获取doctorView拿到顶级布局。在编译生成BR文件包好组件id的时候，还会编译生成DataBinderMapperImpl，其正是绑定核心处理。还会生成对应的databinding.ActivityMainBindingImpl。对应的控件都会取出保存到数组，然后把根据控件id生成的成员对象挨个根据取出来的值赋值，并且清除设置的tag。

双向绑定是 给text或者edittext设置textwatcher监听数据 更改model，model修改后notify通知这边更新。





## Lifecycle架构组件原理



#### ComponentActivity如何实现Lifecycle 源码

![Lifecycle宿主生命周期与宿主状态关系图](.\res\Lifecycle宿主生命周期与宿主状态关系图.png) 

继承实现了LifecycleOwner接口，在成员变量中new了LifecycleRegistry对象，然后创建了一个ReportFragment，在ReportFragment中把自己添加到了activity中，且自身还是不可见的，在fragment中调用了生命周期的事件分发。设计目的是为了兼顾不是继承自AppCompactActivity的场景。因为他在LifecycleDispatcher中获取了context，通过context获取application的context监听activity create，且在里面手动添加了ReportFragment，只要你activity实现了lifecycleOwner接口，就会被监听生命周期。

注册的LifecycleObserver观察者，会绑定一个status（ObserverWithState包装，其中有dispatchEvent方法），如果和宿主的真实状态不符合，就会对齐，并回调前面对应的事件，比如onResume的时候注册，依旧会收到前面的create 和 start。后续变化的时候还会比较每个观察者是否对齐了状态，没有就会通知并对齐。

LifecycleObserver（根据注解来生成对应方法）、FulLifecycleObserver（默认要求实现全量接口）会通过适配器，转换成LifecycleEvnetOberver，LifecycleEvnetOberver中有onStateChanged方法。





## LiveData架构组件原理

通过Lifecycle实现对生命周期的感知，仅仅分发消息给处于活跃状态（即Observer所在宿主处于started、resumed状态）的观察者。

setValue主线程  postValue 主线程子线程都可以（底层使用handle post到主线程发布）

onActive 当且仅当只有一个活跃观察者的时候

inActive 不存在活跃观察者的时候

好处：

1. 页面不可见不会派发消息 
2. 页面可见立刻派发一条最新消息（粘性新注册的observer也能收到前面发的最后一条）
3. 不需要手动处理生命周期
4. 不会内存泄漏



#### 实现原理

当LiveData 注册观察调用 observer方法，会通过lifecycleOwner对象判断是否生命周期已经销毁，销毁就直接return。然后把owner和oberver一起包装成LifecycleBoundObserver类，此类是LifecycleEventObserver的实现类，然后把包装好的boundObserver存储到mObservers这个hashmap中，observer作为键，避免同一个observer对象和不同的owner进来存储，如果有就抛异常，存储好之后把LifecycleBoundObserver注册到owner的lifecycle.addObserver中。

lifecycleBoundObserver的onStateChange方法重写了，判断是否宿主owner是否destroy了，是就移除掉observer观察，然后判断是否是活跃状态。变化成活跃状态后，调用dispatchValue传入this

observerforever不用传入owner，不会关注生命周期，生成一个alwaysactiveobserver的子类，active会返回true。

livedata和observer都有一个version，livedata是mVersion，oberser有一个mLastVersion，每次完整发送前，会判断observer中的mLastVersion是否大于等于mVersion，如果大于等于说明发送过了，就跳过，小于就没发过，可能是新数据或者粘性，然后赋值给observer中。

post的时候，调用handler，然后回到主线程调用setValue，发送mVersion++，存下mData为最新数据，然后调用dispatchValue传入null，null的时候会遍历所有的observer。





## ViewModle结构组件原理

具备宿主生命周期感知能力的数据存储组件

ViewModel保存的数据，在页面因配置变更导致页面销毁重建之后依然也是存在

> [!NOTE]
>
> 配置变更：横竖屏切换、分辨率调整、权限变更、系统字体样式改变..

常规使用：配置变更导致页面销毁重建，复用ViewModel的实例。

  

进阶使用：各种原因内存回收、电量不足等等，都能复用。即使ViewModel实例不同，也能复用。需要引入savedState组件

页面恢复内存复用：

​	ViewModel中传入一个SavedStateHandle 优先从里面取值（HashMap）。然后才考虑远程请求，请求成功后存入SavedStateHandle。



跨Activity数据共享：

​	application中实现ViewModelStoreOwner，然后ViewProvider中取出viewmodel



ViewModelProvider构造方法需要ViewModelStoreOwner（activity默认实现），factory，owner会获取store。ViewModelStore实际上维护了一个HashMap。

provider的get方法，DEFAULT_KEY: 字符串拼接上canonicalName ，然后配合classs对象，从ViewModelStore中去get，如果有实例直接返回，没有就使用factory方法生成一个实例，使用上述生成的键存入ViewModelStore。

ComponentActivity中ViewModelStore存在NonConfigurationInstances对象中，此对象用于保存activty重构时一些需要保存的数据

如有些时候的fragment在页面旋转重建后还是那一个。onRetainNonConfigurationInstance（）以前用于页面销毁恢复时，保存数据的，通过lastNonConfigurationInstance对象获取保存数据，现在废弃，推荐使用ViewModel。使用lastNonConfiguration取回ViewModelStore，所以存储下来了。

![viewmodel保存逻辑](.\res\viewmodel保存逻辑.webp)



#### Activity销毁重建流程

ActivityThread正常启动页面走，handlelaunchActivity，重构页面会走handleRelaunchActivity，在这调用了handleReluanchActivityInner，然后会调用旧acitivty的onPause onStop，onStop中包含onSaveInstanceState，然后handleDstoryActivity，中调用onRetainNonConfigurationInstance。 本质上Activity中的ActivityClientRecord对象是使用Ibinder token来获取的，是同一个数据对象，里面存了销毁的activity部分对象。ActivityClientRecord是通过Ibinder从activitymanagerservice AMS跨进程获取的

IBinder跨进程通信



详细理解：

ActivityThread正常启动页面走，handlelaunchActivity，重构页面会走handleRelaunchActivity

ActivityThread在页面重构时调用handleRelaunchActivity方法，此时如果页面没有paused、stop就会依次调用，stop方法中会触发onSaveInstanceState，然后调用handleDestroyActivity方法，此方法为activity destory时调用，然后传入的getNonConfigInstance为true（正常销毁为false），然后调用performDestroyActivity，如果此时没有stop还是调用一遍，接着触发activity的retainNonConfigurationInstances，此方法会调用自身空实现的onRetainNonConfigurationInstance，而ComponentActivity将其实现后final，但是提供了onRetainCustomNonConfigurationInstance空实现方法，会把其中返回的object返回到NonConfigurationInstances.custom中，另一个就是viewModelStore，用于存放viewmodel的，最后将生成NonConfigurationInstances对象，放入ActivityClientRecord的lastNonConfigurationInstances，ActivityClientRecord在destory执行完后会清理掉activty等成员变量，然后传入handleLaunchActivity方法，启动新页面





#### SavedState

原理：本质上SaveState里有bundle，用于存储数据，而SaveState因为ComponentActivity实现的关系，可以在activity销毁、回收时进行数据存储，然后在重建页面时恢复数据。ViewModel配合使用就可以做到数据恢复。

SaveStateRegistryOwner：  核心接口，用来声明宿主，Activity 和 Fragment 都实现了这个接口，实现接口的同时必须返回一个 SaveStateRegistry，SaveStateRegistry 的创建委托给 SaveStateRegistryController 来完成。

SaveStateRegistryController：控制器，用于创建 SaveStateRegistry，与 Activity、Fragment 建立连接，为了剥离 SaveStateRegistry 与 Activity 的耦合关系。

SaveStateRegistry：核心类，数据存储恢复中心，用于存储、恢复一个 ViewModel 中的 bundle 数据，与宿主生命周期绑定。

SaveStateHandle： 核心类，一个 ViewModel 对应一个 SaveStateHandle，用于存储和恢复数据。真正存储的单元，里面有一个Map对象，以及SavedStateProvider，用于保存时将Map打包成bundle返回出去

SaveStateRegistry模型：一个总 Bundle，key-value 存储着每个 ViewModel 对应的子 bundle。有一个SafeIterableMap<String,SavedStateProvider> mComponents，在注册生成handle（reisterSavedStateProvider）的时候存下来，用于保存时生成bundle，在将每一个SaveStateHandle都打包成一个bundle存在送进来的bundle参数中。

![SavedState架构图.png](.\res\SavedState架构图.png) 

由于页面有可能存在多个 ViewModel，那么每个 ViewModel 当中的数据都会通过 SaveStateHandle 来存储，所以 SaveStateRegistry 的数据结构是一个总的 Bundle，key 对应着 ViewModel 的名称，value 就是每个 SaveStateHandle 保存的数据，这样做的目的是为整存整取。

因为 ViewModel 在创建的时候需要传递一个 SaveStateHandle，SaveStateHandle 又需要一个 Bundle 对象，这个 Bundle 可以从 Bundle mRestoredState 里面获取。它里面存储的 ViewModel 即便被销毁了，那么在 Activity 重建的时候也会复用的。


#### SavedState数据存储

下面进入源码，以 Activity 为例，Fragment 也是类似的。ComponentActivity 实现了 `SavedStateRegistryOwner` 接口，它是一个宿主，用来提供 `SavedStateRegistry` 这个对象的，这个对象就是存储中心：

```java
// 实现了ViewModelStoreOwner和HasDefaultViewModelProviderFactory接口
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory {

    @Override
    public final SavedStateRegistry getSavedStateRegistry() {
        // 委托给SavedStateRegistryController创建数据存储中心SavedStateRegistry
        return mSavedStateRegistryController.getSavedStateRegistry();
    }
}

```

实例的创建委托给 mSavedStateRegistryController，它是一个被 Activity 委托的对象，用来创建 SavedStateRegistry 的，目的是为了剥离 Activity/Fragment 这种宿主与数据存储中心的关系。

数据存储是在哪里呢，其实是在 onSaveInstanceState() 里面：

```java
@Override
protected void onSaveInstanceState(Bundle outState) {
    Lifecycle lifecycle = getLifecycle();
    if (lifecycle instanceof LifecycleRegistry) {
        ((LifecycleRegistry) lifecycle).setCurrentState(Lifecycle.State.CREATED);
    }
    super.onSaveInstanceState(outState);
    // 通过Controller直接将Bundle数据转发
    mSavedStateRegistryController.performSave(outState);
}

//通过 mSavedStateRegistryController.performSave(outState) 直接转发给SavedStateRegistry：
#SavedStateRegistry.java
    
// 以键值对的形式保存着SavedStateProvider
private SafeIterableMap<String, SavedStateProvider> mComponents = new SafeIterableMap<>();
@MainThread
// 数据保存
void performSave(Bundle outBundle) {
    Bundle components = new Bundle();
    if (mRestoredState != null) {
        // 将SavedStateHandle存储到components中
        components.putAll(mRestoredState);
    }
    // 将ViewModel中的数据存储到Bundle中
    for (Iterator<Map.Entry<String, SavedStateProvider>> it =
            mComponents.iteratorWithAdditions(); it.hasNext(); ) {
        Map.Entry<String, SavedStateProvider> entry1 = it.next();
        components.putBundle(entry1.getKey(), entry1.getValue().saveState());
    }
    // 将components存储到outBundle中
    outBundle.putBundle(SAVED_COMPONENTS_KEY, components);
}

//实际存储单元通过 重点，每个 SavedStateHandle 对象中都有一个 SavedStateProvider 对象，并且实现了 saveState() 方法，遍历 mRegular 集合，里面放的就是要缓存的键值对数据，然后打包成一个 Bundle 对象返回:
public final class SavedStateHandle {
    final Map<String, Object> mRegular;
    final Map<String, SavedStateProvider> mSavedStateProviders = new HashMap<>();
    private final Map<String, SavingStateLiveData<?>> mLiveDatas = new HashMap<>();

    // 每个SavedStateHandle对象中都有一个SavedStateProvider对象
    private final SavedStateProvider mSavedStateProvider = new SavedStateProvider() {
        // SavedStateRegistry保存数据时调用，将数据转为Bundel返回
        @Override 
        public Bundle saveState() {
            Map<String, SavedStateProvider> map = new HashMap<>(mSavedStateProviders);
            for (Map.Entry<String, SavedStateProvider> entry : map.entrySet()) {
                Bundle savedState = entry.getValue().saveState();
                set(entry.getKey(), savedState);
            }
            // 遍历mRegular集合，将当前缓存的Map数据转换为Bundle
            Set<String> keySet = mRegular.keySet();
            ArrayList keys = new ArrayList(keySet.size());
            ArrayList value = new ArrayList(keys.size());
            for (String key : keySet) {
                keys.add(key);
                value.add(mRegular.get(key));
            }

            Bundle res = new Bundle();
            // 序列化数据
            res.putParcelableArrayList("keys", keys);
            res.putParcelableArrayList("values", value);
            return res;
        }
    };




```



ComponentActivity的onSaveInstanceState中调用了SaveStateRegistryController的performSave方法，然后调用了SaveStateRegistry的performSave, 其中会将mComponents遍历，再将生成的多个bundle使用key-bundle存入一个新bundle中，放入参数bundle中。

此bundle参数为ActivityClientRecord中的state成员变量，会带入新的activity。

![SaveState数据存储流程.png](.\res\SaveState数据存储流程.png) 

Activity 在内存不足电量不足等系统原因，被回收的情况下，肯定会执行 onSaveInstanceState() 方法，Activity 紧接着就利用 SaveStateRegistryController 转发给 SaveStateRegistry，让它去完成数据存储的工作，SaveStateRegistry 在存储数据的时候会遍历所有注册的 SaveStateProvider 去存储自己的数据，并且返回一个 Bundle 对象，最后合并成一个总的 Bundle，存储到 Activity 的 savedSate 对象当中。




#### SavedState数据的恢复及复用

SavedState 数据复用流程分为两步：

第一步先从 Activity 的 savedState 恢复所有 ViewModel 的数据到 SaveStateRegistry。

需要到 ComponentActivity 的 `onCreate()` 方法中：

```java
#ComponentActivity.java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // 通过SavedStateRegistryController将bundle数据传递给SavedStateRegistry
    mSavedStateRegistryController.performRestore(savedInstanceState);
    //·····
}


//然后通过 SavedStateRegistryController 转发到 SavedStateRegistry 的performRestore()
#SavedStateRegistry.java
// 数据恢复会调用这个方法
void performRestore(Lifecycle lifecycle, Bundle savedState) {
    if (savedState != null) {
        // savedState中根据key获取数据Bundle数据，components对象
        mRestoredState = savedState.getBundle(SAVED_COMPONENTS_KEY);
    }

    lifecycle.addObserver(new GenericLifecycleObserver() {
        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (event == Lifecycle.Event.ON_START) {
                mAllowingSavingState = true;
            } else if (event == Lifecycle.Event.ON_STOP) {
                mAllowingSavingState = false;
            }
        }
    });

    mRestored = true;
}

```

**通过 savedState 取出刚才存储的 components 对象，并且赋值给 mRestoredState，数据的恢复是非常简单的，就是从 Activity 的 savedState 对象中取出前面存储的 Bundle 数据，并且赋值给 mRestoredState**。



第二步：需要去到 Activity 中:

```kotlin
class DemoViewModelActivity : BaseDataBindActivity<ActivityViewmodelBinding>() {
    override fun initView(savedInstanceState: Bundle?) {
        // 获取 SavedState 保存的数据
        val saveViewModel = ViewModelProvider(this).get(MainSaveViewModel::class.java)
    }
}

//假设 onCreate() 是因为被系统原因销毁了重建，才执行过来的：
// ViewModelProvider的构造方法
public ViewModelProvider(ViewModelStoreOwner owner) {
    this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
            ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
            : NewInstanceFactory.getInstance());
}

//owner.getViewModelStore() 这里不仅仅会获取 Activity 缓存里面的 ViewModelStore，还会判断宿主是否实现了 HasDefaultViewModelProviderFactory 接口，ComponentActivity 中是已经实现了该接口的：
// 实现了ViewModelStoreOwner和HasDefaultViewModelProviderFactory接口
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory {

    // DefaultViewModelProviderFactory工厂实现
    @Override
    public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
        if (mDefaultFactory == null) {
            // 创建一个SavedStateViewModelFactory返回
            mDefaultFactory = new SavedStateViewModelFactory(
                    getApplication(),
                    this,
                    getIntent() != null ? getIntent().getExtras() : null);
        }
        return mDefaultFactory;
    }
}
```

这里返回的是一个 SavedStateViewModelFactory，那就是说 SavedStateViewModel 实例都是默认使用这个 Factory 来创建，这个 Factory 有什么不同呢，它的不同之处在于，定义一个 ViewModel 的时候，可以在构造函数里面指定一个参数：

```kotlin
// 指定一个SavedStateHandle参数
class MainSaveViewModel(val savedState: SavedStateHandle) : ViewModel() {}

//指定两个参数Application和SavedStateHandle
class HomeAndroidViewModel(val application: Application, val savedStateHandle: SavedStateHandle) : AndroidViewModel(appInstance) {}

```

SavedStateViewModelFactory 在新建 ViewModel 的时候就会判断你的构造函数有没有参数，如果没有参数就以普通的形式进行反射创建它的实例对象，如果有参数就会判断是不是 SavedStateHandle 类型的，如果是则会从刚才 SavedStateRegistry 当中去取出它所缓存的数据，并构建一个 SavedStateHandle 对象，传递进来。

```kotlin
#ViewModelProvider类
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
    this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
            ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
            : NewInstanceFactory.getInstance());
}

//getDefaultViewModelProviderFactory() 实际是 SavedStateViewModelProviderFactory:
SavedStateViewModelFactory类：

//第一种：有两个参数
private static final Class<?>[] ANDROID_VIEWMODEL_SIGNATURE = new Class[]{Application.class,
        SavedStateHandle.class};
//第二种：只有一个参数
private static final Class<?>[] VIEWMODEL_SIGNATURE = new Class[]{SavedStateHandle.class};

@Override
public <T extends ViewModel> T create(String key, Class<T> modelClass) {
    // 判断是否为AndroidViewModel
    boolean isAndroidViewModel = AndroidViewModel.class.isAssignableFrom(modelClass);
    Constructor<T> constructor;
    // 获取构造器 
    if (isAndroidViewModel && mApplication != null) {
        constructor = findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE);
    } else {
        constructor = findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE);
    }
    // 普通方式创建ViewModel实例
    if (constructor == null) {
        return mFactory.create(modelClass);
    }
    
    // 创建SavedStateHandleController
    SavedStateHandleController controller = SavedStateHandleController.create(
            mSavedStateRegistry, mLifecycle, key, mDefaultArgs);
    try {
        T viewmodel;
        // 根据构造器参数创建viewmodel
        if (isAndroidViewModel && mApplication != null) {
            viewmodel = constructor.newInstance(mApplication, controller.getHandle());
        } else {
            viewmodel = constructor.newInstance(controller.getHandle());
        }
        return viewmodel;
    }
}
```

创建的时候判断 modelClass 是否拥有两种构造函数：

```java
//第一种：有两个参数
private static final Class<?>[] ANDROID_VIEWMODEL_SIGNATURE = new Class[]{Application.class,
        SavedStateHandle.class};
//第二种：只有一个参数
private static final Class<?>[] VIEWMODEL_SIGNATURE = new Class[]{SavedStateHandle.class};
```

如果上面两种都没有，那么在构造实例的时候，就会以普通的形式构造实例 AndroidViewModelFactory，实际上是通过**反射**。

```java
if (constructor == null) {
    return mFactory.create(modelClass);
}
```

`controller.getHandle()` 实际上得到的是 SavedStateHandle，controller 是通过`SavedStateHandleController.create()` 创建，这个类有三个作用：

```java
static SavedStateHandleController create(SavedStateRegistry registry, Lifecycle lifecycle,
        String key, Bundle defaultArgs) {
    // 1.通过key获取到先前保存起来的数据 得到bundle对象
    Bundle restoredState = registry.consumeRestoredStateForKey(key);
    // 2.传递restoredState，创建SavedStateHandle
    SavedStateHandle handle = SavedStateHandle.createHandle(restoredState, defaultArgs);
    // 3.通过key和handle创建SavedStateHandleController对象
    SavedStateHandleController controller = new SavedStateHandleController(key, handle);
    // 4.添加生命周期监听，向SavedStateRegistry数据存中心注册一个SavedStateProvider
    controller.attachToLifecycle(registry, lifecycle);
    tryToAddRecreator(registry, lifecycle);
    return controller;
}
```

在 registry 中获取先前保存的数据，通过 key（ViewModel的名称）得到一个 bundle 对象，接着创建一个 SavedStateHandle 对象，并且把 bundle 数据传递了进去，还会调用controller.attachToLifecycle(registry, lifecycle):

```java
void attachToLifecycle(SavedStateRegistry registry, Lifecycle lifecycle) {
    mIsAttached = true;
    lifecycle.addObserver(this);
    //注册一个 SavedStateProvider，用于实现数据存储的工作
    registry.registerSavedStateProvider(mKey, mHandle.savedStateProvider());
}
```

向 SavedStateRegistry 数据存中心注册一个 SavedStateProvider，用于实现数据存储的工作，那么 SavedStateHandle 被创建完之后，之前存储的数据就被恢复了，然后传递到了 SavedStateViewModelFactory 中的 controller.getHandle() 然后完成了数据的复用。

```java
//SavedStateHandle 对象创建时调用
@MainThread
public void registerSavedStateProvider(String key, SavedStateProvider provider) {
    // 保存SavedStateProvider
    SavedStateProvider previous = mComponents.putIfAbsent(key, provider);
}
```

数据恢复时，ComponentActivity实现了HasDefaultViewModelFactory， 使用了SavedStateViewModelFactory。会在onCreate的时候将saveState中的数据通过controller放入SaveStateRegistry的mRestoredState中，然后controller中会根据viewmodle的名称重新取到存下的bundle，生成SaveStateHandle，并且在attachlifecycle中重新将reisterSavedStateProvider注册到mComponents中。

![savestate原理简易逻辑图](.\res\savestate原理简易逻辑图.webp) 



![viewmodle数据存储区别](.\res\viewmodle数据存储区别.png)



## ActivityResultRegistry

##  为什么废弃 startActivityForResult

`startActivityForResult` 和 `onActivityResult` 有什么缺点：

1. 从哪里发出某个请求必须发送唯一的请求代码，以防使用重复。有时会导致错误的结果
2. 如果重新创建组件，则会丢失结果
3. `onActivityResult` 回调不适用于 `Fragment`