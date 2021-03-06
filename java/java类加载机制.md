基于JVM的语言，如java,kotlin,groovy等语言，在各自编译器编译完成之后，都会编译为```.class```文件，用JVM加载。而class文件只有被确的加载到JVM正中才能运行和使用。虚拟机是如何在家这些文件呢？本文将详细讲解。

# 类的生命周期

一个类从被加载到虚拟机到最后被卸载，生命周期包括：```加载```，```验证```，```准备```，```解析```，```初始化```，```使用```，卸载7个阶段。其中验证，准备，解析3个部分称为````连接```阶段。

![类的生命周期](http://upload-images.jianshu.io/upload_images/1583231-143bb3a388527936.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这7个阶段在实际JVM中并不是按照图中所示的顺序来```开始运行```的，里面存在时间上的交叉进行。但是其中加载，验证，准备，初始化，卸载5个结算的顺序是确定的。

### 加载

这是类生命周期的第一个阶段，那么加载的是什么呢？加载的应该是```一个字节码文件的二进制字节流```。那么此二进制流如何得来呢？java虚拟机规范并没有强制要求，我们可以灵活运用这一特性实现很多的加载源：

* 最常见的，从压缩包获取，比如jar，EAR，WAR等
* 从网络中获取，比如早期嵌入在浏览器中的Applet程序
* 在运行是生成字节码，动态代理技术。如著名的```GCLib```字节码类库，再如现在```Android```中常用的网络请求库```retrofit```中所使用的动态代理```Proxy.newProxyInstance()```中，最终会调用的```sun.misc.ProxyGenerator.generateProxyClass()```方法，该方法在运行时动态产生了一组字节码流（标识为```$Proxy```的代理类）。
* 由其他文件生成，比如由JSP文件生成class类
* 等等等等

在此阶段，开发人员可以使用系统的类加载器进行加载，也可以使用自己定义的类加载器来自定义获取字节码流的方式（重写来加载器的```loadClass```方法）。

加载字节码文件结束后，虚拟机将字节流存储在方法区中，同时在内存中（Hot Spot中实在方法区中）实例化一个```Class```对象，外部可以同过此实例访问该类对象。

在此阶段运行中，验证阶段就已开始，交叉进行。只有通过通过了验证阶段，只有通过了验证阶段，字节流才会进入内存的方法区中进行存储。

### 验证

验证阶段的主要任务是：确保字节码流中包含的信息符合当前版本虚拟机的要求，并不会有危害虚拟机自身安全的行为。

如：将一个对象强转为一个未声明实现的类型，执行一个虚方法，执行一个并不存在的方法。在我们平时编码的经验中，虽然以上这些错误会在编译时报出，无法通过编译；但是，我们上面提到过，class文件是由多种方式得来，对于直接生成```.class```文件、无需编译的方式，验证这一阶段对于虚拟机的保护就显得尤其重要。


简要的概述，虚拟机对类的验证阶段分为以下4个方面，这四个方面层层深入：

#### 文件格式的验证

>针对类文件（字节码流）的验证

验证字节码流是否符合java虚拟机规范中规定的class文件格式，如：
* 魔数是否为CAFEBABY
* 当前虚拟机持否可以处理文件声明的主，次版本号
* 常量池中是否有不被支持的常量类型
* 检查指向常量的索引是否指向了不存在的常量
* CONSTANT_Utf8_info型的常量是否符合Utf8编码
* Class文件中的各个部分是否被删除（class文件是否完整）
* 等等

>通过了验证阶段，字节流会进入内存的方法区中进行存储。以后的验证和其他操作都针对于内存方法区中的数据进行操作，而不针对字节码流。

#### 元数据验证

>针对数据类型的验证

该阶段是进行语义分析验证，以保证其信息符合Java语言规范的要求，比如：

* 检查这个类是否有父类（除了Object之外都应有父类）
* 本类的父类是否继承了不允许被继承的类（被final修饰）
* 如果本类不是抽象类，是否实现了父类中的全部虚方法或接口
* 等等

#### 字节码验证

>针对方法体的验证

此阶段通过数据流和控制流分析，检查程序的语义是合法的，符合逻辑的。保证程序逻辑的正确运行，检验的内容如：
* 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作。（如：不能出现这样的状况：操作栈中放了一个int类型的数据，使用却按照long或者引用类型加载）
* 保证跳转指令不会跳转到方法体以外的字节码指令上
* 类型转换是有效的（如多态）
* 等等
#### 符号引用的验证

>针对常量池匹配的验证

此阶段检查是为了：确保在后续的解析阶段，虚拟机可以顺利的将符号引用转化为直接引用。（关于符号引用与直接引用的概念，祥见下文解析过程）见下图，讲解一下验证内容：

![常量池](http://upload-images.jianshu.io/upload_images/1583231-731c7ee40cbd2392.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

* 符号引用中通过字符串的描述能够找对应的类（如：上图中常量池中有一个指向类型为class的常量```#4 = Class    #17    // java/lang/Object```,应该确保有一个类与之对应,此处为String类）
* 符号引用中通过字符串的描述的能够找到相应的方法（如：上图中```#2 = Methodref          #3.#15    // VinctorTest.test:()V```描述的，需要在VinctorTest类中有一个test方法与之对应）
* 符号引用中的用到的类，方法，字段的访问性（public  private等）确保可以被当前类访问到
* 等等

#### 准备

>针对类变量(static)

经过验证阶段，虚拟机从文件，数据类型，方法逻辑，符号引用等各个方面对类进行了验证，已确保代码的正确性。接下来开始为代码的运行做准备，进入准备阶段。

准备阶段是为正式```类变量```（注意，不是实例变量）分配内存并设置类变量初始值的阶段，注意，此初始值并不是我们java代码中所写的初始值（如 int a=123；），而是java虚拟机规范中规定的初始值，
java体系中各种类型的初始值如下：

![各类型的初始值](http://upload-images.jianshu.io/upload_images/1583231-803d7042c8dc56f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

如果一个变量声明为```  static int a=123```，则在此阶段，声明a的值为0；

注意：如果类变量被```final```修饰，如

>final  static int a=123;

这种情况下，javac编译阶段，将为此变量生成```ConstantValue```属性，在此准备阶段直接将其赋值为123；

#### 解析

>针对常量池

解析阶段是将常量池中符号引用转化成直接引用的过程。主要针对常量池中的类或接口，字段，类方法，接口方法，方法类型，方法句柄，调用限定符

* ```符号引用```：见上文中class文件中常量池的图片，我们可以知道常量池中有描述类，方法，字段等常量，这些常量通过一组符号（比如UTF8字符串）描述所引用的目标。虽然在验证阶段已经对此进行了验证，但是这些毕竟只是一些字符串，并不能拿来直接为虚拟机使用，并不指向任何真实的内存地址。
* ```直接引用```：直接引用则是指向这些目标的指针，偏移量或者句柄。

直接引用指向的目标必须真实存在于内存之中的。在代码运行过程中，会不断产生新对象，故而解析这一过程并不是一次就完成的，其发生的时机不固定。

java虚拟机规范中规定了只有执行了以下字节码指令前才会将所用到的符号引用转化为直接引用：
* ```anewarray```  创建一个```引用类型```的数组
* ```checkcast``` 检查对象是否是给定类型
* ```getfield```  ```putfield```   从对象获取某一个字段  设置对象的字段
* ```getstatic```  ```putstatic``` 从类中获取某一静态变量 设置静态变量
* ```instanceof``` 确定对象是否是给定类型
* ``` invokedynamic``` ```invokeinterface``` ```invokestatic``` ```invokevirtual``` 调用动态方法，接口方法，静态方法，虚方法
* ```invokespecial``` 调用实例化方法，私有方法，父类中的方法
* ```ldc```  ```idc_w```  把常量池中的项压入栈
* ```multianewarray```  创建多为```引用类型```性数组
* ```new``` 实例化对象

在解析过程中，如果需要解析类或接口的的字段，方法，则先查找该字段，方法所属的类或接口是否被解析，如果没有，则先解析类或接口，然后在查找当前的类或接口中是否有该字段或方法，如果没有，则递归向上到父类或父接口中寻找该字段或接口。

#### 初始化

至此，程序终于开始执行我们开发人员写的代码了（等了好久）。此阶段是为类设置类变量的值和一些其他初始化操作的阶段（如执行```static{ }```静态代码块）。

在类编译过充中，编译器为每一个方法生成了一个```<clinit>()```类初始化方法，初始化阶段也是此方法的执行阶段。

>注意```<clinit>()```并不是默认构造方法，前者是类的初始化方法，后者是实例的初始化方法。我们此文讨论的是类的生命周期，而不是实例的生命周期。

```<clinit>()```是如何生成的呢？其中又包含什么呢？

```<clinit>()```方法是在编译阶段，编译器收集整个类中的```类变量```的赋值以及```静态代码块```而形成的。顺序是按照赋值以及静态代码在源文件中出现的顺序生成的。同时，如果一个类有父类，则虚拟机会保证父类的初始化先于子类的初始化执行。

## 使用

至此 一个类已经具备我们使用的条件了，我们可以对这个类进行实例化和其他操作了。

>github上的地址：[DevelopBlog](https://github.com/Vinctor/DevelopBlog)
