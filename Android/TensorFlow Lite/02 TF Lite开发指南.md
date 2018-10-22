# 开发指南

 在移动应用中使用TensorFlow Lite模型需要多个注意事项：您必须选择预先训练的或自定义模型，将模型转换为TensorFLow Lite格式，最后将模型集成到您的应用中。

## 选择一个模型

 您可以选择一种流行的开源模型，例如InceptionV3或MobileNets，并使用自定义数据集重新训练这些模型，甚至可以构建您自己的自定义模型。

- 使用预置训练的模型

	- MobileNets是TensorFlow的移动优先计算机视觉模型系列，旨在有效地最大限度地提高准确性，同时考虑到设备或嵌入式应用程序的受限资源。 MobileNets是小型，低延迟，低功耗模型，参数化以满足各种用途的资源限制。 它们可用于分类，检测，嵌入和分割 - 类似于其他流行的大型模型，例如Inception。 Google为MobileNets提供了16个经过预先培训的ImageNet分类检查点，可用于各种规模的移动项目。

	- Inception-v3是一种图像识别模型，可以实现相当高的准确度，可以识别1000个类别的一般对象，例如“斑马”，“达尔马提亚”和“洗碗机”。 该模型使用卷积神经网络从输入图像中提取一般特征，并基于具有完全连接和softmax层的那些特征对它们进行分类。

	- On Device Smart Reply是一种设备上模型，通过建议与上下文相关的消息，为传入的文本消息提供一键式回复。 该模型专为内存受限设备（如手表和手机）而构建，并已成功用于Android Wear上的Smart Replies。 目前，此模型是特定于Android的。

	[可供下载的预置训练模型](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/lite/g3doc/models.md)

- 为自定义数据集重新训练Inception-V3或MobileNet

	这些预先训练的模型在ImageNet数据集上进行训练，该数据集包含1000个预定义的类。 如果这些类不足以满足您的使用需求，则需要重新训练模型。 这种技术称为转移学习，从已经训练过问题的模型开始，然后在类似的问题上重新训练模型。 从头开始深度学习可能需要数天时间，但转移学习相当快。 为此，您需要生成标有相关类的自定义数据集。

	[TensorFlow for Poets](https://codelabs.developers.google.com/codelabs/tensorflow-for-poets/)代码库逐步完成了重新培训过程。该代码支持浮点和量化推理。

- 训练自定义模型

	开发人员可以选择使用Tensorflow训练自定义模型（有关构建和训练模型的示例，请参阅TensorFlow教程）。 如果您已经编写了模型，则第一步是将其导出到tf.GraphDef文件。 这是必需的，因为某些格式不会将模型结构存储在代码之外，我们必须与框架的其他部分进行通信。 请参阅导出推理图以为自定义模型创建.pb文件。

	TensorFlow Lite目前支持TensorFlow运算符的子集。 有关支持的运算符及其用法，请参阅TensorFlow Lite和TensorFlow兼容性指南。 这套运营商将在未来的Tensorflow Lite版本中继续增长。

## 转换模型格式

 在上一步中生成（或下载）的模型是标准的Tensorflow模型，您现在应该有一个.pb或.pbtxt tf.GraphDef文件。 使用转移学习（重新训练）或自定义模型生成的模型必须转换. 但是，我们必须先冻结图形以将模型转换为Tensorflow Lite格式。 此过程使用多种模型格式：

- tf.GraphDef (.pb)  
	表示TensorFlow训练或计算图的protobuf。 它包含运算符，张量和变量定义。

- CheckPoint (.ckpt)  
	来自TensorFlow图的序列化变量。由于这不包含图形结构，因此无法自行解释。

- FrozenGraphDef  
	GraphDef的子类，不包含变量。可以通过CheckPoint和GraphDef将GraphDef转换为FrozenGraphDef，并使用从CheckPoint检索的值将每个变量转换为常量。

- SaveModel  
	带有签名的GraphDef和CheckPoint，用于标记模型的输入和输出参数。可以从SavedModel中提取GraphDef和CheckPoint。

- TensorFlow Lite model(.tflite)  
	一个序列化的FlatBuffer，包含用于TensorFlow Lite解释器的TensorFlow Lite运算符和张量，类似于FrozenGraphDef。

### Freeze Graph
 
 要将GraphDef .pb文件与TensorFlow Lite一起使用，您必须具有包含经过训练的权重参数的检查点。 .pb文件仅包含图形的结构。 将检查点值与图结构合并的过程称为冻结图。

 您应该有一个检查点文件夹或下载它们以获得预先训练的模型（例如，MobileNets）

 要冻结图形，请使用以下命令（更改参数）：
```
	freeze_graph --input_graph=/tmp/mobilenet_v1_224.pb \
	  --input_checkpoint=/tmp/checkpoints/mobilenet-10202.ckpt \
	  --input_binary=true \
	  --output_graph=/tmp/frozen_mobilenet_v1_224.pb \
	  --output_node_names=MobileNetV1/Predictions/Reshape_1
```
 必须启用input_binary标志，以便以二进制格式读取和写入protobuf。 设置input_graph和input_checkpoint文件。

 在构建模型的代码之外，output_node_names可能并不明显。找到它们的最简单方法是使用TensorBoard或graphviz可视化图形。

 现在，已冻结的GraphDef已准备好转换为FlatBuffer格式（.tflite），以便在Android或iOS设备上使用。 对于Android，Tensorflow Optimizing Converter工具支持浮点和量化模型。 
 要将冻结的GraphDef转换为.tflite格式：
```
	toco --input_file=$(pwd)/mobilenet_v1_1.0_224/frozen_graph.pb \
	  --input_format=TENSORFLOW_GRAPHDEF \
	  --output_format=TFLITE \
	  --output_file=/tmp/mobilenet_v1_1.0_224.tflite \
	  --inference_type=FLOAT \
	  --input_type=FLOAT \
	  --input_arrays=input \
	  --output_arrays=MobilenetV1/Predictions/Reshape_1 \
	  --input_shapes=1,224,224,3
```
 input_file参数应引用包含模型体系结构的冻结GraphDef文件。 这里使用的frozen_graph.pb文件可供下载。 
 output_file是生成TensorFlow Lite模型的位置。 
 除非转换量化模型，否则input_type和inference_type参数应设置为FLOAT。 
 设置input_array，output_array和input_shape参数并不是那么简单。 找到这些值的最简单方法是使用Tensorboard探索图形。 在freeze_graph步骤中重用用于指定推理的输出节点的参数。

 也可以将Tensorflow Optimizing Converter与来自Python或命令行的protobuf一起使用（请参阅toco_from_protos.py示例）。 这允许您将转换步骤集成到模型设计工作流程中，确保模型可以轻松转换为移动推理图。 例如：
```
	import tensorflow as tf

	img = tf.placeholder(name="img", dtype=tf.float32, shape=(1, 64, 64, 3))
	val = img + tf.constant([1., 2., 3.]) + tf.constant([1., 4., 4.])
	out = tf.identity(val, name="out")

	with tf.Session() as sess:
	  tflite_model = tf.contrib.lite.toco_convert(sess.graph_def, [img], [out])
	  open("converteds_model.tflite", "wb").write(tflite_model)
```
 有关用法，请参阅Tensorflow Optimizing Converter[命令行示例](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/lite/toco/g3doc/cmdline_examples.md)。

 有关故障排除帮助，请参阅[Ops兼容性指南](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/lite/g3doc/tf_ops_compatibility.md)

 [开发仓库]()包含一个在转换后可视化TensorFlow Lite模型的工具。要构建[visualize.py](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/lite/tools/visualize.py)工具：
```
	bazel run tensorflow/contrib/lite/tools:visualize -- model.tflite model_viz.html
```
 这将生成一个交互式HTML页面，其中列出了子图，操作和图形可视化。

## 使用TensorFlow Lite模型在移动应用程序中进行推理

 完成前面的步骤后，您现在应该有一个.tflite模型文件。

### Android

 由于Android应用程序是用Java编写的，而核心TensorFlow库是用C ++编写的，因此提供了一个JNI库作为接口。 这仅用于推理 - 它提供加载图形，设置输入和运行模型以计算输出的能力。

 开源Android演示应用程序使用JNI接口，可在[GitHub](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/lite/java/demo/app)上使用。您还可以下载[预建的APK](http://download.tensorflow.org/deps/tflite/TfLiteCameraDemo.apk)。有关详细信息，请参阅[Android演示指南](https://www.tensorflow.org/lite/demo_android)。

 Android移动指南提供了在Android上安装TensorFlow以及设置bazel和Android Studio的说明。
