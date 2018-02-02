# Gradle包装器
 Gradle包装器是是Gradle的一个核心特性，它允许你的机器不休奥安装运行时就能运行Gradle脚本，而且还能确保build脚本运行在指定版本的Gradle。
 他会从中央仓库中自动下载Gradle运行时，解压到你的文件系统，然后用来build。
## 配置Gradle包装器
 在这是包装器之前，需要做两件事：创建一个包装任务，执行这个任务生产包装文件。
 定义一个Wrapper类型的任务，在里面指定你想使用的Gradle版本：  
 ```
	 task wrapper(type: Wrapper){  
	 	gradleVersion = '1.7'  
	 }
```  
 执行这个任务  
 `gradle wrapper`  
 执行完成之后，就可以看wrapper文件和你的构建脚本。  
 只需要运行gradle wrapper一次，以后就可以用wrapper来执行你的任务，下载下来的wrapper文件会被添加到版本控制系统中。
 如果系统已经安装了Gradle运行时，就不需要再添加一个gradle wrapper任务，可以直接运行gradle wrapper任务，这个任务会使用你的Gradle当前笨笨来生成包装文件。  
## 自定义包装器
 只需要修改默认配置：
```
  task wrapper(type: Wrapper){
  	gradleVersion = '1.2'
  	distributionUrl = 'http://myenterprise.com/gradle/dists'
  	distributionPath = 'gradle-dist'
}
```
 还可以了解更多关于包装器DSL的信息：[http://gradle.org/docs/current/dsl/org.gradle.api.tasks.wrapper.Wrapper.html](http://gradle.org/docs/current/dsl/org.gradle.api.tasks.wrapper.Wrapper.html)