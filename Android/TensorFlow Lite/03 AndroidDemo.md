# Android Demo
 
 GitHub上提供了使用TensorFLow Lite的示例Android应用程序。 该演示是一个示例相机应用程序，使用量化的Mobilenet模型或浮点Inception-v3模型连续分类图像。 要运行演示，需要运行Android 5.0（API 21）或更高版本的设备。

 在演示应用程序中，使用TensorFlow Lite Java API进行推理。该演示应用程序实时对帧进行分类，显示最可能的分类。它还显示检测对象所用的时间。

 有三种方法可以将演示应用程序添加到您的设备中:
1. 下载预建的[APK](http://download.tensorflow.org/deps/tflite/TfLiteCameraDemo.apk)
2. 使用Android Studio构建应用程序
3. 下载TensorFlow Lite的源代码和演示，并使用bazel构建它

## 使用Android Studio构建应用程序

 使用Android Studio尝试更改项目代码并编译演示应用程序:
1. 安装最新版本的Android Studio
2. 确保Android SDK版本大于26且NDK版本大于14（在Android Studio设置中）。
3. 将tensorflow / contrib / lite / java / demo目录导入为新的Android Studio项目。
4. 安装它请求的所有Gradle扩展
 
 构建并运行演示应用程序.

 构建过程下载量化的Mobilenet TensorFlow Lite模型，并将其解压缩到assets目录：tensorflow / contrib / lite / java / demo / app / src / main / assets /

 [TF Lite Android App](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/lite/java/demo/README.md)页面上提供了一些其他详细信息

## 使用其他模型

1. Download the floating point [Inception-v3 model](https://storage.googleapis.com/download.tensorflow.org/models/tflite/inception_v3_slim_2016_android_2017_11_10.zip)
2. 将inceptionv3_non_slim_2015.tflite解压缩并复制到assets目录
3. 在Camera2BasicFragment.java中更改所选的分类器  
	from: classifier = new ImageClassifierQuantizedMobileNet(getActivity());  
	to: classifier = new ImageClassifierFloatInception(getActivity());.