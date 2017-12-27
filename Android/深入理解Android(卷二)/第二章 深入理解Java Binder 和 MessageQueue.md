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
