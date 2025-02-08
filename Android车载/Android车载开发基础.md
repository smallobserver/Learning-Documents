# Android车载开发基础

## 常见车载操作系统

车载操作系统主要应用于中控、仪表和 `T-box`， 提供车载信息娱乐服务，可具备网联功能，提供导航、多媒体娱乐、语音、 辅助驾驶、AI 等高级功能。目前常见的车载操作系统有哪些呢？

![车载操作系统对比图.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/013d953c991b420eb3ff2a98fc3a7717~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1253&h=561&s=399529&e=png&b=fcf8f8)

### QNX

`QNX`是一款微内核、嵌入式、非开源、安全实时的操作系统，由加拿大QSSL公司开发，`2004`年，哈曼国际将QNX收入囊中，2010年，BlackBerry母公司RIM又从哈曼国际手中收购QNX。像通用、奥迪、宝马、保时捷等大厂都在使用 QNX。

`QNX`是遵从`POSIX`规范的类`Unix`实时操作系统，目标市场主要是面向嵌入式系统，**具备高运行效率、高可靠性特点，并在工控领域拥有近40年的使用经验**，被广泛应用于汽车、轨道交通、航空航天等对安全性、实时性要求较高的领域。

`QNX`是**微内核架构**，内核一般只有几十KB，驱动程序、协议栈、文件系统、应用程序等都在微内核之外的、受内存保护的空间内运行，可实现组件之间相互独立，避免因程序指针错误造成内核故障。因其内核小巧，运行速度极快，具有独特的微内核架构，安全和稳定性高，不易受病毒破坏系统，是全球首款通过ISO 26262 ASIL-D安全认证的实时操作系统，常用于安全稳定性要求较高的数字仪表中。

但是，  QNX的缺点也十分明显：

**高昂的授权使用费用、安全性带来的兼容性问题以及开放性不足导致的应用生态缺乏**都是`QNX`看得见的天花板。

### Linux

`Linux `作为一款开源、高效、灵活、功能强大的操作系统，**最大优势是具备很强的定制开发灵活度 **。

由于可定制性强，特斯拉在`Linux`基础上开发出了完全适配旗下车辆的车载系统；阿里的`AliOS`也是基于`Linux`开发，目前已经应用在上汽荣威、上汽名爵等多款车型上。

2014年，`Linux`基金会赞助并发布了开源**AGL（Automotive Grade Linux）\**规范 1.0 版本，它是\**首个开放式车载信息娱乐软件规范**。`AGL`是一个协作开源项目，由`Linux`基金会管理，将汽车制造商、供应商和科技公司聚集在一起，以加速开发和使用完全开放的智能网联汽车软件堆栈。`AGL`设立的最初目的，是**提供一个车规级的信息娱乐系统**，但随着自动驾驶的发展，未来还会加入更多的功能，不仅会融合仪表盘、舱内控制的功能，还会覆盖自动驾驶的相关功能。

`AGL`已经获得了多家主机厂的支持，包括丰田、大众、戴姆勒、现代、马自达、本田、三菱、斯巴鲁、日产、上汽等。加入`AGL`的好处是，**70%的操作系统开发代码（包括操作系统、中间件和应用程序框架）已经由AGL编写完成了，剩下30%由车企个性化定制开发**。主机厂不仅获得了操作系统掌控权，还大大缩短了开发进程，降低了开发成本。

### Android：AAOS

`Android`系统是基于`Linux`内核开发的最成功的产品，Google 于 2019 年开放了 `Android Automotive`，原本为移动互联设备开发的 `Android` 应用生态可迁移到 `Android Automotive`。

国内厂家在车载信息娱乐应用中主要采用`Android`系统，尤其是各大互联网巨头、自主品牌和造车新势力纷纷基于`Android`进行定制化改造，推出自己的汽车操作系统。

`Android Automotive OS`(以下简称`AAOS`)，是一款基于 `Android` 的车载信息娱乐系统。车载系统是专为提升驾驶体验而优化的独立` Android` 设备。用户可直接将应用安装到车载系统上，而不是手机上。

`AAOS` 为汽车信息娱乐系统和主机提供了开放性、定制性和扩展性。其中开放性通过在免费和开源代码库中提供基本的汽车信息娱乐功能来提高效率，其定制性和扩展性，可以让开发者根据需要进行定制和扩展特定功能。

优势：开源，定制灵活，应用可移植性强，应用生态最为丰富

缺点：但是安全性和稳定性相对不足。

### 鸿蒙：HarmonyOS

鸿蒙（即HarmonyOS）是华为公司自2012年以来开发的一款可兼容`AOSP`的操作系统。系统性能包括利用“分布式”技术将各款设备融合成一个“超级终端”，便于操作和共享各设备资源。

系统架构支持多内核，包括`Linux`内核、`LiteOS`和鸿蒙微内核，可按各种智能设备选择所需内核，在手机、平板以及`PC`等大内存设备上，系统采用`Linux`内核和`OpenHarmony`框架以运行鸿蒙应用程序，同时利用`AOSP`框架以运行安卓应用。在手表及物联网相关设备上，系统采用`LiteOS`内核以运行轻量的鸿蒙应用程序。

## 车载操作系统架构

车载操作系统架构分为**单系统架构**和**多系统架构**。两类架构都可实现一芯多屏（多屏融合、多屏互动）、单屏多系统（虚拟运行环境、多应用生态融合）、一芯多功能（信息娱乐、T-box 等）。

### 单系统架构

单系统架构仅涉及单个车载操作系统的架构，由车载操作系统内 核、基础库、基础服务、运行环境及程序运行框架组成。车载操作系统对底层硬件和上层应用程序提供统一的接口，实现车载操作系统与 硬件和上层应用程序的解耦。

[车载操作系统架构研究报告](https://link.juejin.cn?target=http%3A%2F%2Fwww.catarc.org.cn%2Fupload%2F202109%2F22%2F202109221131141024.pdf)中，给出了单系统架构和功能要求建议：

![车载研究报告架构图.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b64cf7a3b9e47b7af3a1f332ca7ecaf~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1480&h=1100&s=1665050&e=png&b=fdfdfd)

### 多系统架构

多系统架构是同一套硬件之上运行多个车载操作系统单系统的架构，可分为硬件隔离、虚拟机管理器、容器三类多系统基础架构，以及两类或三类基础架构的混合架构，用于满足不同功能、性能和安全的隔离需求。

关于多系统架构的种类比较多和复杂，这里不做过多讲解，感兴趣的可以通过[车载操作系统架构研究报告](https://link.juejin.cn?target=http%3A%2F%2Fwww.catarc.org.cn%2Fupload%2F202109%2F22%2F202109221131141024.pdf)进一步学习了解。

### 主流操作系统架构

车载操作系统从规范到落地实现，会存在很多不同的架构，但国内主流车载操作系统架构还是很大的相似性，我们可以以下图的架构来进行一个初步的学习和熟悉：

![常见车载操作系统架构.webp](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/afbc8f12c93842cdbfa71c3cde2f6153~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1512&h=1427&s=33366&e=webp&b=fdfdfd)

可以自下而上的看，也就是从硬件到最上面的应用层，会发现很多没见过的概念，不过没关系，下面就会解释下每一层对应的介绍：

#### 硬件

##### T-BOX

`Tbox`是汽车上的一个盒子，指`Telematics Box`，远程通信终端，集成车身网络和无线通讯功能的产品，可提供`Telematics`业务，一般安装在仪表盘下方。`Tbox`是一个基于`Android`、`Linux`操作系统的带通讯功能的盒子，内含一张`SIM`卡，一般是中国联通和移动的`SIM`卡，与这个盒子配套硬件还有`GPS`天线，4G天线等。车机要联网必须有`Tbox`设备才能实现。

`TBOX`的功能模块主要包括4G模块、GPS模块、蓝牙模块、以太网模块、CAN通信模块、电话语言模块、电源模块、Airbag模块、E/B-call模块等，每个模块之间紧密联系在一起，形成一个完整的远程通信终端。

##### VCU

`VCU`是实现整车控制决策的核心电子控制单元，一般仅新能源汽车配备、传统燃油车无需该装置。`VCU`通过采集油门踏板、挡位、刹车踏板等信号来判断驾驶员的驾驶意图；通过监测车辆状态(车速、温度等)信息，由`VCU`判断处理后，向动力系统、动力电池系统发送车辆的运行状态控制指令，同时控制车载附件电力系统的工作模式；`VCU`具有整车系统故障诊断保护与存储功能。

##### ADAS

`ADAS`是`Advanced Driver Assistance System`的简称，翻译成中文的意思就是高级驾驶辅助系统。

翻译成白话文就是，就是利用安装在车上的各式各样传感器收集数据，并结合地图数据进行系统计算，从而预先为驾驶者判断可能发生的危险，保证行车的安全性，

这里我们要明确一个概念，`ADAS`不是现在非常红的自动驾驶，可以说这两者的研究重点完全不同。`ADAS`是辅助驾驶，核心是环境感知，而自动驾驶则是人工智能，体系有很大差别。

##### BMS

`BMS（battery management system）`电池管理系统。

BMS 是一套嵌入式系统，由硬件和软件共同组成。BMS 系统主要完成的功能：管理多节锂电池组成的电池包，实现电池包的充放电管理、安全保护、信息监控等功能。

`BMS`系统主要解决什么问题：

⨳ 充放电保护，如：过充保护、过放保护、过温保护等。

⨳ 充放电信息监控，如：剩余电量检测等。

⨳ 电池本身状态监控，如：剩余电量，电池健康度等。

⨳ 充放电均衡，如：充放电时的主动和被动均衡。

⨳ 与充电桩对接，需要满足充电协议、快充标准等。

##### CAN

`CAN`总线协议（Controller Area Network），控制器局域网总线，是德国BOSCH（[博世](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fen.wikipedia.org%2Fwiki%2FRobert_Bosch_GmbH)）公司研发的一种[串行通讯协议总线](https://link.juejin.cn?target=https%3A%2F%2Fwww.zhihu.com%2Fsearch%3Fq%3D%E4%B8%B2%E8%A1%8C%E9%80%9A%E8%AE%AF%E5%8D%8F%E8%AE%AE%E6%80%BB%E7%BA%BF%26search_source%3DEntity%26hybrid_search_source%3DEntity%26hybrid_search_extra%3D%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A%222604283212%22%7D)，它可以使用双绞线来传输信号，是世界上应用最广泛的现场总线之一。

目前在汽车上使用的高速网络系统采用的都是基于CAN总线的标准，特别是广泛使用的ISO 11898国际标准。CAN总线通常采用屏蔽或非屏蔽的双绞线，总线接口能在极其恶劣的环境下工作。根据ISO 11898的标准建议，即使双绞线中有一根断路，或有一根接出其至两根线短接，总线都必须能继续工作。

##### 以太网

以太网是20世纪80年代初开发的一种通信标准，用于在家庭等本地环境中连接计算机和其他设备。这个本地环境被定义为LAN（Local Area Network），也就是我们平时所说的局域网。在局域网网中，多个设备相连，设备与设备之间可以共享信息。

以太网是一种有线系统，最初使用同轴电缆（Coaxial Cable），现在使用双绞线(Twisted Pair Cable)和光缆(Fiber Optic Cable)。

##### SoC

SoC（System on Chip），翻译过来就是系统级芯片，也有称[片上系统](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fbaike.baidu.com%2Fitem%2F%25E7%2589%2587%25E4%25B8%258A%25E7%25B3%25BB%25E7%25BB%259F)。既然是系统，单个就称不上系统，只有多个个体的组合才能称之为系统，所以，`SOC`强调的是一个整体。用“麻雀虽小五脏俱全”来形容SoC，再确切不过了。

汽车系统级`SoC`主要面向两个领域，一是驾驶舱，二是智能驾驶，两者的界限现在越来越模糊。`SoC`运行`Hypervisor`，在`Hypervisor`之上运行两类操作系统，其中对实时性和安全性要求比较高的安全域模块跑在`QNX`或者`Linux`系统上；对实时性要求不太高、但对生态要求比较高的娱乐域模块跑在`Android`系统上。

> `Hypervisor` 是什么？
>
> 一个 Hypervisor（又被称为 virtual machine monitor、VMM 或 virtualizer）是一种模拟器，它是创建或者运行虚拟机的软件、固件、或者硬件。一个计算机，上面运行着一个 hypervisor，hypervisor 上面又运行着一个或多个虚拟机，该计算机被称为 host machine，每一个虚拟机被叫做guest machine。

##### SPI

`SPI`，`SPI`是串行外设接口（Serial Peripheral Interface），简单来讲就是它一种高速的，全双工，同步的通信总线。

在车载`T-BOX`中， `MCU`和`SoC`之间必然存在数据通信，它们之间就可以基于SPI方式的进行通信。

##### MCU

`MCU(MicroControllerUnit) `中文名称为微控制单元，又称单片微型计算机，是指随着大规模集成电路的出现及其发展，将计算机的`CPU`、`RAM`、`ROM`、定时数器和多种I/O接口集成在一片芯片上，形成芯片级的计算机，为不同的应用场合做不同组合控制。汽车是`MCU`的一个非常重要的应用领域，高端车型中每辆车用到的`MCU`数量接近100个，从行车电脑、液晶仪表，到发动机、底盘，汽车中大大小小的组件都需要`MCU`进行把控。

按应用领域划分，汽车`MCU`又可以分为车身域、动力域、底盘域、座舱域和智驾域。其中对于座舱域和智驾域来说，`MCU`需要有较高的运算能力，并具有高速的外部通讯接口，比如CAN FD和以太网，车身域同样要求有较多的外部通讯接口数量，但对`MCU`的算力要求相对较低，而动力域和底盘域则要求更高的工作温度和功能安全等级。

#### 虚拟机

##### `Hypervisor`

`Hyperviosr`，也称为`VMM`，虚拟机管理器。`Hypervisor` 内部的机制，与单一的任何一种 `OS`类似，简要概括就是调度资源。只不过它发生在更高的层级上，因为我们讨论的不是` OS` 和 多线程，而是多个`OS`，`VMM `调度`OS `的资源，出让`CPU` 的使用权。

`Hypervisor`允许多个操作系统共享一个`CPU`(多核CPU的情况可以是多个CPU)。虽然基本的技术已有半个世纪之久，但是应用到嵌入式领域却是近年才发生的。`Hypervisor`是宽泛的计算概念的一部分，作为虚拟化技术为人所知。基本上`Hypervisor`的目的是共享硬件资源，就像操作系统所做的那样。

对比着看`Hypervior `和 `OS`，我们可以很容易的理解到：

OS 需要为每个线程分配特定的内存资源，同理，`VMM `需要为每个`OS` 分配特定的内存资源，这里特定的内存资源也称之为「资源沙箱」。沙箱以外的任何资源，如`OS `需要使用，都需要经过`VMM` 的调度分配。另外，除了调配任务外，`VMM `还会监控硬件的事件，如定时器的中断。

#### 驱动层

##### Android BSP

**BSP（Board Support Package）**，板级支持包，是介于主板硬件和操作系统中驱动层程序之间的一层，一般认为它属于操作系统一部分，主要是实现对操作系统的支持，为上层的驱动程序提供访问硬件设备寄存器的函数包，使之能够更好的运行于硬件主板。

在嵌入式系统软件的组成中，就有`BSP`。`BSP`是相对于操作系统而言的，不同的操作系统对应于不同定义形式的`BSP`，例如`VxWorks`的`BSP`和`Linux`的`BSP`相对于某一CPU来说尽管实现的功能一样，可是写法和接口定义是完全不同的，所以写`BSP`一定要按照该系统BSP的定义形式来写，这样才能与上层OS保持正确的接口，良好的支持上层OS。

在`Android BSP`中，硬件厂商需要提供自己的硬件抽象层（HAL），以支持特定硬件平台上的设备。`HAL`是一种软件抽象层，用于将硬件和操作系统之间的差异进行抽象，使得`Android`操作系统可以在不同的硬件平台上运行。`HAL`的实现依赖于硬件的特性和功能，因此每个硬件平台需要定制自己的`HAL`。

除了`HAL`之外，`Android BSP`还包含了一些基本的软件组件，如操作系统、驱动程序、库文件、配置文件等。这些组件都需要针对特定的硬件平台进行定制，以确保它们可以正确地运行在该平台上。

#### 框架层

框架层包含 Android Customized Framework、Customized Service、Cluster System Service。

#### 应用层

应用层是应用开发工程师最接近的一层，包含常见的系统应用、三方应用和一些SDK API。其中系统应用是在车载系统中最常定制的，比如Launcher应用、系统设置、SysytemUI等。

1.Launcher

桌面启动器，桌面启动器是帮助用户查找和启动其他应用程序的软件。主要负责摆放小部件，显示其它应用程序入口。

2.系统设置

系统设置是整个IVI（车载信息娱乐系统）的控制中心，车辆的音效、无线通信、状态信息、安全信息等都是需要通过系统设置来查看和操作。

3.SystemUI

常见的状态栏、导航栏、消息中心、 快捷设置、音量调节弹窗、蓝牙连接弹窗等一系列界面都是由 `SystemUI`模块负责管理、显示的。

### 参考

本篇博客参考许多资料，想较为深入理解某个知识可以参考如下文章：

Android for Cars

[developer.android.com/training/ca…](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Ftraining%2Fcars%3Fhl%3Dzh-cn)

智能汽车行业华为产业链深度系列研究-鸿蒙座舱：人车交互新生态

[pdf.dfcfw.com/pdf/H3_AP20…](https://link.juejin.cn?target=https%3A%2F%2Fpdf.dfcfw.com%2Fpdf%2FH3_AP202302071582856037_1.pdf%3F1675791507000.pdf)

车载操作系统架构研究报告

[www.catarc.org.cn/upload/2021…](https://link.juejin.cn?target=http%3A%2F%2Fwww.catarc.org.cn%2Fupload%2F202109%2F22%2F202109221131141024.pdf)

车载联网终端Tbox基本功能介绍

[zhuanlan.zhihu.com/p/497862010](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F497862010)

一文看懂ADAS

[zhuanlan.zhihu.com/p/36903822](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F36903822)

什么是BMS系统？BMS系统的作用和功能是什么？

[zhuanlan.zhihu.com/p/599183106](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F599183106)

通俗讲讲can协议？

[www.zhihu.com/question/39…](https://link.juejin.cn?target=https%3A%2F%2Fwww.zhihu.com%2Fquestion%2F39108503%2Fanswer%2F2604283212)

一文读懂汽车控制芯片(MCU）

[zhuanlan.zhihu.com/p/637567706](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F637567706)

Hypervisor 虚拟化技术与ARM 硬件虚拟化（上）

[zhuanlan.zhihu.com/p/391234199](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F391234199)







## Android音频焦点

在Android系统实际应用中，经常会出现两个应用同时播放声音的场景，此时就需要一种策略来决定如何播放声音。

当两个或两个以上的`Android`应用同时向同一输出流播放音频时，系统会将所有音频流混合在一起。虽然从技术上感觉很牛逼，但实际使用场景会给用户体验是不好的。那么为了避免所有音乐应用同时播放，`Android `引入了“音频焦点”的概念，就是说某一时刻只能有一个应用获得音频焦点。

那么当某个应用需要输出音频时，它需要请求获得音频焦点，获得焦点后，就可以播放声音了。 不过，在获得音频焦点后，可能无法一直持有这个焦点到播放完成，这是因为其他应用在这个过程中也是可以请求焦点，从而占有当前的音频焦点。

如果发生这种抢占焦点的情况，应该将当前应用暂停播放或降低音量，从而让用户听到新的音频源。

所以音频焦点这个测量是一种合作模式。应用是需要遵守音频焦点准则，但系统是不会强制执行这些准则的。 比如说，如果应用想要在失去音频焦点后继续大声播放，系统也不会阻止它。但从用户使用角度来说，这是一种极其难受的体验，那么用户很可能会卸载具有这种影响体验的应用。

行为恰当的音频应用应根据以下一般准则来管理音频焦点：

- 在即将开始播放之前调用 `requestAudioFocus()`，并验证调用是否返回 `AUDIOFOCUS_REQUEST_GRANTED`，比如可以在Media会话的 `onPlay()` 回调中调用 `requestAudioFocus()`。
- 在其他应用获得音频焦点时，停止或暂停播放，或降低音量。
- 播放停止后，放弃音频焦点。

### 请求和放弃焦点

`Android 8.0`之前，调用`requestAudioFocus()` 请求焦点时，需要设置一个焦点变化的监听器 `AudioManager.OnAudioFocusChangeListener`。该监听器可以在媒体会话所在的 Activity 或服务中创建。当其他应用获取或放弃音频焦点时就可以在注册的监听器中收到`onAudioFocusChange()` 回调。

```java
AudioManager audioManager = (AudioManager) context.getSystemService(Context.AUDIO_SERVICE);  
AudioManager.OnAudioFocusChangeListener afChangeListener;  
  
// Request audio focus for playback  
int result = audioManager.requestAudioFocus(afChangeListener,  
        // Use the music stream.  
        AudioManager.STREAM_MUSIC,  
        // Request permanent focus.  
        AudioManager.AUDIOFOCUS_GAIN);

```

但是从 Android 8.0（API 级别 26）开始，当您调用 `requestAudioFocus()`时，需要加上`AudioFocusRequest` 参数。

可以通过使用 `AudioFocusRequest.Builder` 构建 `AudioFocusRequest` 来请求和放弃音频焦点， 由于焦点请求始终必须指定请求的类型，此类型会包含在构建器的构造函数中。使用构建器的方法来设置请求的其他字段。代码示例如下：

```java
AudioManager audioManager = (AudioManager) Context.getSystemService(Context.AUDIO_SERVICE);  
AudioAttributes playbackAttributes = new AudioAttributes.Builder()  
        .setUsage(AudioAttributes.USAGE_GAME)  
        .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)  
        .build();  
AudioFocusRequest focusRequest = new AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN)  
        .setAudioAttributes(playbackAttributes)  
        .setAcceptsDelayedFocusGain(true)  
        .setOnAudioFocusChangeListener(afChangeListener, handler)  
        .build();  
final Object focusLock = new Object();  
  
boolean playbackDelayed = false;  
boolean playbackNowAuthorized = false;  
  
int res = audioManager.requestAudioFocus(focusRequest);  
synchronized(focusLock)  {  
    if (res == AudioManager.AUDIOFOCUS_REQUEST_FAILED) {  
        playbackNowAuthorized = false;  
    } else if (res == AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {  
        playbackNowAuthorized = true;  
        playbackNow();  
    } else if (res == AudioManager.AUDIOFOCUS_REQUEST_DELAYED) {  
        playbackDelayed = true;  
        playbackNowAuthorized = false;  
    }  
}  
  
@Override  
public void onAudioFocusChange(int focusChange) {  
    switch (focusChange) {  
        case AudioManager.AUDIOFOCUS_GAIN:  
            if (playbackDelayed || resumeOnFocusGain) {  
                synchronized (focusLock) {  
                    playbackDelayed = false;  
                    resumeOnFocusGain = false;  
                }  
                playbackNow();  
            }  
            break;  
        case AudioManager.AUDIOFOCUS_LOSS:  
            synchronized (focusLock) {  
                resumeOnFocusGain = false;  
                playbackDelayed = false;  
            }  
            pausePlayback();  
            break;        
        case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT:  
            synchronized (focusLock) {  
                resumeOnFocusGain = true;  
                playbackDelayed = false;  
            }  
            pausePlayback();  
            break;        
        case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:  
            // ... pausing or ducking depends on your app  
            break;  
    }  
}
```

在 Android 8.0（API 级别 26）中，当其他应用使用 `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK` 请求焦点时，系统可以在不调用应用的 `onAudioFocusChange()` 回调的情况下降低和恢复音量。

自动降低音量的行为对于音乐和视频播放应用来说是可接受的，但在播放语音内容时（例如在听书应用中）就没什么用处了。在这种情况下，应用可以选择直接暂停播放。

如果应用在被要求降低音量时暂停播放，还是要创建包含 `onAudioFocusChange()` 回调方法的 `OnAudioFocusChangeListener`，在该回调方法里去实现所需的暂停播放/恢复播放的操作。 调用 `setOnAudioFocusChangeListener()`来注册监听器，然后调用 `setWillPauseWhenDucked(true)` 告诉系统使用您的回调，而不是执行自动降低音量。

#### 放弃焦点

调用 `abandonAudioFocus(listener)`可以放弃焦点

```java
audioManager.abandonAudioFocus(afChangeListener);
```

在Android8.0之后，调用 `abandonAudioFocusRequest()` 释放音频焦点时，需要添加上`AudioFocusRequest` 作为参数。需要注意的是，在请求和放弃焦点时，要使用相同的 `AudioFocusRequest` 实例。

#### 延迟获取焦点

在有些情况下，因为焦点被其他应用“锁定”了，系统不能批准对音频焦点的请求，例如在通话过程中。在这种情况下，`requestAudioFocus()` 会返回 `AUDIOFOCUS_REQUEST_FAILED`。在这种情况下，由于未获得焦点，应用是不会播放音频的。

通过 `setAcceptsDelayedFocusGain(true)`可让应用异步处理焦点请求。设置这个标记后，在焦点锁定时发出的请求会返回 `AUDIOFOCUS_REQUEST_DELAYED`。当锁定音频焦点的情况不再存在时（例如当通话结束时），系统会批准待处理的焦点请求，并调用 `onAudioFocusChange()` 来通知您的应用。

为了处理“延迟获取焦点”，也是需要通过调用 `setOnAudioFocusChangeListener()`来实现所需行为并注册监听器。在`OnAudioFocusChangeListener`监听器的 `onAudioFocusChange()` 回调方法的 `进行处理。

### 响应音频焦点变化

当应用获得音频焦点后，它必须能够在其他应用为自己请求音频焦点时释放该焦点。出现这种情况时，应用就会收到对 `AudioFocusChangeListener` 中的 `onAudioFocusChange()` 方法的调用，在 `onAudioFocusChange()`回调中的 `focusChange` 参数表示所发生的更改类型

```java
AudioManager.OnAudioFocusChangeListener afChangeListener =  
        new AudioManager.OnAudioFocusChangeListener() {  
            public void onAudioFocusChange(int focusChange) {  
                if (focusChange == AudioManager.AUDIOFOCUS_LOSS) {  
                    // Permanent loss of audio focus  
                    // Pause playback immediately                } else if (focusChange == AudioManager.AUDIOFOCUS_LOSS_TRANSIENT) {  
                    // Pause playback  
                } else if (focusChange == AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK) {  
                    // Lower the volume, keep playing  
                } else if (focusChange == AudioManager.AUDIOFOCUS_GAIN) {  
                    // Your app has been granted audio focus again  
                    // Raise volume to normal, restart playback if necessary                }  
            }  
        };
```

#### 暂时性失去焦点

如果焦点更改是暂时性的（`AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK` 或 `AUDIOFOCUS_LOSS_TRANSIENT`），应用一般要降低音量或暂停播放，否则保持相同的状态。

在暂时性失去音频焦点时，您应该继续监控音频焦点的变化，并准备好在重新获得焦点后恢复正常播放。当抢占焦点的应用放弃焦点时，您会收到一个回调 (`AUDIOFOCUS_GAIN`)。此时，您可以将音量恢复到正常水平或重新开始播放。

#### 永久性失去焦点

如果是永久性失去音频焦点 (`AUDIOFOCUS_LOSS`)，则其他应用会播放音频。您的应用应立即暂停播放，因为它不会收到 `AUDIOFOCUS_GAIN` 回调。所以用户必须执行明确的操作，重新开始播放的。

### AAOS的音频焦点

#### 交互类型

为了支持 AAOS，音频焦点请求是根据请求的`CarAudioContext`和当前焦点持有者之间的预定义交互来处理的。交互有三种类型：

1.独占交互

这是 Android 最常用的交互模型。

在独占交互中，一次只允许一个应用程序保持焦点。因此，传入的焦点请求将被授予焦点，而现有的焦点持有者将失去焦点。由于两个应用程序都播放媒体，因此只允许一个应用程序保持焦点。因此，新启动的应用程序的焦点请求将返回`AUDIOFOCUS_REQUEST_GRANTED` ，而当前播放音乐的应用程序会收到焦点更改事件，其状态为与所发出的请求类型相对应的丢失状态。

2.拒绝交互

对于拒绝交互，传入的请求始终被拒绝。例如，在通话过程中尝试播放音乐时。在这种情况下，如果拨号器保持呼叫的音频焦点，并且第二个应用程序请求焦点来播放音乐，则音乐应用程序会收到`AUDIOFOCUS_REQUEST_FAILED`作为对请求的响应。由于焦点请求被拒绝，因此不会将焦点丢失分派给当前焦点持有者。

3.并发交互

`AAOS `的独特之处在于并发交互。这使得请求车内音频焦点的应用程序能够与其他应用程序同时保持焦点。当然如果要发生并发交互，是需要满足以下条件的：

- 传入的焦点请求必须请求`AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK`
- 当前焦点持有者未`setPauseWhenDucked(true)`
- 当前焦点持有者选择不接收鸭子事件

如果满足这些条件，焦点请求会返回`AUDIOFOCUS_REQUEST_GRANTED` ，而当前焦点持有者的焦点没有变化。但是，如果当前焦点持有者选择接收鸭子事件或在鸭子被鸭子时暂停，则当前焦点持有者将失去焦点，就像独占交互时发生的那样。

#### 交互矩阵

`CarAudioService`定义一个交互矩阵。每行代表当前焦点持有者的`CarAudioContext` ，每列代表传入请求的 `CarAudioContext`。

![car-audio-focus.svg](.\res\车载多媒体焦点-交互矩阵.png)

由于是同时进行交互的，因此可能有多个焦点持有者。在这种情况下，在确定要应用什么交互之前，会将传入的焦点请求与每个当前焦点持有者进行比较。在这种情况下，会优秀使用保守的交互：先是拒绝，然后排他，最后并发。

例如，当电话应用程序在持有焦点过程中，音乐媒体请求焦点时，则矩阵指示拒绝音乐媒体的焦点请求，当前只能处理电话的交互。

当音乐媒体应用程序在导航应用程序请求焦点时保持焦点时，假设满足并发交互的其他标准，则矩阵指示两个交互可以同时播放。

#### 多区域焦点管理

对于具有多个音频区域的车辆，每个区域的音频焦点都是独立管理的。对一个区域的请求不会考虑其他区域中持有焦点的内容，也不会导致其他区域中的焦点持有者失去焦点。主舱的焦点可以与后座娱乐系统分开管理，从而不会因焦点改变到另一个区域而中断一个区域的音频播放。

对于所有应用程序， `CarAudioService`会自动管理焦点。焦点请求的音频区域由其关联的`UserId`或`UID`确定，应用程序要同时在多个区域中播放音频，则必须通过在捆绑包中包含`AUDIOFOCUS_EXTRA_REQUEST_ZONE_ID`来请求每个区域的焦点

```java
Bundle bundle = new Bundle();  
bundle.putInt(CarAudioManager.AUDIOFOCUS_EXTRA_REQUEST_ZONE_ID,  
zoneId);  
  
AudioAttributes attributesWithZone = new AudioAttributes.Builder()  
        .setUsage(AudioAttributes.USAGE_MEDIA)  
        .addBundle(bundle)  
        .build();
```





## 车载音频系统

Android Automotive OS （AAOS） 是在核心 Android 音频堆栈的基础之上打造而成。AAOS 负责实现信息娱乐声音（即媒体、导航和通讯声音），但不直接负责具有严格可用性和计时要求的铃声和警告。

虽然 AAOS 提供了信号和机制来帮助车辆管理音频，但最终还是由车辆来决定应为驾驶员和乘客播放什么声音，从而确保对保障安全至关重要的声音和监管声音能被确切听到，而不会中断。

当 Android 管理车辆的媒体体验时，应通过应用来代表外部媒体来源（例如电台调谐器），这类应用可以处理该来源的音频焦点和媒体键事件。

### 车载音频架构

汽车音频系统可以处理多种声音和声音流，管理来自 Android 应用的声音，同时控制这些应用，并根据其声音类型将声音路由到 HAL 中的输出设备

![声音流中心架构图.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/595aa9fa55654d99bbc8199e4b9c4f2d~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=917&h=529&s=105905&e=png&a=1&b=fbfbfb)

### 声音流

声音流包含逻辑声音流和物理声音流：

- **逻辑声音流**：也称为“声源”，使用**音频属性**进行标记，音频系统会使用这些属性做出混音决策并将系统状态通知给应用。
- **物理声音流**：也称为“设备”，在混音后没有上下文信息。

系统实现者必须提供一个混音器，用于接受来自 Android 的一个或多个声音输入流，然后以合适的方式将这些声音流与车辆所需的外部声源组合起来。

为了确保可靠性，**外部声音**（来自独立声源，例如安全带警告铃声）在 Android 外部（HAL 下方，甚至是在单独的硬件中）进行管理。

HAL 实现和外部混音器负责确保对保障安全至关重要的外部声音能够被用户听到，而且负责在 Android 提供的声音流中进行混音，并将混音结果路由到合适的音响设备。

### 车载音频配置

车载音频服务会读取车载音频配置文件来为设备设置音频。

将车载音频配置文件放在设备的 `vendor\etc\` 或 `system\etc\` 中，车载音频服务会首先在 `vendor\etc\` 中搜索该文件。车载音频服务会读取 `car_audio_configuration.xml` 来确定音频配置。

在 Android 10 中，`car_audio_configuration.xml` 取代了 `car_volumes_groups.xml` 和 `IAudioControl.getBusForContext`。`audio_policy_configuration.xml` 通常包含在 vendor 分区中，表示主板的音频硬件配置。

`car_audio_configuration.xml` 中引用的所有设备都必须在 `audio_policy_configuration.xml` 中进行定义。

在 Android 14 中，AAOS 引入了原始设备制造商 （OEM） 插件服务，可以更主动地管理由车载音频服务监督的音频行为。除了新的插件服务之外，车载音频配置文件中还添加了以下更改：

- OEM 定义的车载音频上下文
- 非主音频区动态配置

### 音频上下文

为了简化 AAOS 音频的配置，用法均已归入 CarAudioContexts。这些音频上下文会在整个 CarAudioService 中使用，以定义路由、音量组、音频焦点和冲突管理。

下图列出了 AAOS 中的静态音频上下文

![Pasted image 20240624074643.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3df46426ee514a60ab02db6a8595d7c0~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=903&h=822&s=80915&e=png&b=fffcfa) 

### 音频控制HAL

Android 9 中引入了音频控制 HAL，车载音频服务会与音频控制 HAL 进行通信。从 Android 14 开始，音频控制 HAL 支持：

1. 淡变和平衡：与 Android 中已提供的通用音效不同，此机制允许系统应用通过 `CarAudioManager` API 设置音频平衡和淡出；
2. HAL 音频焦点请求：与 Android 类似，AAOS 依靠应用对音频焦点的积极参与来管理车载音频播放。焦点信息用于管理要进行音量和闪避控制的音频流
3. 设备静音和闪避：Android 12 引入了音量组静音功能，以便在用户的音频互动期间实现更全面的静音控制。这样一来，音频控制 HAL 便可以接收车载音频服务截获的静音事件；Android 12 引入了车载音频闪避，以优化对音频流并发播放的控制。这样一来，OEM 可以根据汽车的物理音频配置和当前播放状态（由车载音频服务确定）实现自己的闪避行为。
4. 音频设备增益变化：可以使用这种机制将车载音频系统的音频增益变化传达给车载音频服务。该机制会暴露音频增益音量指数的变化，并提供增益发生变化的对应原因
5. 音频端口配置更改

### CarAudioManager

#### 获取CarAudioManager

获取`CarAudioManager`对象，首先要获取`Car`对象，然后通过`mCar.getCarManager（Car.AUDIO_SERVICE）`

```java
Car mCar;
CarAudioManager mAudioManager;
private void setup(){
    mCar = Car.createCar(mContext, mConnectionCallbacks); 
    mCar.connect();
}

private ServiceConnection mConnectionCallbacks = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        Log.d(TAG, "onServiceConnected: ");
        mCarAudioManager = (CarAudioManager) mCar.getCarManager(Car.AUDIO_SERVICE);
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        Log.d(TAG, "onServiceDisconnected: ");
    }
};
```

#### CarAudioManager初始化过程

```java
public Object getCarManager(String serviceName) {
    CarManagerBase manager;
    ICar service = getICarOrThrow();
    synchronized (mCarManagerLock) {
        manager = mServiceMap.get(serviceName);
        if (manager == null) {
            try {
                IBinder binder = service.getCarService(serviceName);
                if (binder == null) {
                    Log.w(CarLibLog.TAG_CAR, "getCarManager could not get binder for service:" +
                          serviceName);
                    return null;
                }
                manager = createCarManager(serviceName, binder);
                if (manager == null) {
                    Log.w(CarLibLog.TAG_CAR,
                          "getCarManager could not create manager for service:" +
                          serviceName);
                    return null;
                }
                //用来回调onCarDisconnected
                mServiceMap.put(serviceName, manager);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
    }
    return manager;
}
private CarManagerBase createCarManager(String serviceName, IBinder binder) {
    CarManagerBase manager = null;
    switch (serviceName) {
        case AUDIO_SERVICE:
            manager = new CarAudioManager(binder, mContext, mEventHandler);
            break;
    }
}
private void tearDownCarManagers() {
    synchronized (mCarManagerLock) {
        for (CarManagerBase manager: mServiceMap.values()) {
            //回调所有的CarManagerBase
            manager.onCarDisconnected();
        }
        mServiceMap.clear();
    }
}
```

#### 对外的API

通过`CarAudioManager`可以通过提供的AP去实现相关的功能，比如音量的调节。 它实际是对CarAudioServive的跨进程调用

![AudioManager-API.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ff8c70cf2d9642ee9c8d36d2bf51c061~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=833&h=1245&s=195910&e=png&b=f9f9f9)   

### CarAudioService

`AAOS`中对原有的音频机制进行扩充，在`CarService`中加入了`CarAudioService`，对音频设备进行更加细致的管理。

#### 音量控制相关的方法

音量相关方法需要关注的是参数，包含`zoneId`、`groundId`和`usage`

```java
void setGroupVolume(int zoneId, int groupId, int index, int flags);
  int getGroupMaxVolume(int zoneId, int groupId);
  int getGroupMinVolume(int zoneId, int groupId);
  int getGroupVolume(int zoneId, int groupId); 
  int getVolumeGroupCount(int zoneId);
  int getVolumeGroupIdForUsage(int zoneId, int usage);
  int[] getUsagesForVolumeGroupId(int zoneId, int groupId);
```

#### groupId的获取：

通过`getVolumeGroupIdForUsage`通过传入的`usage`和`zoneId`进行遍历查找，返回对应的`groupID`：

```java
CarVolumeGroup[] groups = mCarAudioZones[zoneId].getVolumeGroups();
for (int i = 0; i < groups.length; i++) {
    int[] contexts = groups[i].getContexts();
    for (int context : contexts) {
        if (getContextForUsage(usage) == context) {
            return i;
        }
    }
}
```

#### zoneId的获取

`setZoneIdForUid`、`getZoneIdForUid`和`clearZoneIdForUid`是对`mUidToZoneMap`查询维护，然后进行音频焦点`AudioFocus`的管理

```java
 int[] getAudioZoneIds();
 int getZoneIdForUid(int uid);
 boolean setZoneIdForUid(int zoneId, int uid);
 boolean clearZoneIdForUid(int uid);
 int getZoneIdForDisplayPortId(byte displayPortId);
```

通过`mUidToZoneMap`维护`uid`和`zoneId`的映射map，通过getZoneIdForUid就可以获取到相应的zoneId

#### 音量的监听

```java
public void registerVolumeCallback(@NonNull IBinder binder) {}
public void unregisterVolumeCallback(@NonNull IBinder binder) {}
```

传统模式的音量监听回调是在init方法，通过广播监听音量变化，然后循环遍历binder集合进行回调

```java
public void init() {
	if (mUseDynamicRouting) {
    }else{
        setupLegacyVolumeChangedListener();
    }
}
private void setupLegacyVolumeChangedListener() {
    IntentFilter intentFilter = new IntentFilter();
    intentFilter.addAction(AudioManager.VOLUME_CHANGED_ACTION);
    intentFilter.addAction(AudioManager.MASTER_MUTE_CHANGED_ACTION);
    mContext.registerReceiver(mLegacyVolumeChangedReceiver, intentFilter);
}
private final BroadcastReceiver mLegacyVolumeChangedReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        final int zoneId = CarAudioManager.PRIMARY_AUDIO_ZONE;
        switch (intent.getAction()) {
            case AudioManager.VOLUME_CHANGED_ACTION:
                int streamType = intent.getIntExtra(AudioManager.EXTRA_VOLUME_STREAM_TYPE, -1);
                int groupId = getVolumeGroupIdForStreamType(streamType);
                if (groupId == -1) {
                    Log.w(CarLog.TAG_AUDIO, "Unknown stream type: " + streamType);
                } else {
                    callbackGroupVolumeChange(zoneId, groupId, 0);
                }
                break;
            case AudioManager.MASTER_MUTE_CHANGED_ACTION:
                callbackMasterMuteChange(zoneId, 0);
                break;
        }
    }
};
private void callbackGroupVolumeChange(int zoneId, int groupId, int flags) {
    for (BinderInterfaceContainer.BinderInterface<ICarVolumeCallback> callback :
         mVolumeCallbackContainer.getInterfaces()) {
        try {
            callback.binderInterface.onGroupVolumeChanged(zoneId, groupId, flags);
        } catch (RemoteException e) {
            Log.e(CarLog.TAG_AUDIO, "Failed to callback onGroupVolumeChanged", e);
        }
    }
}
```

动态路由的音量监听是在`setupDynamicRouting`里，通过`AudioPolicy`的`builder`设置的`mAudioPolicyVolumeCallback`，之后再循环遍历其他监听`binder`集合

```java
private void setupDynamicRouting(SparseArray<CarAudioDeviceInfo> busToCarAudioDeviceInfo) {
        final AudioPolicy.Builder builder = new AudioPolicy.Builder(mContext);
        builder.setAudioPolicyVolumeCallback(mAudioPolicyVolumeCallback);
}
    private final AudioPolicy.AudioPolicyVolumeCallback mAudioPolicyVolumeCallback =
            new AudioPolicy.AudioPolicyVolumeCallback() {
        @Override
        public void onVolumeAdjustment(int adjustment) {
            final int usage = getSuggestedAudioUsage();
            Log.v(CarLog.TAG_AUDIO,
                    "onVolumeAdjustment: " + AudioManager.adjustToString(adjustment)
                            + " suggested usage: " + AudioAttributes.usageToString(usage));
            // TODO: Pass zone id into this callback.
            final int zoneId = CarAudioManager.PRIMARY_AUDIO_ZONE;
            final int groupId = getVolumeGroupIdForUsage(zoneId, usage);
            final int currentVolume = getGroupVolume(zoneId, groupId);
            final int flags = AudioManager.FLAG_FROM_KEY | AudioManager.FLAG_SHOW_UI;
            switch (adjustment) {
                case AudioManager.ADJUST_LOWER:
                    int minValue = Math.max(currentVolume - 1, getGroupMinVolume(zoneId, groupId));
                    setGroupVolume(zoneId, groupId, minValue , flags);
                    break;
                case AudioManager.ADJUST_RAISE:
                    int maxValue =  Math.min(currentVolume + 1, getGroupMaxVolume(zoneId, groupId));
                    setGroupVolume(zoneId, groupId, maxValue, flags);
                    break;
                case AudioManager.ADJUST_MUTE:
                    setMasterMute(true, flags);
                    callbackMasterMuteChange(zoneId, flags);
                    break;
                case AudioManager.ADJUST_UNMUTE:
                    setMasterMute(false, flags);
                    callbackMasterMuteChange(zoneId, flags);
                    break;
                case AudioManager.ADJUST_TOGGLE_MUTE:
                    setMasterMute(!mAudioManager.isMasterMute(), flags);
                    callbackMasterMuteChange(zoneId, flags);
                    break;
                case AudioManager.ADJUST_SAME:
                default:
                    break;
            }
        }
    };
```

#### 多音区音频

随着现在的汽车领域发展，多个用户同时与平台互动并且每个用户都希望使用单独媒体的需求出来了。比如，后座上的乘客在后座显示屏上观看视频时，司机可以在驾驶舱中播放音乐。

所以多区音频的技术也就应需而生，它通过允许不同的音频源在车辆的不同音频区同时进行播放来实现此目的。

从 Android 10 开始提供的多区音频让原始设备制造商 （OEM） 能够将音频配置到单独的音频区。每个音频区由车辆内的一组设备组成，并且有各自的音量组、上下文路由配置以及焦点管理。通过这种方式，可以将主驾驶舱配置为一个音频区，而将后座显示屏的耳机插孔配置为第二个音频区。

每个音频区的焦点也是单独维护的。这使得不同音频区中的应用可以单独生成音频，而不会彼此干扰，同时让应用保持关注其所在音频区内焦点的变化。`CarAudioService` 内中的 `CarZonesAudioFocus` 负责管理每个音频区的焦点。

这些音频区被定义为 `car_audio_configuration.xml` 的一部分。然后，`CarAudioService` 读取该配置，并帮助 `AudioService `根据关联的音频区路由音频流。每个音频区仍会根据上下文和应用 UID 定义路由规则。创建播放器时，`CarAudioService` 会确定播放器与哪个音频区相关联，然后根据用法确定 `AudioFlinger `应将音频路由到哪个设备。

![多区音频1.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ba0b4b8a3294ddcab80a682d1a371af~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=951&h=472&s=113676&e=png&a=1&b=fbf4f1)

### 参考资料

1. 音频属性的介绍 [source.android.google.cn/docs/core/a…](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.google.cn%2Fdocs%2Fcore%2Faudio%2Fattributes%3Fhl%3Dzh-cn)
2. 车载音频配置 [source.android.com/docs/automo…](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.com%2Fdocs%2Fautomotive%2Faudio%2Faudio-policy-configuration)
3. Android Q CarAudio 汽车音频学习笔记 [blog.csdn.net/sinat_18179…](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fsinat_18179367%2Farticle%2Fdetails%2F103875807)



## 车载术语

### 操作系统和平台

在车载系统中，有多种操作系统和平台被广泛使用，以满足不同车辆制造商和应用需求。以下是一些主要的车载操作系统和平台：

#### 1. Android Automotive OS

- **描述**：一种专门为汽车设计的嵌入式操作系统，基于Android定制，直接运行在车辆的硬件上，而不依赖于智能手机。
- **特点**：提供丰富的多媒体、导航和通讯功能，并允许开发者为车载环境开发应用。
- **应用**：用于车载信息娱乐系统（IVI），支持广泛的第三方应用。

#### 2. QNX

- **描述**：一种基于微内核的实时操作系统，由BlackBerry旗下的QNX软件系统公司开发。
- **特点**：高度稳定和可靠，支持实时操作，非常适合安全性要求高的汽车应用。
- **应用**：广泛用于车辆的电子控制单元（ECU）、信息娱乐系统、仪表盘等。

#### 3. Automotive Grade Linux (AGL)

- **描述**：一个开源的Linux项目，旨在为汽车应用创建一个共享的软件平台。
- **特点**：由Linux基金会推动，提供一个灵活且可扩展的框架，支持定制和多供应商协作。
- **应用**：适用于信息娱乐系统、仪表盘、车载信息处理等。

#### 4. GENIVI

- **描述**：一个开源软件平台，旨在支持车载信息娱乐系统的快速开发和部署。
- **特点**：促进跨供应商的互操作性和标准化，支持多种硬件和软件环境。
- **应用**：主要用于信息娱乐系统和车载通讯。

#### 5. Microsoft Windows Embedded Automotive

- **描述**：微软为汽车制造商提供的定制化嵌入式操作系统解决方案。
- **特点**：基于Windows Embedded平台，提供强大的开发工具和支持，易于集成和定制。
- **应用**：曾用于福特Sync等车载信息娱乐系统。

#### 6. VxWorks

- **描述**：由风河公司（Wind River）开发的实时操作系统（RTOS）。
- **特点**：高可靠性、低延迟，广泛应用于安全关键系统。
- **应用**：用于ECU、信息娱乐系统、仪表盘等。

#### 7. ROS (Robot Operating System)

- **描述**：一种开源的机器人操作系统，越来越多地应用于自动驾驶和高级驾驶辅助系统（ADAS）。
- **特点**：提供丰富的开发工具和库，支持复杂的传感器融合和数据处理。
- **应用**：用于自动驾驶车辆的研究和开发。

#### 8. Ubuntu Automotive

- **描述**：基于Ubuntu Linux的定制版本，专为汽车应用设计。
- **特点**：提供一个开放的开发环境，支持广泛的硬件和软件集成。
- **应用**：适用于信息娱乐系统、自动驾驶平台等。

#### 9. BlackBerry IVY

- **描述**：BlackBerry与Amazon Web Services（AWS）合作开发的智能汽车数据平台。
- **特点**：提供实时数据处理和分析，支持云连接和边缘计算。
- **应用**：用于车载数据管理、分析和服务提供。

#### 10. Tizen IVI (In-Vehicle Infotainment)

- **描述**：一个基于Linux的开源操作系统，由Tizen项目开发，专注于车载信息娱乐系统。
- **特点**：灵活且可扩展，支持多种应用和服务。
- **应用**：适用于车载信息娱乐系统、仪表盘等。

### 车载互联解决方案

#### 1.Carwith

`Carwith` 是小米公司推出的车载互联系统，通过智能手机与车载系统的连接，实现导航、音乐、通讯等功能的无缝集成，提升了驾驶和乘车的智能化体验。

- **智能互联**：集成小米生态系统，支持多种智能设备的连接和控制。
- **语音助手**：集成小爱同学，支持语音控制导航、音乐等功能。
- **导航和媒体**：支持高德地图、百度地图等导航应用，以及小米音乐等媒体应用。

#### 2.HiCar

`HiCar` 是华为公司的车载解决方案，通过智能手机与车载系统的深度集成，提供全面的车载应用和服务，支持语音助手和多种第三方应用的集成。

- **智能协同**：利用华为的IoT平台，实现多设备的智能协同工作。
- **语音控制**：集成华为语音助手，支持语音导航、音乐播放等操作。
- **导航和媒体**：支持高德地图、百度地图等导航应用，以及华为音乐等媒体应用。

#### 3.Apple CarPlay

Apple CarPlay 是苹果公司的车载信息娱乐系统，通过连接支持CarPlay的iPhone，将iPhone的界面映射到车载显示屏上，提供导航、音乐、通讯等功能。

- **iOS集成**：与iPhone深度集成，支持苹果地图、音乐和通讯应用。
- **语音助手**：集成Siri，支持语音控制导航、音乐等功能。
- **第三方应用**：支持第三方应用的集成和操作。

#### 4.Android Auto

Android Auto 是Google的车载互联解决方案，允许支持Android Auto的Android手机与车载系统连接，提供导航、音乐、通讯等功能，支持Google Assistant语音控制和第三方应用的集成。

- **Android集成**：与Android手机深度集成，支持Google地图、音乐和通讯应用。
- **语音助手**：集成Google Assistant，支持语音导航、音乐播放等操作。
- **第三方应用**：支持第三方应用的集成和操作。

#### 5.Baidu CarLife

`Baidu CarLife `是百度公司的车载互联解决方案，支持Android和iOS设备，集成了百度地图和语音助手，提供导航、音乐、通讯等功能。

- **百度生态集成**：支持百度地图、语音助手等百度生态系统的功能。
- **语音控制**：支持语音导航、音乐播放等操作。
- **第三方应用**：支持第三方应用的集成和操作。

#### 6.对比分析

| 特点 / 解决方案    | Carwith                            | HiCar                              | Apple CarPlay                     | Android Auto                                      | Baidu CarLife                      |
| ------------------ | ---------------------------------- | ---------------------------------- | --------------------------------- | ------------------------------------------------- | ---------------------------------- |
| **厂商**           | 小米                               | 华为                               | 苹果                              | Google                                            | 百度                               |
| **主要特点**       | 智能互联、语音助手、导航和媒体播放 | 智能协同、语音控制、导航和媒体播放 | iOS集成、Siri语音控制、第三方应用 | Android集成、Google Assistant语音控制、第三方应用 | 百度地图集成、语音助手、第三方应用 |
| **兼容性**         | 小米手机                           | 华为手机                           | iPhone                            | Android手机                                       | Android和iOS手机                   |
| **第三方应用支持** | 有限                               | 有限                               | 丰富                              | 丰富                                              | 有限                               |
| **语音助手**       | 小爱同学                           | 华为语音助手                       | Siri                              | Google Assistant                                  | 百度语音助手                       |

### 通信协议

#### 1.CAN

**CAN（Controller Area Network）** 是一种串行通信协议，最初由德国的Bosch公司在20世纪80年代开发，旨在汽车内部的控制器和设备之间实现高效、可靠的数据通信。CAN已经成为汽车行业的标准通信协议，并广泛应用于工业自动化、医疗设备、楼宇自动化等领域。

##### 工作原理

CAN网络是一种多主方式的总线系统，允许多个节点（即控制器、传感器、执行器等）连接到同一总线上进行通信。每个节点都可以在总线上发送和接收数据，网络上的节点根据报文的优先级进行仲裁，确保高优先级的报文能够优先传输。

其多主架构和强大的错误检测机制使其成为车载网络通信的首选解决方案。随着CAN FD的推出，CAN的应用范围和性能得到了进一步提升，能够满足更高数据速率和更大数据量的需求。

##### 应用场景

- **动力系统**：发动机控制单元（ECU）、变速器、燃油系统等。
- **车身电子**：车灯、车窗、空调、座椅调节等。
- **安全系统**：防抱死制动系统（ABS）、安全气囊、电子稳定程序（ESP）等。

#### 2.LIN

**LIN（Local Interconnect Network）** 是一种用于汽车低速通信的串行通信协议，主要用于车内各类传感器和执行器的网络。它由多个汽车制造商和供应商共同开发，旨在作为更简单、成本更低的替代方案，用于一些不需要高带宽和实时性的应用场景。

##### 工作原理

LIN是一种单主多从（master-slave）通信系统，其中只有一个主节点（通常是控制单元）和多个从节点（传感器、执行器等）。主节点负责调度所有的通信，从节点只在接收到主节点的请求时才响应。

##### 应用场景

1. **车身电子**： 车窗升降、座椅调节、车灯控制、雨刷控制等不需要高速通信的功能。
2. **空调控制**：空调系统中的各类传感器和执行器之间的数据通信。
3. **车内网络**：车内娱乐系统、门锁控制、镜子调节等功能的通信。
4. **CAN Bus（Controller Area Network）**：一种车辆内部的通信协议，用于各个电子控制单元（ECU）之间的数据传输。
5. **MOST (Media Oriented Systems Transport)**：用于多媒体数据传输的高带宽汽车通信标准。
6. **Ethernet AVB (Audio Video Bridging)**：用于音频和视频数据流的汽车以太网协议。
7. **LIN（Local Interconnect Network）**：一种用于低速通信的车载网络协议，通常用于简单设备的通信。
8. **FlexRay**：一种高速、时间触发的车载通信协议，常用于需要高可靠性和实时性的应用。

#### 3.MOST

**MOST（Media Oriented Systems Transport）** 是一种用于车载多媒体和信息娱乐系统的高带宽串行通信协议。MOST最初由德国公司OASIS SiliconSystems开发，目前由MOST Cooperation管理。该协议主要用于传输音频、视频、语音和数据，广泛应用于汽车行业的高端多媒体和信息娱乐系统。

##### 工作原理

`MOST`网络是一种环形网络架构，使用光纤、电缆或同轴电缆进行数据传输。它采用同步传输方式，确保数据以恒定的速率传输，适合高质量多媒体数据的传输。每个设备在网络中作为节点，通过网络接口发送和接收数据。

##### 应用场景

1. **车载信息娱乐系统**：用于传输音频、视频和数据，实现高质量的多媒体播放、导航和通信功能。
2. **高级驾驶辅助系统：** 用于传输实时视频和数据，支持驾驶辅助功能。
3. **车载网络**：在高端汽车中，用于连接多媒体设备、导航系统、电话和其他信息娱乐设备。

#### 4.FlexRay

`FlexRay`是一种用于汽车的高速、实时、可靠的串行通信协议。它由`FlexRay Consortium`（包括宝马、戴姆勒、博世、飞思卡尔、恩智浦等公司）开发，旨在满足现代汽车对电子控制系统越来越高的要求。

`FlexRay`特别适用于高级驾驶辅助系统（ADAS）、车身控制、动力系统控制等对数据速率和可靠性有高要求的应用。

##### 工作原理

`FlexRay`网络采用时分多址（TDMA）技术，将通信时间划分为多个时隙，每个时隙分配给网络中的一个或多个节点。这种方式确保了网络的确定性和实时性，适合时间关键型应用。`FlexRay`还支持双通道通信，进一步提高了系统的可靠性和冗余度。

##### 应用场景

1. **高级驾驶辅助系统（ADAS）**： 高速、可靠的数据传输，用于传感器融合、环境感知等功能。
2. **动力系统控制**： 实时性和可靠性要求高的应用，如发动机控制、变速器控制。
3. **车身控制**：高速数据传输需求的应用，如电子稳定控制系统（ESC）、电动助力转向（EPS）。
4. **信息娱乐系统**：高带宽需求的应用，如车载视频、音频系统。

#### 5.Automotive Ethernet

`Automotive Ethernet`是将以太网技术应用于汽车内部通信的一种技术标准。由于汽车中的电子系统日益复杂，数据带宽需求迅速增加，传统的车载通信协议（如CAN、LIN、MOST等）在带宽和速度上已不能满足需求。`Automotive Ethernet`以其高带宽、低延迟和灵活的架构，成为现代汽车网络通信的主要发展方向。

##### 工作原理

Automotive Ethernet采用标准的以太网协议，但为了适应汽车环境，对物理层和部分协议层进行了改进。其典型拓扑结构包括星形、菊花链和环形，能够灵活适应不同的应用场景。

##### 应用场景

1. **高级驾驶辅助系统（ADAS）**： 高速传输传感器数据和视频流，实现实时环境感知和决策。
2. **自动驾驶**：实时传输大量数据，包括传感器数据、地图数据、车辆状态信息等。
3. **车载信息娱乐系统**： 高带宽传输音频、视频和互联网数据，提供高质量的信息娱乐体验。
4. **车内网络**：连接各类电子控制单元（ECU）、传感器和执行器，实现高效的数据交换和控制。
5. **车载诊断和维护**：实时监控车辆状态和故障诊断，提高车辆维护效率。

#### 6.车载通信协议的对比

| 特点 / 协议  | Automotive Ethernet  | CAN            | LIN               | FlexRay                  | MOST               |
| ------------ | -------------------- | -------------- | ----------------- | ------------------------ | ------------------ |
| **传输速率** | 最高可达10 Gbps      | 最高可达1 Mbps | 通常在20 kbps左右 | 最高可达20 Mbps          | 最多150 Mbps       |
| **通信方式** | 全双工，交换技术     | 多主           | 单主多从          | 双通道，时分多址（TDMA） | 同步传输，多种拓扑 |
| **实时性**   | 高                   | 高             | 较低              | 高                       | 高                 |
| **可靠性**   | 高                   | 高             | 较低              | 高                       | 高                 |
| **复杂度**   | 中                   | 中             | 低                | 高                       | 高                 |
| **应用场景** | 高带宽、高实时性应用 | 高速控制系统   | 低速、低带宽应用  | 高速、实时性应用         | 高质量多媒体传输   |

### OTA

**OTA（Over The Air）** 是指通过无线通信网络对设备进行远程更新和管理的一种技术。它允许设备制造商或服务提供商通过无线方式推送软件、固件、配置文件或数据更新到用户的设备，而无需用户进行物理连接或手动操作。这种技术在智能手机、车载系统、物联网设备、智能家居和其他电子设备中得到了广泛应用。

#### FOTA（Firmware Over-The-Air）

FOTA是指通过无线通信网络对设备的固件进行远程更新。固件是嵌入在设备硬件中的底层软件，负责设备的基本功能和控制。

- **目的**：更新车辆的电子控制单元（ECU）固件，通常涉及车辆的底层功能，如引擎控制、变速箱管理和安全系统。
- **特点**：确保车辆运行的核心系统保持最新状态，修复漏洞，提升性能和增加新功能。

#### SOTA（Software Over-The-Air）

SOTA是指通过无线通信网络对设备的软件进行远程更新。软件包括操作系统、应用程序和中间件等，负责设备的高级功能和用户界面。

- **目的**：更新车载信息娱乐系统的软件，包括操作系统、应用程序和用户界面。
- **特点**：改善用户体验、修复软件问题、添加新功能和支持新设备或服务。

#### 特点对比

| 特点           | FOTA（Firmware Over-The-Air）        | SOTA（Software Over-The-Air）        |
| -------------- | ------------------------------------ | ------------------------------------ |
| **更新内容**   | 固件（控制单元的嵌入式软件）         | 应用软件和操作系统                   |
| **应用范围**   | 车载电子控制单元（ECU）              | 信息娱乐系统、导航系统、应用软件等   |
| **更新频率**   | 较低（通常是关键更新或重要修复）     | 较高（包括功能更新、性能改进等）     |
| **复杂度**     | 较高（涉及底层系统，更新失败风险大） | 较低（涉及上层应用，更新相对安全）   |
| **实时性要求** | 较高（可能影响车辆核心功能）         | 较低（更新失败风险较小）             |
| **安全性**     | 高（需确保固件完整性和真实性）       | 较高（需确保软件安全性）             |
| **数据量**     | 较小（通常为固件补丁或升级包）       | 较大（可能涉及多个应用或系统的更新） |
| **依赖性**     | 高（依赖于硬件和底层软件）           | 较低（依赖于操作系统和平台）         |



### 

## 汽车模块介绍

​    汽车模块指的是车辆中独立的电子控制单元（ECUs），如发动机控制单元（ECU）、车身控制模块（BCM）等，它们负责特定的功能或系统。

### 一、控制类模块

​    这些模块主要用于控制车辆的不同系统，确保车辆各部分的正常运行。

#### 1、ECM

​    ECM（Electronic Control Module，电子控制模块）通常指的是专门用于控制发动机运行的电子装置，即发动机控制单元（Engine Control Unit）。不过，ECM这个术语有时也被用作电子控制模块的通用术语，可以涵盖多种类型的控制模块。在这里，我们将重点介绍ECM作为发动机控制模块的角色。

> - 发动机管理：ECM 负责管理发动机的各项功能，包括燃油喷射、点火正时、怠速控制等。
> - 数据采集与处理：ECM 从各种传感器（如节气门位置传感器、氧传感器、曲轴位置传感器等）收集数据，并根据这些数据来调整发动机的工作状态。
> - 故障诊断：ECM 能够监测发动机系统的运行状态，并在检测到故障时生成诊断故障代码（DTCs），帮助维修人员快速定位问题所在。
> - 性能优化：通过精确控制发动机的各项参数，ECM 能够优化发动机的性能，提高燃油经济性和减少排放。
> - 通信功能：ECM 通过车载网络（如 CAN 总线）与其他 ECU（电子控制单元）进行通信，实现信息共享和协同工作。

#### 2、TCU

​    TCU（Transmission Control Unit，变速箱控制单元）是自动变速箱车辆中的一个关键电子控制组件。TCU 负责监控和控制自动变速箱的操作，确保车辆在不同的驾驶条件下能够平顺地换挡，并且保持最佳的性能和燃油效率。在车联网环境中，TCU 的作用不仅限于传统的变速箱控制，还可以与车辆的其他系统以及云端平台进行数据交换，以提供更多的增值服务。

> - 换挡控制：TCU 根据车辆的速度、发动机负载、驾驶者的意图等因素来决定何时换挡。
> - 离合器和制动器控制：在自动变速箱中，TCU 控制着离合器和制动器的动作，以实现平顺的换挡过程。
> - 燃油经济性优化：TCU 通过智能控制换挡时机，帮助车辆在不同驾驶条件下实现最佳的燃油效率。
> - 驾驶舒适性：TCU 可以根据驾驶者的驾驶风格调整换挡策略，提供更为舒适的驾驶体验。
> - 故障诊断：TCU 能够监测变速箱的状态，并在检测到故障时记录错误代码，便于维修诊断。

#### 3、VCU

​    VCU（Vehicle Control Unit，整车控制器）是电动汽车和其他新能源汽车中的核心控制单元。VCU作为车辆的“大脑”，负责协调和管理车辆各个子系统的运行，确保车辆的整体性能达到最佳状态。

> - 协调管理：VCU 负责协调管理车辆的各个子系统，如动力系统、电池管理系统、驱动电机控制系统等，确保它们协同工作。
> - 信号采集与处理：VCU 采集来自车辆不同部位的信号，如加速踏板、制动踏板、方向盘位置、电池状态等，并对这些信号进行处理和分析。
> - 决策与控制：基于采集到的信号和车辆的状态，VCU 做出相应的决策，并控制车辆的各个控制器执行相应的动作，如调整电机转速、管理电池充放电等。
> - 安全监控：VCU 监控车辆的安全状态，如在紧急情况下切断电源或激活安全系统，确保车辆和乘客的安全。
> - 通信管理：VCU 负责管理车辆内部的通信网络，如 CAN（Controller Area Network）总线，确保各个 ECU（Electronic Control Unit）之间的数据交换。 

#### 4、ICM

​    ICM（Instrument Cluster Module，组合仪表模块）是汽车仪表盘的核心组件，负责显示车辆的各种关键信息，如车速、发动机转速、油量、里程、故障警告灯以及其他与驾驶相关的数据。ICM是驾驶员了解车辆运行状态的重要途径，对于保证行车安全至关重要。

> - 显示基本信息：显示车速表、转速表、油量表等基本信息，帮助驾驶员随时掌握车辆状态。
> - 警告与指示：通过各种指示灯和警告信息提醒驾驶员注意潜在的问题，如低油量、发动机故障等。
> - 交互反馈：提供对驾驶员操作的即时反馈，如换挡提示、巡航控制状态等。
> - 集成信息：可能集成其他系统的信息，如导航提示、电话状态、多媒体播放信息等。
> - 个性化设置：允许驾驶员根据个人偏好调整显示的内容和布局。

#### 5、IPM

​    IPM（Integrated Panel Module，集成面板模块）通常指的是集成在汽车内的控制面板，特别是指空调控制面板。然而，需要注意的是，在不同的上下文中，IPM 可能有不同的含义。例如，在某些情况下，IPM 也可能指的是其他类型的集成模块，但在这里我们将重点讨论其作为空调控制面板的角色。

> - 温度控制：允许驾驶员和乘客调节车厢内的温度设定。
> - 模式选择：提供不同的气流模式选择，如脸部、脚部、挡风玻璃除霜等。
> - 风速调节：控制送风风扇的速度，以适应不同的需求。
> - 信息显示：显示当前设置的温度、风速、模式等信息。
> - 自动控制：在一些高级系统中，IPM 可以自动调节温度和气流，以维持设定的舒适度。
> - 集成其他功能：在一些车型中，IPM 可能还会集成其他功能，如座椅加热、通风等。 

### 二、车身控制模块 

​    这类模块主要负责管理车辆的非动力系统功能，如灯光、门锁等。

#### 1、BCM

​    BCM（Body Control Module，车身控制模块）是现代汽车电子系统中的一个重要组成部分，它是一个电子控制单元（ECU），专门用于管理和控制汽车的各种车身功能。BCM 的功能涵盖了多种与车辆舒适性和安全性相关的系统，包括但不限于以下几个方面：

> - 车门锁控制：BCM 可以控制车门的上锁和解锁，包括通过遥控钥匙进行的无钥匙进入系统。
> - 车窗管理：管理车窗的升降，包括一键式车窗控制和防夹功能。
> - 灯光控制：控制车辆的外部和内部照明系统，如大灯、尾灯、转向灯、室内灯等。
> - 雨刷和洗涤器：控制雨刷的速度和洗涤器喷水的时间。
> - 空调系统：与空调系统配合，管理车内温度和通风。
> - 座椅调节：在一些高级车辆中，BCM 还可以参与控制电动座椅的位置记忆和调节功能。
> - 电源管理系统：管理车辆的电源分配，包括电池监控、休眠模式下的电流消耗控制等，以延长电池寿命。
> - 报警系统：在车辆发生异常情况时，如未关闭车门或车窗时，BCM 可以激活报警系统。

​    BCM 通过车辆的总线系统（如 CAN 总线）与其他电子控制单元进行通信，确保所有车身功能的协调工作。随着汽车电子技术的发展，BCM 的功能也在不断增加，其复杂性和集成度也随之提高。BCM 的设计和开发需要考虑车辆的不同需求，以确保所有功能都能高效、可靠地运行。

#### 2、DSM

​    DSM（Driver Seat Module，驾驶员座椅模块）是车联网技术中的一部分，主要负责管理和控制驾驶员座椅的各项功能。DSM 模块通常包括座椅位置调整、加热、通风、按摩等功能，并且可能还包括座椅安全相关的特性，如安全带预紧器等。在现代汽车中，驾驶员座椅模块不仅提升了驾驶舒适性，还增强了车辆的安全性。

> - 座椅位置调整：DSM 可以控制电动座椅的位置调整，包括前后移动、上下升降、靠背倾斜角度等。一些高端车型还支持记忆功能，能够存储多个驾驶者的座椅位置偏好。
> - 加热与通风：DSM 可以控制座椅加热功能，通过内置的加热元件在寒冷天气下为驾驶员提供温暖。同样，DSM 也可以控制座椅通风功能，通过内置的风扇在炎热天气下为驾驶员提供凉爽。
> - 按摩功能：一些高级车型的 DSM 模块还提供了按摩功能，通过座椅内的振动装置来缓解驾驶员的疲劳。
> - 安全功能：DSM 可能包括安全带预紧器，能够在碰撞发生时迅速收紧安全带，减少对驾驶员的伤害。一些 DSM 模块还可能集成座椅位置传感器，确保安全气囊在碰撞时能够正确展开。
> - 舒适性调整：DSM 可以控制座椅的侧翼支撑、腰部支撑等，以提供更好的包裹性和舒适性。
> - 个性化设置：DSM 支持个性化设置，允许驾驶员保存自己的座椅偏好设置，并在不同驾驶员之间切换时自动调整。

#### 3、RRM

​    RRM（Radio Receiver Module，无线电接收模块）是一个重要的组件，它负责接收无线电信号，并将其转换为可以被车载系统处理的信息。RRM 在车联网中的应用非常广泛，从传统的广播接收，到现代的车辆间通信（V2V）、车辆与基础设施通信（V2I）等，都是依靠 RRM 来实现信号的接收和处理。

> - 信号接收：RRM 接收来自不同来源的无线电信号，这些信号可以是广播信号、车辆间通信信号或来自基础设施的信号。
> - 信号处理：RRM 将接收到的无线电信号进行解调，提取有用的信息。
> - 数据传递：RRM 将处理后的数据传递给车载计算机或其他系统，以供进一步处理或使用。
> - 通信协议支持：RRM 支持多种通信协议，确保能够与不同的设备和服务进行通信。

### 三、安全类模块

​    这些模块专注于提高车辆的安全性能，保护驾驶员和乘客的安全。

#### 1、ABM

​    ABM (Air Bag Module，气囊控制器模块) 指的是现代汽车中一个至关重要的安全组件。气囊控制器模块通常包含了安全气囊系统的中央处理器，它连接着各种传感器，例如加速度传感器、碰撞传感器等，这些传感器能够检测到车辆碰撞的情况。

> - 碰撞检测：通过内置的传感器（如加速度传感器）来检测车辆是否发生了碰撞。
> - 气囊部署：一旦检测到碰撞并且达到了预设的阈值，ABM就会立即启动气囊展开程序。
> - 安全气囊控制：控制车辆中所有安全气囊的展开顺序和速度，确保气囊能够在正确的时间以适当的力量展开。
> - 故障诊断：持续监测气囊系统的工作状态，并在系统发生故障时存储故障码，以便维修时查找问题原因。
> - 通信功能：与车辆的其他控制系统（如 ECU）进行通信，共享碰撞信息以及其他相关数据。

​    当车辆发生碰撞时，ABM 会迅速分析来自传感器的数据，判断碰撞的严重程度，并决定是否需要启动安全气囊。如果满足预设的安全气囊展开条件，ABM 就会触发气囊充气，以保护驾驶员和乘客免受伤害。此外，ABM 还可能监控气囊的状态，确保它们处于良好的工作状态，并且能够在必要时正确展开。

​    气囊控制器模块是汽车安全系统的一部分，它通过车辆的总线系统（如 CAN 总线）与其他车辆系统通信，确保在碰撞发生时能够协调工作，比如与座椅安全带预紧器、门锁释放系统等协同动作，提供全面的安全保护。

#### 2、SAM

​    SAM（Steering Angle Module，方向盘转角模块）通常指的是方向盘转角传感器（Steering Angle Sensor，简称 SAS），尽管术语上略有差异，但两者常常指的是同一个设备。方向盘转角传感器用于测量汽车转向时方向盘的旋转角度，为车辆提供关键的转向信息。在现代汽车尤其是那些具备高级驾驶辅助系统（ADAS）和自动驾驶功能的车辆中，SAM 起着至关重要的作用。

> - 测量转向角度：SAM 能够测量方向盘相对于中心位置的角度变化，即方向盘的旋转角度。
> - 检测转向方向：SAM 可以检测方向盘的旋转方向，即向左转还是向右转。
> - 提供实时数据：SAM 提供实时的方向盘角度数据，这对于动态稳定控制（DSC）、主动悬挂系统、车道保持辅助等功能至关重要。
> - 辅助驾驶系统：SAM 的数据对于诸如自适应巡航控制（ACC）、车道保持辅助（LKA）、盲点监测（BSD）等高级驾驶辅助系统至关重要。

### 四、中央管理模块

​    这类模块通常作为车辆内部网络的中心节点，管理不同网络之间的通信。

#### 1、GWM

​    GWM（Gateway Module，网关模块）是车辆电子系统中的一个核心组件，它充当不同车载网络之间的通信桥梁。GWM 的主要职责是管理不同车载网络（如 CAN 总线）之间的数据交换，确保信息能够准确无误地在各个电子控制单元（ECUs）之间传递。

> - 网络管理：GWM 负责管理多个车载网络，如高速 CAN、低速 LIN 等，确保不同网络之间的数据能够正确传输。
> - 数据路由：作为车辆内部网络的中心节点，GWM 能够识别和转发来自不同 ECU 的数据包，实现跨网络的数据交换。
> - 协议转换：在不同网络之间可能存在不同的通信协议，GWM 能够进行必要的协议转换，确保不同网络之间的兼容性。
> - 安全控制：GWM 还承担着安全控制的任务，比如过滤掉无效或恶意的数据包，防止未经授权的访问。
> - 故障诊断：支持 OBD-II 等诊断标准，允许外部诊断工具通过 GWM 访问车辆的各个系统进行故障检测。
> - 数据汇总与报告：收集来自各个 ECU 的数据，进行汇总分析，并向上层系统（如 T-Box）报告车辆状态信息。

#### 2、CEM

​    CEM（Central Electronic Module，中央电子模块）也被称为域控制器，在车联网中扮演着重要的角色。它是汽车电子系统的一个中心化管理单元，负责连接和管理车辆内部的多个子系统和电子控制单元（ECUs）。CEM 的主要目的是简化车辆的电气架构，提高系统的集成度和效率，同时减少线束的数量和复杂性。

> - 集中管理：CEM 作为中央节点，可以集中管理车辆中的多个 ECU，协调它们之间的通信和功能。
> - 通信枢纽：CEM 通常作为车辆内部 CAN（Controller Area Network）或其他车载网络的通信枢纽，负责数据的转发和管理。
> - 电源管理：CEM 可以负责车辆电气系统的电源管理，控制不同 ECU 的供电状态，比如在车辆关闭后切断不必要的电源供应。
> - 诊断功能：CEM 可以执行故障诊断功能，收集来自不同 ECU 的状态信息，并向维修人员提供诊断数据。
> - 安全功能：CEM 可以实现一些安全相关功能，如防盗系统、车门锁控制等。
> - 舒适性功能：CEM 还可以管理诸如灯光控制、雨刷控制等与车辆舒适性相关的功能。

​    随着汽车电子技术的发展，未来的 CEM 可能会变得更加智能化和集成化，不仅仅局限于当前的功能，还将融合更多的智能计算能力和数据处理能力，以支持更高级别的自动驾驶和智能互联功能。此外，随着 5G 等新一代通信技术的应用，CEM 将能够更快地与云端进行数据交换，提供更为丰富和实时的服务。

### 五、通信类模块

​    这类模块主要负责车辆与外界的通信，确保车辆能够与外部网络进行数据交换。

#### 1、用户身份模块

​    SIM（Subscriber Identification Module，用户身份模块）卡是一种集成电路卡，它被用于移动通信设备中，以存储用户的身份信息和其他数据。在车联网环境中，SIM卡的作用类似于传统移动电话中的 SIM 卡，但其用途更为专业化，主要用于车辆与互联网或其他车辆之间的通信。

> - 身份认证：SIM 卡存储用户的识别信息，如 IMSI（国际移动用户识别码），用于在网络中识别用户身份。
> - 加密通信：SIM 卡包含加密密钥，用于保护通信的安全性，防止数据被非法拦截。
> - 存储数据：SIM 卡可以存储联系人信息、短信等数据，但在车联网中更多的是存储与车辆相关的数据和服务信息。
> - 网络接入：SIM 卡允许车辆通过移动通信网络接入互联网，实现数据传输。

​    SIM卡通过其存储用户身份信息和提供安全通信的能力，为车联网提供了多种功能和服务。无论是远程信息处理、车辆互联还是车载娱乐，SIM卡都在其中扮演着重要角色。 

​    随着技术的发展，这些模块的功能将会变得更加智能化，并且在车联网环境下，它们之间的协作也会更加紧密。



### IHU

**汽车IHU（Infotainment Hub Information & Entertainment Central Unit）是车载系统中的一个重要模块，主要负责信息娱乐功能。**‌‌

#### IHU模块的功能和作用

IHU模块在车载系统中扮演着核心角色，它集成了信息娱乐系统的各种功能，包括音频、视频、导航、电话等。通过与车辆的其他控制系统交互，IHU模块能够提供丰富的娱乐体验和车辆信息显示功能。

#### IHU模块与其他汽车模块的关系

虽然搜索结果中没有直接提到IHU与其他汽车模块的具体关系，但可以推测IHU模块与其他车载系统模块（如[CEM](https://www.baidu.com/s?wd=CEM&usm=1&ie=utf-8&rsv_pq=dd6063f7007b7c14&oq=汽车IHU是什么模块&rsv_t=6aafib2MYQbzPj2QV%2BTAtuQ8h7yDnx0PApP60iNo08hq1dS1S9GPktlq2U4&rsv_dl=re_dqa_generate&sa=re_dqa_generate)）之间存在密切的交互和依赖关系。例如，在某些情况下，IHU模块出现问题时，可能需要更换CEM等其他模块来解决。此外，IHU模块与[发动机控制模块](https://www.baidu.com/s?wd=发动机控制模块&usm=1&ie=utf-8&rsv_pq=dd6063f7007b7c14&oq=汽车IHU是什么模块&rsv_t=a67d4JFA8AvdAbyMZz7qxe%2FlHt2uQk4Dd8F9zGqMMcJjn3%2Be36Lut6xCMNY&rsv_dl=re_dqa_generate&sa=re_dqa_generate)和[点火模块](https://www.baidu.com/s?wd=点火模块&usm=1&ie=utf-8&rsv_pq=dd6063f7007b7c14&oq=汽车IHU是什么模块&rsv_t=7a18GdCZCFRjAH2FUjw9%2F4UW8EaektvBOGTx7rfm3np2shZX2pjsvC1IiXI&rsv_dl=re_dqa_generate&sa=re_dqa_generate)等共同工作，确保车辆的正常运行和安全。





## Selinux

### 1. SELinux简介

SELinux最早由美国国家安全局（NSA）开发，旨在通过精细的权限控制提高Linux系统的安全性。SELinux采用**强制访问控制（MAC）**模型，确保即便是具有root权限的进程，也无法随意访问敏感资源。

`MAC（Mandatory Access Control，强制访问控制）`是一种访问控制模型，它通过系统强制施加的安全策略来控制用户和进程对资源的访问。与常见的自主访问控制（DAC）不同，MAC并不允许用户自己决定资源的访问权限，而是由系统的安全策略严格控制和管理。MAC主要用于需要高度安全的场景中，例如军队和政府系统，确保系统内的每个用户和进程只能访问被授予的资源，且权限配置无法被普通用户更改。

在`MAC模型`中，每个资源（文件、进程、网络连接等）都被分配了一个安全标签或标记，而每个用户或进程也有自己的安全标签。MAC模型会根据这些标签，结合预设的访问策略，来判断是否允许某个用户或进程访问特定资源。

### 2. Android与SELinux

Android 4.3引入SELinux的目标是增强Android系统的安全性。

在Android的SELinux中，MAC模型用于控制应用程序和系统进程之间的访问。例如，系统可以通过SELinux策略限制某个应用访问系统资源或其他应用的数据，即使该应用拥有高权限，也只能在SELinux策略允许的范围内活动。这种机制有效降低了恶意软件的危害，防止它们随意访问系统资源。

SELinux在Android中的作用主要体现在以下几个方面：

#### 强化安全性：保护系统和用户数据

SELinux为Android带来了更细粒度的安全控制。在引入SELinux之前，Android主要依赖自主访问控制（DAC）机制，这种机制允许应用请求权限从而访问系统资源或用户数据。而SELinux通过强制访问控制（MAC），限制了即便获得某些权限的应用，也不能随意访问特定资源。这种控制特别适用于防范恶意软件或有意泄露用户隐私的应用。

#### 隔离应用和系统进程

SELinux可以将应用和系统进程分隔在各自的权限边界内。通过设置严格的策略，SELinux确保不同应用之间无法直接访问彼此的数据，也无法对系统的核心资源造成影响。这种隔离机制降低了应用对系统的破坏性，确保了Android系统的稳定和安全性。

#### 控制关键系统服务的访问权限

Android中的系统服务，例如媒体服务、相机和电话等，通常具有较高的权限。在SELinux的保护下，这些服务的访问被严格控制，即使是具有root权限的进程，也仅限于预先定义的策略中允许的操作。SELinux的这种强制性控制减少了关键系统服务受到恶意攻击的风险。

#### 运行时策略管理

SELinux在Android中可以运行在“宽容模式（Permissive）”或“强制模式（Enforcing）”。宽容模式下，SELinux仅记录违反策略的行为，适用于开发和调试；强制模式下，SELinux会严格执行策略，禁止未授权的行为，这是生产环境的标准配置。Android 5.0版本及以上默认启用SELinux强制模式，以确保系统的运行安全。

#### 提供安全日志与审计功能

SELinux为Android系统提供了详细的安全日志记录。每当某个进程尝试执行未授权的操作时，SELinux都会生成审计日志，帮助开发者和系统管理员识别并修正潜在的安全问题。这些日志为后续的安全分析提供了重要数据，可以帮助改进和优化SELinux策略。

### 3.SELinux的三种模式

SELinux有三种模式，它们分别提供不同级别的安全控制，适用于系统的不同需求。

#### 禁用模式（Disabled）

在这种模式下，SELinux被完全关闭，系统不加载任何SELinux策略。禁用模式下，系统只依赖传统的自主访问控制（DAC）机制来管理权限，没有强制访问控制的保护。**禁用模式并不推荐使用**，因为它会导致系统暴露于潜在的安全风险之下。

#### 宽容模式（Permissive）

在宽容模式下，SELinux会加载并执行安全策略，但不会阻止任何操作。相反，它会记录所有违反策略的操作，将其记录为日志信息。这种模式常用于开发和调试，帮助开发者了解哪些操作可能违反SELinux策略。宽容模式能够帮助识别和调整策略，而不影响系统的正常运行，但**不适合生产环境**，因为并未真正阻止未授权的操作。

- **适用场景**：开发和调试阶段，测试新策略或检查权限问题。
- **命令查看**：`adb shell getenforce` 输出 `Permissive` 表示系统处于宽容模式。

#### 强制模式（Enforcing）

强制模式是SELinux在生产环境中的标准配置。在该模式下，SELinux严格执行所有安全策略，任何不符合策略的操作都会被立即阻止，并记录在日志中。强制模式可以有效限制未授权的操作，为系统提供强有力的安全保障。自Android 5.0以来，Google默认将Android设备的SELinux配置为强制模式，以保证系统的安全性。

- **适用场景**：生产环境和正式发布版本，确保应用和系统进程的权限被严格控制。
- **命令查看**：`adb shell getenforce` 输出 `Enforcing` 表示系统处于强制模式。

#### 切换SELinux模式的命令

在开发和调试阶段，管理员可以通过以下命令切换SELinux模式（需要root权限）：

```bash
bash 代码解读复制代码# 切换到宽容模式
setenforce 0

# 切换到强制模式
setenforce 1
```

### 4. SELinux的工作原理

以下是SELinux在Android中的一个典型工作流程：

1. 一个应用进程尝试读取系统文件（比如`/system/data`）。
2. SELinux检查该进程的安全上下文，假设它的类型是`untrusted_app`。
3. SELinux检查系统文件的安全上下文，假设其类型是`system_data_file`。
4. SELinux在策略文件中检查是否允许`untrusted_app`访问`system_data_file`。
5. 如果策略未授权该访问，SELinux会阻止该操作并在日志中记录“avc: denied”信息。

SELinux在Android中涉及到如下概念：安全上下文、策略文件、类型强制

#### 安全上下文（Security Context）

每个进程、文件、设备、网络接口等在SELinux中都有一个**安全上下文**。安全上下文包含了标签，用于标记对象和操作的类型，帮助SELinux识别和控制权限。通常，安全上下文由以下部分组成：

- **用户（User）**：指执行进程或访问资源的用户角色。
- **角色（Role）**：规定了用户的权限范围，限定用户可以使用的资源。
- **类型（Type）**：定义了进程、文件和资源的具体类型，类型强制（Type Enforcement）是SELinux控制权限的核心部分。
- **敏感度（Sensitivity）**：用来描述安全级别，帮助细化访问控制。

#### 策略文件（Policy Files）

SELinux策略文件是控制访问权限的核心配置。它们定义了每个安全上下文之间的交互关系，例如哪些进程可以访问哪些文件、使用哪些系统资源等。常见的SELinux策略文件包括：

- **sepolicy**：SELinux的主策略文件，涵盖了系统和应用的权限控制规则。
- **file_contexts**：规定文件和目录的安全上下文。
- **service_contexts**：定义了系统服务的安全上下文。

在Android设备启动时，系统会加载策略文件，并将相应的安全上下文应用于每个资源。管理员也可以在开发过程中自定义这些策略文件，以满足不同应用场景的需求。

#### 类型强制（Type Enforcement）

类型强制（Type Enforcement, TE）是SELinux在Android中工作的核心机制。每个进程和资源都被赋予一个特定的“类型”，SELinux通过策略文件定义不同类型间的访问权限。类型强制确保了即使具有高权限的应用或进程，也只能在类型策略允许的范围内进行访问。

> 例如，假设文件类型为“app_data_file”，进程类型为“untrusted_app”，如果策略未允许“untrusted_app”类型的进程访问“app_data_file”，SELinux会阻止该进程的操作。

#### 策略执行

Android中的SELinux通常运行在**强制模式（Enforcing）**，确保所有操作都经过策略验证，任何不符合策略的操作都会被阻止。Android系统会在开机时加载SELinux策略，并根据策略规则对进程进行管理。SELinux支持两种执行模式：

- **宽容模式（Permissive）**：仅记录违规行为而不阻止操作，适合测试和调试。
- **强制模式（Enforcing）**：严格执行SELinux策略，阻止任何违规操作，是生产环境的推荐模式。

当进程尝试访问资源时，SELinux会通过以下过程来进行权限判断：

1. **安全上下文检查**：SELinux读取该进程和目标资源的安全上下文。
2. **匹配策略规则**：根据策略文件中的规则，确定进程是否有权限进行该操作。
3. **日志记录**：如果操作被拒绝，SELinux会将相关信息记录到系统日志中，以便管理员检查和分析。

> 例如，如果一个应用尝试读取系统文件而未获得授权，SELinux会阻止这一操作并在日志中记录详细的“拒绝访问”信息。

#### 日志与审计

SELinux在Android系统中的审计功能非常强大。系统会记录所有的访问违规行为到日志中，常见的日志信息包括`avc: denied`（访问控制被拒绝）。审计日志是解决SELinux权限问题的主要依据，开发者可以使用`adb logcat | grep "avc: denied"`命令来查看拒绝访问的详细信息。

对于开发人员来说，利用日志分析工具如`audit2allow`可以将日志中的拒绝事件转换成策略语句，用于生成允许访问的策略条目。但在生产环境中，管理员应尽量避免通过放宽策略来绕过权限控制，以确保系统安全。

### 5.配置和管理SELinux策略

配置和管理SELinux策略在Android系统中相对复杂，但可以通过自定义策略文件和工具来实现。SELinux策略管理的目的是制定合适的权限规则，以确保系统的安全性，并限制应用程序对资源的访问。

下面介绍下配置和管理SELinux策略的关键步骤和工具

#### SELinux策略文件概述

在Android中，SELinux策略文件通常包含在AOSP（Android Open Source Project）源码中。常见的策略文件有：

- **sepolicy**：主策略文件，定义系统的访问控制规则。
- **file_contexts**：定义文件和目录的安全上下文。
- **service_contexts**：定义系统服务的安全上下文。
- **property_contexts**：定义系统属性的安全上下文。

这些文件位于`/system/etc/selinux/`目录下，开发者可以在源码中自定义这些策略文件，编译后部署到设备上。

#### 基本配置步骤

配置SELinux策略需要修改或添加相应的策略规则，以适应应用和系统服务的需求。以下是基本的配置步骤：

##### (1)创建和编辑策略文件

在AOSP中，可以通过在`system/sepolicy`目录下添加或修改`.te`文件来定义策略规则。例如，要为一个新的应用定义权限规则，可以创建一个新的`.te`文件，并指定应用的上下文和权限。

```bash
bash 代码解读复制代码# example_app.te
type example_app, domain;
allow example_app example_file:dir { read write };
```

这条规则允许名为`example_app`的应用读取和写入`example_file`目录。

##### (2) 定义安全上下文

为了让SELinux识别新应用或服务，需在`file_contexts`和`service_contexts`文件中定义其安全上下文。例如，要定义`/data/example`文件的上下文，可以在`file_contexts`文件中添加如下规则：

```bash
bash

 代码解读
复制代码/data/example(/.*)? u:object_r:example_data_file:s0
```

这样，`/data/example`目录及其子目录都将分配到`example_data_file`上下文。

##### (3) 编译和部署

完成策略编辑后，可以通过编译AOSP源码来生成更新的`sepolicy`文件。然后将新策略文件刷写到设备中，通常位于`/system/etc/selinux/`目录下。

#### 调试SELinux策略

配置SELinux策略后，通常需要进行测试和调试，确保策略规则设置合理。调试SELinux策略主要有以下几个步骤：

(1) **查看SELinux日志**

使用`adb logcat | grep "avc: denied"`命令查看被拒绝的操作。每当某个操作被SELinux阻止时，系统会生成`avc: denied`日志，显示被拒绝的进程、资源和操作。

(2)**使用audit2allow**：

`audit2allow`工具可以将日志中的`denied`信息转换为相应的策略语句。这在调试阶段非常有用，帮助开发者生成允许访问的规则，但生产环境应慎用。

```perl
perl 代码解读复制代码# 生成策略规则
adb logcat | grep "avc: denied" | audit2allow -M mycustompolicy
```

(3)**在宽容模式下测试**

在强制模式（Enforcing）下，所有未授权操作会被阻止，而在宽容模式（Permissive）下，系统只记录违规操作而不阻止它们。在测试策略时，可以通过`setenforce 0`将SELinux临时切换到宽容模式，确认策略设置的效果。

#### te文件编写

在编写`.te`文件时，需要遵循特定的语法和规则，以确保SELinux能够正确解析并执行这些策略。以下是编写`.te`文件的基本规则：

##### 声明类型（Type Declaration）

每个进程、文件、目录等资源在SELinux中都有一个类型，类型是SELinux策略的核心。首先需要为你的应用或资源声明类型。

```bash
bash 代码解读复制代码type example_app, domain;
type example_data_file, file_type;
```

- `example_app`为进程类型，用`domain`来指定。
- `example_data_file`为文件类型，用`file_type`来指定。

##### 允许规则（allow）

`allow`语句是`.te`文件中最常见的规则，定义了允许一个类型对另一个类型执行的操作。

```xml
xml

 代码解读
复制代码allow <subject_type> <object_type>:<class> <permissions>;
```

- **subject_type**：发起操作的类型（如进程的类型）
- **object_type**：操作目标的类型（如文件的类型）
- **class**：对象的类别（如文件、目录、进程等）
- **permissions**：允许的具体权限（如读、写、执行等）

比如：允许`example_app`类型的进程对`example_data_file`类型的目录执行读取和写入操作。

```arduino
arduino

 代码解读
复制代码allow example_app example_data_file:dir { read write };
```

##### 角色规则（Role Rules）

角色控制用户能执行的操作，通常用于定义域间的访问规则。常见的写法如下：

```ini
ini

 代码解读
复制代码role example_role types example_app;
```

上述规则将角色`example_role`与`example_app`类型的域关联，使得该角色下的用户可以使用`example_app`类型的进程。

##### 特权规则（Attribute）

属性是一种标签，用于将多个类型分组，便于权限管理。例如，可以创建一个`data_file_type`属性，分配给多个文件类型，这样就能为所有数据文件统一设置权限。

```ini
ini 代码解读复制代码# 定义属性
attribute data_file_type;

# 分配属性
typeattribute example_data_file data_file_type;

# 使用属性
allow example_app data_file_type:file { read write };
```

在上述规则中，`example_app`可以对所有具有`data_file_type`属性的文件执行读写操作。

##### 类型转换规则（type_transition）

类型转换用于自动将一个对象分配特定类型。例如，当`example_app`类型的进程在某目录中创建新文件时，可以自动将这些新文件标记为`example_data_file`类型。

```css
css

 代码解读
复制代码type_transition example_app example_dir:file example_data_file;
```

当`example_app`类型的进程在`example_dir`目录下创建文件时，自动将新文件分配为`example_data_file`类型。

##### 示例

下面一个完整的`.te`文件示例，展示了如何为`example_app`进程配置权限和上下文：

```ini
ini 代码解读复制代码# 定义进程类型和文件类型
type example_app, domain;
type example_data_file, file_type;

# 允许example_app类型的进程读取、写入example_data_file类型的文件
allow example_app example_data_file:dir { read write };

# 自动将example_app进程在/data/example目录中创建的文件分配为example_data_file类型
type_transition example_app example_data_file:file example_data_file;

# 定义角色
role example_role types example_app;
```

#### 注意事项

- **策略最小化**：尽量制定最小化的权限策略，避免授予不必要的权限。策略越精细，系统越安全。
- **生产环境谨慎放宽策略**：`audit2allow`生成的规则应仔细审查后再应用到生产环境，以避免意外放宽系统权限。
- **频繁测试与审计**：SELinux策略配置完成后，应进行定期测试和审计，以确保策略配置的有效性和安全性。

#### SELinux的挑战和未来

虽然`SELinux`为`Android`带来了显著的安全性提升，但它也增加了系统配置的复杂性。每个版本的Android都在不断优化`SELinux`，以更好地兼顾安全性和性能。未来，随着`Android`系统的逐步完善，`SELinux`将在增强系统防护方面发挥越来越重要的作用。

## 参考

SELinux 概念 [source.android.com/docs/securi…](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.com%2Fdocs%2Fsecurity%2Ffeatures%2Fselinux%2Fconcepts%3Fhl%3Dzh-cn)

SELinux实现 [source.android.com/docs/securi…](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.com%2Fdocs%2Fsecurity%2Ffeatures%2Fselinux%2Fimplement%3Fhl%3Dzh-cn)

SELinux验证 [source.android.com/docs/securi…](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.com%2Fdocs%2Fsecurity%2Ffeatures%2Fselinux%2Fvalidate%3Fhl%3Dzh-cn)

Android SELinux开发多场景实战指南 [blog.csdn.net/tkwxty/arti…](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Ftkwxty%2Farticle%2Fdetails%2F104538447)





作者：树獭非懒
链接：https://juejin.cn/post/7317281367113285651
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。