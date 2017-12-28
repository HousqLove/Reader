# 深入理解Java Binder 和 MessageQueue

本章重点分析两个基础点:  
- Binder系统在Java世界如何布局和工作  
- MessageQueue的新职责  

## Java层中Binder架构分析
  Java层中Binder实际上也是一个C/S架构, 并且在其类的命名上尽量和Native层保持一致,因此可以认为,Java层的Binder架构是Native层Binder架构的一个镜像.
- 系统定义了一个IBinder接口类以及DeathRecipient接口.
- Binder类和BinderProxy类分别实现了IBinder接口.其中,Binder类作为服务端的Bn代表,而BinderProxy作为客户端Bp的代表.
- 系统还定义了一个BinderInternal类.该类是一个仅供Binder架构使用的类.它内部有一个GcWatcher类,该类专门用于处理和BInder架构相关的垃圾回收.
- Java层同样提供一个用于承载通信数据的Parcel类.

### 初始化Java层Binder框架
  Android系统中,Java世界初创时期,系统会提前注册一些JNI函数,其中一个函数专门负责搭建Java Binder和Native Binder的交互,该函数就是register_android_os_Binder,代码如下:
```
  [android_util_binder.cpp::register_android_os_binder]
  int register_android_os_Binder(JNIEnv* env){
    //初始化Java Binder类和Native层的关系
    if(int_register_android_os_Binder(env) < 0)
       return -1;
    //初始化Java BinderInternal类和Native层的关系  
    if(int_register_android_os_BinderInternal(env) < 0)
      return -1;
    //初始化Java BinderProxy类和Native层的关系
    if(int_register_android_os_BinderProxy(env) < 0)
      return -1;
    //初始化Java Parcel类和Native层的关系
    if(int_register_android_os_Parcel(env) < 0)
      return -1;
  }
```
  由以上代码可知,register_android_os_binder函数完成了Java层Binder架构的最重要4个类的初始化工作.
#### Binder类的初始化
  在Binder的初始化函数中,通过gBinderOffsets对象保存了和Binder类相关的某些JNI层中使用的信息.包括:execTransact()函数,mObject属性
#### BinderInternal类的初始化
  和BInder的初始化类似,也保存了一些有用的methodID和fieldID.表明JNI层一定会向上调用Java层的函数.
  注册相关类中native函数的实现.
