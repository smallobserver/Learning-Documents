# HarmonyOS学习



## ArtTS学习

简述：ArtTS引申于 TypeScript语言简称TS，在其类似JS灵活的基础上进行了一定的限制，使其接近Java这种严格的面相对象的语言，比如无法动态修改类中的属性（增加减少或者修改属性类型）。



### 基础语法

#### 类型

声明使用**`let`**关键字

基本类型：string，number，boolean

引用类型：Array，自定义类等

枚举类型：Enum

联合类型：Union（允许变量的值为多个类型）

​	比如有个变量可以是数字或者字符串，且可以切换

类型别名：Aliases （允许给一个类型取一个别名，方便理解和复用，使用关键字**`type`**声明）

#### 空安全

```typescript
let name: string | null = null
//使用?? null时返回右边 
const res = name ?? '';
//使用？可选链，如果是null，返回undefined
let len = name?.length;
```

ArkTS支持自动类型推导，会根据上下文自动选择合适的类型。

条件表达式 -》 三元运算符支持

for...of循环语句



面相对象三大特征，在ArkTS中和java基本一致，封装继承多态。

需要在模块外使用的类，需要使用export



### 声明式UI语法

@Component 声明结构体为一个自定义组件声明，才可以作为组件使用

​	自定义组件需要用**`struct`**关键字来定义

@Entry 声明为入口页面，可以通过UI Ability进行加载

在组件中的build()函数，来加载子组件，类似于Flutter中的页面开发。

@State 表示组件中的状态变量，状态变量变化后会触发ui刷新

- [@Builder](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-builder-V5)/[@BuilderParam](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-builderparam-V5)：特殊的封装UI描述的方法，细粒度的封装和复用UI描述。
- [@Extend](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-extend-V5)/[@Styles](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-style-V5)：扩展内置组件和封装属性样式，更灵活地组合内置组件。
- [stateStyles](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-statestyles-V5)：多态样式，可以依据组件的内部状态的不同，设置不同样式。



ForEach(array，()=>{}) 语法可以在build函数中使用，批量生成对应的ui



#### build()函数

所有声明在build()函数的语句，我们统称为UI描述，UI描述需要遵循以下规则：

- @Entry装饰的自定义组件，其build()函数下的根节点唯一且必要，且必须为容器组件，其中ForEach禁止作为根节点。

  @Component装饰的自定义组件，其build()函数下的根节点唯一且必要，可以为非容器组件，其中ForEach禁止作为根节点。

- ```typescript
  @Entry@Componentstruct MyComponent {  build() {   
      // 根节点唯一且必要，必须为容器组件    
      Row() {      ChildComponent()     } 
  }}
  @Componentstruct ChildComponent {  build() {    
      // 根节点唯一且必要，可为非容器组件    
      Image('test.jpg')  
  }}
  ```
  
- 不允许声明本地变量，反例如下。

  ```typescript
  build() {  
      // 反例：不允许声明本地变量  
      let a: number = 1;
  }
  ```

- 不允许在UI描述里直接使用console.info，但允许在方法或者函数里使用，反例如下。

  ```typescript
  build() {  
      // 反例：不允许console.info  
      console.info('print debug log');
  }
  ```

- 不允许创建本地的作用域，反例如下。

  ```typescript
  build() {  
      // 反例：不允许本地作用域  
      {    ...  }
  }
  ```

- 不允许调用没有用@Builder装饰的方法，允许系统组件的参数是TS方法的返回值。

  ```typescript
  @Componentstruct ParentComponent {  doSomeCalculations() {  }
    calcTextValue(): string {    return 'Hello World';  }
    @Builder doSomeRender() {    Text(`Hello World`)  }
    build() {    Column() {      
        // 反例：不能调用没有用@Builder装饰的方法      
        this.doSomeCalculations();      
        // 正例：可以调用      
        this.doSomeRender();      
        // 正例：参数可以为调用TS方法的返回值      
        Text(this.calcTextValue())    
    }  }}
  ```

- 不允许使用switch语法，如果需要使用条件判断，请使用if。示例如下。

  ```typescript
  build() {  Column() {    
      // 反例：不允许使用switch语法    
      switch (expression) {      
          case 1:        
              Text('...')        
              break;      
          case 2:        
              Image('...')        
              break;      
          default:        
              Text('...')        
              break;    
      }   
      // 正例：使用if   
      if(expression == 1) {     
          Text('...')
      } else if(expression == 2) {
          Image('...')    
      } else {
          Text('...')    
      }  
  }}
  ```

- 不允许使用表达式，反例如下。

  ```typescript
  build() {  Column() {    
      // 反例：不允许使用表达式    
      (this.aVar > 10) ? Text('...') : Image('...')  
  }}
  ```

- 不允许直接改变状态变量，反例如下。详细分析见[@State常见问题：不允许在build里改状态变量](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-state-V5#不允许在build里改状态变量)

  ```typescript
  @Componentstruct CompA {  @State col1: Color = Color.Yellow;  @State col2: Color = Color.Green;  @State count: number = 1;  build() {   
      Column() {      
      	// 应避免直接在Text组件内改变count的值      
      	Text(`${this.count++}`)        
          .width(50)        
          .height(50)        
          .fontColor(this.col1)        
          .onClick(() => {          
          	this.col2 = Color.Red;        
      	})      
      	Button("change col1").onClick(() =>{
          	this.col1 = Color.Pink;      
      	})    
  	}.backgroundColor(this.col2)  
  }}
  ```



### 页面和自定义组件生命周期

- 自定义组件：@Component装饰的UI单元，可以组合多个系统组件实现UI的复用，可以调用组件的生命周期。
- 页面：即应用的UI页面。可以由一个或者多个自定义组件组成，[@Entry](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-create-custom-components-V5#自定义组件的基本结构)装饰的自定义组件为页面的入口组件，即页面的根节点，一个页面有且仅能有一个@Entry。只有被@Entry装饰的组件才可以调用页面的生命周期。

#### 流程

流程图：

![UI组件生命周期流程图](.\res\UI组件生命周期流程图.png)

页面生命周期，即被@Entry装饰的组件生命周期，提供以下生命周期接口：

- [onPageShow](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-custom-component-lifecycle-V5#onpageshow)：页面每次显示时触发一次，包括路由过程、应用进入前台等场景。
- [onPageHide](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-custom-component-lifecycle-V5#onpagehide)：页面每次隐藏时触发一次，包括路由过程、应用进入后台等场景。
- [onBackPress](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-custom-component-lifecycle-V5#onbackpress)：当用户点击返回按钮时触发。

组件生命周期，即一般用@Component装饰的自定义组件的生命周期，提供以下生命周期接口：

- [aboutToAppear](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-custom-component-lifecycle-V5#abouttoappear)：组件即将出现时回调该接口，具体时机为在创建自定义组件的新实例后，在执行其build()函数之前执行。
- [onDidBuild](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-custom-component-lifecycle-V5#ondidbuild12)：组件build()函数执行完成之后回调该接口，不建议在onDidBuild函数中更改状态变量、使用animateTo等功能，这可能会导致不稳定的UI表现。
- [aboutToDisappear](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-custom-component-lifecycle-V5#abouttodisappear)：aboutToDisappear函数在自定义组件析构销毁之前执行。不允许在aboutToDisappear函数中改变状态变量，特别是@Link变量的修改可能会导致应用程序行为不稳定。



#### 自定义组件的创建和渲染流程

1. 自定义组件的创建：自定义组件的实例由ArkUI框架创建。
2. 初始化自定义组件的成员变量：通过本地默认值或者构造方法传递参数来初始化自定义组件的成员变量，初始化顺序为成员变量的定义顺序。
3. 如果开发者定义了aboutToAppear，则执行aboutToAppear方法。
4. 在首次渲染的时候，执行build方法渲染系统组件，如果子组件为自定义组件，则创建自定义组件的实例。在首次渲染的过程中，框架会记录状态变量和组件的映射关系，当状态变量改变时，驱动其相关的组件刷新。
5. 如果开发者定义了onDidBuild，则执行onDidBuild方法。

#### 自定义组件重新渲染

当事件句柄被触发（比如设置了点击事件，即触发点击事件）改变了状态变量时，或者LocalStorage / AppStorage中的属性更改，并导致绑定的状态变量更改其值时：

1. 框架观察到了变化，将启动重新渲染。
2. 根据框架持有的两个map（[自定义组件的创建和渲染流程](https://developer.huawei.com/consumer/cn/doc/#自定义组件的创建和渲染流程)中第4步），框架可以知道该状态变量管理了哪些UI组件，以及这些UI组件对应的更新函数。执行这些UI组件的更新函数，实现最小化更新。

#### 自定义组件的删除

如果if组件的分支改变，或者ForEach循环渲染中数组的个数改变，组件将被删除：

1. 在删除组件之前，将调用其aboutToDisappear生命周期函数，标记着该节点将要被销毁。ArkUI的节点删除机制是：后端节点直接从组件树上摘下，后端节点被销毁，对前端节点解引用，前端节点已经没有引用时，将被JS虚拟机垃圾回收。
2. 自定义组件和它的变量将被删除，如果其有同步的变量，比如[@Link](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-link-V5)、[@Prop](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-prop-V5)、[@StorageLink](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-appstorage-V5#storagelink)，将从[同步源](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-state-management-overview-V5#基本概念)上取消注册。

不建议在生命周期aboutToDisappear内使用async await，如果在生命周期的aboutToDisappear使用异步操作（Promise或者回调方法），自定义组件将被保留在Promise的闭包中，直到回调方法被执行完，这个行为阻止了自定义组件的垃圾回收。



### 应用框架和应用模型

#### 应用的多Module设计机制

- **支持模块化开发：** 一个应用通常会包含多种功能，将不同的功能特性按模块来划分和管理是一种良好的设计方式。在开发过程中，我们可以将每个功能模块作为一个独立的Module进行开发，Module中可以包含源代码、资源文件、第三方库、配置文件等，每一个Module可以独立编译，实现特定的功能。这种模块化、松耦合的应用管理方式有助于应用的开发、维护与扩展。
- **支持多设备适配：** 一个应用往往需要适配多种设备类型，在采用多Module设计的应用中，每个Module都会标注所支持的设备类型。有些Module支持全部类型的设备，有些Module只支持某一种或几种类型的设备（比如平板），那么在应用市场分发应用包时，也能够根据设备类型做精准的筛选和匹配，从而将不同的包合理的组合和部署到对应的设备上。

#### Module类型

Module按照使用场景可以分为两种类型：

- **Ability类型的Module：** 用于实现应用的功能和特性。每一个Ability类型的Module编译后，会生成一个以.hap为后缀的文件，我们称其为HAP（Harmony Ability Package）包。HAP包可以独立安装和运行，是应用安装的基本单位，一个应用中可以包含一个或多个HAP包，具体包含如下两种类型。

  - entry类型的Module：应用的主模块，包含应用的入口界面、入口图标和主功能特性，编译后生成entry类型的HAP。每一个应用分发到同一类型的设备上的应用程序包，只能包含唯一一个entry类型的HAP。
  - feature类型的Module：应用的动态特性模块，编译后生成feature类型的HAP。一个应用中可以包含一个或多个feature类型的HAP，也可以不包含。

- **Library类型的Module：** 用于实现代码和资源的共享。同一个Library类型的Module可以被其他的Module多次引用，合理地使用该类型的Module，能够降低开发和维护成本。Library类型的Module分为Static和Shared两种类型，编译后会生成共享包。

  - Static Library：静态共享库。编译后会生成一个以.har为后缀的文件，即静态共享包HAR（Harmony Archive）。
  - Shared Library：动态共享库。编译后会生成一个以.hsp为后缀的文件，即动态共享包HSP（Harmony Shared Package）。

  说明

  实际上，Shared Library编译后除了会生成一个.hsp文件，还会生成一个.har文件。这个.har文件中包含了HSP对外导出的接口，应用中的其他模块需要通过.har文件来引用HSP的功能。为了表述方便，我们通常认为Shared Library编译后生成HSP。

  HAR与HSP两种共享包的主要区别体现在：

  | 共享包类型 | 编译和运行方式                                               | 发布和引用方式                                               |
  | ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | HAR        | HAR中的代码和资源跟随使用方编译，如果有多个使用方，它们的编译产物中会存在多份相同拷贝。注意：[编译HAR](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/har-package-V5#编译)时，建议开启混淆能力，保护代码资产。 | HAR除了支持应用内引用，还可以独立打包发布，供其他应用引用。  |
  | HSP        | HSP中的代码和资源可以独立编译，运行时在一个进程中代码也只会存在一份。 | HSP一般随应用进行打包，当前支持应用内和[集成态HSP](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/integrated-hsp-V5)。应用内HSP只支持应用内引用，集成态HSP支持发布到ohpm私仓和跨应用引用。 |

   HAR和HSP在APP包中的形态示意图如下：

​	![HAR与HSP共享包在APP包中的形态示意图](.\res\HAR与HSP共享包在APP包中的形态示意图.png)



#### 工程目录结构

项目工程结构示意图Demo如下：

 ![项目工程结构示意图Demo](.\res\项目工程结构示意图Demo.png)

> [!WARNING]
>
> - AppScope目录由DevEco Studio自动生成，不可更改。（AppScope下有个**`app.json5`** 为应用配置文件）
> - Module目录名称可以由DevEco Studio自动生成（比如entry、library等），也可以自定义。为了便于说明，下表中统一采用Module_name表示。
>



| 文件类型      | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| 配置文件      | 包括应用级配置信息、以及Module级配置信息：<br>- **AppScope > app.json5**：[app.json5配置文件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/app-configuration-file-V5)，用于声明应用的全局配置信息，比如应用Bundle名称、应用名称、应用图标、应用版本号等。<br>- **Module_name > src > main > module.json5**：[module.json5配置文件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/module-configuration-file-V5)，用于声明Module基本信息、支持的设备类型、所含的组件信息、运行所需申请的权限等。 |
| ArkTS源码文件 | **Module_name > src > main > ets**：用于存放Module的ArkTS源码文件（.ets文件）。 |
| 资源文件      | 包括应用级资源文件、以及Module级资源文件，支持图形、多媒体、字符串、布局文件等，详见[资源分类与访问](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/resource-categories-and-access-V5)。<br>- **AppScope > resources** ：用于存放应用需要用到的资源文件。<br>- **Module_name > src > main > resources** ：用于存放该Module需要用到的资源文件。 |
| 其他配置文件  | 用于编译构建，包括构建配置文件、编译构建任务脚本、混淆规则文件、依赖的共享包信息等。<br>- **build-profile.json5**：工程级或Module级的构建配置文件，包括应用签名、产品配置等。<br>- **hvigorfile.ts**：应用级或Module级的编译构建任务脚本，开发者可以自定义编译构建工具版本、控制构建行为的配置参数。<br>- **obfuscation-rules.txt**：混淆规则文件。混淆开启后，在使用Release模式进行编译时，会对代码进行编译、混淆及压缩处理，保护代码资产。<br>- **oh-package.json5**：用于存放依赖库的信息，包括所依赖的三方库和共享包。 |

其中**`resoureces`** 用于放置资源文件 

- **`base`**	
  - **`element`** 元素资源目录，用于放置颜色、数字、字符串等
  - **`media`** 媒体资源目录，用于放置图片、音频、视频等
  - **`profile`** 自定义配置文件目录，包含页面配置、卡片配置等配置文件
    - **`main_pages.json`** 默认生成包含page路由配置
- **`en_US`**、**`zh_CN`** 国际化多语言 和android一致
- **`rawfile`** 其下的资源文件会直接打包进应用，不用编译和赋予资源ID，和android raw一致

​	

#### 编译态包结构

不同类型的Module编译后会生成对应的HAP、HAR、HSP等文件，开发态视图与编译态视图的对照关系如下：

![img](.\res\开发态与编译态的工程结构视图.png)

从开发态到编译态，Module中的文件会发生如下变更：

- **ets目录**：ArkTS源码编译生成.abc文件。
- **resources目录**：AppScope目录下的资源文件会合入到Module下面资源目录中，如果两个目录下存在重名文件，编译打包后只会保留AppScope目录下的资源文件。
- **module配置文件**：AppScope目录下的app.json5文件字段会合入到Module下面的module.json5文件之中，编译后生成HAP或HSP最终的module.json文件。

> [!WARNING]
>
> **在编译HAP和HSP时，会把他们所依赖的HAR直接编译到HAP和HSP中。**





#### 发布态包结构

每个应用中至少包含一个.hap文件，可能包含若干个.hsp文件、也可能不含，一个应用中的所有.hap与.hsp文件合在一起称为**Bundle**，其对应的bundleName是应用的唯一标识（详见[app.json5配置文件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/app-configuration-file-V5)中的bundleName标签）。

当应用发布上架到应用市场时，需要将Bundle打包为一个.app后缀的文件用于上架，这个.app文件称为**App Pack**（Application Package），与此同时，DevEco Studio工具自动会生成一个**pack.info**文件。**pack.info**文件描述了App Pack中每个HAP和HSP的属性，包含APP中的bundleName和versionCode信息、以及Module中的name、type和abilities等信息。

> [!IMPORTANT]
>
> - App Pack是发布上架到应用市场的基本单元，但是不能在设备上直接安装和运行。
> - 在应用签名、云端分发、端侧安装时，都是以HAP/HSP为单位进行签名、分发和安装的。



![img](.\res\编译发布与上架部署流程图.png)

#### 选择合适的包类型

HAP、HAR、HSP三者的功能和使用场景总结对比如下：

| Module类型     | 包类型                                                       | 说明                                                         |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Ability        | [HAP](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/hap-package-V5)    | 应用的功能模块，可以独立安装和运行，必须包含一个entry类型的HAP，可选包含一个或多个feature类型的HAP。 |
| Static Library | [HAR](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/har-package-V5) | 静态共享包，编译态复用。<br>- 支持应用内共享，也可以发布后供其他应用使用。<br>- 作为二方库，发布到[OHPM](https://ohpm.openharmony.cn/)私仓，供公司内部其他应用使用。<br>- 作为三方库，发布到[OHPM](https://ohpm.openharmony.cn/)中心仓，供其他应用使用。<br>- 多包（HAP/HSP）引用相同的HAR时，会造成多包间代码和资源的重复拷贝，从而导致应用包膨大。<br>- 注意：[编译HAR](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/har-package-V5#编译)时，建议开启混淆能力，保护代码资产。 |
| Shared Library | [HSP](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/in-app-hsp-V5) | 动态共享包，运行时复用。<br>- 当前仅支持应用内共享。<br>- 当多包（HAP/HSP）同时引用同一个共享包时，采用HSP替代HAR，可以避免HAR造成的多包间代码和资源的重复拷贝，从而减小应用包大小。 |

HAP、HSP、HAR支持的规格对比如下，其中“√”表示是，“×”表示否。

开发者可以根据实际场景所需的能力，选择相应类型的包进行开发。在后续的章节中还会针对如何使用[HAP](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/hap-package-V5)、[HAR](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/har-package-V5)、[HSP](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/in-app-hsp-V5)分别展开详细介绍。

| 规格                                                         | HAP  | HAR  | HSP  |
| ------------------------------------------------------------ | ---- | ---- | ---- |
| 支持在配置文件中声明[UIAbility](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/uiability-overview-V5)组件与[ExtensionAbility](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/extensionability-overview-V5)组件 | √    | ×    | ×    |
| 支持在配置文件中声明[pages](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/module-configuration-file-V5#pages标签)页面 | √    | ×    | √    |
| 支持包含资源文件与.so文件                                    | √    | √    | √    |
| 支持依赖其他HAR文件                                          | √    | √    | √    |
| 支持依赖其他HSP文件                                          | √    | √    | √    |
| 支持在设备上独立安装运行                                     | √    | ×    | ×    |



> [!WARNING]
>
> - HAR虽然不支持在配置文件中声明pages页面，但是可以包含pages页面，并通过[命名路由](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-routing-V5#命名路由)的方式进行跳转。
> - 由于HSP仅支持应用内共享，如果HAR依赖了HSP，则该HAR文件仅支持应用内共享，不支持发布到二方仓或三方仓供其他应用使用，否则会导致编译失败。
> - HAR和HSP均不支持循环依赖，也不支持依赖传递。





### UIAbility组件基础

概述：UIAbility组件是一种包含UI的应用组件，主要用于和用户交互，是应用程序的入口

UIAbility的设计理念：

​	原生支持应用组件级的跨端迁移和多端协同。

​	支持多设备和多窗口形态。

UIAbility划分原则与建议：

​	UIAbility组件是系统调度的基本单元，为应用提供绘制界面的窗口。一个应用可以包含一个或多个UIAbility组件。例如，在支付应用中，可以将入口功能和收付款功能分别配置为独立的UIAbility。

每一个UIAbility组件实例都会在最近任务列表中显示一个对应的任务。

对于开发者而言，可以根据具体场景选择单个还是多个UIAbility，划分建议如下：

- 如果开发者希望在任务视图中看到一个任务，建议使用“一个UIAbility+多个页面”的方式，可以避免不必要的资源加载。

- 如果开发者希望在任务视图中看到多个任务，或者需要同时开启多个窗口，建议使用多个UIAbility实现不同的功能。

  例如，即时通讯类应用中的消息列表与音视频通话采用不同的UIAbility进行开发，既可以方便地切换任务窗口，又可以实现应用的两个任务窗口在一个屏幕上分屏显示。

> [!WARNING]
>
> 任务视图用于快速查看和管理当前设备上运行的所有任务或应用。



#### 声明配置

为使应用能够正常使用UIAbility，需要在[module.json5配置文件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/module-configuration-file-V5)的[abilities标签](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/module-configuration-file-V5#abilities标签)中声明UIAbility的名称、入口、标签等相关信息。

```typescript
{
  "module": {
    // ...
    "abilities": [
      {
        "name": "EntryAbility", // UIAbility组件的名称
        "srcEntry": "./ets/entryability/EntryAbility.ets", // UIAbility组件的代码路径
        "description": "$string:EntryAbility_desc", // UIAbility组件的描述信息
        "icon": "$media:icon", // UIAbility组件的图标
        "label": "$string:EntryAbility_label", // UIAbility组件的标签
        "startWindowIcon": "$media:icon", // UIAbility组件启动页面图标资源文件的索引
        "startWindowBackground": "$color:start_window_background", // UIAbility组件启动页面背景颜色资源文件的索引
        // ...
      }
    ]
  }
}
```



#### UIAbility生命周期

UIAbility的生命周期包括Create、Foreground、Background、Destroy四个状态，如下图所示：

 ![UIAbility生命周期状态](.\res\UIAbility生命周期状态.png)



##### WindowStage状态

上图中不包含 WindowStageCreate和WindowStageDestroy状态，详细如下： ![UIAbility生命周期状态](.\res\WindowStageCreate和WindowStageDestroy状态.png)



在onWindowStageCreate()回调中通过[loadContent()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-window-V5#loadcontent9)方法设置应用要加载的页面，并根据需要调用[on('windowStageEvent')](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-window-V5#onwindowstageevent9)方法订阅[WindowStage的事件](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-window-V5#windowstageeventtype9)（获焦/失焦、切到前台/切到后台、前台可交互/前台不可交互）。

```typescript

onWindowStageCreate(windowStage: window.WindowStage): void {
    // 设置WindowStage的事件订阅（获焦/失焦、切到前台/切到后台、前台可交互/前台不可交互）
    try {
      windowStage.on('windowStageEvent', (data) => {
        let stageEventType: window.WindowStageEventType = data;
        switch (stageEventType) {
          case window.WindowStageEventType.SHOWN: // 切到前台
            hilog.info(DOMAIN_NUMBER, TAG, `windowStage foreground.`);
            break;
          case window.WindowStageEventType.ACTIVE: // 获焦状态
            hilog.info(DOMAIN_NUMBER, TAG, `windowStage active.`);
            break;
          case window.WindowStageEventType.INACTIVE: // 失焦状态
            hilog.info(DOMAIN_NUMBER, TAG, `windowStage inactive.`);
            break;
          case window.WindowStageEventType.HIDDEN: // 切到后台
            hilog.info(DOMAIN_NUMBER, TAG, `windowStage background.`);
            break;
          case window.WindowStageEventType.RESUMED: // 前台可交互状态
            hilog.info(DOMAIN_NUMBER, TAG, `windowStage resumed.`);
            break;
          case window.WindowStageEventType.PAUSED: // 前台不可交互状态
            hilog.info(DOMAIN_NUMBER, TAG, `windowStage paused.`);
            break;
          default:
            break;
        }
      });
    } catch (exception) {
      hilog.error(DOMAIN_NUMBER, TAG,
        `Failed to enable the listener for window stage event changes. Cause: ${JSON.stringify(exception)}`);
    }
    hilog.info(DOMAIN_NUMBER, TAG, `%{public}s`, `Ability onWindowStageCreate`);
    // 设置UI加载
    windowStage.loadContent('pages/Index', (err, data) => {
      // ...
    });
  }
```

> [!TIP]
>
> WindowStage的相关使用请参见[窗口开发指导](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/application-window-stage-V5)。

对应onWindowStageWillDestroy()回调，在WindowStage销毁前执行，此时WindowStage可以使用。此时可以释放通过windowStage对象获取的资源。

对应于onWindowStageCreate()回调，在UIAbility实例销毁之前，则会先进入onWindowStageDestroy()回调，可以在该回调中释放UI资源。例如在onWindowStageDestroy()中注销WindowStage事件订阅（获焦/失焦、切到前台/切到后台、前台可交互/前台不可交互）



##### Foreground和Background状态

Foreground和Background状态分别在[UIAbility](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5)实例切换至前台和切换至后台时触发，对应于[onForeground()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5#uiabilityonforeground)回调和[onBackground()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5#uiabilityonbackground)回调。

onForeground()回调，在UIAbility的UI可见之前，如UIAbility切换至前台时触发。可以在onForeground()回调中申请系统需要的资源，或者重新申请在onBackground()中释放的资源。

onBackground()回调，在UIAbility的UI完全不可见之后，如UIAbility切换至后台时候触发。可以在onBackground()回调中释放UI不可见时无用的资源，或者在此回调中执行较为耗时的操作，例如状态保存等。

例如应用在使用过程中需要使用用户定位时，假设应用已获得用户的定位权限授权。在UI显示之前，可以在onForeground()回调中开启定位功能，从而获取到当前的位置信息。

当应用切换到后台状态，可以在onBackground()回调中停止定位功能，以节省系统的资源消耗。

```typescript
import { UIAbility } from '@kit.AbilityKit';
export default class EntryAbility extends UIAbility {  // ...
  onForeground(): void {    
      // 申请系统需要的资源，或者重新申请在onBackground()中释放的资源  
  }
  onBackground(): void {    
      // 释放UI不可见时无用的资源，或者在此回调中执行较为耗时的操作    // 例如状态保存等  
  }
}
```

当应用的UIAbility实例已创建，且UIAbility配置为[singleton](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/uiability-launch-type-V5#singleton启动模式)启动模式时，再次调用[startAbility()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-inner-application-uiabilitycontext-V5#uiabilitycontextstartability)方法启动该UIAbility实例时，只会进入该UIAbility的[onNewWant()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5#uiabilityonnewwant)回调，不会进入其[onCreate()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5#uiabilityoncreate)和[onWindowStageCreate()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5#uiabilityonwindowstagecreate)生命周期回调。应用可以在该回调中更新要加载的资源和数据等，用于后续的UI展示。

```typescript
import { AbilityConstant, UIAbility, Want } from '@kit.AbilityKit';
export default class EntryAbility extends UIAbility {  // ...
  onNewWant(want: Want, launchParam: AbilityConstant.LaunchParam) {    
      // 更新资源、数据  
  }}
```



##### Destroy状态

Destroy状态在[UIAbility](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5)实例销毁时触发。可以在onDestroy()回调中进行系统资源的释放、数据的保存等操作。

例如，调用[terminateSelf()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-inner-application-uiabilitycontext-V5#uiabilitycontextterminateself)方法停止当前UIAbility实例，执行onDestroy()回调，并完成UIAbility实例的销毁。

```typescript
import { UIAbility } from '@kit.AbilityKit';
export default class EntryAbility extends UIAbility {  // ...
  onDestroy() {    
      // 系统资源的释放、数据的保存等  
  }}
```





#### **UIAbility组件启动模式**

- singleton（单实例模式）
- multiton（多实例模式）
- specified（指定实例模式）



##### singleton启动模式

singleton启动模式为单实例模式，也是默认情况下的启动模式。

每次调用startAbility()方法时，如果应用进程中该类型的UIAbility实例已经存在，则复用系统中的UIAbility实例。系统中只存在唯一一个该UIAbility实例，即在最近任务列表中只存在一个该类型的UIAbility实例，如果再次调用startAbility()方法启动该UIAbility实例。由于启动的还是原来的UIAbility实例，并未重新创建一个新的UIAbility实例，此时只会进入该UIAbility的onNewWant()回调，不会进入其onCreate()和onWindowStageCreate()生命周期回调。



##### multiton启动模式

multiton启动模式为多实例模式，每次调用startAbility()方法时，都会在应用进程中创建一个新的该类型UIAbility实例。即在最近任务列表中可以看到有多个该类型的UIAbility实例。这种情况下可以将UIAbility配置为multiton（多实例模式）。



##### specified启动模式

specified启动模式为指定实例模式，针对一些特殊场景使用（例如文档应用中每次新建文档希望都能新建一个文档实例，重复打开一个已保存的文档希望打开的都是同一个文档实例）。

简单来说，创建UIAbility实例之前，开发者可以为该实例指定一个唯一的字符串Key，这样在调用startAbility()方法时，应用就可以根据指定的Key来识别响应请求的UIAbility实例。在EntryAbility中，调用startAbility()方法时，可以在[want](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-want-V5)参数中增加一个自定义参数，例如instanceKey，以此来区分不同的UIAbility实例。如果匹配到已启动的实例，那么只会拉回到前台，并且获得焦点，如果没有匹配到才会创建。

```typescript
Button()// ...
           .onClick(() => {
             let context: common.UIAbilityContext = getContext(this) as common.UIAbilityContext;
             // context为调用方UIAbility的UIAbilityContext;
             let want: Want = {
               deviceId: '', // deviceId为空表示本设备
               bundleName: 'com.samples.stagemodelabilitydevelop',
               abilityName: 'SpecifiedFirstAbility',
               moduleName: 'entry', // moduleName非必选
               parameters: {
                 // 自定义信息
                 instanceKey: this.KEY_NEW
               }
             };
             context.startAbility(want).then(() => {
               hilog.info(DOMAIN_NUMBER, TAG, 'Succeeded in starting SpecifiedAbility.');
             }).catch((err: BusinessError) => {
               hilog.error(DOMAIN_NUMBER, TAG, `Failed to start SpecifiedAbility. Code is ${err.code}, message is ${err.message}`);
             })
             this.KEY_NEW = this.KEY_NEW + 'a';
           })
```



#### **UIAbility组件基本用法**

##### 指定UIAbility的启动页面

应用中的UIAbility在启动过程中，需要指定启动页面，否则应用启动后会因为没有默认加载页面而导致白屏。可以在UIAbility的onWindowStageCreate()生命周期回调中，通过WindowStage对象的loadContent()方法设置启动页面。

```typescript
import { UIAbility } from '@kit.AbilityKit';
import { window } from '@kit.ArkUI';

export default class EntryAbility extends UIAbility {
  onWindowStageCreate(windowStage: window.WindowStage): void {
    // Main window is created, set main page for this ability
    windowStage.loadContent('pages/Index', (err, data) => {
      // ...
    });
  }
  // ...
}
```



##### 获取UIAbility的上下文信息

UIAbility类拥有自身的上下文信息，该信息为UIAbilityContext类的实例，UIAbilityContext类拥有abilityInfo、currentHapModuleInfo等属性。通过UIAbilityContext可以获取UIAbility的相关配置信息，如包代码路径、Bundle名称、Ability名称和应用程序需要的环境状态等属性信息，以及可以获取操作UIAbility实例的方法（如startAbility()、connectServiceExtensionAbility()、terminateSelf()等）。

如果需要在页面中获得当前Ability的Context，可调用getContext接口获取当前页面关联的UIAbilityContext或ExtensionContext。

- 在UIAbility中可以通过this.context获取UIAbility实例的上下文信息。

  ```typescript
  import { UIAbility, AbilityConstant, Want } from '@kit.AbilityKit';
  
  export default class EntryAbility extends UIAbility {
    onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
      // 获取UIAbility实例的上下文
      let context = this.context;
      // ...
    }
  }
  ```

- 在页面中获取UIAbility实例的上下文信息，包括导入依赖资源context模块和在组件中定义一个context变量两个部分。

  ```typescript
  import { common, Want } from '@kit.AbilityKit';
  
  @Entry
  @Component
  struct Page_EventHub {
    private context = getContext(this) as common.UIAbilityContext;
  
    startAbilityTest(): void {
      let want: Want = {
        // Want参数信息
      };
      this.context.startAbility(want);
    }
  
    // 页面展示
    build() {
      // ...
    }
  }
  ```

  也可以在导入依赖资源context模块后，在具体使用UIAbilityContext前进行变量定义。

  ```typescript
  import { common, Want } from '@kit.AbilityKit';
  
  @Entry
  @Component
  struct Page_UIAbilityComponentsBasicUsage {
    startAbilityTest(): void {
      let context = getContext(this) as common.UIAbilityContext;
      let want: Want = {
        // Want参数信息
      };
      context.startAbility(want);
    }
  
    // 页面展示
    build() {
      // ...
    }
  }
  ```



#### **UIAbility组件与UI的数据同步**

基于当前的应用模型，可以通过以下几种方式来实现UIAbility组件与UI之间的数据同步。

- [使用EventHub进行数据通信](https://developer.huawei.com/consumer/cn/doc/#使用eventhub进行数据通信)：在[基类Context](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/application-context-stage-V5)中提供了[EventHub](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-inner-application-eventhub-V5)对象，可以通过发布订阅方式来实现事件的传递。在事件传递前，订阅者需要先进行订阅，当发布者发布事件时，订阅者将接收到事件并进行相应处理。
- [使用AppStorage/LocalStorage进行数据同步](https://developer.huawei.com/consumer/cn/doc/#使用appstoragelocalstorage进行数据同步)：ArkUI提供了[AppStorage](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-appstorage-V5)和[LocalStorage](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-localstorage-V5)两种应用级别的状态管理方案，可用于实现应用级别和UIAbility级别的数据同步。



基于当前的应用模型，可以通过以下几种方式来实现[UIAbility](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5)组件与UI之间的数据同步。

- [使用EventHub进行数据通信](https://developer.huawei.com/consumer/cn/doc/html/harmonyos-guides-V5/uiability-data-sync-with-ui-V5#使用eventhub进行数据通信)：在[基类Context](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/application-context-stage-V5)中提供了[EventHub](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-inner-application-eventhub-V5)对象，可以通过发布订阅方式来实现事件的传递。在事件传递前，订阅者需要先进行订阅，当发布者发布事件时，订阅者将接收到事件并进行相应处理。
- [使用AppStorage/LocalStorage进行数据同步](https://developer.huawei.com/consumer/cn/doc/html/harmonyos-guides-V5/uiability-data-sync-with-ui-V5#使用appstoragelocalstorage进行数据同步)：ArkUI提供了[AppStorage](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-appstorage-V5)和[LocalStorage](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-localstorage-V5)两种应用级别的状态管理方案，可用于实现应用级别和UIAbility级别的数据同步。



##### 使用EventHub进行数据通信

[EventHub](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-inner-application-eventhub-V5)为[UIAbility](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5)组件提供了事件机制，使它们能够进行订阅、取消订阅和触发事件等数据通信能力。

在[基类Context](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/application-context-stage-V5)中，提供了EventHub对象，可用于在UIAbility组件实例内通信。使用EventHub实现UIAbility与UI之间的数据通信需要先获取EventHub对象，本章节将以此为例进行说明。

1. 在UIAbility中调用[eventHub.on()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-inner-application-eventhub-V5#eventhubon)方法注册一个自定义事件“event1”，eventHub.on()有如下两种调用方式，使用其中一种即可。

   ```typescript
   import { hilog } from '@kit.PerformanceAnalysisKit';
   import { UIAbility, Context, Want, AbilityConstant } from '@kit.AbilityKit';
   
   const DOMAIN_NUMBER: number = 0xFF00;
   const TAG: string = '[EventAbility]';
   
   export default class EntryAbility extends UIAbility {
     onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
       // 获取eventHub
       let eventhub = this.context.eventHub;
       // 执行订阅操作
       eventhub.on('event1', this.eventFunc);
       eventhub.on('event1', (data: string) => {
         // 触发事件，完成相应的业务操作
       });
       hilog.info(DOMAIN_NUMBER, TAG, '%{public}s', 'Ability onCreate');
     }
   
     // ...
     eventFunc(argOne: Context, argTwo: Context): void {
       hilog.info(DOMAIN_NUMBER, TAG, '1. ' + `${argOne}, ${argTwo}`);
       return;
     }
   }
   ```

2. 在UI中通过[eventHub.emit()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-inner-application-eventhub-V5#eventhubemit)方法触发该事件，在触发事件的同时，根据需要传入参数信息。

   ```typescript
   import { common } from '@kit.AbilityKit';
   import { promptAction } from '@kit.ArkUI';
   
   @Entry
   @Component
   struct Page_EventHub {
     private context = getContext(this) as common.UIAbilityContext;
   
     eventHubFunc(): void {
       // 不带参数触发自定义“event1”事件
       this.context.eventHub.emit('event1');
       // 带1个参数触发自定义“event1”事件
       this.context.eventHub.emit('event1', 1);
       // 带2个参数触发自定义“event1”事件
       this.context.eventHub.emit('event1', 2, 'test');
       // 开发者可以根据实际的业务场景设计事件传递的参数
     }
   
     build() {
       Column() {
         // ...
         List({ initialIndex: 0 }) {
           ListItem() {
             Row() {
               // ...
             }
             .onClick(() => {
               this.eventHubFunc();
               promptAction.showToast({
                 message: 'EventHubFuncA'
               });
             })
           }
   
           // ...
           ListItem() {
             Row() {
               // ...
             }
             .onClick(() => {
               this.context.eventHub.off('event1');
               promptAction.showToast({
                 message: 'EventHubFuncB'
               });
             })
           }
           // ...
         }
         // ...
       }
       // ...
     }
   }
   ```

3. 在UIAbility的注册事件回调中可以得到对应的触发事件结果，运行日志结果如下所示。

   ```json
   [Example].[Entry].[EntryAbility] 1. []
   [Example].[Entry].[EntryAbility] 1. [1]
   [Example].[Entry].[EntryAbility] 1. [2,"test"]
   ```

4. 在自定义事件“event1”使用完成后，可以根据需要调用[eventHub.off()](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-inner-application-eventhub-V5#eventhuboff)方法取消该事件的订阅。

   ```typescript
   import { UIAbility } from '@kit.AbilityKit';
   
   export default class EntryAbility extends UIAbility {
     // ... 
     onDestroy(): void {
       this.context.eventHub.off('event1');
     }
   }
   ```



##### 使用AppStorage/LocalStorage进行数据同步

ArkUI提供了[AppStorage](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-appstorage-V5)和[LocalStorage](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-localstorage-V5)两种应用级别的状态管理方案，可用于实现应用级别和[UIAbility](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-app-ability-uiability-V5)级别的数据同步。使用这些方案可以方便地管理应用状态，提高应用性能和用户体验。其中，AppStorage是一个全局的状态管理器，适用于多个UIAbility共享同一状态数据的情况；而LocalStorage则是一个局部的状态管理器，适用于单个UIAbility内部使用的状态数据。通过这两种方案，开发者可以更加灵活地控制应用状态，提高应用的可维护性和可扩展性。详细请参见[应用级变量的状态管理](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-application-state-management-overview-V5)。



#### **UIAbility组件间交互（设备内）**

UIAbility是系统调度的最小单元。在设备内的功能模块之间跳转时，会涉及到启动特定的UIAbility，包括应用内的其他UIAbility、或者其他应用的UIAbility（例如启动三方支付UIAbility）。

- 启动应用内的UIAbility
- 启动应用内的UIAbility并获取返回结果
- 启动UIAbility的指定页面

##### 启动应用内的UIAbility

可以使用startAbility()函数，其中want参数可以携带数据，类似android的startActivity

```typescript
{
            // context为Ability对象的成员，在非Ability对象内部调用需要
            // 将Context对象传递过去
            let wantInfo: Want = {
              deviceId: '', // deviceId为空表示本设备
              bundleName: 'com.samples.stagemodelabilitydevelop',
              moduleName: 'entry', // moduleName非必选
              abilityName: 'FuncAbilityA',
              parameters: {
                // 自定义信息
                info: '来自EntryAbility Page_UIAbilityComponentsInteractive页面'
              },
            };
            // context为调用方UIAbility的UIAbilityContext
            this.context.startAbility(wantInfo).then(() => {
              hilog.info(DOMAIN_NUMBER, TAG, 'startAbility success.');
            }).catch((error: BusinessError) => {
              hilog.error(DOMAIN_NUMBER, TAG, 'startAbility failed.');
            });
          }

//接收UIAbility
import { AbilityConstant, UIAbility, Want } from '@kit.AbilityKit';

export default class FuncAbilityA extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    // 接收调用方UIAbility传过来的参数
    let funcAbilityWant = want;
    let info = funcAbilityWant?.parameters?.info;
  }
  //...
}
```

> [!WARNING]
>
> - 在被拉起的FuncAbility中，可以通过获取传递过来的want参数的parameters来获取拉起方UIAbility的PID、Bundle Name等信息
> - 在FuncAbility业务完成之后，如需要停止当前UIAbility实例，在FuncAbility中通过调用terminateSelf()方法实现
> - 调用terminateSelf()方法停止当前UIAbility实例时，默认会保留该实例的快照（Snapshot），即在最近任务列表中仍然能查看到该实例对应的任务。如不需要保留该实例的快照，可以在其对应UIAbility的module.json5配置文件中，将abilities标签的removeMissionAfterTerminate字段配置为true
> - 如需要关闭应用所有的UIAbility实例，可以调用ApplicationContext的killAllProcesses()方法实现关闭应用所有的进程



##### 启动应用内的UIAbility并获取返回结果

使用startAbilityForResult()函数启动Ability，类似android的startActivityforResult

```typescript
 {
            let context: common.UIAbilityContext = getContext(this) as common.UIAbilityContext; // UIAbilityContext
            const RESULT_CODE: number = 1001;
            let want: Want = {
              deviceId: '', // deviceId为空表示本设备
              bundleName: 'com.samples.stagemodelabilitydevelop',
              moduleName: 'entry', // moduleName非必选
              abilityName: 'FuncAbilityA',
              parameters: {
                // 自定义信息
                info: '来自EntryAbility UIAbilityComponentsInteractive页面'
              }
            };
            context.startAbilityForResult(want).then((data) => {
              if (data?.resultCode === RESULT_CODE) {
                // 解析被调用方UIAbility返回的信息
                let info = data.want?.parameters?.info;
                hilog.info(DOMAIN_NUMBER, TAG, JSON.stringify(info) ?? '');
                if (info !== null) {
                  promptAction.showToast({
                    message: JSON.stringify(info)
                  });
                }
              }
              hilog.info(DOMAIN_NUMBER, TAG, JSON.stringify(data.resultCode) ?? '');
            }).catch((err: BusinessError) => {
              hilog.error(DOMAIN_NUMBER, TAG, `Failed to start ability for result. Code is ${err.code}, message is ${err.message}`);
            });
          }



```



而在FuncAbility停止自身时，需要调用terminateSelfWithResult()方法，入参abilityResult为FuncAbility需要返回给EntryAbility的信息

```typescript
{
            let context: common.UIAbilityContext = getContext(this) as common.UIAbilityContext; // UIAbilityContext
            const RESULT_CODE: number = 1001;
            let abilityResult: common.AbilityResult = {
              resultCode: RESULT_CODE,
              want: {
                bundleName: 'com.samples.stagemodelabilitydevelop',
                moduleName: 'entry', // moduleName非必选
                abilityName: 'FuncAbilityB',
                parameters: {
                  info: '来自FuncAbility Index页面'
                },
              },
            };
            context.terminateSelfWithResult(abilityResult, (err) => {
              if (err.code) {
                hilog.error(DOMAIN_NUMBER, TAG, `Failed to terminate self with result. Code is ${err.code}, message is ${err.message}`);
                return;
              }
            });
          }
```



##### 启动UIAbility的指定页面

UIAbility的启动分为两种情况：UIAbility冷启动和UIAbility热启动。

- UIAbility冷启动：指的是UIAbility实例处于完全关闭状态下被启动，这需要完整地加载和初始化UIAbility实例的代码、资源等。
- UIAbility热启动：指的是UIAbility实例已经启动并在前台运行过，由于某些原因切换到后台，再次启动该UIAbility实例，这种情况下可以快速恢复UIAbility实例的状态。

###### 目标UIAbility冷启动

目标UIAbility冷启动时，在目标UIAbility的onCreate()生命周期回调中，接收调用方传过来的参数。然后在目标UIAbility的onWindowStageCreate()生命周期回调中，解析调用方传递过来的want参数，获取到需要加载的页面信息url，传入windowStage.loadContent()方法。

###### 目标UIAbility热启动

在应用开发中，会遇到目标UIAbility实例之前已经启动过的场景，这时再次启动目标UIAbility时，不会重新走初始化逻辑，只会直接触发onNewWant()生命周期方法。为了实现跳转到指定页面，需要在onNewWant()中解析参数进行处理。

> [!WARNING]
>
> 当被调用方UIAbility组件启动模式设置为multiton启动模式时，每次启动都会创建一个新的实例，那么onNewWant()回调就不会被用到。



#### 布局开发流程，常用文档

| 任务                         | 简介                                                         | 相关指导                                                     |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 学习ArkTS                    | 介绍了ArkTS的基本语法、状态管理和渲染控制的场景。            | - [基本语法](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-basic-syntax-overview-V5)- [状态管理](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-state-management-overview-V5)- [渲染控制](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-rendering-control-ifelse-V5) |
| 开发布局                     | 介绍了几种常用的布局方式。                                   | - [常用布局](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-layout-development-overview-V5) |
| 添加组件                     | 介绍了几种常用的系统组件、以及通过API方式支持的界面元素。    | - [常用组件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-common-components-button-V5)- [自定义组件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-create-custom-components-V5)- [气泡和菜单](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-popup-and-menu-components-popup-V5) |
| 设置组件导航和页面路由       | 介绍了如何设置组件间的导航以及页面路由。                     | - [组件导航（推荐）](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-navigation-navigation-V5)- [页面路由](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-routing-V5) |
| 使用文本                     | 介绍了输入框、富文本和属性字符串等文本组件的使用方法。       | - [文本显示](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-common-components-text-display-V5)- [文本输入](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-common-components-text-input-V5)- [富文本](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-common-components-richeditor-V5)- [图标小符号](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-common-components-symbol-V5)- [属性字符串](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-styled-string-V5) |
| 使用弹窗                     | 介绍了模态弹窗和自定义弹窗的使用方法。                       | - [模态弹窗](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-modal-dialog-V5)- [自定义弹窗](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-uicontext-custom-dialog-V5) |
| 显示图形                     | 介绍了如何显示图片、绘制自定义几何图形以及使用画布绘制自定义图形。 | - [几何图形](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-geometric-shape-drawing-V5)- [画布](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-drawing-customization-on-canvas-V5) |
| 使用动画                     | 介绍了组件和页面使用动画的典型场景。                         | - [属性动画](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-attribute-animation-overview-V5)- [转场动画](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-transition-overview-V5)-[粒子动画](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-particle-animation-V5)- [组件动画](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-component-animation-V5)- [动画曲线](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-traditional-curve-V5)- [动画衔接](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-animation-smoothing-V5)- [动画效果](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-blur-effect-V5)- [帧动画](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-animator-V5) |
| 绑定事件                     | 介绍了事件的基本概念和如何使用通用事件和手势事件。           | - [通用事件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-common-events-touch-screen-event-V5)- [手势事件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-gesture-events-binding-V5) |
| 使用自定义能力               | 介绍了自定义能力的基本概念和如何使用自定义能力。             | - [自定义节点](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-user-defined-node-V5)- [自定义扩展](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-user-defined-modifier-V5) |
| 使用镜像能力                 | 介绍了镜像能力的基本概念和如何使用镜像能力。                 | - [使用镜像能力](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-mirroring-display-V5) |
| 支持适老化                   | 介绍了适老化的使用场景和使用方法。                           | - [支持适老化](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkui-support-for-aging-adaptation-V5) |
| 主题设置                     | 介绍了应用级和页面级的主题设置能力。                         | - [应用深浅色适配](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-light-dark-color-adaptation-V5)- [设置主题换肤](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/theme_skinning-V5) |
| Stage模型下ArkUI全局接口开发 | 介绍了如何使用UIContext中对应的接口获取与实例绑定的对象。    | - [Stage模型下ArkUI全局接口开发指导](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-global-interface-V5) |
| 使用NDK接口构建UI            | 介绍了ArkUI NDK接口提供的能力，以及如何通过NDK接口创建UI界面。 | - [接入ArkTS页面](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ndk-access-the-arkts-page-V5)- [添加交互事件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ndk-listen-to-component-events-V5)- [使用动画](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ndk-use-animation-V5)- [使用懒加载开发长列表界面](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ndk-loading-long-list-V5)- [构建弹窗](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ndk-build-pop-up-window-V5)- [构建自定义组件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ndk-build-custom-components-V5)- [嵌入ArkTS组件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ndk-embed-arkts-components-V5) |



#### 通用规则

- **默认单位**

  表示长度的入参单位默认为vp，即入参为number类型、以及[Length](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-types-V5#length)和[Dimension](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/ts-types-V5#dimension10)类型中的number单位为vp。

- **异常值处理**

  输入的参数为异常（undefined，null或无效值）时，处理规则如下：

  （1）对应参数有默认值，按默认值处理；

  （2）对应参数无默认值，该参数对应的属性或接口不生效。



##### 像素单位相关

| 名称  | 描述                                                         |
| :--- | :----------------------------------------------------------- |
| px    | 屏幕物理像素单位。                                           |
| vp    | 屏幕密度相关像素，根据屏幕像素密度转换为屏幕物理像素，当数值不带单位时，默认单位vp。<br>**说明：**vp与px的比例与屏幕像素密度有关。 |
| fp    | 字体像素，与vp类似适用屏幕密度变化，随系统字体大小设置变化。 |
| lpx   | 视窗逻辑像素单位，lpx单位为实际屏幕宽度与逻辑宽度（通过[designWidth](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/module-configuration-file-V5#pages标签)配置）的比值，designWidth默认值为720。<br>当designWidth为720时，在实际宽度为1440物理像素的屏幕上，1lpx为2px大小。 |



| 接口                            | 描述                                                         |
| :------------------------------ | :----------------------------------------------------------- |
| vp2px(value : number) : number  | 将vp单位的数值转换为以px为单位的数值。<br>**说明：**默认使用当前UI实例所在屏幕的虚拟像素比进行转换，UI实例未创建时，使用默认屏幕的虚拟像素比进行转换。 |
| px2vp(value : number) : number  | 将px单位的数值转换为以vp为单位的数值。<br>**说明：**默认使用当前UI实例所在屏幕的虚拟像素比进行转换，UI实例未创建时，使用默认屏幕的虚拟像素比进行转换。 |
| fp2px(value : number) : number  | 将fp单位的数值转换为以px为单位的数值。                       |
| px2fp(value : number) : number  | 将px单位的数值转换为以fp为单位的数值。                       |
| lpx2px(value : number) : number | 将lpx单位的数值转换为以px为单位的数值。                      |
| px2lpx(value : number) : number | 将px单位的数值转换为以lpx为单位的数值。                      |



#### Web加载相关

ArkWeb提供了web组件，及webview两个来进行web的加载。

通常使用web组件，可以加载在线网页、本地网页、HTML格式文本数据。

使用**`WebviewController`**来进行能力管理。