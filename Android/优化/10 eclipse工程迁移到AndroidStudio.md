# 记录工作项目从Eclipse迁移到AndroidStudio的过程

 这是一个古老而庞大的项目，中间几经变迁，最终发展成现在的样子。如今在我手上要迁移到Android studio上，以此记录。

## 项目概述

 本身是一个Android项目（如果不是也不会有这次迁移了），支持最小版本14，目标版本19，Eclipse依赖的Android api是4.4.2。  

 项目包含工程依赖（依赖多个库，传递依赖），jni使用，第三方库等。麻雀虽小，五脏俱全了。

## 迁移流程及问题

 肯定会遇到问题的，不然也不会有本次博客了。

 本身Android Studio提供的有eclipse项目迁移的功能的。可是发现导入的工程会是多个module并行存在的情况，跟新创建的Project下包含不同的module的架构有点不同，虽然没什么问题，可是看了浑身不舒服，于是采用新建项目，新建module，复制代码的笨方法。

 搬砖开始...

### 问题1：乱码问题

 终于把代码复制到新工程下，问题接踵而至，“编码不匹配”，这个划时代的问题还是让我碰到了，没错，之前的项目是GBK编码的（!_!）。

 于是乎，```reload in another encoding```,选择GBK，然后提示reload or convert，点击reload，然后重新加载文件。文件已经能正常显示了。可是我要的是葡萄，于是选中文件，菜单栏：File -> File Encoding -> utf8，再次提示reload or convert,选择convert。emm..葡萄就到手了。

 看起来那么完美，可是鬼知道我这个项目有多少文件，于是把eclipse里文件的内容复制，在studio中找到对应的文件粘贴，竟然也能拿到葡萄。

 于是发扬码农精神，左边eclipse，右边studio，撸起袖子加油干，幸亏我除了笔记本之外还有个显示器,程序员果然应该多显示器，前人诚不欺我。。。

### 问题2：HttpClient找不到

 没错，我还在用httpClient，eclipse下的依赖api是4.4.2，迁移到studio 3.0.1之后没有强行改低版本，使用的编译版本是26（8.0），步子迈的大了。

 谷哥在6.0的时候移除了HttpClient的相关类，推荐使用HttpUrlConnection,但是我怎么能轻易就妥协呢，于是在对应module的gradle文件里android配置项内增加```useLibrary 'org.apache.http.legacy'```,然后就能愉快的使用HttpClient了。

### 问题3：依赖项配置

 库工程还没有健康起来，还是有文件没找到，```org.simalliance.openmobileapi.SEService```，就是SmartcardAPI相关的类找不到了，这是什么玩意？

 知道的人已经知道了，不知道的人还不知道。

 我的项目需要访问SIM卡，而SmartcardAPI就是访问SIM卡所需的包。他是Open Mobile API的一个实现，使Android应用程序能够和安全原件进行通信，这是一个开源项目，如今各大手机厂商基本都会使用到这个包。所以，我的项目里安全原件就是SIM卡了。同时这个包是不需要打包进apk的，因为手机里已经有了，这也是在这里单独拿出来的原因。

 列举一下gradle配置依赖项的选项（强行学习）：

- implementation：替换弃用的compile。  
  依赖项在编译时对模块可用，并且仅在运行时对模块的消费者可用。对于大型多项目的构建，使用implementation而不是api/compile可以显著缩短构建时间，因为他可以减少构建系统需要编译的项目量。大多数应用和测试模块都应该使用此配置。

- api：替换弃用的compile。  
  依赖项在编译时对模块可用，并且在编译时和运行时还对模块的消费者可用。此配置的行为类似于compile（已弃用），一般情况下，应当仅在库模块中使用他。应用模块使用implementation，除非你想将其api公开给单独的测试模块。
- compileOnly：替换弃用的provided。  
  依赖项仅在编译时对模块可用，并且在编译和运行时对其消费者不可用，此配置的行为类似于provided（已弃用）。

- runtimeOnly：替换弃用的apk。  
  依赖项仅在运行时对模块及其消费者可用。

  综上所述：我应该把SmartcardAPI的jar包单独拿出来使用compileOnly依赖。

### 问题4：资源重复定义

 重新编译，还是有问题，没有问题才怪。。。

 看控制台报Duplicate resources错误，原来是app module下的字符串资源定义重复了，原谅我的项目的字符串资源在多个文件下面，eclipse下面咋就不报错呢？迷茫脸。。。

 不过小问题而已，干掉一个解决之。。。

### 问题5：.9文件不合法

 重新编译，出错，内心毫无波动。

 看控制台```error: image must be at least 3x3 (1x1 image with 1 pixel border).```，看是看懂了，大概就是要用至少3*3的图片，而我用的图片不合法。那么是哪个module？不知道，哪个图片？也没说。wtf。。。

 不过肯定难不倒我程序员的，看我手动DFS。

 终于让我找到了一些只有三个边有黑线的图片，加上第四条黑线，幸亏AS有.9的编辑功能，感谢谷哥，感谢png。

 编译，问题还在，clean project，问题还在，删除build文件夹再编译，问题还在，settings.gradle文件下注释掉其他module，只保留app再编译，问题还在。。。

 难道真的有宽或高小于3像素的图片存在？事实证明他是真的存在。

 重命名再编译，还有问题但已经不是这个了，我有句mmp不知当讲不当讲，老老实实当个png不好么，瞎凑什么热闹

### 问题:6：帧动画放置位置不正确

 果然解决了上一个问题就可以解决下一个问题了，报一个帧动画的xml文件未找到，原来是之前AS提示我帧动画应该放到drawable文件夹下面的时候我移过去了，修改下引用到drawable资源解决，又涨姿势了。不过这个错误不处理也行，不影响运行。

### 问题:7：权限申请重复

 所幸只是个警告，和问题6一块发现的。

### 问题8：style参数未找到

 问题不断，```Error:(2783) error: style attribute '@android:attr/windowEnterAnimation' not found.```参数未找到。

 AS3.0之后已经不支持@开头使用android自带的属性去掉@改为```<item name="android:windowEnterAnimation">@anim/slide_from_bottom</item>```解决。果然还是应该早点迁移过来啊

### 问题9：帧动画资源未找到

 app没有依赖添加的lib工程，吓得我赶快把依赖上。编译之。。。

 不要问为啥app使用的资源在库工程中，我是不会回答的。

 随着问题的解决，构建时发现问题的时间也越来越长，最终编译结束了，并没有提示错误，程序已经能运行了吗？还差的远呢。

 即使如此，也强制跑一下吧，这时才发现，由于创建工程的时候选择的是没有Activity，虽然后来在清单文件中添加了Launcher，配置并没有同步更新，这时点击File -> Invalidate Caches / Restart -> 选择丢弃缓存并重启。然而并没有什么卵用，看来只能设置启动指定的Activity了，指定MainActivity（理论default）启动。

### 问题10：反射调用方法传参错误

 不跑不知道，一跑吓一跳。预期的崩溃并没有出现，因为编译都没通过，哈哈哈。。。

 错误是：
> 警告: 最后一个参数使用了不准确的变量类型的 varargs 方法的非 varargs 调用;  
>										methods[i].invoke(o, null));  
>    									                     ^  
> 对于 varargs 调用, 应使用 Object  
> 对于非 varargs 调用, 应使用 Object[], 这样也可以抑制此警告

 上面的^指向的是null，原谅我不会使用markdown。

 红彤彤的错误显示着，仍大言不惭说警告，你怎么能这么任性。

 原来代码中通过反射调用方法的时候，需要传```Object...```的方法参数，而我传的是null。你是IDE，听你的。

 改成```new Object[]{}```，通过。

### 问题11：Base64找不到方法。

> Base64.encodeBase64URLSafeString() 找不到符号
 
 RSA加密解密时，需要Base64编码，eclipse时使用的common包，在AS下报找不到方法的错误。网上有人说是android包含了老版本的包，导致程序找不到方法。不知道为啥会影响到我的程序。问题不能不解决，android提供了Base64的类，全称是```android.util.Base64```使用这个类进行编码，不知道会不会出问题，先记录下。修改如下：

```
	Base64.encode("foobar".getBytes(), Base64.Base64.NO_WRAP);
```

### 问题12：依赖传递问题

 eclipse中有依赖传递的机制，就是如果依赖库工程就会把库工程的代码，jar包，资源等全都拿来用，如果主工程中也添加相同的jar包反而会报错。在AS中好像不是这么回事。并且eclipse可以依赖任何目录下的jar包，AS只能依赖各自module下的jar包。

 简单介绍下项目依赖，库工程B依赖库工程A，主工程app依赖库工程B、库工程C,D。大概就这样吧。

 那么问题来了，我的项目中引用依赖，传递依赖，应有尽有。下面允许我列一个表格：

| 序号 | eclipse中依赖情况 | AS报错 | 解决方案 |
|------|-------------------|--------|----------|
|  1   | A中存在的jar包，A、B都有使用    | B中报找不到类的错误 | A中使用api依赖jar包 |
|  2   | app中存在的jar包，B中使用 | B中报找不到类 | 移动jar包到B中，并api依赖 |
|  3   | A，B中都存在的jar包，A、B都有使用 | 无 | 修改A中jar包使用api依赖，同时删除B中的jar包 |
|  4   | app中存在的jar包，B和app中使用 | B中找不到类 | 移动jar包到B并api依赖，同时删除app中的jar包 |
|  5   | app依赖B，B依赖A | app找不到A中的类 | B使用api依赖A |
|  6   | 

和问题3应该属同一类问题。刚开始的时候还想各个module下各放一个jar包，并使顶层库使用implementation依赖，下层库或app使用compileOnly依赖，真是惭愧。

### 问题13：库工程权限问题

 以为只要在主工程中添加相应权限就行了，结果AS报错了，那就在module中再加一次吧。

### 问题14：修改库工程依赖为radle依赖

 之前接入过神策的数据统计库，因为之前在eclipse中所以直接通过下载了源码接入，这次直接通过gradle引入，操作参考接入文档

### 问题15：Unable to merge dex

>Error:Execution failed for task ':app:transformDexArchiveWithExternalLibsDexMergerForDebug'.
> java.lang.RuntimeException: com.android.builder.dexing.DexArchiveMergerException: Unable to merge dex

 上面是问题描述，谷歌百度，有的说是方法数超过了65535，有的说重复依赖了不同版本的库，好像都解决不了我的问题，探索中...

 找到了解决问题12序号3时遗留的jar包，编译问题还在。。

 找到了A库中libs下的support-v4的jar包和gradle自动导入的jar包同时存在的情况，于是删除A中的jar，在gradle依赖v4，编译通过了。

 好吧，我的问题还是逃不出上面的范畴

### 问题16：java.lang.UnsatisfiedLinkError

 没想到竟然直接编译成apk安装的手机上了，不过运行肯定是不行的。看日志是对应的so文件没找到。

 期待已久的NDK支持果然姗姗来迟。

 由于决定采用CMake的方式构建NDK，将步骤记录于此：

1. 新建源文件目录。在```src\main```新建目录cpp，并将.cpp,.h文件复制过来
2. 配置ndk编译配置。在app目录下新建文件CMakeLists.txt, 在其中编辑ndk的编译配置，包括cmake_minimum_required、add_library、find_library、target_link_libraries等。
	我的项目比较简单只配置了部分选项，如下：

```
	cmake_minimum_required(VERSION 3.4.1)

	add_library( # Sets the name of the library.
         transport

         # Sets the library as a shared library.
         SHARED

         # Provides a relative path to your source file(s).
         src/main/cpp/transport.cpp)

    find_library( # Sets the name of the path variable.
         log-lib

         # Specifies the name of the NDK library that
         # you want CMake to locate.
         log )

    target_link_libraries( # Specifies the target library.
        transport

        # Links the target library to the log library
        # included in the NDK.
        ${log-lib}
        )
```

3. 将ndk与gradle关联。在gradle文件的android选项中添加externalNativeBuild选项，并在其中添加cmake选项，配置gradle的ndk外部编译为CMake。在defaultConfig选项中添加externalNativeBuild选项，配置编译参数，此参数可在构建类型和风味中复写。在defaultConfig选项中添加ndk选项，并在其中配置abiFilters，配置目标abi。具体配置如下：
```
	android{
		defaultConfig{
			...


	//        externalNativeBuild{
	//            cmake{
	//                // Passes optional arguments to CMake.
	//                arguments "-DANDROID_ARM_NEON=TRUE", "-DANDROID_TOOLCHAIN=clang"
	//
	//                // Sets optional flags for the C compiler.
	//                cFlags "-D_EXAMPLE_C_FLAG1", "-D_EXAMPLE_C_FLAG2"
	//
	//                // Sets a flag to enable format macro constants for the C++ compiler.
	//                cppFlags "-D__STDC_FORMAT_MACROS"
	//            }
	//        }

	        ndk {
	            abiFilters 'x86', 'x86_64', 'armeabi', 'armeabi-v7a', 'arm64-v8a'
	        }
		}

		externalNativeBuild{
	        cmake{
	            path 'CMakeLists.txt'
	        }
    	}
	}
```

### 问题17：签名文件替换

 经过多次clean、build程序终于在手机上跑起来了，不过提示签名校验失败。

 因为我的项目对要使用专门的签名文件才行，写到这里突然想到（测试环境要校验毛的签名文件啊。。。。。。）

 既然写到这里了还是先换掉签名文件吧。同样在gradle文件中修改，在android选项下增加signingConfigs选项，在里面配置不同的签名信息。在buildTypes选项的不同构建类型中选择不同的签名配置。中如下：

```
	android{
		...
		singingConfigs {
			xian{
				keyAlias 'androiddebugkey'
				keyPassword 'android'
				storeFile file('...')
				storePassword 'android'
			}
			...
		}

		buildTypes{
			debug {
				signingConfig signingConfigs.xian
			}
		}
	}
```

### 问题18：混淆

 由于eclipse的机制问题，编译打包的时候没法混淆库工程，所以采用proguardGui工具进行混淆，到了AS当然要借助gradle这个自动化工具了。

 在复制之前的混淆配置到库工程的proguard-rules.pro文件里，至于会不会生效后续再验证。

### 问题19：The same input jar is specified twice

 打release包的时候会报jar包在配置混淆的时候指定了两次，原因是因为在gradle文件中配置了
```
	dependencies{
		implenmentation fileTree(include:['*.jar'], dir: 'libs')
	}
```
 里面的jar包已经默认配置到打包脚本中了，所以不需要再次手动添加，将里面的```-libraryjars```注释掉即可。
 
### 问题20：运行

 修改完了之后再次生成release包，2m42s之后提示成功，安装运行也成功。长吁一口气.jpg

### 问题21：配置打包

 终于到了使用AS的真正目的的时候了，gradle的自动化构建。还是先学习一波。

#### （1）构建流程

 将项目转换成Android应用软件包（APK）的流程。通常步骤如下：

1. 编译器将源代码转换成DEX（Dalivk Executable）文件。将所有其他内容转换成已编译资源。
2. APK打包器将DEX文件和已编译资源合并成单个APK。
3. APK打包器使用秘钥签署APK
4. 打包器使用zipalign对应用优化。

#### （2）自定义构建配置

 Gradle和Android插件可帮助完成以下方面的配置：

1. 构建类型  
 构建类型定义Gradle在构建和打包应用时使用的某些属性，通常针对开发生命周期的不同阶段进行配置。例如：调试构建类型支持调试选项，使用调试秘钥签署apk；而发布构建类型则可压缩，混淆apk以及使用发布秘钥签署apk进行分发。

2. 产品风味  
 产品风味代表你可以发布给用户的不同应用版本，例如免费和付费。你可以将产品风味定义成使用不同的代码和资源，同时对所有应用版本共有的部分加以共享和重复利用。

3. 构建变体
 构建变体是构建类型和产品风味交叉的产物，是gradle在构建应用时使用的配置。您可以利用构建变体在开发时构建产品风味的调试版本，或者构建已签署的产品风味发布版本进行分发。构建变体并不是直接配置的，而是配置组成构建变体的构建类型和产品风味。

4. 清单条目
 您可以为构建变体配置中清单文件的一些属性指定值。这些构建值会替换清单文件中的现有值。

5. 依赖项
 构建系统管理来自您的本地文件系统以及来自远程存储区的项目依赖项。

6. 签署
 构建系统能够让您在构建配置中指定签署设置，并可在构建过程中自动签署apk。

7. ProGuard
 构建系统让您能够为每个构建变体指定不同的ProGuard规则文件。构建系统可在构建过程中运行ProGuard对类进行压缩和混淆处理。

8. APK拆分
 构建系统能够自动构建不同的apk，并且每个apk只包含特定屏幕密度或应用二进制接口所需的代码和资源。

 而我的项目需求也是很特殊的。测试模式下，一个是需要使用A库编译出的不同代码
