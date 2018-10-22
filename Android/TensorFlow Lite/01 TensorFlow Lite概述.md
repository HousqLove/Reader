# TensorFlow Lite highlights

1. 一组核心运算，包括量化和浮动，其中许多已经针对移动平台进行了调整。这些可用于创建和运行自定义模型。开发人员还可以编写自己的自定义运算符并在模型中使用它们。
2. 一种新的基于FlatBuffers的模型文件格式
3. 具有内核优化的设备上解释器，可在移动设备上更快地执行。
4. TensorFlow转换器将TensorFlow训练的模型转换为TensorFlow Lite格式。
5. 更小的尺寸：当支持所有的运算被链接时，TensorFlow Lite小于300KB，当仅使用支持InceptionV3和Mobilenet所需的运算时，小于200KB。
6. 预训练模型  
	以下所有型号均可保证开箱即用：
	- Inception V3: 用于检测图像中存在的主要对象的流行模型。
	- MobileNets: 一系列移动优先计算机视觉模型，旨在有效地最大限度地提高准确性，同时注意到设备或嵌入式应用程序的受限资源。 它们是小型，低延迟，低功耗模型，参数化以满足各种用例的资源限制。 它们可以用于分类，检测，嵌入和分割。 	MobileNet模型比Inception V3更小但精度更低。
	- On Device Smart Reply: 一种设备上的模型，通过建议上下文相关的消息，为传入的文本消息提供一键式回复。 该模型专为内存受限设备（如手表和手机）而构建，并已成功用于将Android Wear上的智能回复呈现给所有第一方和第三方应用。

	[TensorFlow Lite's supported models](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/lite/g3doc/models.md)包括型号，性能编号和可下载的模型文件
7. MobileNet模型的量化版本，其运行速度比CPU上的非量化（浮点）版本快。
8. 新的Android演示应用程序，用于说明使用TensorFlow Lite和量化的MobileNet模型进行对象分类。
9. Java and C++ API support 

# Getting Started

 我们建议您使用上面指出的预测试模型试用TensorFlow Lite。 如果您有现有模型，则需要测试您的模型是否与转换器和支持的运算集兼容。 要测试您的模型，请[参阅GitHub上的文档](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/lite)。

## 为自定义数据集重新调整Inception-V3或MobileNet

 上面提到的预训练模型已经在ImageNet数据集上进行了训练，该数据集由1000个预定义类组成。 如果这些类对您的用例不相关或有用，则需要重新训练这些模型。 这种技术称为转移学习，它从已经训练过一个问题的模型开始，然后再对类似的问题进行再训练。 从头开始深度学习可能需要数天时间，但转移学习可以相当快地完成。 为此，您需要生成标有相关类的自定义数据集。

 [TensorFlow for Poets](https://codelabs.developers.google.com/codelabs/tensorflow-for-poets/)代码库逐步完成了这个过程。重新训练代码支持对浮点和量化推理进行重新训练。

## TensorFlow Lite 结构

 TensorFlow Lite架构设计.jpg

 从磁盘上经过训练的TensorFlow模型开始，您将使用TensorFlow Lite转换器将该模型转换为TensorFlow Lite文件格式（.tflite）。然后，您可以在移动应用程序中使用该转换后的文件。

 部署TensorFlow Lite模型文件使用:

1. Java API: 围绕Android上的C ++ API的便利包装器
2. C++ API: 加载TensorFlow Lite模型文件并调用解释器。 Android和iOS都提供相同的库
3. Interpreter: 使用一组内核执行模型。解释器支持选择性内核加载;没有内核它只有100KB，加载了所有内核300KB。这比TensorFlow Mobile要求的1.5M显着降低。
4. On select Android devices: Interpreter将使用Android神经​​网络API进行硬件加速，如果没有，则默认为CPU执行。

 您还可以使用可由Interpreter使用的C ++ API实现自定义内核。

# Future Work

 在未来的版本中，TensorFlow Lite将支持更多模型和内置运算符，包含定点和浮点模型的性能改进，对工具的改进，以便更轻松地开发工作流程以及支持其他更小的设备等。 在我们继续开发的过程中，我们希望TensorFlow Lite能够大大简化针对小型设备模型的开发人员体验。

 未来的计划包括使用专门的机器学习硬件来为特定设备上的特定模型获得最佳性能。