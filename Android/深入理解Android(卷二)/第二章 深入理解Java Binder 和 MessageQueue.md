# 第二章 深入理解Java Binder 和 MessageQueue

本章重点分析两个基础点:  
- Binder系统在Java世界如何布局和工作  
- MessageQueue的新职责  

## 2.2 Java层中Binder架构分析
  Java层中Binder实际上也是一个C/S架构, 并且在其类的命名上尽量和Native层保持一致,因此可以认为,Java层的Binder架构是Native层Binder架构的一个镜像.
- 系统定义了一个IBinder接口类以及DeathRecipient接口.
- Binder类和BinderProxy类分别实现了IBinder接口.其中,Binder类作为服务端的Bn代表,而BinderProxy作为客户端Bp的代表.
- 系统还定义了一个BinderInternal类.该类是一个仅供Binder架构使用的类.它内部有一个GcWatcher类,该类专门用于处理和BInder架构相关的垃圾回收.
- Java层同样提供一个用于承载通信数据的Parcel类.

### 2.2.2 初始化Java层Binder框架
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
#### 1. Binder类的初始化
  在Binder的初始化函数中,通过gBinderOffsets对象保存了和Binder类相关的某些JNI层中使用的信息.包括:execTransact()函数,mObject属性
#### 2. BinderInternal类的初始化
  和BInder的初始化类似,也保存了一些有用的methodID和fieldID.表明JNI层一定会向上调用Java层的函数.
  注册相关类中native函数的实现.
#### 3. BinderProxy类的初始化
  BInderProxy的初始化函数中,除了初始化BinderProxy类外,还获取了WeakReference类和Error类的一些信息.故BinderProxy对象的生命周期会委托给WeakReference来管理.

### 2.2.3 addService分析
  通过ActivityManagerService(AMS)揭示Java层Binder的工作原理.分析步骤如下:
1. 分析AMS如何将自己注册到ServiceManager.
2. 分析AMS如何响应客户端的Binder调用.
```
public static void setSystemService(){
	try{
		ActivityManagerService m = mSelf;
		ServiceManager.addService("activity", m);
		...
	}
}
```
  上面代码的目的是将AMS服务注册到ServiceManager中.
  整个Android系统中有一个Native的ServiceManager(SM)进程.它统筹管理Android系统上的所以Service.成为一个Service的必要条件是在SM中注册.
#### 1. 在SM中注册服务
  (1) 创建ServiceManagerProxy
  在SM中注册的函数为addService.
  在addService的过程中,BinderInternal.getContextObject完成了两个工作:
- 创建了一个Java层的BinderProxy对象.
- 通过JNI,该BinderProxy对象和一个Native的BpProxy对象挂钩,而该BpProxy对象的通信目标就是ServiceManager.
  所以Java层的Binder架构最终还是要借助Native的Binder架构进行通信.

### 2.2.4 Java层Binder架构总结  

- 对于代表客户端的BinderProxy来说,Java层的BinderProxy在Native层对应一个BpBinder对象.凡是从Java层发出的请求,首先从Java层的BinderProxy传递到Native层的BpBinder,继而由BpBinder传递到Binder驱动.
- 对于代表服务端的Service来说,Java层的Binder在Native层有一个JavaBBinder对象,而JavaBBinder仅起到中转作用,即把来自客户端的请求从Native层传递到Java层
- 系统依然只有一个Native的ServiceManager.

## 2.3 心系两界的MessageQueue  
  MessageQueue封装了与消息队列有关的操作.Android2.3之前,只有Java层才能向MessageQueue添加消息,从2.3开始,MessageQueue的核心下移至Native层,Native层也能利用消息循环来处理事情.因此现在的MessageQueue心系Native层和Java层两个世界.
### 2.3.1 MessageQueue的创建
  Native层创建了一个与MessageQueue对应的NativeMessageQueue对象,同样也有一个Looper循环处理消息队列中的消息.Looper保存在线程本地存储空间中.*保存在TLS中是一种很常见的以线程为单位的单利模式*  
  Native的Looper是Native层中参与消息循环的重要角色,但是和Java层的Looper无任何关系.
### 2.3.2 提取消息
  准备就绪后,Java层的Looper会在一个循环中通过next函数提取消息.当消息队列为空时,next函数就会阻塞.MessageQueue同时支持Java层和Native层的事件.  
  在next函数中会调用nativePollOnce函数取回消息.  
#### 1 在Java层投递Message  
  MessageQueue的enqueueMessage函数完成将一个Message投递到MessageQueue中的工作.
  主要功能是:
- 将Message按执行时间排序,并加入消息队列
- 根据情况调用NativeWake函数,以触发nativePollOnce函数,结束等待.
#### 2 nativeWake函数分析
  层层调用,最终执行Native层Looper的wake函数. wake函数仅仅向管道的写端写入一个字符'W',这样管道的读端就会因为有数据可读而从等待状态醒来.
### 2.3.3 nativePollOnce函数分析
  最终会调用Native层Looper的pollOnce函数,函数原型如下:  
``` int pollOnce(int timeoutMillis, int* outFd, int* outEvent, void** outData) ```  
  其中:
- timeoutMillis为超时等待时间.如果是-1,表示无限等待.如果为0表示立即返回.
- outFd用来存储发生事件的那个文件描述符
- outEvent用来存储在该文件描述符上发生了哪些事件,目前支持可读,可写,错误和终端4个事件.这4个事件其实是从epoll事件转化而来.
- outData用来存储上下文数据,上下文数据是由用户在添加监听句柄时传递的,他的作用和pthread_create函数最后一个参数param一样,用来传递用户自定义的数据.
  另外,pollOnce函数的返回值也具有特殊的意义:
- 当返回值是ALOOPER_POLL_WAKE时,表示返回由wake触发,也就是管道写段的那次写事件触发的.
- 当返回值是ALOOPER_POLL_TIMEOUT时,表示等待超时
- 当返回值是ALOOPER_POLL_ERROR表示等待过程发生错误
- 当返回值是ALOOPER_POLL_CALLBACK时,表示被监听的句柄因某种原因被触发.这时outFd参数用于存储事件发生的文件句柄,outEvent用于存储发生的事件.
#### 1 epoll基础知识
  epoll机制提供了Linux平台最高效的I/O复用机制.
  从调用方法上看,epoll的用法和select/poll非常类似,其主要作用就是I/O复用,即在一个地方等待多个文件句柄的I/O事件.

  epoll的效率之所以高,是因为epoll只在epoll_ctl进行加入操作的时候才复制一次.另外epoll内部用于保存事件的数据结构使用的是红黑树,查找速度很快.
#### 2 pollOnce函数分析  

#### 3 添加监控请求  
  
#### 4 处理监控请求  
  
#### 5 Native的SendMessage
  
### 2.3.4 MessageQueue总结
  
#### 1 消息处理大家族合照  
- Java层提供了Looper类和MessageQueue类,其中Looper类提供循环处理消息的机制,MessageQueue类提供一个消息队列,以及插入,删除和提取消息的函数接口.
- MessageQueue内部通过mPtr变量保存一个Native层的NativeMessageQueue对象,mMessages保存来自Java层的Message消息.
- NativeMessageQueue保存一个native的Looper对象,该Looper对象从ALooper派生,提供pollOnce和addFd等函数
- Java层有Message类和Handler类,而Native层对应也有Message类和MessageHandler抽象类.在编码时,一般使用的是MessageHandler的派生类WeakMessageHandler类.
#### 2 MessageQueue处理流程总结
  MessageQueue核心逻辑下移到Native层后,极大地拓展了消息处理范围.
- MessageQueue继续支持来自Java层的Message消息.
- MessageQueue在Native层的代表是NativeMessageQueue支持来自Native层的Message,是通过Native的Message和MessageHandler来处理的.
- NativeMessageQueue还处理通过addFDF添加的Request.
- 从处理逻辑上看,先是处理Native的Message,然后是处理Native的Request,最后才是Java的Message.
## 2.4 本章小结
  Java层的Binder架构在通信上还是依赖Native层的Binder架构