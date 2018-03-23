# JNI 
 JNI是Java Native Interface。他为Android编写的字节码定义了一种方法，使Android管理的代码（Java或kotlin编写）能和本地代码（C/C++编写）进行交互。
 JNI是独立于供应商的，他支持从动态共享库加载代码，虽然有时非常麻烦，但效率相当高。

# 基本原则
 尽量减少JNI层的占用空间。有几个方面需要考虑。你的JNI解决方案应该遵循这些准则（按照重要性从高到低）：
* 尽量减少JNI层资源的编组。跨越JNI层的编组具有不小的开销。尝试设计一个接口，以最大限度地减少您需要编组的数据量和您必须编组数据的频率
* 尽量避免上层语言和C++代码直接的异步通信。这将使你的JNI 接口更易于维护。一般通过上层语言的异步更新来简化JNI的UI更新。比如在java代码中，不再UI线程中通过JNI调用C++代码，最好在Java语言的两个线程之间回调，其中一个线程阻塞调用C++，然后在调用完成时通知UI线程。
* 尽量减少需要接触或被JNI接触的线程数量。如果你确实需要在Java和C++之间使用线程池，请尝试在池的所有者之间保持通信，而不是在各个工作线程之间通信。
* 将你的接口代码保存在少量易于识别的C++和Java源码中，以方便未来重构。适当的考虑使用JNI自动生成库。

# JavaVM 和 JNIEnv
 JNI定义了两个关键的数据结构，JavaVM和JNIEnv。这两个实际上都是指向函数表指针的指针。（在C++版本中，他们是具有指向函数表指针和贯穿该表的每个JNI函数的类）。JavaVM提供了调用接口，允许你创建和销毁JavaVM。理论上，一个进程可以有多个JavaVM，但是Android只允许一个。

 JNIEnv提供了大部分JNI函数。你的native函数都会手袋一个JNIEnv作为第一个函数。

 JNIEnv保存在thread-local中。因此，不能在线程之间共享一个JNIEnv。如果在一段代码没有其他方法获得JNIEnv，你应该共享JavaVM，并使用GetEnv来发现线程的JNIEnv。

 C里面定义的JNIEnv和JavaVM和C++中不同。“jni.h”根据是否包含C或C++提供不同的的类型定义。因此，在两种语言的头文件中包含JNIEnv参数是一个坏主意。

# Threads
 所有的线程都是Linux线程，由内核调度。他们通常从管理代码（Thread.start）开始，但是也可以在别处创建然后连接到JavaVM。例如，可以通过pthread_create开始一个线程，然后通过JNI AttachCurrentThread或AttachCurrentThreadAsDaemon来连接。连接之前，这个线程没有JNIEnv，也不能进行JNI调用。

 连接一个native创建的线程会导致构造一个java.lang.Thread对象并将其添加到“main ThreadGroup”,使其对调试器可见。在已连接的线程上调用AttachCurrentThread是无效的，不会进行任何操作。

 Android不暂停执行native代码的线程。如果GC执行或者调试器发出暂停请求，Android会在下次执行JNI调用的时候暂停该线程。

 通过JNI连接的线程必须在退出钱调用DetachCurrentThread。如果直接编码比较困难，在Android 2.0和更高版本中，可以使用pthread_key_create来定义在线程退出之前调用的析构函数，并从中调用DetachCurrentThread。（使用pthread_setspecific作为key将JNIEnv存储在thread-local-storage中，这样他就会作为参数传入你的析构函数中。）