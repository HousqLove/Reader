# JNI 
 JNI是Java Native Interface。他为Android编写的字节码定义了一种方法，使Android管理的代码（Java或kotlin编写）能和本地代码（C/C++编写）进行交互。
 JNI是独立于供应商的，他支持从动态共享库加载代码，虽然有时非常麻烦，但效率相当高。

# 基本原则
 尽量减少JNI层的占用空间。有几个方面需要考虑。你的JNI解决方案应该遵循这些准则（按照重要性从高到低）：

* 尽量减少JNI层资源的编组（marshalling 文档用的是这个单词，有编组，组装的意思，例如将军整编部队等，我觉得应该是指Java和JNI的调用或者编译时JNI和Java的调用定义问题）。跨越JNI层的编组具有不小的开销。尝试设计一个接口，以最大限度地减少您需要编组的数据量和您必须编组数据的频率
* 尽量避免上层语言和C++代码直接的异步通信。这将使你的JNI 接口更易于维护。一般通过上层语言的异步更新来简化JNI的UI更新。比如在java代码中，不再UI线程中通过JNI调用C++代码，最好在Java语言的两个线程之间回调，其中一个线程阻塞调用C++，然后在调用完成时通知UI线程。
* 尽量减少需要接触或被JNI接触的线程数量。如果你确实需要在Java和C++之间使用线程池，请尝试在池的所有者之间保持通信，而不是在各个工作线程之间通信。
* 将你的接口代码保存在少量易于识别的C++和Java源码中，以方便未来重构。适当的考虑使用JNI自动生成库。

# JavaVM 和 JNIEnv
 JNI定义了两个关键的数据结构，JavaVM和JNIEnv。这两个实际上都是指向函数表指针的指针。（在C++版本中，他们是具有指向函数表指针和贯穿该表的每个JNI函数的类）。JavaVM提供了调用接口，允许你创建和销毁JavaVM。

 理论上，一个进程可以有多个JavaVM，但是Android只允许一个。

 JNIEnv提供了大部分JNI函数。你的native函数都会收到一个JNIEnv作为第一个函数。

 JNIEnv保存在thread-local中。因此，不能在线程之间共享一个JNIEnv。如果在一段代码没有其他方法获得JNIEnv，你应该共享JavaVM，并使用GetEnv来发现当前线程的JNIEnv。

 C里面定义的JNIEnv和JavaVM和C++中不同。“jni.h”根据是否包含C或C++提供不同的的类型定义。因此，在两种语言的头文件中包含JNIEnv参数是一个坏主意。

# Threads
 所有的线程都是Linux线程，由内核调度。他们通常从管理代码（Thread.start）(指的是编译托管的上层代码，如：Java）开始，但是也可以在别处创建然后连接到JavaVM。例如，可以通过pthread_create开始一个线程，然后通过JNI AttachCurrentThread或AttachCurrentThreadAsDaemon来连接。连接之前，这个线程没有JNIEnv，也不能进行JNI调用。

 连接一个native创建的线程会导致构造一个java.lang.Thread对象并将其添加到“main” ThreadGroup,使其对调试器可见。在已连接的线程上调用AttachCurrentThread是无效的，不会进行任何操作。

 Android不会暂停正在执行native代码的线程。如果GC执行或者调试器发出暂停请求，Android会在下次执行JNI调用的时候暂停该线程。

 通过JNI连接的线程必须在退出前调用DetachCurrentThread。如果直接编码比较困难，在Android 2.0和更高版本中，可以使用pthread_key_create来定义在线程退出之前调用的析构函数，并从中调用DetachCurrentThread。（使用pthread_setspecific作为key将JNIEnv存储在thread-local-storage中，这样他就会作为参数传入你的析构函数中。）

# jclass, jmethodID, and jfieldID

 如果你想在native代码中访问一个对象的成员变量，你需要：

- 使用findClass得到这个类的对象引用
- 使用getFieldID得到成员变量的fieldID
- 使用getIntField之类的方法得到这个field里的内容

 同样，要调用一个方法，你首先需要一个类对象的引用，然后获得一个methodID，这些ID通常只是指向内部运行时数据结果的指针。查看他们可能需要几次字符串的比较，但是一旦有了他们，实际调用来获取字段或调用方法是非常快的。

 如果性能很重要，那么查找一次这些值并缓存在native代码中会很有用。因为一个进程只能有一个JavaVM，所以将这些数据存储在静态本地结构中是合理的。

 在类被卸载之前，类引用，fieldID和methodID一直有效。只有被类加载器装载的所有类都在要回收的范围内，类才会被回收，在Android中这很少见，但不是不可能。注意：即使jclass是一个类引用，也必须通过调用NewGlobalRef来保护（参阅下一节）。

 如果你想在类加载时缓存这些ID，然后在类被卸载又重新加载是重新缓存他们，那么正确方法是把这些代码添加到合适的class内，如：

```
	private static native void nativeInit();

	static {
		nativeInit();
	}
```

 在执行ID查找的C/C++代码中创建一个nativeClassInit方法。代码将在类初始化时执行一次。

# Local and Global References

 传递给native方法的每个参数和JNI返回的每个大部分对象都是本地引用。这就意味着他们只在当前线程的当前方法中有效。即使在native方法返回后，对象本身还在活动，这个引用也是无效的。当启用JNI检查时，运行时会警告大多数引用错误，这对jobject的所有子类都有效，比如jclass，jstring，jarray。

 获得非本地引用的方法是通过NewGlobalRef和NewWeakGlobalRef。

 如果你想持有更长期的引用，就必须使用全部引用。NewGlobalRef函数将本地引用作为参数并返回一个全局引用。全局引用在调用DeleteGlobalRef之前一直有效。如：

```
	jclass localClass = env->FindClass("MyClass");
	jcalss globalClass = reinterpret_cast<jclass>(env->NewGlobalRef(localClass));
```
 
 所有的JNI方法都接受本地和全局引用作为参数。对同一对象的引用有可能具有不同的值。比如，在同一个对象上连续调用NewGlobalRef的返回值可能不同。想要确认两个引用是否引用的是同一个对象，必须使用IsSampleObject函数而不能使用==。

 这样做的结果就是你不能认为对象引用在native代码中是常量或唯一的。有可能表示32位值的对象在下一次调用方法之后就不同了，也有可能两个不同的对象在连续调用时可能具有相同的32位值。

 所以不要使用jobject作为key。

 程序员需要做到不过度分配本地引用。这意味着，如果你创建大量的本地引用，你应该使用DeleteLocalRef手动释放他们，而不是让JNI为你做释放。该实现仅需要为16个本地引用预留插槽，因此如果您需要更多，则应该随时删除或使用EnsureLocalCapacity / PushLocalFrame预留更多。

 jfieldID和jmethodID是不透明类型，不是对象引用，不应该传递给NewGlobalRef。像GetStringUTFChars和GetByteArrayElements这样的函数返回的原始数据指针也不是对象。但他们可以在线程之间传递，并且直到调用释放都有效。

 如果使用AttachCurrentThread连接一个native线程，则在线程detac之前，运行的代码永远不会自动释放本地引用。你创建的任何本地引用都必须手动删除。一般情况下，任何native代码中，在循环中创建的本地引用可能都需要手动删除。

 小心使用全局引用。全局引用可能不可避免，但他们难以调试，并且可能造成难以诊断的内存（错误）行为。所有其他条件相同的条件下，全局引用较少的解决方案可能更好。

# UTF-8 and UTF-16 Strings

 Java编程语言使用UTF-16.为了方便，JNI提供了与修改的UTF-8一起使用的方法。修改的代码对于C代码非常有用，因为它将\ u0000编码为0xc0 0x80而不是0x00。关于这一点的好处是，您可以依靠C风格的零终止字符串，适用于标准的libc字符串函数。不利的一面是，你不能将任意的UTF-8数据传递给JNI，并期望它能正常工作。

 如果可能的话，使用UTF-16字符串操作通常会更快。Android目前不需要GetStringChars中的副本，而GetStringUTFChars需要分配和转换为UTF-8。请注意，UTF-16字符串不是零终止的，而\ u0000是允许的，因此您需要挂上字符串长度以及jchar指针。

 不要忘记释放你获得的字符串。字符串函数返回jchar *或jbyte *，它们是指向原始数据的C型指针而不是本地引用。它们在调用Release之前保证有效，这意味着它们在native方法返回时不会被释放。

 传递给NewStringUTF的数据必须采用Modified UTF-8格式。一个常见的错误是从文件或网络流中读取字符数据，并将其传递到NewStringUTF而不进行过滤。除非您知道数据是7位ASCII，否则您需要去除高位ASCII字符或将其转换为适当的修改后的UTF-8格式。如果你不这样做，UTF-16转换可能不会成为你期望的。扩展的JNI检查将扫描字符串并警告您有关无效数据，但它们不会捕获所有内容。

# Primitive Arrays

 JNI提供了访问数组对象内容的函数。虽然对象数组必须一次访问一个entry，但是基元数组可以直接读取和写入，就好像它们是在C中声明的一样。

 为了使接口在不限制VM实现的情况下尽可能高效，Get <PrimitiveType> ArrayElements调用系列允许运行时返回指向实际元素的指针，或者分配一些内存并进行复制。 无论哪种方式，返回的原始指针保证有效，直到发出相应的Release调用（这意味着，如果数据未被复制，则数组对象将被固定并且不能作为压缩 堆）。 你必须释放你获得的每一个数组。 另外，如果Get调用失败，则必须确保您的代码不会尝试稍后释放NULL指针。

 您可以通过为isCopy参数传入非NULL指针来确定数据是否已被复制。 这很少用。

 Release调用采用可以具有三个值之一的模式参数。 运行时执行的操作取决于它是否返回了指向实际数据的指针或它的副本：