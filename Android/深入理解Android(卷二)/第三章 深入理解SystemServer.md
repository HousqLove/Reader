# 深入理解SystemServer

## 3.1 概述
  SystemServer是Android Java的两大支柱之一。另外一个支柱是专门负责孵化Java进程的Zygote。若Android Java真的崩溃了，Linux系统中的进程init会重新启动两大支柱以重建Android Java。  
  SystemServer和系统服务有着重要关系。Android中几乎所有的核心服务都在这个进程中。
## 3.2 SystemServer分析
  SystemServer是由Zygote孵化而来的一个进程，进程名为system_server。
### 3.2.1 main函数分析
  main函数是SystemServer核心逻辑的入口函数。首先做一些初始化工作，然后加载动态库libandroid_server.so，最后调用native的init1函数。
  SystemServer的main函数通过init1函数从java层穿越到native层，做了一些初始化工作后，又通过JNI从Native层穿越到Java层去调用init2函数。
  init2函数返回后，最终又回到Native层。
### 3.2.2 Service群英会
  
  init2函数中启动了一个线程ServerThread。Android平台众多Service都汇聚在这个线程中。  

- 第一类服务：ActivityManagerService、PowerManagerService、PackageManagerService、WindowManagerService
- 第二类服务：NetWorkManagerService、NetworkTimeUpdateService、NetworkPolicyManagerService、NetworkStatsService、WiFiService、TelephoneRegister
- 第三类服务：ContentService、AccountManagerService、MountService、AudioService、TextServicesManagerService、SearchManagerService、LocationManagerService、AccessibilityManagerService、UsbService、DevicePolicyManagerService、RecognitionManagerService
- 第四类服务：BatteryService、LightService、AlarmManagerService、VibratorService
- 第五类服务：EntropyService、ClipboardService、DiskStatsService、DropBoxManagerService、SamplingProfilerService、DeviceStorageMonitorService、Watchdog
- 第六类服务：BluetoothA2dpService、BluetoothService
- 第七类服务：AppwidgetService、InputMethodManagerService、UIModeManagerService、StatusBarManagerService、NotificationManagerService、WallpaperManagerService
  第一大类是Android的核心服务。第二大类是和通讯相关的服务。第三大类是和系统功能相关的服务。第四打雷是BatteryService。VibratorService等服务。第五打雷是EntropyService，DiskStatsServiceWatchdog等相对独立的服务。第六大类是蓝牙服务。第七大类是和UI紧密相关的服务。

## 3.3 EntropyService分析
  根据物理学基本原理，一个系统的熵越大，该系统越不稳定。在Android中，目前也只有随机数常处于这种不稳定的系统中了。
  EntropyService构造函数一次调用了4个关键函数。

- loadInitialEntropy函数：将entropy.dat文件的内容写到urandom设备。这样可以增加系统的随机性。
- addDeviceSpecificEntropy函数：将一些和设备相关的信息写入urandom。即使向urandom的Entropy pool中写入固定信息，也能增加随机数生成的随机性。
- writeEntropy函数：读取urandom设备的内容到entropy.dat文件。
- scheduleEntropyWriter函数：向EntropyService内部的Handler发生衣蛾ENTROPY_WHAT消息。该消息每3小时发送一次。收到消息后，EntropyService会再次调用writeEntropy函数，将urandom设备的内容写到entropy.dat中。

## 3.4 DropBoxManagerService分析
  DropBoxManagerService（DBMS）用于生成和管理系统运行时的一些日志文件。这些日志文件大多记录的是系统或某个应用程序出错时的信息。
### 3.4.1 DBMS构造函数分析
  DBMS注册一个BroadcastReceiver对象，同时会监听Settings数据库的变动。系统启动完毕时，设备存储空间不足时，Settings数据库相应项发生变化时，会触发onRecive函数。onReceive函数主要功能是存储空间不足时，需要删除一些旧的日志文件以节省存储空间。
### 3.4.2 Dropbox日志文件的添加
  当某个应用因为发生异常而崩溃时，ActivityManagerService（AMS）的handleApplicationCrash函数被调用，里面会调用addErrorToDropBox函数。
### 3.4.3 DBMS和Settings数据库
  DBMS的运行依赖一些配置项。其实除了DBMS，SystemServer中很多服务都依赖相关的配置项。这些配置项都是通过SettingsProvider操作Settings数据库来设置和查询的。SettingsProvider是系统中很重要的一个APK，如果将其删除，系统就不能正常启动了。  
  这里总结一下和DBMS相关的配置项。
  //用来判断是否允许记录该tag类型的日志文件。默认是允许生成任何tag类型的文件
  Secure.DROPBOX_TAG_PREFIX+tag:"dropbox:" + tag
  //用于控制每个日志文件的存活时间，默认是三天。大于三天的日志文件就会被删除以节省空间
  Secure.DROPBOX_AGE_SECONDS:"dropbox_age_seconds"
  //用于控制系统保存的日志文件个数，默认是1000文件
  Secure.DROPBOX_MAX_FILES:"dropbox_max_files"
  //用于控制dropbox目录最多占存储空间容量的比例，默认是10%
  Secure.DROPBOX_QUOTE_PERCENT:"dropbox_quote_percent"
## 3.5 DiskStatsService和DeviceStorageMonitorService分析
  DiskStatsService、DeviceStorageMonitorService与系统内部存储管理、监控有关。
### 3.5.1 DiskStatsService分析
  DiskStatsService从Binder派生，却没有实现任何接口