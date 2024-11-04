### Debian12工作机学习笔记

#### 系统安装：

​	略



#### 环境变量配置：

#####  java环境：

​	情况概述：根据实际情况，目前的安卓开发，可能会需要用到1.8及11的版本。11其实Android Studio中是含有的，以及17的版本都是包含的，但是为了整体环境的可靠，感觉还是需要安装比较好，其中1.8版本可选，如果有老项目需要使用才需要安装。

​	多版本切换：因为是需要同时使用多个版本的，如果单纯安装，手动配置java home能够实现，但是每次切换版本都需要手动去更改环境变量。使用`update-alternatives`工具来切换版本。

​	如果直接使用apt去下载安装java，那么直接通过`sudo update-alternatives --config java`指令进行安装版本的选择切换就好。

​	如果是手动下载的java，就需要通过 update-alternatives工具的指令来创建软索引。



##### Android环境：

​	前提是java环境需要配置好。

​	下载并解压Android Studio，指定好Android SDK的位置

​	设置环境变量：

```tex
		#android sdk 替换为自己的SDK路径
		export ANDROID_SDK=...
		export PATH=${PATH}:$ANDROID_SDK/tools:$ANDROID_SDK/platform-tools:$ANDROID_SDK/tools/bin:$ANDROID_SDK/emulator
```

​	此时adb等可正常使用。



##### Flutter环境：

​	如果需要使用到多个flutter版本，可以使用`fvm`工具。

​	首先导入镜像，由于在国内访问Flutter有时可能会受到限制，Flutter官方为中国开发者搭建了临时镜像，大家可以将如下环境变量加入到用户环境变量中：

```tex
#Flutter 
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn

```

​	下载源码: https://flutter.dev/docs/development/tools/sdk/releases?tab=linux#macos

​	

###### 	安装fvm：

​		在fvm工具的官网中，https://fvm.app 中获取fvm的包。

​		使用fvm需要安装dart环境。

###### 		安装dart：

​			在dart官网，https://dart.cn/get-dart/ 中获取安装dart的指令。

​		安装好dart之后，配置fvm的环境变量即可。



```
#set fvm
export FVM_CACHE_PATH=...
export PATH=${PATH}:$FVM_CACHE_PATH
```

​		

​		在fvm下建立versions文件夹，导入下载好的flutter版本，文件夹以flutter的数字版本号命名，例如：2.2.0

​		然后检查是否正常，终端输入fvm list，正常输出版本成功。

​		fvm global 配置全局的flutter版本

​		fvm use在对应的flutter项目下使用，可以给项目设置使用的flutter版本

​		fvm配置完成。



flutter 配置好之后，可以使用flutter doctor来检查flutter配置的情况。

