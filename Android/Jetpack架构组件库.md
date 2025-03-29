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





## StartUp

#### 官方的定义如下：

> The App Startup library provides a straightforward, performant way to initialize components at application startup. Both library developers and app developers can use App Startup to streamline startup sequences and explicitly set the order of initialization.
>  Instead of defining separate content providers for each component you need to initialize, App Startup allows you to define component initializers that share a single content provider. This can significantly improve app startup time.

##### 翻译如下：

应用程序启动库提供了一种在应用程序启动时初始化组件的简单、高效的方法。库开发人员和应用程序开发人员都可以使用`StartUp`来简化启动序列并显式设置初始化顺序。
 `StartUp`允许您定义共享单个内容提供程序的组件初始化程序，而不是为每个需要初始化的组件定义单独的`content provider`。这可以显著缩短应用程序启动时间。

简单的说就是通过一个公共的`content provider`来集中管理需要初始化的组件，从而提高应用的启动速度。

### StartUp的使用方法

- 添加所需依赖

> dependencies {
>  implementation "androidx.startup:startup-runtime:1.0.0"
>  }

- 为需要的每一个组件定义一个`component initializer`，假设存在A，B，C，D四个需要初始化的组件，这时候就需要定义四个`initializer`.



```kotlin
class ASdk {
    //假设这里是我们需要初始化的组件A
    companion object {
        fun getInstance(): ASdk {
            return Instance.instance
        }
    }
    private object Instance {
        val instance = ASdk()
    }
}
```

每一个需要初始化的组件我们需要创建一个`class`去实现`Initializer<T>`接口，它所对应的`Initializer`如下：



```kotlin
class ASdkInitializer : Initializer<ASdk> {
    override fun create(context: Context): ASdk {
        Log.i("gj","ASdkInitializer create()方法执行" )
        return ASdk.getInstance()
    }
    override fun dependencies(): MutableList<Class<out Initializer<*>>> {
        Log.i("gj","ASdkInitializer dependencies()方法执行" )
        return mutableListOf()
    }

}
```

可以看到只有两个方法需要我们去实现，`create ()`方法和`dependencies()`方法。

- `create ()`方法包含初始化组件所需的所有操作，并返回T的实例。
- `dependencies()`方法返回的是一个`Initializer<T>`的`list`,这个集合当中包含了当前的`Initializer<T>`所依赖的其他的`Initializer<T>`，由此可见该方法的作用是让我们可以控制在程序启动时的组件的初始化顺序。

比如我们的组件A，B，C的初始化之间存在着C依赖B，B依赖A的这么一种关系，这时B和C组件的`Initializer<T>`就应该写成如下：



```kotlin
/**
* B组件的初始化Initializer，依赖A
*/
class BSdkInitializer : Initializer<BSdk> {
  override fun create(context: Context): BSdk {
      Log.i("gj","BSdkInitializer create()方法执行" )
      return BSdk.getInstance()
  }

  override fun dependencies(): MutableList<Class<out Initializer<*>>> {
      Log.i("gj","BSdkInitializer dependencies()方法执行" )
      return mutableListOf(ASdkInitializer::class.java)
  }
}
```

因为B组件依赖以A组件的初始化完成，所以在`dependencies()`方法中，我们需要返回的是`ASdkInitializer`,同理C的`Initializer`如下：



```kotlin
/**
 * C组件的初始化Initializer，依赖B
 */
class CSdkInitializer : Initializer<CSdk> {
    override fun create(context: Context): CSdk {
        Log.i("gj", "CSdkInitializer create()方法执行")
        return CSdk.getInstance()
    }

    override fun dependencies(): MutableList<Class<out Initializer<*>>> {
        Log.i("gj", "CSdkInitializer dependencies()方法执行")
        return mutableListOf(BSdkInitializer::class.java)
    }
}
```

至此，我们的对组件的定义基本完成，接下来`StartUp`是怎么启动的？

------

`StartUp`为我们提供了两种方式来启动，一种是自动启动，一种是手动调用启动。

- 自动启动的方式如下：
   我们只需要在AndroidManifest中对`InitializationProvider`添加对应声明，如下：



```xml
        <provider
                android:name="androidx.startup.InitializationProvider"
                android:authorities="${applicationId}.androidx-startup"
                android:exported="false"
                tools:node="merge">
            <meta-data
                    android:name="com.yy.myapplication.CSdkInitializer"
                    android:value="androidx.startup"/>
            <meta-data
                    android:name="com.yy.myapplication.DSdkInitializer"
                    android:value="androidx.startup"
                    tools:node="remove" />
        </provider>
```

`meta-data`标签下是我们`Initializer`的路径，`value`需要注意的是必须为`androidx.startup`，可以看到在上面，A，B，C组件中我只对C的`Initializer`做了声明，这是因为B和C都可以通过`dependencies()`的链式调用进行初始化，当然，如果A，B，C之间不存在依赖关系的话，则需要对每一个对应的`Initializer`进行声明。而A，B，C的执行顺序则与我们的声明顺序保持一致。
 至此，程序启动的时候就会按照我们既定的顺序进行初始化操作。
 运行后日志如下：

![img](https:////upload-images.jianshu.io/upload_images/6065113-846fa511c4d81560.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

1611313866000.jpg

从日志不难看出：`creat()`的执行顺序为A-->B-->C,`dependencies()`的执行顺序为C-->B-->A，符合我们的预期效果。



- 手动控制方式如下：
   比如我们还定义了一个`DSdkInitializer`,这时候需要对D组件手动进行初始化，我们就可以直接对其进行初始化调用：



```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        //手动启动sdk的初始化
        AppInitializer.getInstance(this).initializeComponent(DSdkInitializer::class.java)
    }
}
```

从日志也可以看到D组件是最后被初始化的。

> 这里我们需要注意的是，我们虽然在`AndroidManifest`中声明了`DSdkInitializer`但是这个声明并不是了为了D组件的初始化所做的，可以看到我们加了一个属性`tools:node="remove"`这个标签的作用是为了防止在其他引用的三方库中有对相同组件的一个初始化，保证该组件的自动初始化真正的被关闭。

------

- 关闭startup的所有组件的自动初始化，我们除了可以上诉一个一个关闭的方法，还可以调用如下的方法：



```bash
        <provider android:name="androidx.startup.InitializationProvider"
                  android:authorities="${applicationId}.androidx-startup" 
                  tools:node="remove"/>
```

这样就可以做到真正的关闭 Startup 的所有自动初始化逻辑。

------

### StartUp 源码详解

![img](https:////upload-images.jianshu.io/upload_images/6065113-437471cf9385ab32.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/692/format/webp)

1611555500174.jpg

由上图可以看到StartUp包含的类只有五个`AppInitializer`,`InitializationProvider`,`Initializer`,`StartupException`,`StartupLogger`下面我们依次对这五个类进行详细的介绍。

------

#### AppInitializer

这个类是StartUp类库的核心类。



```java
public final class AppInitializer {
              ...
private static AppInitializer sInstance;

    /**
     * Guards app initialization.
     */
    private static final Object sLock = new Object();

    @NonNull
    final Map<Class<?>, Object> mInitialized;

    @NonNull
    final Context mContext;

    /**
     * Creates an instance of {@link AppInitializer}
     *
     * @param context The application context
     */
    AppInitializer(@NonNull Context context) {
        mContext = context.getApplicationContext();
        mInitialized = new HashMap<>();
    }

    /**
     * @param context The Application {@link Context}
     * @return The instance of {@link AppInitializer} after initialization.
     */
    @NonNull
    @SuppressWarnings("UnusedReturnValue")
    public static AppInitializer getInstance(@NonNull Context context) {
        synchronized (sLock) {
            if (sInstance == null) {
                sInstance = new AppInitializer(context);
            }
            return sInstance;
        }
    }
                ...
}
```

首先通过该类的`getInstance(@NonNull Context context)`方法获取我们所需的动态实例。
 这里注意一下这个参数的作用`Map<Class<?>, Object> 用map去存储已经被初始化过的组件。



```dart
	@NonNull
    @SuppressWarnings("unused")
    public <T> T initializeComponent(@NonNull Class<? extends Initializer<T>> component) {
        return doInitialize(component, new HashSet<Class<?>>());
    }

    @NonNull
    @SuppressWarnings({"unchecked", "TypeParameterUnusedInFormals"})
    <T> T doInitialize(
            @NonNull Class<? extends Initializer<?>> component,
            @NonNull Set<Class<?>> initializing) {
        synchronized (sLock) {
            boolean isTracingEnabled = Trace.isEnabled();
            try {
                if (isTracingEnabled) {
                    // Use the simpleName here because section names would get too big otherwise.
                    Trace.beginSection(component.getSimpleName());
                }
                if (initializing.contains(component)) {
                    String message = String.format(
                            "Cannot initialize %s. Cycle detected.", component.getName()
                    );
                    throw new IllegalStateException(message);
                }
                Object result;
                if (!mInitialized.containsKey(component)) {
                    initializing.add(component);
                    try {
                        Object instance = component.getDeclaredConstructor().newInstance();
                        Initializer<?> initializer = (Initializer<?>) instance;
                        List<Class<? extends Initializer<?>>> dependencies =
                                initializer.dependencies();

                        if (!dependencies.isEmpty()) {
                            for (Class<? extends Initializer<?>> clazz : dependencies) {
                                if (!mInitialized.containsKey(clazz)) {
                                    doInitialize(clazz, initializing);
                                }
                            }
                        }
                        if (StartupLogger.DEBUG) {
                            StartupLogger.i(String.format("Initializing %s", component.getName()));
                        }
                        result = initializer.create(mContext);
                        if (StartupLogger.DEBUG) {
                            StartupLogger.i(String.format("Initialized %s", component.getName()));
                        }
                        initializing.remove(component);
                        mInitialized.put(component, result);
                    } catch (Throwable throwable) {
                        throw new StartupException(throwable);
                    }
                } else {
                    result = mInitialized.get(component);
                }
                return (T) result;
            } finally {
                Trace.endSection();
            }
        }
    }
```

从上面的代码可以看到`doInitialize(component, new HashSet<Class<?>>())`方法最终调用的是`result = initializer.create(mContext);`不难看出这个方法最终的目的就是完成所有依赖项的初始化。
 大致的流程如下：

- 首先会判断正在进行初始化的`Initializer`集合，如果集合中存在`component`,说明当前`Initializer`之间存在着循环依赖，会抛出一个循环依赖的异常，如果没有依赖则继续向下执行。
- 如果已经初始化过的`mInitialized`中不包含`component`，说明当前的`Initializer`并为进行初始化，将其加到正在初始化的集合`initializing`中，之后通过反射调用`component`的构造方法进行初始化，同时获取`Initializer`的所有的依赖项记作`dependencies`，如果`dependencies`当中存在依赖，则对每个依赖项通过递归调用`doInitialize(clazz, initializing)`的方式进行初始化。
- 最终调用`initializer.create(mContext)`完成初始化，并将已经初始化的`Initializer`从集合`initializing`移除，同时将初始化完成的`component`放到`mInitialized`中保存起来。
- 如果已经在`mInitialized`中包含了`component`，就只需要从`result = mInitialized.get(component);`中获取缓存即可。

------

下面看一下`discoverAndInitialize()`方法做了什么？
 该方法是由`InitializationProvider`进行调用，最终会调用的是我们上面提到过的方法`doInitialize(component, initializing);`



```dart
    @SuppressWarnings("unchecked")
    void discoverAndInitialize() {
        try {
            Trace.beginSection(SECTION_NAME);
            ComponentName provider = new ComponentName(mContext.getPackageName(),
                    InitializationProvider.class.getName());
            ProviderInfo providerInfo = mContext.getPackageManager()
                    .getProviderInfo(provider, GET_META_DATA);
            Bundle metadata = providerInfo.metaData;
            String startup = mContext.getString(R.string.androidx_startup);
            if (metadata != null) {
                Set<Class<?>> initializing = new HashSet<>();
                Set<String> keys = metadata.keySet();
                for (String key : keys) {
                    String value = metadata.getString(key, null);
                    if (startup.equals(value)) {
                        Class<?> clazz = Class.forName(key);
                        if (Initializer.class.isAssignableFrom(clazz)) {
                            Class<? extends Initializer<?>> component =
                                    (Class<? extends Initializer<?>>) clazz;
                            if (StartupLogger.DEBUG) {
                                StartupLogger.i(String.format("Discovered %s", key));
                            }
                            doInitialize(component, initializing);
                        }
                    }
                }
            }
        } catch (PackageManager.NameNotFoundException | ClassNotFoundException exception) {
            throw new StartupException(exception);
        } finally {
            Trace.endSection();
        }
    }
```

- 首先，会获取到`InitializationProvider`中所有的`metadata`
- 接着遍历所有的`metadata`，找到属于startup的所有`metadata`，并通过包名路径查看是否是`Initializer`的实现类，如果是的话就进行初始化的操作。

------

#### InitializationProvider

`InitializationProvider`继承自`ContentProvider`，主要作用就是触发对StartUp的整个初始化。



```java
/**
 * The {@link ContentProvider} which discovers {@link Initializer}s in an application and
 * initializes them before {@link Application#onCreate()}.
 *
 * @hide
 */
@RestrictTo(RestrictTo.Scope.LIBRARY)
public final class InitializationProvider extends ContentProvider {
    @Override
    public boolean onCreate() {
        Context context = getContext();
        if (context != null) {
            AppInitializer.getInstance(context).discoverAndInitialize();
        } else {
            throw new StartupException("Context cannot be null");
        }
        return true;
    }
       .....
}
```

`onCreate()`方法由系统主动触发。

------

#### Initializer

该类就是StartUp提供的需要我们去声明初始化的组件以及初始化的依赖关系和顺序的。



```dart
/**
 * {@link Initializer}s can be used to initialize libraries during app startup, without
 * the need to use additional {@link android.content.ContentProvider}s.
 *
 * @param <T> The instance type being initialized
 */
public interface Initializer<T> {

    /**
     * Initializes and a component given the application {@link Context}
     *
     * @param context The application context.
     */
    @NonNull
    T create(@NonNull Context context);

    /**
     * @return A list of dependencies that this {@link Initializer} depends on. This is
     * used to determine initialization order of {@link Initializer}s.
     * <br/>
     * For e.g. if a {@link Initializer} `B` defines another
     * {@link Initializer} `A` as its dependency, then `A` gets initialized before `B`.
     */
    @NonNull
    List<Class<? extends Initializer<?>>> dependencies();
}
```

- `T create(@NonNull Context context);`方法中完成初始化并返回。
- `List<Class<? extends Initializer<?>>> dependencies();`方法中指定当前`Initializer`的依赖关系。

------

#### StartupException



```kotlin
@RestrictTo(RestrictTo.Scope.LIBRARY)
@SuppressWarnings("WeakerAccess")
public final class StartupException extends RuntimeException {
    public StartupException(@NonNull String message) {
        super(message);
    }

    public StartupException(@NonNull Throwable throwable) {
        super(throwable);
    }

    public StartupException(@NonNull String message, @NonNull Throwable throwable) {
        super(message, throwable);
    }
}
```

`RuntimeException`的一个自定义的子类，用于StartUp初始化过程中遇到错误的抛出类。

------

#### StartupLogger



```php
public final class StartupLogger {

    private StartupLogger() {
        // Does nothing.
    }

    /**
     * The log tag.
     */
    private static final String TAG = "StartupLogger";

    /**
     * To enable logging set this to true.
     */
    static final boolean DEBUG = false;

    /**
     * Info level logging.
     *
     * @param message The message being logged
     */
    public static void i(@NonNull String message) {
        Log.i(TAG, message);
    }

    /**
     * Error level logging
     *
     * @param message   The message being logged
     * @param throwable The optional {@link Throwable} exception
     */
    public static void e(@NonNull String message, @Nullable Throwable throwable) {
        Log.e(TAG, message, throwable);
    }
}
```

这个类就是一个普通的工具类，没什么好说的。







## Hilt

### 什么是IOC

IOC是Inversion of Control的缩写，翻译为控制反转，是面向对象编程中的一种设计原则，可以用来降低代码之间的耦合度。

因此IOC中最常见的方式叫做依赖注入（Dependency Injection，简称DI），所谓依赖注入，就是由IOC容器在运行期间，动态地将某种依赖关系注入到对象之中。

类里面声明的变量叫做依赖，通常依赖需要自己创建对象，自己进行管理。依赖注入，就是IOC容器在运行期间，让外部帮你初始化你的依赖，然后注入到你声明的变量上。

让外部初始就算依赖注入，那么工厂模式，Builder模式也是依赖注入。那么Hilt就是使用了注解的方式，让依赖注入变得更方便。

### 为什么要用依赖注入

如果一个对象需要被共享，或者可能在多个类中被使用，为了解耦，你就可以使用以来注入来初始化它。

### Hilt 是什么

Hilt 是 Android 的依赖注入库，其实是基于 Dagger 。可以说 Hilt 是专门为 Andorid 打造的。

Hilt 创建了一组标准的 组件和作用域。这些组件会自动集成到 Android 程序中的生命周期中。在使用的时候可以指定使用的范围，事情作用在对应的生命周期当中。

### Hilt支持的Android组件

![img](https:////upload-images.jianshu.io/upload_images/8669504-27afdb323d8b508a.png?imageMogr2/auto-orient/strip|imageView2/2/w/936/format/webp)

### 引入 Hilt

- 在项目的build.gradle中添加引用



```bash
buildscript {
    dependencies {
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.42'
    }
}
```

- 在App的build.gradle中添加plugin的引用和依赖：



```bash
    id 'kotlin-kapt'
    id 'dagger.hilt.android.plugin'

dependencies {
    implementation "com.google.dagger:hilt-android:2.42"
    kapt "com.google.dagger:hilt-android-compiler:2.42"
}
```

#### 常用注解

Hilt常用注解主要是：@HiltAndroidApp、@AndroidEntryPoint、@Inject、@Module、@InstallIn、@Binds、@Provides、@EntryPoint 等等。

- Hlit初始化
   所有使用Hilt的项目都需要包含一个带有@HiltAndroidApp注释的Application类．

@HiltAndroidApp会触发Hilt代码生成，生成的代码包含应用的一个基类，该基类充当应用级依赖项容器。



```kotlin
@HiltAndroidApp
class MyApplication: Application() {

    override fun onCreate() {
        super.onCreate()
        
    }
}
```

只有 Application 这个入口点是使用 @HiltAndroidApp 注解来声明的。其他的所有入口点，都是用 @AndroidEntryPoint 注解来声明的。

###### Hilt为何要增加@HiltAndroidApp注解

１：将ApplicationContextModule添加至应用组件中，获取应用组件applicationContext
 ２：自动实现了依赖注入，免去了类似Dagger的手动调用

##### 将依赖项注入Android类 @AndroidEntryPoint

Hilt目前支持以下Android类：

> Application（通过使用 @HiltAndroidApp）
>  Activity
>  Fragment
>  View
>  Service
>  BroadcastReceiver

为什么需要在使用依赖注入的组件上添加AndroidEntryPoint注解？因为Hilt需要找到使用了该注解的类，自动找到合适的位置，比如反射activity的oncreate方法中，进行初始化方法调用。

##### @Inject

如需从组件获取依赖项，可以使用@Inject注释执行字段注入。



```kotlin
class Truck @Inject constructor(val driver: Driver) {

    fun deliver() {
        Log.i("minfo", "Truck is delivering cargo driver by $driver")
    }
}

class Driver @Inject constructor() {

}
```

MainActivity中使用注入truck对象，并调用deliver()方法。



```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var truck: Truck

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        truck.deliver()
    }
}
```

##### Hilt模块 @Module

有时，类型不能通过构造函数注入。例如，您不能通过构造函数注入接口。此外，您也不能通过构造函数注入不归您所有的类型，如来自外部库的类。在这些情况下，您可以使用Hilt @Module向Hilt提供绑定信息。比如我们常用于创建依赖类的对象(例如第三方库OkHttp、Retrofit等等)，使用@Module注解的类，需要使用@InstallIn注解指定module的范围。

##### Hilt内置组件和组件作用域

InstallIn，就是安装到的意思。那么@InstallIn(ActivityComponent::class)，就是把这个模块安装到Activity组件当中。
 Hilt一共内置了7种组件类型，分别用于注入到不同的场景，如下表所示。

![img](https:////upload-images.jianshu.io/upload_images/8669504-462b93283723deb7.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

###### @Singleton

Hilt会为每次的依赖注入行为都创建不同的实例。这种默认行为在很多时候确实是非常不合理的，比如我们提供的Retrofit和OkHttpClient的实例，理论上它们全局只需要一份就可以了，每次都创建不同的实例明显是一种不必要的浪费。

如后面写到的NetworkModule内部创建OkHttpClient、Retrofit使用了注解@Singleton。



```kotlin
@Module
@InstallIn(SingletonComponent::class)
// 把NetworkModule绑定到HiltApp 的生命周期。
object NetworkModule {
}
```

##### @Binds 注入接口实例

接口是无法通过构造方法进行注入的，需要通过在Hilt 模块内创建一个带有@Binds注释的抽象函数进行注入。@Binds 注释会告知Hilt在需要提供接口的实例时要使用哪种实现。

示例：



```kotlin
//需要注入接口
interface Engine {
    fun start()
    fun shutdown()
}

//接口实现类
class GasEngine @Inject constructor() : Engine {

    override fun start() {
        Log.i("minfo", "GasEngine start")
    }

    override fun shutdown() {
        Log.i("minfo", "GasEngine shotdown")
    }

}

@Module
@InstallIn(ActivityComponent::class)
abstract class EnginModule {

    @Binds
    abstract fun bindEngin(gasEngine: GasEngine): Engine   //指定子类和实现类
    //@Binds 注释会告知Hilt在需要提供接口的实例时要使用哪种实现。
    //@Binds告知 需要提供 Engin接口的实例是有该抽象方法的参数类型对象提供，所以GasEngine需要实现Engine接口
}
```

在Truck类中注入engine接口：



```kotlin
class Truck @Inject constructor(val driver: Driver) {
    @Inject
    lateinit var engine: Engine

    fun deliver() {
        engine.start()
        Log.i("minfo", "Truck is delivering cargo driver by $driver")
        engine.shutdown()
    }
}
```

##### @Provides 注入实例

第三方库如：Retrofit、OkHttpClient 或 Room数据库等都是无法通过构造函数注入，我们去改不了三方库的代码，在里面加入@Inject，这时就需要使用@Provides注入实例，在Provides注解的方法中，给出具体的实现代码。



```kotlin
@Module
@InstallIn(SingletonComponent::class)
// 把NetworkModule绑定到HiltApp 的生命周期。
object NetworkModule {

    @Provides
    @Singleton
    fun getOkhttpClient() : OkHttpClient {
        return OkHttpClient.Builder().build()
    }

    @Provides
    @Singleton
    fun createRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .client(okHttpClient)
            .baseUrl("https://www.wanandroid.com")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun getService(): ApiService {
        return createRetrofit(getOkhttpClient()).create(ApiService::class.java)
    }
    
}
```

如上方式，类上添加@InstallIn(SingletonComponent::class) 以及提供对象添加@Singleton注解，就是提供单例依赖注入的写法。如果需要每次使用初始化，就去掉这两个注解。

其他生命周期，@InstallIn可将注解改为：ActivityComponent/FragmentComponent/ViewComponent/ActivityRetainedComponent的生命周期。

##### 使用注入对象

直接在activity中进行注入：



```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var truck: Truck

    @Inject
    lateinit var apiService: ApiService

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        truck.deliver()
    }
}
```

##### Hilt 中的预定义限定符

如果有个我们想要依赖注入的类，它又是依赖于Context的，这个情况要如何解决呢？

Hilt提供了一些预定义的限定符。例如，由于我们可能需要来自应用或Activity的Context类，因此Hilt提供了@ApplicationContext和 @ActivityContext 限定符。



```kotlin
@Singleton
class Driver @Inject constructor(val context: Context) {
}
```

现在你编译一下项目一定会报错，原因也很简单，Driver类无法被依赖注入了，因为Hilt不知道要如何提供Context这个参数。其实只需要在Context参数前加上一个@ApplicationContext注解，代码就能编译通过了。



```kotlin
@Singleton
class Driver @Inject constructor(@ApplicationContext val context: Context) {
}

@Singleton
class Driver @Inject constructor(@ActivityContext val context: Context) {
}
```

##### Hilt在ViewModel的使用



```kotlin
@ActivityRetainedScoped
class MyViewModel @Inject constructor() : ViewModel() {

    fun loadData() {
        viewModelScope.launch {

        }
    }
}
```

在activity中使用viewmodel



```kotlin
    @Inject
    lateinit var viewModel: MyViewModel
```

##### @EntryPoint注解

Hilt支持最常见的Android类Application、Activity、Fragment、View、Service、BroadcastReceiver 等等，但是我们可能需要在Hilt 不支持的类中执行依赖注入，在这种情况下可以使用@EntryPoint注解进行创建，Hilt会提供相应的依赖。

#### 原理篇

###### @HiltAndroidApp注解做了什么事

首先会创建一个包含很多接口和抽象类的 java 文件 MyApplication_HiltComponents，用来规范好所有的类关系和行为。

然后会为 Application 创建一个注射器 MyApplication_GeneratedInjector，负责将这个 Application 注入。

然后再创建一个 Application 的父类 Hilt_Application，用来替换掉源代码的继承关系，以此实现将 Hilt 插入到正常代码逻辑中，实现 Hilt 的功能。原有application对象的父类就变成了Hilt_Application。

###### @AndroidEntryPoint注解了MainActivity之后Hilt帮我们做了什么？

1.Hilt自动帮我们生成了一个继承自AppCompatActivity的名称为Hilt_MainActivity.java类。
 然后让原本的Mainactivity继承于Hilt_MainActivity，Hilt_MainActivity会在onCreate方法中调用自己的inject方法进行注入，将声明依赖进行赋值。

Hilt_MainActivity类中：



```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
  inject();
  super.onCreate(savedInstanceState);
}
```

![img](https:////upload-images.jianshu.io/upload_images/8669504-4fbe9381e7f6c4cd.png?imageMogr2/auto-orient/strip|imageView2/2/w/509/format/webp) 

1. 对使用了@Inject注解的依赖，生成了XXX_Factory类，类中是对该对象的创建方法。







## **DataStore 原理**

`DataStore` 是 Android Jetpack 中一个用于存储数据的新组件，它替代了传统的 `SharedPreferences`，提供了更强大的类型安全、异步和持久化的功能。`DataStore` 提供了两种主要的实现方式：**Preferences DataStore** 和 **Proto DataStore**，两者都可以用于存储和检索小型的结构化数据，但它们有不同的用途和实现原理。

### **1. DataStore 的基本概念**

`DataStore` 是基于 Jetpack 中的 **Coroutines** 和 **Flow** 的异步数据存储解决方案，确保了在操作过程中不阻塞主线程，因此适合用于应用的背景数据存储。

- **Preferences DataStore**：用于存储键值对数据，类似于 `SharedPreferences`。
- **Proto DataStore**：使用 `Protocol Buffers`（ProtoBuf）来存储结构化数据，更加灵活且类型安全。

### **2. DataStore 的工作原理**

#### **异步操作**

`DataStore` 采用异步读写操作，避免了传统同步存储（如 `SharedPreferences`）对 UI 线程的阻塞。它依赖于 Kotlin 的 **Coroutines** 和 **Flow**，使得所有的存储操作都可以在后台线程执行。

- **Flow**：`DataStore` 使用 `Flow` 来处理数据的读取和监听。`Flow` 是一个冷流，当你调用它时，它会返回一个 **异步的流**，允许你在其中收集数据并响应数据变化。

#### **类型安全**

与 `SharedPreferences` 不同，`DataStore` 提供了强类型的 API，可以存储复杂的对象和自定义的数据结构。对于 **Proto DataStore**，它支持通过 **Protocol Buffers** 序列化和反序列化对象，提供更强的类型安全性和结构化存储。

#### **持久化**

`DataStore` 是持久化的，它将数据存储到设备的本地磁盘上，保证数据在应用退出或设备重启后依然可以访问。

#### **原子操作**

`DataStore` 保证数据写入操作是原子的，确保写入操作不会产生中间状态或数据不一致的情况。它使用类似事务的机制来管理数据的读写。

------

### **3. Preferences DataStore**

`Preferences DataStore` 存储数据为键值对，类似于 `SharedPreferences`，但它提供了更好的性能和 API 设计，且支持异步操作。

#### **存储和读取原理**

- **存储数据**：你可以使用 `DataStore` 提供的 API 来写入键值对数据。它通过内部的文件存储（通常是 `.preferences` 文件）将数据保存在磁盘上。
- **读取数据**：数据读取是通过 `Flow` 来实现的，这意味着你可以异步地从数据源中读取数据，而不需要阻塞主线程。

#### **示例**：

```
kotlin复制代码// Preferences DataStore 初始化
val dataStore: DataStore<Preferences> = context.createDataStore(name = "user_prefs")

// 写入数据
suspend fun saveUserData(userName: String) {
    dataStore.edit { preferences ->
        preferences[KEY_USER_NAME] = userName
    }
}

// 读取数据
val userNameFlow: Flow<String> = dataStore.data
    .map { preferences ->
        preferences[KEY_USER_NAME] ?: "Default Name"
    }
```

- **数据类型**：数据存储为 `Preferences`，键是 `Preferences.Key<T>`，`T` 可以是 `String`、`Int`、`Boolean` 等基本数据类型。

#### **写入操作**：

- 使用 `dataStore.edit {}` 来执行写入操作。
- 数据是通过修改 `Preferences` 对象中的键值对来更新的。

#### **读取操作**：

- 使用 `dataStore.data` 访问存储的所有数据，该属性返回一个 `Flow<Preferences>`，你可以使用 `collect()` 或 `map()` 来获取和响应数据变化。

------

### **4. Proto DataStore**

Proto DataStore 是一种基于 **Protocol Buffers** 的数据存储方式，能够存储结构化数据。与 `Preferences DataStore` 的键值对形式不同，Proto DataStore 允许你定义自己的数据模型（类），并通过 `ProtoBuf` 序列化和反序列化来实现数据的存储和读取。

#### **ProtoBuf 的优点**：

- **类型安全**：你可以使用自定义数据类，而不是简单的键值对，支持更加复杂的数据结构。
- **性能**：Protocol Buffers 是一种高效的序列化格式，适合存储结构化数据，特别是对于大规模数据和高频率访问时非常高效。
- **兼容性**：ProtoBuf 是跨平台的，允许你在其他平台（例如 Java 或 Python）上共享数据模型。

#### **Proto DataStore 的存储原理**：

- 使用 Protocol Buffers 来序列化和反序列化数据模型。
- 数据是通过与 `DataStore` 一起定义的 `.proto` 文件中的消息类型进行序列化存储的。
- 写入和读取数据时，`Proto DataStore` 会自动处理序列化和反序列化过程。

#### **Proto DataStore 使用示例**：

1. **定义 Proto 文件**： 你需要先定义 `.proto` 文件，描述你的数据模型。

   ```Proto 
   syntax = "proto3";
   
   option java_package = "com.example.datastore";
   option java_multiple_files = true;
   
   message User {
       string name = 1;
       int32 age = 2;
   }
   ```

2. **生成数据类**： 使用 `protoc` 编译器来生成 Java 或 Kotlin 类，这些类将用于序列化和反序列化数据。

   ```kotlin
   // User Proto 数据类
   val user = User.newBuilder()
       .setName("Alice")
       .setAge(25)
       .build()
   
   // Proto DataStore 的初始化
   val userDataStore: DataStore<User> = context.createDataStore(
       fileName = "user_data.pb", 
       serializer = UserSerializer
   )
   
   // 写入数据
   suspend fun saveUser(user: User) {
       userDataStore.updateData { currentUser ->
           user
       }
   }
   
   // 读取数据
   val userFlow: Flow<User> = userDataStore.data
   ```

   - **UserSerializer** 是一个自定义的 Proto 序列化器，用于将 `User` 对象转换为字节流（在存储时）和从字节流还原成对象（在读取时）。

3. **写入和读取操作**：

   - `updateData {}` 用于更新数据。
   - 读取操作通过 `dataStore.data` 来访问。

------

### **5. DataStore 的优势**

#### **相比 SharedPreferences**

- **异步操作**：`DataStore` 完全基于异步操作，避免了主线程阻塞的问题，而 `SharedPreferences` 是同步操作，可能会导致 ANR（应用无响应）。
- **类型安全**：`DataStore` 支持类型安全的数据存储，特别是 Proto DataStore，支持存储复杂的对象，而 `SharedPreferences` 只能存储基本类型的键值对。
- **性能**：`DataStore` 提供了更高效的存储和访问方式，特别是在处理较大的数据时，比 `SharedPreferences` 更加高效。
- **可扩展性**：Proto DataStore 可以存储结构化数据，可以轻松管理复杂的数据模型。

#### **对比 Room 数据库**

- `Room` 主要用于存储结构化的大型数据（如数据库表），而 `DataStore` 适用于存储轻量级的数据（如配置、用户设置、偏好等）。
- `Room` 提供了关系型数据存储，而 `DataStore` 更适用于键值对或序列化数据的存储。

------

### **总结**

- **DataStore** 是 Android 的新型数据存储解决方案，相较于 `SharedPreferences` 提供了更现代的 API 和更强的功能，支持异步操作、类型安全、持久化存储和可靠性。
- **Preferences DataStore** 用于存储简单的键值对数据，而 **Proto DataStore** 适用于存储结构化的复杂数据。
- `DataStore` 基于 Kotlin Coroutines 和 Flow，使得所有数据存取操作都变得更加流畅、异步且不阻塞主线程，适合高效存储和管理应用的轻量级数据。



## WorkManager 

WorkManager 根据不同api地城调度服务

  ![overview-criteria.png](https://i-blog.csdnimg.cn/blog_migrate/bcd26ba87c2614979b86e04006d26cad.png) 