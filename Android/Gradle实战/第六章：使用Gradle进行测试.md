# 第六章：使用Gradle进行测试
 Gradle集成了很多Java和Groovy测试框架，包括JUnit、TestNG、Spock等。
## 自动化测试
 自动化测试是你工具箱里面非常关键的一个部分。
### 自动化测试类型
 自动化测试通常在作用域、实现方式和执行时间上有所差异，可简单分为：单元测试、集成测试和功能测试。

- 单元测试用于测试你代码的最小单元，基于java的项目中这个单元就是一个方法，在单元测试中你会避免与其他类或者外部的系统打交道。单元测试很容易编写，执行起来非常快速，能够在开发阶段给你代码的正确性提供反馈。
- 集成测试用于测试有一个组件或者子系统。你想确保不同类之间的交互能够按照预期一样，一个典型的情况就是逻辑层需要和数据库打交道。因此相关的子系统、资源文件和服务层必须在测试执行阶段是可访问的。集成测试通常比单元测试运行更慢，更难维护，出现错误也比较难诊断。
- 功能测试用于测试一个应用的功能，包括外部系统的交互。功能测试是最难实现也是运行最慢的，因为他们需要模仿用户交互的过程，在web开放的情况，功能测试应该包括用户点击链接，输入数据或者在浏览窗口提交表单这些情形，因为用户接口可能随着时间改变，功能测试的维护将会很困难。
### 自动化测试的金字塔
 你需要写多少测试取决于编写和维护测试的时间消耗。测试越简单就越容易执行，一般来讲你的项目应该包含很多单元测试，少量的集成测试和更少的功能测试。
## 测试Java应用
 一些开源的测试框架比如JUnit，TestNG能够帮助你编写可复用的结构化的测试，为了运行这些测试，你要先编译他们，就像编译源代码一样。  
 测试代码的作用仅仅用于测试的情况，通过会把源代码和测试代码分开来。Gradle的标准项目布局中测试代码的路径是```src/test/java```。
### 项目布局
 默认的项目布局，源代码是```src/main/java```，资源文件是在```src/main/resources```，测试源代码```src/test/java```，资源文件```src/test/resources```，编译之后测试的class文件在```build/classes/test```下。  
  
 所有的测试框架都会生产至少一个文件用来说明测试执行的结果，最普遍的格式是xml格式，可以在```build/test-results```路径下找到这些文件。许多测试框架都允许把测试结果转换成报告，比如JUnit可以生成html格式的报告，Gradle把测试报告放在```build/report/test```下。如下图：  
 ![Image](https://github.com/HousqLove/Reader/blob/master/Android/Gradle%E5%AE%9E%E6%88%98/images/gradle-6-1.png)
### 测试配置
 Java插件引入可两个配置来声明测试代码的编译期和运行期依赖：```testCompile```和```testRuntime```，如下：  
```
	dependencies {
		testCompile 'junit:junt:4.11'
	}
```  
 testRuntime用来声明那些编译期用不到但运行期需要的依赖。对于测试依赖来讲测试配置继承了源代码相关配置，比如testCompile继承了compile配置的依赖，testRuntime继承了runtime和testCompile和他们的父类，他们父类的依赖会自动传递到testCompile或testRuntime中，如下图：  
 ![Image](https://github.com/HousqLove/Reader/blob/master/Android/Gradle%E5%AE%9E%E6%88%98/images/gradle-6-2.png)
### 测试任务
 测试编译和测试执行阶段是在源代码被编译和打包之后的，如果你想避免执行测试阶段你可以在命令行执行gradle jar或者让你的任务依赖jar任务。
### 自动测试检查
 对于```build/classes/test```目录下的所有编译的测试类，Gradle会检查下面几条规则来确认是否要执行。

- 任何继承子junit.framework.TestCase或groovy.util.GroovyTestCase的类
- 任何被@RunWith注解的子类
- 任何至少包含一个被@Test注解的类  
 如果没有找到符合的，测试就不会执行。
## 单元测试
 为之前的ToDo应用的存储类InMemoryTODORepository.java编写单元测试。  
 在```src/test/java```目录下创建一个名叫InMemoryToDoRepositoryTest.java的类，在代码中添加适当的断言。
```
	public class InMemoryToDoRepositoryTest {
		private ToDoRepository inMemoryToDoRepository;

		//用这个注解标识的都会在类的所有测试方法之前执行
		@Before
		public void setUp() {
			inMemoryToDoRepository = new InMemoryToDoRepository();
		}

		//用这个注解的都会作为测试用例
		@Test
		public void insertToDoItem(){
			ToDoItem new ToDoItem = new ToDoItem();
			newTODoItem.setName("Write unit tests");
			Long newId = inMemoryToDoRepository.insert(newToDoItem);
			//错误的断言会导致测试失败
			assertNull(newId);
			ToDoItem persistedToDoItem = inMemoryToDoRepository.findById(newId);
			assertNotNull(persistedToDoItem);
			assertEquals(newToDoItem, persistedToDoItem);
		}
	}
```
 接下来在依赖配置中添加JUnit的依赖：
```
	dependencies {
		testCompile 'junit:junit:4.11'
	}
```
 在任务中使用-i选项来打印日志输出：```gradle :repository:test -i```   
 在build/report/test查看html的测试报告。
### 使用testNG
 使用testNG的注解标识相应方法，同时还需要做两件事：
1. 声明对testNG库的依赖
2. 调用test.useTestNG()方法来声明测试过程中使用testNG框架
 如下：  
```
	dependencies {
		testCompile 'org.testng:testng:6.8'
	}
	//设置使用testNG来测试
	test.useTestNG()
```