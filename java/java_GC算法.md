>github上的地址：[DevelopBlog](https://github.com/Vinctor/DevelopBlog)

与C语言不同，Java内存（堆内存）的回收由JVM垃圾收集器自动完成，不需要程序开发者手动释放内存。

从Java内存模型（[链接](http://www.jianshu.com/p/860c259c8aad)）一文中，我们知道，java中几乎所有的对象实例存储在堆内存中，故而堆内存是JVM垃圾回收的主要阵地。

# 哪些对象需要被回收？

在讨论GC之前我们需要考虑一个问题？如何确定一个对象是否需要被回收？

两个方法：```引用计数法```和```可达性分析法```。

## 引用计数法

给对象添加一个引用计数器，每当有一个地方引用它时，计数器+1，当引用失效时，计数器-1；当计数器为0，则表明它没有被引用，也就是说可以被GC回收。

优点：此方法简单粗暴，效率很高。很多其他语言也是用这一方法进行对象的回收判断。

缺点：此方法无法解决对象之间的相互循环引用问题。例如：

    public class GCVinctorTest {
    
        private Object instance = null;
        
        public static void main(String[] args) {
            GCVinctorTest objA = new GCVinctorTest();
            GCVinctorTest objB = new GCVinctorTest();

            objA.instance = objB;
            objB.instance = objA;

            objA = null;
            objB = null;

            System.gc();
        }
    }


![image.png](http://upload-images.jianshu.io/upload_images/1583231-7231320273b6f0cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


* step1：实例A的引用计数加1，实例A的引用计数=1；
* step2：实例B的引用计数加1，实例B的引用计数=1；
* step3：实例A的引用计数加1，实例A的引用计数=2；
* step4：实例B的引用计数加1，实例B的引用计数=2；

执行

            objA = null;
            objB = null;

之后：
* 实例A的引用计数减1，实例A的引用计数=1；
* 实例B的引用计数减1，实例B的引用计数=1；

此时，objA与objB相互引用，他们的引用计数器都不是0，除此之外，再无任何其他实际引用，但是引用计数法无法通知GC收集器回收他们。

## 可达性分析算法

可达性分析算法 通过一系列的被称为```GC Roots```的对象爱你个作为起始点（相当于根），其他对象（相当于树枝或树叶）直接或间接都与这个```GC Roots```相连。如果一个对象与root不相连，则就说明这个对象是不可用的，GC就可以将它回收。如图所示


![image.png](http://upload-images.jianshu.io/upload_images/1583231-b256deacbf6b8a76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

图中obj1,obj2,obj3都与roots有直接或间接关联，不可被回收的存活对象，obj6，obj4，obj5虽然这三者之间有关联，但是与roots已经断开，故而可被回收对象。

在```HotSpot```中，JVM使用```OopMap```的数据结构来记录对象内什么偏移量存储的是什么类型的数据的映射关系，在JIT编译过程中，也会在特定位置记录下栈和寄存器中的那些位置和引用的，这样GC在扫描时就可以直接获得这些信息，而不需要去遍历GC roots的引用链，提高了回收效率。

在```HotSpot```中，使用可达性分析算法进行可回收对象的标记。

#### GC ROOTS分类:

*　虚拟机栈（栈帧中的本地变量表）中引用的对象
*　方法区中类静态属性引用的对象
*　方法区中常量引用的对象
*　本地方法栈中JNI（Natice方法）中引用的对象

（finalize()方法不讨论）



![image.png](http://upload-images.jianshu.io/upload_images/1583231-259584b7fad9f6c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


从从上图看出，存在3个GC Root：

静态变量```RefV1```指向堆中实例1；

局部变量```RefV2```指向堆中实例2；

Jni变量```RefV3```指向堆中实例3，实例3指向了实例4；

故而，实例1，2，3，4都是可以存活的对象；

而实例5不存在GC Root，故实例5可以被回收，虽然实例6被实例5引用，但是实例5没有Gc Root（整条链无Root），故实例6也可以被回收。


#### 引用的四种类型：
从强到弱分为：强引用，软引用（soft），弱引用（weak），虚引用

* 强引用：在开发中经常的写法，类似于```BeanDemo demo=new BeanDemo();```这种写法，极为强引用。只要这种引用还存在，该实例就不会被标记为可回收，垃圾回收器也就不会回收掉该对象。
* 软引用：用来引用一些有用但是非必需的对象。```在系统将要发生内存溢出OOM时```，有这种引用的对象，将要被回收。
* 弱引用：用来引用一些有用但是非必需的对象。但是与```软引用在OOM进行回收```不同，有这些引用的对象，只要发生垃圾回收，该对象将被回收。
* 虚引用：无法通过虚引用获得一个对象的实例，设置虚引用的目的就是能在这个对象被收集器回收时收到一个系统通知。

# 垃圾回收算法

### 标记—清除算法(mark-and-sweep)

顾名思义：先标记后清除（废话）。JVM首先标记出所有需要回收的对象，在标记完成之后，在下一次垃圾回收的时候统一进行回收这些对象。
如图所示：

清除前

![](http://upload-images.jianshu.io/upload_images/1583231-2ca5d672994ed679.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

清除后

![](http://upload-images.jianshu.io/upload_images/1583231-85a460bb1beafa2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

![](http://upload-images.jianshu.io/upload_images/1583231-befa7019e6f03700.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)


可以看到，垃圾回收之后，剩余的存货对象分布比较杂乱，产生大量的碎片。碎片如果太多，将会导致，内存空间的不连续性，如果这时出现一个大对象（如数组），将会无法为其分配空间。

### 复制算法(Copying)

为了解决```标记—清除算法```的弊端，出现了```复制算法```。该算法首先将内存空间分为两部分，一次只使用其中的一块。当GC发生之时，对象被清理之后，将剩余的存活对象复制到另一部分内存空间上，再把当前的内存空间清空，这样就解决了碎片过多的问题。如图所示：

![](http://upload-images.jianshu.io/upload_images/1583231-959442b70ec908aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

此方法虽然解决了碎片过多的问题，但是另一个显著问题又出现了：内存利用率太低！

在同一时间，只有一部分的内存在被使用，如果对象存活率比较高的时候，复制算法将会进行较高的复制操作，复制算法将会非常低。考虑[上篇文章](http://www.jianshu.com/p/860c259c8aad)的老年代与新生代的各自不同特点，可见此算法不适用于老年代。

### 标记—整理算法

为了解决复制算法的利用率低的问题，提出了```标记—整理算法```算法，与```标记—清除算法```一样，首先对对象进行标记，但后续步骤则不是对可回收对象进行清理，而是让存活的对象向内存空间的一端就行移动，然后清理掉端边界以外的内存。


![](http://upload-images.jianshu.io/upload_images/1583231-d0ffbbc9a1ddfe2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 分代收集

这是[上篇文章](http://www.jianshu.com/p/860c259c8aad)已经提及的，根据对象的存活周期将内存划分为新生代与老年代。然后根据各个年代不同的特点采用最适合的手机算法。

例如：
* 新生代的对象```朝生夕死```，时时产生新对象，时时又会有大批对象死去，这个区域就可以使用复制算法；
* 老年代因为对象存活率特别高，不容易被回收，需要使用```标记—清除```或```标记—整理```算法进行回收。


上文中提到，在GC过程中使用```OopMap```记录对象 的偏移和类型信息，随着程序的执行，无时不刻都有可能会产生新的对象，如果每进行一步操作，都生成相应的```OopMap```，那会需要大量的额外空间，GC成本变的非常高。在实际的```HotSpot```中，JVM只在程序执行到特定的位置才生成```OopMap```，这些位置称为“```安全点```”。

当程序执行到安全点之后，由于需要进行GC ROOTS统计，需要暂停进程中所有的线程，如果不暂停，就会在统计的过程中不断产生的新的对象，使得统计无法得到准确的结果。

安全点的选定既不能太少，会导致GC等待时间太长；又不能太多，导致太过于频繁而增大运行负荷。故而选定的标准为：是否有让程序城市间执行的特征。一般最明显的就是制定序列的服用（指令将在以后介绍），如：方法调用，循环跳转，异常跳转等。

线程如何达到安全点呢？JVM在安全点的地方设置一个中断标志，当线程执行到这个标志时，线程自己中断挂起。

这里还有一个问题，当一个线程没有处于运行状态的时候，如SLeep或者Blocked状态，那么如何才能中断自身呢？GC也不可能等待该线程重新抢占CPU再进行中断，这时```安全区域```的概念产生了。安全区域同安全点的概念一样，只不过安全区域是一段代码片段。

上文提过，为了保证GC ROOTS统计的完整性，统计时产生新的引用关系，才提出的安全点这一概念。同样，我们选定安全区域的标准也是```该段代码并不会产生新的引用关系```。

在线程执行到安全区域中的代码时，首先标记一下自己已经进入了安全地带了，这样JVM进行GC时，就不用考虑该线程的引用关系了。当线程```将要离开```该安全区域的时候，该线程首先检查自己是否已经完成了GC ROOTS的统计枚举，如果完成了，那就继续往下执行代码，并将之前进入安全区域的标识去掉；如果没有完成GC ROOTS的枚举，则该线程中断、等待，直到收到GC ROOTS枚举已经统计完成的信号，才可以继续执行下面的代码。

就酱，本文介绍了GC如何识别出那些需要回收和存活下来的对象，并从理论上介绍了GC的回收算法，下篇文章将详细介绍JVM中的一些具体的垃圾回收器。


























(部分图片源于网络，如果侵犯到您的权益，请联系本人删除。)

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
