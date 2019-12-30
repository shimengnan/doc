# JVM 基础

## 1.JAVA 是如何运行的

从虚拟机视角来看,执行java 代码,首先需要将它编译成的class文件加载到java 虚拟机中.加载后的java 类会存在方法区(Method Area) 中.实际运行时,虚拟机会执行方法区内的代码.

JVM 内存划分为:

- 线程共享:方法区
- 线程共享:堆
- 线程私有:程序计数器
- 线程私有:java 方法栈
- 线程私有:native 本地方法栈

在运行过程中,每当调用进入一个 JAVA 方法,Java 虚拟机会在当前线程的java方法栈中生成一个栈帧,用以存放局部变量以及字节码操作数.(JVM 不要求栈帧在内存空间连续分布).当退出当前执行的方法时,不管是正常返回还是异常返回,JVM 均会弹出当前线程栈帧.



硬件无法直接执行JAVA 字节码,JVM 需要将字节码翻译成机器代码.这个翻译过程有如下两种:

- 解释执行---无需等待编译
- 即时执行(Just-In-Time compilation ,JIT)---实际运行速度快

HotSpot默认采用混合模式,结合解释执行与即时执行的优点,先解释执行,而后将其中反复执行的热点代码,以方法为单位进行即时编译.

```
即时编译建立在程序符合二八定律的假设上,也就是百分制二十的代码占据了百分制八十的计算资源.对于占据大部分的不常用代码,无需耗时编译成机器代码,而是采取解释执行的方式运行,另一方面,对于仅占据一小部分热点代码,将其编译成机器代码,达到理想运行速度.
```

HotSpot 内置了多个即时编译器:C1,C2 和 Graal.Graal 是JAVA 10 引入的.

- C1 client 编译器		面向启动性能有要求的客户端GUI 程序
- C2 Server 编译器      面向对峰值性能有要求的服务器端程序,优化手段负责,编译时间长,执行效率高

从Java7 开始HotSpot 默认采用分层编译方式:热点方法首先会被C1 编译,而后热点方法会进一步被C2 编译

为了不干扰应用的正常运行,HotSpot 的即时编译是放在额外的编译线程中进行的.HotSpot会根据CPU 数量设置编译线程数目,并按照1:2比例分配给C1,C2.

## 2.JAVA 基本类型



JAVA 基本数据类型有八种:boolean,byte,short,int,long,double,float,char

### boolean 类型的特殊

在JAVA中,boolean 的值只有两种可能 true/false.JVM显然不能直接使用 true/false 标识boolean 类型

在JVM 中,boolean 的类型被映射成了int 类型, true 用 1表示,false 用0表示.



### Java 基本类型说明

| 类型    | 值域                | 默认值   | 虚拟机内部符号 |
| ------- | ------------------- | -------- | -------------- |
| boolean | {false,true}        | false    | Z              |
| byte    | [-128,127]          | 0        | B              |
| short   | [-32768,32767]      | 0        | S              |
| char    | [0,65535]           | '\u0000' | C              |
| int     | [-2^31,2^31 -1]     | 0        | I              |
| long    | [-2^63,2^63 -1]     | 0L       | L              |
| float   | ~[-3.4E38,3.4E38]   | +0.0F    | F              |
| double  | ~[-1.8E308,1.8E308] | +0.0D    | D              |

### Java 基本类型的大小

JVM 每调用一个JAVA 方法,变回创建一个栈帧,此处仅讨论解释栈帧.

这种栈帧有两个主要组成部分,**局部变量区,字节码的操作数栈.**这里的局部变量是广义的,除了普通的局部变量,还包含实例方法的**"this指针"**以及方法所接收参数.

在JVM 规范中,局部变量区等价于一个数组,并且可以用正整数来索引.除了long double 值用两个数组单元存储,其他基本类型以及引用类型的值均占用一个数组单元.

说明,boolean ,byte short char 这四种类型,在栈上用的空间和int 是一样的,和引用类型也是一样的.因此,在32位HotSpot 这些类型占用**4个字节**,在64位HotSpot 占用**8个字节**.

注意,这种情况仅存在与局部变量中,而并不会出现在存储于对堆中的字段或者数组元素上.对于byte,char,short 这三种类型的字段或者数组单元,它们在堆上占用的空间分别为一字节,两字节,两字节,与类型匹配.



### Java 基本类型的加载

JVM 的算术运算几乎全部依赖于操作数栈.也就是说,我们需要将堆中的 boolean,byte,char 以及 short 加载到操作数栈上,而后将栈上的值当做int 进行运算.

对于 boolean 与 char 这两个**无符号类型**来说,加载伴随着零扩展(值在低位,高位填充零)

对于byte,short 来说,加载伴随着符号扩展,(保持符号位不变,低字节为值,高字节填充零)

## Java 运行时数据区域

![img](C:/Users/Administrator.20180329-134844/AppData/Local/YNote/data/shimengnan007@163.com/647f0c4e524546c08afb9427305246af/clipboard.png)

### 程序计数器

线程私有内存，用于存储 线程间由于cpu 时间片执行，切换后的恢复作用。此区域不会出现OutOfMemoryError 情况的区域。

### Java 虚拟机栈

线程私有，生命周期与线程相同。描述Java 方法执行的内存模型。 用来执行方法

java 方法执行时会创建一个 栈帧，存储 局部变量表，操作数栈，动态链接，方法出口 等信息

线程请求的栈深度大于虚拟机规定的深度，抛出 StackOverflowError 异常

在扩展栈的时候，如果无法申请到足够内存，抛出 OutOfMemoryError 异常	

### Java堆

所有的对象及数组 均在堆上分配，对是虚拟机管理内存中最大的一块。（新生代 Eden  ，From Survivor，To Survivor）  用来存储对象实例

会抛出OutOfMemoryError：  java heap space

### native本地方法栈

与 虚拟机栈类似，区别在于 java 虚拟机栈执行Java 方法，而本地方法栈则为执行虚拟机使用到的Native 方法。

同Java 虚拟机栈一样会抛出 StackOverflowError 或OutOfMemoryError

### 方法区

用来存储 虚拟机加载的类信息、常量、静态变量（虚拟机规范定义为堆的一部分，别名叫做非堆，用来区分java 堆）

会抛出OutOfMemoryError : PermGen space

### 运行时常量池

运行时常量池，是方法去中的一部分，在类加载后会把class 中的常量池 放到 运行时常量池中。

（Class 文件中信息包括，版本，字段，方法，接口，常量池，常量池用来存放编译器的各种字面量与符号引用）

### 直接内存

直接内存不是JVM 运行时数据区的一部分,也不是JVM 规范中定义的内存区域.可能导致OutOfMemoryError 出现.

如Nio 中的堆外内存

## 3.JVM 如何加载JAVA 类

在JAVA 中 类型可以分为

**基本类型**:略

**引用类型**:类,接口,数组类(数组类是由JVM 直接生成,其余两者有对应字节流)

### JVM 加载类的过程

#### 加载

加载是指,查找字节流,并创建类的过程.

**类加载器:**

>BootStrap ClassLoader 启动类加载器,由C++实现,没有对应JAVA 对象(JRE lib 目录下jar,或-Xbootclasspath 指定的类)
>
>Extension ClassLoader 扩展类加载器 (JRE lib/ext 目录下的jar,或者由 java.ext.dirs 指定的类)
>
>ApplicationClassLoader 应用类加载器 (- classpath,系统变量java.class.path 或环境变量 CLASSPATH 所指定的路径)

​	

除了BootStrap ClassLoader 加载器,其余类加载器都是 java.lang.ClassLoader 的子类,因此有对应的JAVA 	对象,这些类加载器需要先由另一个类加载器加载至JVM 中,才能执行类加载.

在JVM 中,类的唯一性是由类加载器实例,以及类的全名一同确定的,即便字节码相同,经不同类加载器加载,也会得到两个不同的类.(在大型应用中,借助这一特性,运行同一个类的不同版本)

注: 自定义类加载器,可以对class 文件进行加密,加载时,在利用自定义的类加载器对其解密.

**双亲委派机制:**

每当一个类加载器接收到加载请求时,会先将请求转发给父类加载器,在父类加载器没有找到所请求类的情况	下,该加载器才会尝试去加载.

#### 链接

链接,指将创建成的类合并至JAVA虚拟机中,使之能够执行的过程,分为 验证,准备,解析三个阶段.

##### 验证

确保被加载的类满足JVM 约束条件

##### 准备

为被加载类的静态字段分配内存(对静态字段的具体初始化,会在后面的初始化阶段进行),JVM 也会在此阶段构造其他数据结构,如实现虚方法的动态绑定的方发表.

##### 解析

```
在class 文件被加载至JVM之前,这个类无法知道其他类及其方法,字段所对应的具体地址,甚至不知道自己方法,字段,地址.因此,在需要引用这些成员是,JAVA 编译器会生成一个符号引用.在运行阶段,这个符号引用能够无歧义的定位到具体目标上.
```

解析阶段的目的,正是将符号引用解析成实际引用.如果符号引用指向一个未被加载的类,或未被加载的类的字段或方法,那么将触发这个类的加载(但未必触发这个类的链接以及初始化)

JVM 规范并没有要求在链接过程中完成解析,仅规定,如果某些字节码使用了符号引用,那么在执行这些字节码之前,需要完成对这些符号引用的解析.

#### 初始化

在JAVA 代码中,若要初始化一个静态字段,可以再声明时直接赋值,也可以在静态代码块中对其赋值.

如果直接赋值的静态字段被final 所修饰,并且它的类型是基本类型或字符串,该字段便会被Java 编译器标记成常量值(ConstantValue),其初始化直接由JVM 完成.除此之外的直接赋值操作,以及所有静态代码块中的代码,则会被JAVA 编译器置于同一方法中,命名为clInit

总结来说,初始化就是为标记为常量值的字段赋值,以及执行clInit方法的过程,JVM 会通过加锁来确保类的clInit方法仅被执行一次.

JVM 规范枚举了如下种情况会触发初始化:

- 虚拟机启动,初始化用户指定的主类
- 新建目标类实例的new 指令时,初始化new 指定的目标类
- 调用静态方法的指令时,初始化该静态方法所在的类
- 调用静态字段的指令时,初始化该静态字段所在的类
- 子类的初始化会触发父类的初始化
- 一个接口定义了default 方法,直接或间接实现该接口的类的初始化,会触发该接口的初始化
- 使用反射API对某个类进行反射调用时,初始化这个类
- 当除此调用MethodHandle,初始化该MethodHandle 指向方法所在类(invoke dynamic)

如下是借助类初始化线程安全,且仅被执行一次的特性写的单例模式

```java
public class Singleton {
    private static class LazyHolder{
         static final Singleton INSTANCE = new Singleton();
    }
    private Singleton(){
    }
    public static Singleton getInstance(){
        return LazyHolder.INSTANCE;
    }
}
```

## 4.JVM 如何执行方法调用

### 重载

同一类中,方法名相同,参数类型不同(继承关系也算)

重载的方法在编译过程中即可确认调用.选取过程如下三个阶段:

1. 不考虑基本类型自动拆装箱,可变长参数选取重载方法
2. 第一个阶段没有适配方法,允许自动拆装箱,不允许可变长参数选取重载方法
3. 第二个阶段没有适配方法,允许自动拆装箱以及可变长参数选取重载方法

若JAVA编译器在同一阶段找到了多个适配方法,那么它会在其中选择最为贴切的.贴切的程度就是形式参数类型的继承关系

### 重写

子类中定义与父类中同方法名,同参数的方法.

若子类重写了父类的静态方法,那么子类中的方法会隐藏父类的静态方法.

若子类重写了父类的非静态,非私有方法,那么子类重写了父类的该方法

注:重写是JAVA 多肽的一种重要体现方式

### JVM 的静态绑定和动态绑定

JVM 识别方法在于:类名,方法名,方法描述符.方法描述符由方法的参数类型以及返回类型所构成.



在同一个类中,如果同时出现多个同名且同描述符的方法,JVM 在类的验证阶段报错



JVM 与 JAVA  不一样,JVM 不限制 同名,同参数,但返回值不同的方法在同一类中.对于调用这些方法的字节码

字节码所附带的方法描述符包含了返回类型,所以JVM 可以准确识别目标方法.



JVM中关于重写的判定同样基于方法描述符,若子类定义了父类中非私有非静态的同名方法,只有当这两个方法的参数类型以及返回类型一致,JVM 才会认为是重写.

对于JAVA 中认为是重写,而JVM 中认定是非重写的情况,编译器会通过桥接方法来实现.

重载方法的区分在编译阶段已经完成.故可以认为在JVM 中不存在重载这一概念

在JVM 中有个不完全正确的说法:

​	重载:静态绑定,编译时多肽

​	重写:动态绑定,运行时多肽

这是因为,父类方法可能被子类方法重写,因此java 编译器将对所有非私有实例方法调用为动态绑定的类型.



### JAVA 字节码中调用相关的指令

| 名称            | 备注                                                         |
| --------------- | ------------------------------------------------------------ |
| invokestatic    | 调用静态方法                                                 |
| invokespecial   | 调用私有实例方法,new 关键字,super 关键字,所实现接口的默认方法 |
| invokeinterface | 调用接口方法                                                 |
| invokevirtual   | 调用非私有实例方法                                           |
| invokedynamic   | 调用动态方法(MethodHandle)                                   |



对于 invokestatic 与 invokespecial 来说JVM 能够直接识别目标方法

对于invokevirtual与 invokeinterface 来说JVM 需要在执行过程中根据调用者动态类型来执行具体目标方法.



### 调用指定的符号引用

在编译过程中,不能确定目标方法的具体地址.因此JAVA 编译器会暂时使用符号引用来标识该目标方法.

符号引用:包括目标方法所在类或接口名字,目标方法的方法名和方法描述符

符号引用存储在class 文件的常量池中,根据目标方法是否为接口方法,引用可区分为 :**接口符号引用与非接口符号引用**

非接口符号引用:

​	假定该符号引用所指向的类C,则JVM 会按照如下步骤进行查找.

1. 在C 中查找符合名字以及描述符的方法

2. 如果没找到,在C 的父类中继续查找,直至Object类

3. 如果没找到,在C的直接实现或间接实现的接口中搜索,这一步搜索得到的目标方法必须是非私有,非静态的.并且,如果目标方法在间接实现的接口中,则需要满足C 与 该接口之间没有其他符合条件的目标方法.如果有多个符合的目标方法,任意返回其中一个.

   注:静态方法可以通过子类调用,子类的静态方法会覆盖父类的静态方法

接口符号引用:

​	假定该符号引用所指向的接口未I,则JVM 会按照如下步骤查找.

1. 在I中查找符合名字以及描述符的方法
2. 如果没找到,在Object 中的公有实例方法中搜索
3. 如果没找到,在I的超接口中搜索,过程与非接口符号引用的3一致

对于静态绑定的方法调用,实际引用是一个指向方法的指针.

对于动态绑定的方法调用,实际引用则是一个方发表的索引.



### 虚方法调用

上文中提到过,JAVA中所有非私有实例方法调用都会被编译成invokevirtual指令,而接口方法调用会被编译成invokeinterface指令.

invokevirtual 与 invokeinterface 这两种指令均属于JVM 中虚方法的调用.

在绝大多数情况下,JVM 需要根据调用者的动态类型,来确定虚方法调用的目标方法.这个过程称之为动态绑定.

相对于静态绑定的非虚方法调用来说,虚方法调用更加耗时.

在JVM 中,静态绑定包括用于调用静态方法的invokestatic 指令,和用于调用构造器,私有实例方法以及超类非私有实例方法 invokespecial,若JVM 调用指向一个标记为final 的方法,那么JVM 也可以静态绑定该虚方法调用的目标方法.

JVM 会采用空间换时间的策略来实现动态绑定,它为每个类生成一张方发表,用以快速定位目标方法.



### 方发表

在类加载的准备阶段,除了为静态字段分配内存,还会构造与该类关联的方发表

这个数据结构,是JVM 实现动态绑定的关键所在.下文中将用invokevirtual 锁使用的虚方法表(virtual method table vtable)为例介绍方发表用法,invokeinterface 所适用的接口方法表稍微复杂,原理类似.

方发表本质上是一个数组,每个数组元素指向一个当前类及其父类中非私有的实例方法.

这些方法可能是具体的可执行方法,也可能是没有相应字节码的抽象方法.方法表满足两个特质:

1. 子类方法表中包含父类方法表所有方法
2. 子类方法在方法表中的索引值,与它所重写的父类方法的索引值相同

上文中曾经说过,方法调用指令中的符号引用会在执行前解析成实际引用.对于静态绑定的方法调用,实际引用指向目标具体方法.对于动态绑定的方法调用,实际引用指向方法表的索引值(实际上 并不仅是索引值)

**动态绑定:**在实际实行过程中,JVM 将会获取调用者的实际类型,并在该实际类型的虚方法表中,根据索引值获得目标方法.

注: 实际上,使用了方法表的动态绑定与静态绑定相比,仅仅多出几个内存解引用操作:访问栈上的调用者,读取调用者的动态类型,读取该类型的方法表,读取方法表中某个索引值所对应的目标方法.但是,不能认同虚方法调用对性能没有太大影响,上述优化过程中仅存在与解释执行过程中,或即时编译最坏情况.即时编译还有另外两种性能更好的优化手段:内联缓存(inlining cache),方法内联(method inlining)

### 内联缓存

内联缓存是一种加快动态绑定的优化技术,它能够缓存虚方法调用中调用者的动态类型,以及该类型所对应的目标方法.在之后的执行过程中,若碰到已缓存的类型,内联缓存变会直接调用该类型所对应的目标方法.若没有碰到已缓存的类型,内联缓存会退化至**基于方法表的动态绑定**

针对多态的优化手段中,通常涉及如下三个术语:

1. 单态	仅有一种状态
2. 多态    有限数量种状态的情况
3. 超多态 更多的状态情况,通常会有一个具体数值来区分多肽和超多态,数值下成为多态,反之

对于内联缓存来说分别对应了,单态,多态,超多态内联缓存.在实践中,大部分的虚方法调用时单态的,也就是只有一种动态类型,为了节省内存空间,JVM 只采用单态内联缓存.

当内联缓存没有命中的情况下,JVM 需要重新使用方法表进行动态绑定.对于内联缓存汇总的内容,有两种选择.

1. 替换单态内联缓存中的记录,但是在最坏情况下,我们使用两种不同的调用者,轮流执行该方法调用,那么每次进行方法调用都将替换内联缓存.只有写缓存的开销,但是缓存却没有利用上.
2. JVM 劣化为超多态,处于这种状态下的内联缓存,放弃了优化机会,将直接访问方法表,来动态绑定目标方法



注意:虽然内联缓存 有内联二字,但是它并没有内联目标方法.这里需要明确的是,任何方法调用除非被内联,否则都会有固定开销.这些开销来源于保存程序在该方法中执行位置,以及新建,压入和弹出新方法所使用的栈帧. 对于及其简单的方法而言,如getter/setter 这部分固定开销占据的CPU 时间甚至超过了方法本身.在即时编译中,方法内联不仅仅能够消除方法调用的固定开销,而且还增加了进一步优化的可能性.

## 5.JVM是如何处理异常的

### 异常处理的两种方式:

1. 抛出异常	抛出异常使用**throw** 关键字
2. 捕获异常    捕获异常使用 **try**{}**catch**(){}**finally**{} 三种代码块

### 异常的基本概念:

在JAVA 语言规范中,所有异常都是Throwable类或者子类的实例,Throwable 有两大子类.

1. Error     一般抛出Error 时,执行状态已经无法恢复,需要终止线程甚至是终止虚拟机.
2. Exception  程序需要捕获并且处理的异常



### Exception 划分:

1. 编译期异常

   需要使用try,catch显示的捕获或者throws 关键字在方法声明中标注

2. 运行期异常

   RuntimeException

异常实例的构造十分昂贵,因为在构造异常实例时,JVM需要生成该异常的**栈轨迹** .该操作会注意访问当前线程的栈帧,并且记录下各种调试信息,包括栈帧所指向方法的名字,方法所在类,文件名,以及在触发异常的代码行数.(在生成栈轨迹时,JVM会忽略掉异常构造器以及填充栈帧的JAVA方法-Throwable.fillInStackTrace)

### JVM 是如何捕获异常的

在编译生成的字节码中,每个方法都会附带一个异常表,如下

```java
    Exception table:
       from    to  target type
           6     8    13   Class java/lang/Exception
           6     8    22   any
          13    17    22   any
          22    24    22   any
```

异常表中的每一个条目代表一个异常处理器,并且由from指针,to指针,target指针 以及所捕获的异常类型构成

这些类型的值是字节码索引,用以定位字节码

当程序触发异常时,JVM 会从上至下遍历异常表中的条目,当触发异常的字节码的索引值在某一行的from,to 范围内时,JVM 会判断抛出的异常与 该条目捕获的异常是否匹配,若匹配,JVM将控制转移至该条目target指针指向的字节码.

若遍历完所有异常条目,JVM 仍未匹配到异常处理器,那么会弹出当前方法对应的JAVA 栈帧,并且在调用者(caller)中重复上述操作.最坏情况下,JVM 需要遍历当前线程JAVA 栈上的所有方法异常表.

finally 代码块的编译比较复杂,当前版本JAVA 编译器的做法是复制finally 代码块的内容,分别放在 try-catch 代码块所有正常执行路径以及异常执行路径的出后中.

注:若catch 代码块中捕获了异常,并且触发了另一个异常,那么finally 捕获并且重新抛出的是新的异常,原有的异常会被忽略掉.

### JAVA7 语法糖

try-with-resources 语法糖,用来隐式的释放资源:

​	在try 关键字后声明并实例化实现了AutoCloseable 接口的类,编译器将自动添加对应的close 操作.

## 6.JVM 是如何处理反射的

反射是JAVA 中的重要特性,它允许正在运行的JAVA 程序观测,甚至是修改程序行为.

举例来说,我们可以通过Class 对象枚举该类中的所有方法,还可以通过Method.setAccessible 绕过Java 语言的访问权限,访问私有方法.

在Web 开发中,经常接触到各种配置通用框架,为了保证框架的可扩展性,一般借助JAVA 反射机制,来根据不同配置加载不同的类.Spring 的依赖反转(IOC)便是依赖反射机制

### 反射调用的实现

```java
public final class Method extends Executable {
	...
    @CallerSensitive
    public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException{
        ...
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
            ma = acquireMethodAccessor();
        }
        return ma.invoke(obj, args);
    } 
    ...
}
```

从上述源码可得,Method 的invoke 方法实际上委派给MethodAccessor 来处理.MethodAccessor 是一个接口,有两个具体实现

1. 通过本地方法来实现反射调用.		<本地实现>		调用native 本地方法
2. 使用了委派模式                                <委派实现>        

Method 实例的第一次反射调用都会生成一个委派实现,委派具体实现是一个本地实现.本地实现可以理解成--当进入了JVM内部后,便拥有了Method 实例所指向方法的具体地址.这时候,反射调用就是将传入的参数准备好,然后调用进入目标方法.

例V0版本ReflectTest:

```java
public class ReflectTest {
    public static void target(int i){
        new Exception("#"+i).printStackTrace();
    }
    public static void main(String[] args) throws ClassNotFoundException, 	
    		NoSuchMethodException, 
    		InvocationTargetException, 
    		IllegalAccessException {
        Class klass = Class.forName("jvmtest.ReflectTest");
        Method method = klass.getMethod("target",int.class);
        method.invoke(null,0);

    }
}
```



执行上述代码后,会抛出调用的堆栈信息如下:

```java
java.lang.Exception: #0
	at jvmtest.ReflectTest.target(ReflectTest.java:14)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:483)
	at jvmtest.ReflectTest.main(ReflectTest.java:19)
```

从上述堆栈信息中可以看出,反射调用过程如下:

1. 执行Method.invoke
2. 执行DelegationMethodAccessorImpl.invoke-- 委派实现
3. 执行NativeMethodAccessorImpl.invoke--本地实现

为什么反射调用还要采取委派实现作为中间层?

Java 的反射调用机制还设立了另一种动态字节码的实现,直接使用invoke指令来调用目标方法,之所以采取委派实现,便是为了能够在本地实现以及动态字节码实现中切换.

动态实现与本地实现相比,其运行效率要快上20倍左右.这是因为动态实现无需经过JAVA 到C++ 再到JAVA 的切换,但由于生成字节码十分耗时,仅调用一次的话,反而是本地实现要快上3到4倍.

考虑到许多反射调用仅调用一次,JAVA 虚拟机设置了一个阈值15(可以通过 **-Dsun.reflect.inflationThreshold=value** 调整),当某个反射的调用次数在15之下时,采用本地实现,当达到15时,便开始动态生成字节码,并将委派实现对象切换至动态实现,这个过程称为Inflation(扩充).



例V1版本ReflectTest:

```java
public class ReflectTest {
    public static void target(int i){
        new Exception("#"+i).printStackTrace();
    }
    public static void main(String[] args) throws ClassNotFoundException, 
    		NoSuchMethodException, 
    		InvocationTargetException, 
    		IllegalAccessException {
        Class klass = Class.forName("jvmtest.ReflectTest");
        Method method = klass.getMethod("target",int.class);
        for(int i=0;i<20;i++) {
            method.invoke(null, i);
        }

    }
}
```

第15,16次执行堆栈信息如下:

```
java.lang.Exception: #15
	at jvmtest.ReflectTest.target(ReflectTest.java:14)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:483)
	at jvmtest.ReflectTest.main(ReflectTest.java:20)
java.lang.Exception: #16
	at jvmtest.ReflectTest.target(ReflectTest.java:14)
	at sun.reflect.GeneratedMethodAccessor1.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:483)
	at jvmtest.ReflectTest.main(ReflectTest.java:20)
```

可以看到,在第十五次反射调用,触发了动态实现的生成,这是JVM 加载了不少类,如GeneratedMethodAccessor1,并且从第16次反射调用开始,便切换至这个刚刚生成的动态实现.

反射调用的Inflation机制是可以通过参数(**-DSun.reflect.noInflation=true**)来关闭的,这样在反射调用一开始便会直接生成动态实现,而不会使用委派实现或者本地实现.

### 反射调用的开销

在前面写的例子中,我们先后进行了 Class.forName,Class.getMethod,Method.invoke 三个操作.其中Class.forName 会调用本地方法,Class.getMethod 则会遍历该类的公有方法,若未查询到会遍历父类的公有方法.这两个操作都异常耗时.

注意:getMethod 为代表的的查找方法,会返回当前查找得到结果的一份拷贝.因此避免在热点代码中使用返回Method 数组的getMethods或getDeclaredMethods,以减少不必要的对空间消耗.获取同一个方法的两个Method 对象,equals 相同.

在实践中,我们往往会在应用程序中缓存Class.forName 和 Class.getMethod 的结果.因此我们只关注反射调用本身的性能开销.

为了比较直接调用和反射调用的性能差距,将前面的例子修改为下面的V2版本.

```java
public class ReflectTest2 {
    public static void target(int i){
    }
    public static void main(String[] args) throws ClassNotFoundException, 
    		NoSuchMethodException, 
    		InvocationTargetException, 
    		IllegalAccessException {
        Class klass = Class.forName("jvmtest.ReflectTest2");
        Method method = klass.getMethod("target",int.class);
        long current = System.currentTimeMillis();
        for(int i=1;i<2_000_000_000;i++) {
            if(i%100_000_000 == 0){
                long temp=System.currentTimeMillis();
                System.out.println(temp-current);
                current = temp;
            }
            method.invoke(null, 128);
        }

    }
}
```

上述代码一共将反射调用二十亿次,并且记录下每跑一亿次的时间.我们取最后五个记录的平均值,作为预热后的峰值性能.一亿次执行大概需要440ms.这个不调用的时间基本是一致的.原因在于这段代码属于热循环同样会触发即时编译.并且及时编译会将对Test.target 的调用内联,从而消除调用的开销.

```java
    Code:
       0: ldc           #2                  // String jvmtest.ReflectTest2
       2: invokestatic  #3                  // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
       5: astore_1      
       6: aload_1       
       7: ldc           #4                  // String target
       9: iconst_1      
      10: anewarray     #5                  // class java/lang/Class
      13: dup           
      14: iconst_0      
      15: getstatic     #6                  // Field java/lang/Integer.TYPE:Ljava/lang/Class;
      18: aastore       
      19: invokevirtual #7                  // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
      22: astore_2      
      23: invokestatic  #8                  // Method java/lang/System.currentTimeMillis:()J
      26: lstore_3      
      27: iconst_1      
      28: istore        5
      30: iload         5
      32: ldc           #9                  // int 2000000000
      34: if_icmpge     88
      37: iload         5
      39: ldc           #10                 // int 100000000
      41: irem          
      42: ifne          63
      45: invokestatic  #8                  // Method java/lang/System.currentTimeMillis:()J
      48: lstore        6
      50: getstatic     #11                 // Field java/lang/System.out:Ljava/io/PrintStream;
      53: lload         6
      55: lload_3       
      56: lsub          
      57: invokevirtual #12                 // Method java/io/PrintStream.println:(J)V
      60: lload         6
      62: lstore_3      
      63: aload_2       
      64: aconst_null   
      65: iconst_1      
      66: anewarray     #13                 // class java/lang/Object
      69: dup           
      70: iconst_0      
      71: sipush        128
      74: invokestatic  #14                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      77: aastore       
      78: invokevirtual #15                 // Method java/lang/reflect/Method.invoke:(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;
      81: pop           
      82: iinc          5, 1
      85: goto          30
      88: return        
```

上方是将V2版本代码后编译形成的字节码.从图中可以看出如下:

1. 由于Method.invoke 是一个变长参数方法,在字节码层面它的最后一个参数是Object 数组.Java 编译器会在方法调用处生成一个长度为传入参数的Object 数组.并将传入参数存储进数组中.
2. 由于Object 不能存储基本类型.Java 编译器会对传入的基本类型参数进行自动装箱.

注:对于自动装箱,JAVA 缓存了[-128,127] 中所有整数对应Integer 对象.当需要自动装箱的数值在这个范围时,便返回缓存的Integer.可以使用**-Djava.lang.Integer.IntegerCache.high**=128 来调整Integer 的最大值.

调整IntegerCache 最大值后.耗时没有太大变化,前两次的耗时明显降低.通过GC log 来看,发生GC次数大大减少.

```java
public class ReflectTest3 {
    public static void target(int i){
    }
    public static void main(String[] args) throws ClassNotFoundException, 
    		NoSuchMethodException, 
    		InvocationTargetException, 
    		IllegalAccessException {
        Class klass = Class.forName("jvmtest.ReflectTest3");
        Method method = klass.getMethod("target",int.class);
        Object [] objArgs = new Object[1];
        objArgs[0]=128;
        long current = System.currentTimeMillis();
        for(int i=1;i<2_000_000_000;i++) {
            if(i%100_000_000 == 0){
                long temp=System.currentTimeMillis();
                System.out.println(temp-current);
                current = temp;
            }
            method.invoke(null, objArgs);
        }

    }
}
```

上方是V3版本代码,声明了objArgs 作为变长参数.耗时没有太大变化.GC 次数大大减少.可以关闭反射调用Inflation机制,取消委派实现,直接使用动态字节码的方式.会更快.

### 增加反射调用的开销

上述的例子中,之所以反射调用这么快,主要是因为即时编译器中的方法内联.在关闭了Inflation 的情况下,内联的瓶颈与Method.invoke 方法中对MethodAccessor.invoke 方法的调用.

在生产环境中.我们往往拥有对多个不同的反射调用,对应多个GeneratedMethodAccessor--动态字节码实现

```java
public class ReflectTest4 {
    public static void target(int i) {
    }

    public static void target1(int i) {
    }

    public static void target2(int i) {
    }
    public static void polluteProfile() throws Exception {
        Method method1 = ReflectTest4.class.getMethod("target1", int.class);
        Method method2 = ReflectTest4.class.getMethod("target2", int.class);
        for (int i = 0; i < 2000; i++) {
            method1.invoke(null, 0);
            method2.invoke(null, 0);
        }
    }
    public static void main(String[] args) throws Exception {
        Class klass = Class.forName("jvmtest.ReflectTest4");
        Method method = klass.getMethod("target", int.class);
        Object[] objArgs = new Object[1];
        objArgs[0] = 128;
        polluteProfile();
        long current = System.currentTimeMillis();
        for (int i = 1; i < 2_000_000_000; i++) {
            if (i % 100_000_000 == 0) {
                long temp = System.currentTimeMillis();
                System.out.println(temp - current);
                current = temp;
            }
            method.invoke(null, objArgs);
        }
    }
}
```



由于JVM 关于调用点的**类型Profile**(对于invokevirtual或者invokeinterface,JVM 会记录下调用者的具体类型)无法同时记录这么多类,因此可能造成所测试的反射调用无法被内联.

注:通过-XX:TypeProfileWidth(默认2)可以调整JVM关于每个调用能够记录的类型数目.

l

### 反射API 简介

获取Class 对象:

1. 使用静态方法Class.forName来获取
2. 调用对象的getClass 方法
3. 直接用类名+".class" 访问,对于基本类型来说,它们的包装类型(wrapper classes) 拥有一个名为"TYPE"的final 静态字段,指向该基本类型对应的Class 对象(如,Integer.TYPE 指向int.class,对于数组类型来说,  可以使用 类名+"[].class"来访问,如int[].class)

注:Class 类和java.lang.reflect 包中还提供了许多返回Class 对象的方法,如对于数组类的Class 对象,调用Class.getComponentType() 方法可以获得数组元素的类型.

获得了Class 对象便可以使用反射功能了.下面为较为常用几项:

1. 使用newInstance 来生成一个该类的实例.(要求无参构造器)
2. 使用instance(Object) 判断一个对象是否为该类的实例,(语法上等同于instanceof 关键字)
3. 使用Array.newInstance(Class,int) 来构造该类型的数组
4. 使用getFields/getConstructors/getMethods 来访问类的成员

注:方法名中带Declared 的不会返回父类成员,但是会返回私有成员,而不带Declared 的相反



当获得类成员之后,可以进一步做如下操作.

- 使用Constructor/Field/Method.setAccessible(true)绕过Java 语言的访问限制.
- 使用Constructor.newInstance(Object [])来生成该类实例
- 使用Field.get/set(Object)访问字段值
- 使用Method.invoke(Object,Object[])来调用方法

## 7.JVM 是如何执行invokedynamic

## 8.Java 对象的内存布局

### 在Java 中,创建新对象的方式如下:

1. 使用new 关键字 (调用构造器)
2. 使用反射newInstance (调用构造器)
3. Object.clone (复制已有数据,来初始化实例字段)
4. 反序列化  (复制已有数据,来初始化实例字段)
5. Unsafe.allocateInstance  只实例化空间,没有初始化字段

### Java 构造方法约束

1. 若一个类没有定义任何构造方法,Java 编译器会自动添加一个无参构造方法
2. 子类构造方法需要调用父类构造方法,如果父类存在无参构造方法,调用时隐式的,Java 编译器会自动添加对父类构造方法的调用.但如果父类没有无参构造方法,那么子类构造方法需要显示调用父类构造方法.(否则编译报错)
3. 显示调用可以通过this 调用同类中其余构造方法,可以通过super 调用父类构造方法, 但都必须作为构造方法中的第一条语句.
4. 总结来说,当调用一个构造方法时,将优先调用父类构造方法,直至Object 类,这些构造方法的调用者皆为统一对象,通过new 指令新建的对象.
5. 通过 new 指令创建出来的对象,它的内存涵盖了父类中的所有实例字段,虽然子类无法访问父类的私有实例字段,或者子类的实例字段隐藏了父类的同名实例字段,但是子类的实例还是会为这些父类实例分配内存的.



### 压缩指针

在JVM 中每个对象都有一个**对象头**(**object header**),由**标记字段**和**类型指针**构成

标记字段:

> 存储JVM有关该对象的运行数据:哈希码,GC 信息,锁信息,

类型指针:

> 类型指针指向该对象的类



在64位JVM 中,对象头的标记字段占64位,而类型指针又64位,说明,每个Java 对象在内存中额外开销就是16个字节.以Integer 为例,它仅有一个int 类型的私有字段,占4个字节.因此每一个Integer 对象的额外内存开销是400%.(这也是Java 引入基本数据类型的原因之一)

为了尽量减少对象的内存使用量,64位的JVM引入了压缩指针的概念(-**XX:+UseCompressedOops**,默认开启),将堆中原本64位的Java 对象压缩成32位.这样对象头中的类型指针也被压缩成32位,使得对象头的大小由16个字节降至12个字节.压缩指针不仅可以作用于对象头的类型指针,还可以作用于引用类型的字段,以及类型数组

### 压缩指针的原理

JVM 有32位与 64位:

1. 32位对应4G 内存寻址空间
2. 64位对应 32G内存寻址空间(正常来讲是8G,开启压缩指针达到了 32G)

64位JVM 在支持更大寻址空间的同时,对象引用占用了8个字节,带来了如下问题:

1. 增加了GC 开销,对象引用挤占了对象数据存储,更易触发GC
2. 64位对象引用增大了,CPU 能缓存的对象指针(OOP)将会更少,降低了CPU 缓存的效率

为了保持,32位的性能,对象指针必须保留32位,如何使用32位OOP来引用更大的堆内存呢->通过压缩指针

JVM 的实现方式是,不在保存所有引用,而是每隔8个字节保存一个引用(通过移位实现),堆中的引用实际存储时还是按照0x0,0x1,0x2 ...进行存储.只不过当引用被存入64位寄存器的时候,JVM 会将其左移3位,转换成 0x0,0x8,0x10.而当从寄存器读出时,JVM 右移3位.

因此计算:

​	32位JVM: 2^32=4G 寻址空间,

​	64位JVM:2^32 * 3(通过偏移)=32G 寻址空间

上述的模型,移位同时需要有个前提,对象的起始地址需要对齐至8的倍数,如果一个对象用不到8N个字节,那么空间就浪费了.浪费的空间成为对象间的填充padding.(对应 -XX:ObjectAlignmentInBytes,默认为8),把这个概念成为**内存对齐**



注:就算关闭了压缩指针,JVM 还是会进行**内存对齐**,此外,内存对齐不仅存在于对象与对象之间,也存在于对象中的字段之间,字段**内存对齐**的其中一个原因,是让字段只出现在同一CPU 缓存行.为了防止出现**CPU伪共享缓存行**问题

> Java8 还引入了一个新的注释@Contended ,用来解决CPU 伪共享问题:
>
> 两个线程访问同一对象中不同的volatile 字段,逻辑上,它们没有共享内容不需要同步,然后若这两个字段恰好在同一CPU缓存行中,那么对这些字段的写操作会导致缓存行的写回

### 字段重排列

JVM 重新分配字段的先后顺序,达到内存对齐的目的.JVM 中有三种排列方法(对应-XX:FieldsAllocationStyle 默认为1),但都会遵循如下两个原则:

- 如果一个字段占据C个字节,那么该字段的偏移量需要对齐至NC,这里偏移量指的是字段地址与对象的起始地址差值

  > 以Long 类为例,它仅有一个long 类型实例字段,在使用压缩指针的64位虚拟机中,尽管对象头的大小为12个字节,该long 类型字段的偏移量 也只能是16 ,中间空着4个字节会被浪费掉.

- 子类所继承字段的偏移量,需要与父类对应字段偏移量保持一致

  > 在具体实现中,JVM 还会对齐子类字段的起始位置,对于使用了压缩指针的64位JVM,子类第一个字段对齐至4N;而对于关闭了压缩指针的64位JVM,子类第一个字段需要对齐至8N

### Java 的对象访问

1. 使用句柄访问

   > Java 堆中会划分一块内存来作为句柄池,reference 中存储的就是句柄池中的句柄对象地址

2. 直接访问

   > reference 存储的直接就是内存地址

## 9.垃圾回收

### 引用计数法与可达性分析

如何判断一个对象是否需要销毁:

引用计数法:

> 为每个对象添加一个引用计数器,用来统计指向该对象的引用个数,一旦某个对象的引用计数器为0,说明对象死亡,可以回收.
>
> but,引用计数法无法处理循环引用对象,造成内存泄露

可达性分析算法:

> 将一系列GC Roots作为初始的存活对象合集(live set),然后从该集合出发,探索所有能够被该集合引用到的对象,并将其加入到该集合中.这个过程我们称之为标记.最终未被探索到的对象便是死亡的,可以回收.
>
> 可达性分析可以解决引用计数法无法解决的循环引用问题.

GC Roots ,可以理解为堆外指向堆内的引用,一般包括但不限于如下几种:

1. Java 方法栈帧中的局部变量
2. 已加载类的静态变量
3. JNI handles
4. 已启动且未停止的Java 线程

### Stop the world 以及安全点

在JVM 中,传统的垃圾回收算法采用的是,简单粗暴方式,Stop the world,停止非垃圾回收线程的工作,直到完成垃圾回收.这也就造成了垃圾回收所谓的暂停时间(GC pause)

JVM 中,Stop the world 是通过安全点(safepoint)机制来实现的,当JVM 收到Stop the world 请求,会等待所有的线程都到达安全点,才允许请求Stop the world 的线程进行独占工作.

安全点的初始目的不是让其他线程停下来,而是找到一个稳定的执行状态,在这个执行状态下,JVM 的堆栈不会发生变化.此时,垃圾回收期便能够安全的执行可达性分析.

可以作为安全点的:

执行JNI 本地代码api入口处,解释执行字节码,执行即时编译器生成的机器码和线程阻塞,其余运行状态,需要JVM 保证在可预见的时间内进入安全点,否则垃圾回收线程可能长期处于等待所有线程进入安全点的状态,从而提高了垃圾回收的暂停时间.



### 垃圾回收的三种方式

#### 清除-sweep

​	造成内存碎片,由于JVM 堆中对象必须是连续分布的,因此可能出现总空闲空间足够,但是无法分配的极端情况

#### 压缩-compact

​	可以解决碎片化问题,但是压缩算法的性能开销较大

#### 复制-copy

​	内存区域分为两等分,分别用两个指针from,to 维护,并且只用from 指针指向的内存区域分配内存,当发生垃圾回收时,把存活的对象复制到to 指针指向的区域中,并且交换from 指针和to指针的内容

​	可以解决碎片化问题,堆空间利用率低下.

### 内存分代回收

​	经过统计分析,大部分的Java 对象只存活一小段时间,而存活下来的小部分Java 对象会存活很长一段时间.

​	这也引出了JVM分代回收思想.将堆空间划分为两代,新生代,老年代.新生代存储新建对象,老年代存储常驻对象.

​	JVM 可以再不同代使用不同的回收算法,

​		对于新生代,可以频繁地采用耗时较短的垃圾回收算法--Minor GC

​		对于老年代,老年代中的对象在发生垃圾回收时大概率存活,当真正触发对老年代的回收时,代表堆空间已经耗尽

​		JVM 需要做一次全堆扫描,--Full GC

### JVM 的堆划分

JVM 分为新生代和老年代.

新生代划分为:Eden区,以及两个大小相同的Survivor 区(from,to 典型的 复制算法).

> -XX:+UsePSAdaptiveSurvivorSizePolicy 根据生成对象的速率,以及Survivor 的使用情况,动态调整Eden 区域与Survivor 的比例
>
> -XX:SurvivorRatio 可以设置 Eden 区域 Survivor 比例.

![](C:\Users\Administrator.20180329-134844\Desktop\堆空间内存划分.png)

 通常来说,当调用new 指令时,会在Eden 区中划出一块作为存储对象的内存,由于堆空间是线程共享的,因此在同一时间可能会有多个线程在堆上申请空间,因此,每次对象分配都需要同步,而在竞争激励的场合分配的效率会下降.

JVM 通过 TLAB(Thread Local Allcoation Buffer ,对应 -XX:+UseTLAB,默认开启),**线程本地缓存**解决此问题.

TLAB 本身占用Eden 区空间,在开启TLAB 的情况下.JVM 会为每个Java 线程分配一块TLAB空间.TLAB 空间的内存非常小,缺省情况下仅占有整个Eden空间的1%,(通过 -XX:TLABWasteTargetPercent)设置TLAB 空间所占用Eden空间的百分比大小.

由于TLAB 空间一般不大,因此大对象无法在TLAB 上进行分配,总会直接分配在堆上.TLAB 空间由于较小,如:100K空间,已经使用了80K,需要再分配一个30K 的对象时,JVM 会有两种选择

1. 废弃当前TLAB 这样会浪费20KB 空间
2. 将30KB 直接分配在堆上,保留当前TLAB,20KB 空间可能还会被使用.

注:JVM 内部会维护refill_waste 的值,当请求对象大于refill_waste 时,会选择在堆中分配,若小于该值,则会废弃当前TLAB,新建TLAB 来分配对象(-XX:-ResizeTLAB 可以禁用TLAB 大小调整,-XX:TLABSize 可以手工指定TLAB 大小)



当Eden 区空间耗尽,这时JVM 会触发一次Minor GC,来收集新生代垃圾,过程如下:

> Eden 区 和 from 指向的Survivor 中存活的对象会被复制到to 指定的Survivor ,然后交换 from 和to 指针,保证
>
> 下一次Minor GC 时,to 指向的Survivor 区还是空的.

在Minor GC 的过程中,JVM 会记录Survivor 中的存活对象经历垃圾回收的次数,若大于15

(-XX:+MaxTenuringThreshold),那么此对象将晋升至老年代,另外如果单个Survivor 区已经被占用50%,

(-XXTargetSurvivorRatio),那么较高复制次数的对象也会被晋升至老年代

### 卡表

Minor GC 是不需要对整个堆进行扫描回收的(仅Eden 与 from),会引出一个问题,若老年代中的对象引用新生代的对象,当标记存活对象的时候,需要扫描老年代中的对象. 若该对象拥有对新生代对象的引用,那么这个引用会被作为GC Roots. 这样会扫描老年代,进行了一次全堆扫描.



JVM 的解决方式是 卡表(card table),将堆空间划分为一个大小为512 字节的卡表,用来存储每张卡的一个标识位,这个标识位代表对应的卡是否可能存有指向新生代对象的引用,在执行Minor GC 的时候,便不用扫描所有老年代,只需要扫描卡表中,标识位为脏的对应的老年代堆空间,将其中对象加入到GC Roots 里,当完成卡表扫描后,清空卡表中的标志位.



### 垃圾回收器

#### 新生代

Serial 								复制-单线程

Parallel Scavenge			复制-多线程-吞吐率高-不能与CMS 一起使用

Parallel New					 复制-多线程

#### 老年代

Serial Old						   标记压缩-单线程

Parallel Old						标记压缩-多线程

CMS									标记清除-并发 已经在Java9中废弃使用G1 替代



#### G1

Garbage First 是一个横跨新生代和老年代的垃圾回收器,实际上,它已经打乱了前面所说的堆结构,直接将堆分成及其多个区域,每个区域都可以充当Eden区,Survivor 区,或者老年代.采用的是标记压缩算法



## 10.Java 内存模型

为了让应用程序免于数据竞争的干扰,Java 5 引入了明确定义的Java 内存模型.其中最为重要的概念便是 happens-before 关系.

### happens-before 

happens-before 关系是用来描述两个操作的内存可见性.若 操作X happens-before 操作Y ,那么X 的结果于Y可见.

在同一线程中,字节码的先后顺序也暗含了happens-before 关系:在程序控制流路径中靠前的字节码happens-before 靠后的字节码.注:可能会出现指定重排.

在多线程中,happens-before 关系如下:

1. 解锁操作happens-before对同一把锁的加锁操作
2. volatile 字段的写操作happens-before同一字段的读操作.
3. 线程的启动操作(Thread.start())happens-before该线程第一个操作
4. 线程的最后一个操作 happens-before 它的终止时间
5. 线程对其他线程的终端操作 happens-before 被中断线程所收到的中断事件
6. 构造方法的最后一个操作happens-before 构造方法的第一个操作

Java 内存模型是通过内存屏障(memory barrier 来禁止重排序的),对于即时编译器来说,它会针对前面提到的每一个happens-before 关系,向正在编译的目标方法中插入相应的读读,读写,写读以及写写的内存屏障.

这些内存屏障会限制即时编译器的重排序操作.以volatile 字段访问为例,所插入的内存屏障将不允许volatile 字段写操作之前的内存访问被重排序至其之后,也将不允许volatile 字段读操作之后的内存访问被重排序至其之前.



### 锁,volatile字段,final 字段 与安全发布

#### 锁

锁操作同样具备happens操作关系,解锁操作happens-before 之后对同一把锁的加锁操作.在解锁时JVM 需要强制刷新缓存,使得当前线程锁修改的内存对其他线程可见.

注:若编译器能够(通过逃逸分析)证明某把锁仅被同一线程持有,那么它可以移除响应的加锁解锁操作,因此不会强制刷新缓存.

#### volatile

volatile 字段可以看成一种轻量级的,不保证原子性的同步,其性能往往由于所操作,然而,频繁的访问volatile字段也会因为不断的强制刷新缓存而严重影响程序的性能.

在x86_64 平台,只有volatile 字段的写操作会强制刷新缓存,因此理想情况下对volatile 字段的使用应道多读少写,并且只有一个线程进行写操作.

volatile 字段的另一个特性是无法将其分配到**寄存器**中,volatile 字段的读写每次访问均需从内存中读写.

#### final

final 实例字段设计新建对象的发布问题,当一个对象包含final 实例字段时,我们希望其他线程只能看到已初始化的final 实例字段.因此Java 编译器会在final 字段的写操作后插入一个写写屏障,防止优化将新建对象的发布重排序至final 字段的写操作之前.(在x86_64平台,写写屏障是空操作)

当发布一个已初始化对象时,我们希望所有已初始化的实例字段对其他线程可见,否则,其他线程可能见到一个部分初始化的新建对象,从而造成程序错误.可以参考https://vlkan.com/blog/post/2014/02/14/java-safe-publication/

注:对象在构造方法执行返回后才会达到一个稳定的发布状态.**对象发布** 在构造方法返回后执行.

但是,如果在执行构造方法过程中对象还没构件结束,发生了逃逸,如下:

```java
public ThisEscape(EventSource source) {
    source.registerListener(
        new EventListener() {
            public void onEvent(Event event) {
                doSomething(event);
            }
        });
    }
}
```

上述代码,当ThisEscape 对象发布EventListener时,可能整暗地里发布发布ThisEscape 对象.因为内部类实例包含一个隐藏封闭类实例引用.

可以通过如下方式避免在构造方法执行过程中的对象逃逸行为.

```java
public class SafeListener {
    private final EventListener listener;

    private SafeListener() {
        listener = new EventListener() {
            public void onEvent(Event event) {
                doSomething(event);
            }
        }
    }

    public static SafeListener newInstance(EventSource source) {
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
```

上述代码存在线程安全问题.可以通过synchronized 解决.或者依靠类的内部类的方式解决.

## 11.JVM 是如何实现Synchronized

Synchronized 关键字可以对程序进行加锁,可以 声明代码块,或直接标记静态方法与实例方法.

### synchronized 声明代码块



当声明synchronized 代码块时,编译而成的字节码会包含 `monitorenter` 与 `monitorexit` 指令.这两种指令会消耗操作数栈上的一个引用类型的元素(synchronized 关键字括号内的引用),作为所要锁操作的对象

```java
    public void foo(Object lock) {
        synchronized (lock) {
            lock.hashCode();
        }
    }

	//字节码如下
 	public void foo(java.lang.Object);
    Code:
       0: aload_1       
       1: dup           
       2: astore_2      
       3: monitorenter  
       4: aload_1       
       5: invokevirtual #2                  // Method java/lang/Object.hashCode:()I
       8: pop           
       9: aload_2       
      10: monitorexit   
      11: goto          19
      14: astore_3      
      15: aload_2       
      16: monitorexit   
      17: aload_3       
      18: athrow        
      19: return        
    Exception table:
       from    to  target type
           4    11    14   any
          14    17    14   any
```

上文为synchronized 的代码块,以及它所编译成的字节码.从字节码可知,synchronized 包含一个monitorenter以及多个monitorexit指令.因为JVM 要确保获得的锁在正常执行路径,以及异常执行路径都能解锁.

### synchronized 标记方法

当用synchronized 标记方法时,在字节码中方法的访问标记包括ACC_SYNCHRONIZED.该标记标识在进入该方法时,JVM 需要进行monitorenter 操作.而再退出该方法时,不管是否正常返回,还是向调用者抛异常.JVM 均需要进行monitorexit操作.



```java
public synchronized void foo(Object lock) { 
	lock.hashCode();
}
// 上面的 Java 代码将编译为下面的字节码 
public synchronized void foo(java.lang.Object); 
	descriptor: (Ljava/lang/Object;)V 
	flags: (0x0021) ACC_PUBLIC, ACC_SYNCHRONIZED 
	Code: 
		stack=1, locals=2, args_size=2 
        0: aload_1 
        1: invokevirtual java/lang/Object.hashCode:()I 
        4: pop 
        5: return
```

注:如上反编译结果为文中所述,在Jdk8中反编译synchronized 在方法上是,并未出现ACC_SYNCHRONIZED标识.怀疑是在JVM 关键字中进行的处理,或反编译时候没哟短处方法描述信息.TODO

当**synchronized**修饰方法时有如下两种情况:

1. 修饰实例方法		锁对象等同于this
2. 修饰公共方法        锁对象等同于Class 实例.

关于monitorenter 和monitorexit 的作用:

> 抽象理解成每个锁对象拥有一个锁计数器和一个指向持有该锁的线程的指针.
>
> 当执行monitorenter时,如果目标锁对象的计数器为0,那么说明没有被其余线程持有,在这个情况下,JVM 会将该锁对象的持有线程设置为当前线程,并且将其计数器加1.
>
> 若锁对象的计数器不为0时,如果锁对象的持有线程是当前线程(锁重入),那么JVM 将其计数器加1,否则需要等待,直至持有线程释放该锁
>
> 当执行monitorexit 时JVM 将锁对象的计数器减1.当计数器减为0时,标识该锁已经被释放掉了.



### 重量级锁

重量级锁是JVM 中最为基础的锁实现.在这种情况下,JVM 会阻塞加锁失败的线程,并且在目标锁被释放的时候,唤醒这些线程.

Java 线程的阻塞以及唤醒,都是依靠操作系统来完成的.(这些操作涉及操作系统调用,需要从操作系统的用户态切换至内核态,其开销非常之大)

为了避免昂贵的线程阻塞,唤醒操作,JVM 会在线程进入阻塞状态前,以及唤醒后竞争不到锁的情况下,进入自旋状态,在处理器上空跑并且轮训锁是否释放.如果此时锁恰好释放了,那么当前线程变无需进入阻塞状态,而是直接获得这把锁.

与线程阻塞相比,自旋状态可能会浪费大量CPU 资源.这是因为当前线程仍处于运行状况,只不过跑的是无用指令.它期望在运行无用指令的过程中,锁能够被释放出来.

JVM 的自适应自旋,根据以往自旋等待时是否能够获得锁,来动态调整自旋的时间(循环数目).

自旋的另一个副作用是不公平的锁机制.处于阻塞状态的线程,并没有办法like竞争被释放的锁.然而处于自旋状态的线程,则可能有限获得这把锁.

### 轻量级锁

竞争不激烈的情况下,采用的锁方式:多个线程在不同的时间段请求同一把锁,没有锁竞争,这时JVM 采用轻量级锁便面重量级锁的阻塞与唤醒.

#### JVM 如何区分轻量级锁与和重量级锁

前文Java 对象的内存布局中对象头提到的 标记字段(mark down).它的最后两位便被用来表示该对象的锁状态.其中00 代表轻量级锁,01代表无锁(或偏向锁),10代表重量级锁,11则跟垃圾回收算法的标记有关.

当进行加锁操作时,JVM 会判断是否是重量级锁,如果不是,会在当前线程的栈帧中划出一块空间,作为锁的锁记录,并且将锁对象的标记字段复制到该锁记录中.然后JVM 会尝试用CAS 操作替换锁对象的标记字段.具体过程如下:

> 假设当前锁对象的标记字段值为  A,JVM 会比较该字段 的最后两位是否为01.如果是,则替换为刚才分配的锁记录的地址.由于内存对齐的缘故,它的最后两位为00.此时该线程已经获得这把锁,可以继续执行.
>
> 若A最后两位不是01,会继续判断 是否是该线程持有该锁(锁重入),此时,JVM 会将锁记录清零,代表该锁被重复获取.若其他线程持有该锁,此时JVM 会将该锁膨胀为重量级锁,并且阻塞当前线程.
>
> 当进行解锁操作时,如果当前锁记录的值为0,则代表重复进入同一把锁,直接返回即可.
>
> 否则,JVM 会使用CAS 操作,比较锁对象标记字段的值是否与当前锁记录的地址,若是则替换锁记录中的值,也就是锁对象原本的标记字段,此时该线程已经成功释放这把锁.
>
> 如果不是,则意味着这把锁已经被膨胀成重量级锁,此时JVM 会进入重量级锁释放的过程,唤醒因竞争该锁而被阻塞了的线程

### 偏向锁

从始至终只有一个线程请求某一锁.

在线程进行加锁时,如果该锁对象支持偏向锁,那么JVM 会通过CAS 操作,将当前线程的地址记录在锁对象的标记字段中,并且标记字段的最后三位设置为101.

在接下来的运行过程汇总,每当有线程请求这把锁,JVM 只需要判断锁对象标记字段中最后三位是否为101,是否包含当前线程地址,以及epoch 值是否和锁对象的epoch 值相同,如果都满足当前线程持有该偏向锁,可以直接返回.

epoch 值概念如下:

> 偏向锁的撤销:当请求加锁的线程与锁对象标记字段保持的线程地址不匹配时,(而且 epoch 值相等,如若不等,那么当前线程可以将该锁重偏向至自己)JVM 需要撤销该偏向锁.这个撤销过程非常麻烦,它要求持有偏向锁的线程到达安全点,再将偏向锁替换成轻量级锁.
>
> 若某一类锁对象的总撤销数超过了一个阈值(对应JVM 参数-XX:BiasedLockingBulkReiasThreshold 默认为20)
>
> 那么JVM 会将这个类的偏向锁失效.
>
> 具体做法是每个类中维护一个epoch 值,可以理解为第几代偏向锁,当设置偏向锁时JVM 需要将该epoch 值复制到锁对象的标记字段中.在宣布某个类的偏向锁失效时,JVM 需要将该类的epoch 值加1,标识之前那一段偏向锁已经失效.而新设置的偏向锁需要复制新的epoch 值.
>
> 为了保证当前持有偏向锁且已经加锁的线程不至于丢锁,JVM 需要遍历所有线程栈,找出该类加锁的实例并将它们标记字段中的epoch 加1,该操作需要线程处于安全点.
>
> 当总撤销数超过另一个阈值(对应JVM 参数-XX:BiasedLockingBulkRevokeThreshold 默认为40),那么JVM 会认为这个类已经不适合偏向锁,此时会撤销该类实例偏向锁,并且在后续的加锁过程中直接设置为轻量级锁.

## 12.Java 语法糖与Java 编译器

### 自动拆装箱

Java 8个基本类型,每个基本类型都有对应的包装(wrapper)类型.之所以引入包装类型,是因为许多Java 和兴类库的API 都是面向对象的.

例如容器类,仅支持引用类型.但是可以 直接添加基本类型,Java 编译器会隐式的进行装箱操作.

```java
    public int foo(){
        List<Integer> list = new ArrayList<>();
        list.add(0);
        int value = list.get(0);
        return value;
    }
  public int foo();
    Code:
       0: new           #2                  // class java/util/ArrayList
       3: dup           
       4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
       7: astore_1      
       8: aload_1       
       9: iconst_0      
      10: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      13: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
      18: pop           
      19: aload_1       
      20: iconst_0      
      21: invokeinterface #6,  2            // InterfaceMethod java/util/List.get:(I)Ljava/lang/Object;
      26: checkcast     #7                  // class java/lang/Integer
      29: invokevirtual #8                  // Method java/lang/Integer.intValue:()I
      32: istore_2      
      33: iload_2       
      34: ireturn       
}
```

从上述代码与反编译所得字节码可知,在向泛型参数为Integer 的list 添加int值时,有自动装箱操作,调用了Integer.valueOf方法,将int 转换为Integer 存储至容器中.

当从泛型参数为Integer 的list去除元素时,实际得到的是Integer 对象,如果接收是int 会发生自动拆箱.

将执行Integer.intValue方法.

### 泛型 与 类型擦除

从上述反编译字节码中可知,ArrayList get 方法 与 add 方法,返回结果 与 参数 均是Object,但是由于泛型类型是Integer,在调用get 方法时,需要将Object 转化成Integer `checkcast java/lang/Integer`

在编码中使用到的泛型,在经过编译器变异后会进行 `泛型的类型擦除` ,当然,并不是每一个泛型参数被擦除后都会变成Object 类,对于限定了继承类的泛型参数,经过类型擦除后,所有的泛型参数将变成所限定的继承类.Java 编译器将选取该泛型所指代的所有类中层次最高的那个,作为泛型替代类.



对于泛型方法,为了保证重写的语义,.子类中会生成 父类的泛型方法,并桥接到其子类重写方法中.

子类中重写父类的方法不能显示的直接调用,可以通过反射调用.

## 13.即时编译

HotSpot JVM 包含多个及时即时编译器 C1,C2以及Graal

Graal 是一个实验性质的即时编译器,可以通过 XX:+UnlockExperimentalVMOptions -XX:+UseJVMCIComplier 启用,并替换C2

在Java7 之前,需要根据程序特性选择对应的即时编译器.对于执行时间较短的,或者启动性能有要求的程序,采用编译效率较块的C1 (GUI,Swing 客户端),对于执行时间较长的,或者峰值性能有要求的程序,采用生成代码执行效率较块的C2.



在Java 7,引入了分层编译(对应参数 -XX:+TieredCompilation) 的概念,综合了C1的启动性能优势 和C2 的峰值性能优势.

分层编译将JVM 的执行状态分为五个层次.(`C1 代码` 指C1生成的机器码,`C2代码`指C2生成的机器码)五个层次如下:

1. 解释执行			`执行profiling收集`
2. 执行不带 `profiling` 的C1代码
3. 执行仅带方法调用次数以及循环回边执行次数 `profiling` 的C1 代码   `执行profiling收集`
4. 执行带所有profiling 的C1代码 `执行profiling收集`
5. 执行C2代码

通常 C2代码的执行效率比C1代码高30%以上.然而,对于C1代码的3种状态,执行效率从高至低是 2层>3层>4层

其中2层比3层稍微高一些,3层比4层高出30%左右.

`profiling`是指在程序执行过程中,收集能够反映程序执行状态的数据.这里所收集的数据我们称之为程序的profile

在5个层次的执行状态中,2层与5层是终止状态.当一个方法被终止状态编译过后,如果编译后的代码没有失效,那么JVM是不会再次发出该方法的编译请求的.

下面例句4个不同的编译路径:

1. 通常情况下,热点方法会被4层的C1编译,然后被5层的C2编译
2. 如果方法的字节码数目比较少,而且4层的profiling 没有可收集的数据,那么JVm判定该方法对于C1 与C2 执行效率基本相同.这种情况下Java 虚拟机会在4层编译之后,直接使用2层的C1编译,由于这是终止状态,JVM 不会再使用5层的C2编译
3. 在C1忙碌的情况下,JVM 会在解释执行过程中对程序进行profiling,而后直接由5层的C2编译.
4. 在C2忙碌的情况下,方法会被3层的C1编译,然后在被4层的C1编译,以减少方法在4层的执行时间

Java8 默认开启了分层编译,不管开启还是关闭分层编译,原本用来选择即时编译器的参数-client 和 -server 都是无效的,在关闭分层编译器的情况下JVM 将直接使用C2.



### 即时编译的触发

JVM 是根据**方法的调用次数** 以及**循环回边的执行次数**来触发即时编译的.前面提到,JVM 在1,3,4层执行时会进行 profiling 数据收集.这就包含方法的调用次数和循环回边的执行次数.

循环回边可以简单理解成在字节码中往回跳转的指令.(循环)

具体来说,在不启用分层编译的情况下,当方法的调用次数和循环回边的次数的和,超过由参数-XX:CompileThreshold 指定的阈值时(使用C1时该值为1500,使用C2时,该值为10000),便会触发即时编译.

当启用分层编译时,JVM 将不再采用由参数 -XX:CompileThreshold 指定的阈值,而是使用另一个阈值,这个阈值时动态调整的.



### Profiling

分层编译中的1层,3层和4层都会进行profiling,收集能够反映程序执行状态的数据.最为基础的为便是方法的调用次数与循环回边的执行次数.它们被用于触发即时编译.

此外1层和4层还会收集用于5层C2编译的数据,比如说分支跳转字节码的分支profile(branch profile),包括跳转次数与不跳转次数,以及非私有实例方法调用指令,强制类型转换指令,类型测试instanceof 指令.和引用类型的数组存储aastore 指令的类型profile.

分支profile 和 类型profile 的收集将给应用程序带来不少的性能开销.4层C1代码执行的性能比3层C1代码低30%

通常情况,不会再解释执行过程中收集分支profile以及类型profile,只有在方法触发C1编译后,JVM认为该方法可能被C2编译,才在该方法的C1代码中收集这些profile.

### 基于分支profile的优化(剪枝优化)

查看如下逻辑代码:

```java
    public static int foo(boolean f, int in) {
        int v;
        if (f) {
            v = in;
        } else {
            v = (int) Math.sin(in);
        }
        if (v == in) {
            return 0;
        } else {
            return (int) Math.cos(v);
        }
    }

	//下方是反编译字节码
  	public static int foo(boolean, int);
    Code:
       0: iload_0       
       1: ifeq          9
       4: iload_1       
       5: istore_2      
       6: goto          16
       9: iload_1       
      10: i2d           
      11: invokestatic  #2                  // Method java/lang/Math.sin:(D)D
      14: d2i           
      15: istore_2      
      16: iload_2       
      17: iload_1       
      18: if_icmpne     23
      21: iconst_0      
      22: ireturn       
      23: iload_2       
      24: i2d           
      25: invokestatic  #3                  // Method java/lang/Math.cos:(D)D
      28: d2i           
      29: ireturn     
```

![](C:\Users\Administrator.20180329-134844\Desktop\logic.png)



假设应用程序调用该方法时,所传入的boolean 值皆为true.那么上述两个分支判断的false 都不会执行.字节码中偏移量为1,与18 的跳转你指令所对应的分支profile 中,跳转次数都为0.

C2 可以根据这两个分支profile 做出假设,在接下来的执行过程中,这两个条件跳转指令仍旧不会发生跳转.基于这个假设,C2便不再编译这两个条件跳转语句所对应的false 分支了.

(目前暂时忽略上述假设的错误情况).经过剪枝后,在第二个条件跳转处,v的值只有可能为所输入的int值.因此,该条件跳转可以进一步被优化,最终的优化结果,在第一个条件跳转后,c2代码将直接返回0.



根据条件跳转指令的分支profile,即时编译器可以将从未执行过的分支剪掉.避免编译这些很有可能不会用到的代码,从而节省编译时间以及部署代码所消耗的内存空间.此外,剪枝将精简程序的数据流,从而触发更多的优化.

除了剪枝优化,对于分支profile,C2还会计算每一条程序执行路径的概率,以便某些编译器优化优先处理概率较高的路径.



### 基于类型profile 的优化

```java
    public static int hash(Object in) {
        if (in instanceof Exception) {
            return System.identityHashCode(in);
        } else {
            return in.hashCode();
        }
    }
 	public static int hash(java.lang.Object);
    Code:
       0: aload_0       
       1: instanceof    #4                  // class java/lang/Exception
       4: ifeq          12
       7: aload_0       
       8: invokestatic  #5                  // Method java/lang/System.identityHashCode:(Ljava/lang/Object;)I
      11: ireturn       
      12: aload_0       
      13: invokevirtual #6                  // Method java/lang/Object.hashCode:()I
      16: ireturn       
```

假设应用程序调用该方法时,所传入的Object 皆为Integer 实例.那么偏移量为1 的instanceof 指令的类型profile 仅包含Integer,偏移量为4的分支跳转语句的分支profile 中不跳转的次数为0,偏移量为13的方法调用指令的类型profile仅包含Integer.

![](C:\Users\Administrator.20180329-134844\Desktop\logic1.png)

在JVM 中,如果instanceof 的目标类型是final类型,那么JVM仅需比较测试对象的动态类型是否为final类型.

如果目标类型不是final类型,比如上述代码中的Exception,那么JVM需要从测试对象的动态类型开始,依次测试该类,父类,祖先类,该类所直接实现或者间接实现的接口是否与目标类型一致.

不过在我们的例子中,instanceof 指令的类型profile仅包含Integer.根据这个信息,即时编译器可以假设,在接下来的执行过程中,所输入的Object对象仍为Integer 实例.

因为,生成的代码将测试所输入的对象的动态类型是否为Integer.如果是,则继续执行接下来的代码.(该优化院子Graal,C2无法复现),然后,即时编译器会采用和第一个例子中一致的针对分支profile 的优化,以及对方法调用条件去虚化内联.

### 去优化

当假设失败的情况下,程序将何去何从.

JVM 的解决方案是去优化.即从执行即时编译生成的机器码,切换回解释执行.

在C2生成的机器码中,即时编译器将在假设失败的位置上插入一个陷阱(trap).该陷阱实际上是一条call 指令,调用至JVM 里专门负责去优化的方法.与普通的call 指令不一样.去优化方法将更改栈上的返回地址,并不在返回即时编译器生成的机器码中.



## 14.JAVA 字节码

### 操作数栈

JAVA 字节码是JVM 所使用的指令集.因此它与JVM基于栈的计算模型是密不可分的.

在解释执行的过程中,每当为Java 方法分配栈帧时,JVM 往往会开辟一块额外空间作为操作数栈,来存放计算的操作数以及返回结果.

具体如下:

​	执行每一条指令前,JVM 要求该指令的操作数,已经压入操作数栈中,在执行指令时,JVM 会将该指令所需的操作数弹出,并将指令的结果重新压入操作数栈.

![1564102525711](C:\Users\Administrator.20180329-134844\AppData\Roaming\Typora\typora-user-images\1564102525711.png)

如上iadd 指令,假设在执行该指令前,栈定的两个元素分别为int 1 与 int 2,那么iadd 指令将弹出这两个int,并将求得的和 int 3 压入栈中.

![1564102617684](C:\Users\Administrator.20180329-134844\AppData\Roaming\Typora\typora-user-images\1564102617684.png)

由于iadd 指令只消耗栈顶的两个元素,因此对于离站定距离为2的元素,即图中的问号,iadd 指令并不关心它是否存在,更加不会对其进行修改.

Java 字节码中好几条指令是直接作用在操作数栈上的.最常见的 dup:复制栈顶元素,以及pop :舍弃栈顶元素.

dup 指令常用语复制new 指令所生成的未经初始化的引用.(将其引用入栈调用,构造方法),如下,当执行new 指令时,JVM 将指向一块已分配的,未初始化的内存引用压入操作数栈中.接下来需要以这个引用为调用者,调用其构造器,也就是字节码中invokespecial指令.要注意,该指令将消耗操作数栈上的元素,作为它的调用者以及参数

```java
    public void foo(){
        Object obj = new Object();
    }
	//对应字节码如下
    public void foo();
    Code:
       0: new           #2                  // class java/lang/Object
       3: dup           
       4: invokespecial #1                  // Method java/lang/Object."<init>":()V
       7: astore_1      
       8: return   
```

因此,我们需要使用dup 指令,复制一份new指令的结果,并用来调用构造方法.当调用返回之后,操作数栈上仍有原本new 指令生成的引用,既用于接下来的操作(即偏移量为7的字节码)



pop 指令则常用语舍弃调用指令的返回结果.如下代码,调用静态foo 方法,但是不使用其返回值.

```java
    public void foo(Object test){
        bar();
    }
  	public static boolean bar();
    Code:
       0: iconst_0      
       1: ireturn       

  	public void foo(java.lang.Object);
    Code:
       0: invokestatic  #2                  // Method bar:()Z
       3: pop           
       4: return       
```

dup 与 pop 指令只能处理非long 或者非double 类型的值,这是因为long 类型或 double 类型的值,需要占据两个栈单元.当遇到这些值时,我们需要同时复制栈顶两个单元的 dup2 指令,以及弹出栈顶两个单元的pop2 指令.

除此之外,不常见但是也能直接作用于操作数栈的还有swap指令,它将交换栈顶两个元素的值

在字节码中有一部分指令可以直接将常量加载到操作数栈上.以int 类型为例,JVM 既可以通过iconst 指令加载-1到5之间的int 值,也可以通过binpush,sipush 加载一个字节,两个字节所能代表的int 值

JVM 还可以通过idc 加载常量池中的常量值,例如 ldc #18 将加载常量池中的第18项.

| 类型                          | 常数指令 | 范围                         |
| ----------------------------- | -------- | ---------------------------- |
| int (boolean,byte,char,short) | iconst   | [-1,5]                       |
|                               | bipush   | [-128,127]                   |
|                               | sipush   | [-32768,32767]               |
|                               | ldc      | any int value                |
| long                          | lconst   | 0,1                          |
|                               | ldc      | any long value               |
| float                         | fconst   | 0,1,2                        |
|                               | ldc      | any float value              |
| double                        | dconst   | 0,1                          |
|                               | ldc      | any double value             |
| reference                     | aconst   | null                         |
|                               | ldc      | String literal,Class literal |

### 常量加载指令表.

正产情况下,操作数栈的压入,弹出都是一条条指令完成的.唯一的例外情况是在抛出异常时,JVM 会清除操作数栈上的所有内容.而后将异常实例压入操作数栈上.

### 局部变量区

Java 方法栈帧的另一个重要组成部分则是局部变量区,字节码程序可以将计算的结果缓存在局部变量区中.

实际上,JVM 将局部变量区当成一个数组,一次存放this 指针(仅非静态方法),所传入的参数,以及字节码中的局部变量

与操作数栈相同,long 类型以及double 类型占用两个单元,其余类型占用一个单元.

```java
	public void foo(long l,float f){
        {
            int i=0;
        }
        {
            String s ="Hello,World";
        }
    }
```

上述代码foo 方法为例,由于它是一个实例方法,因此局部变量数组的第0个单元存放着this指针.

第一个参数类型为long类型,于是数组的1,2 两个单元存放着所传入的long类型参数的值.第二个参数则是float 类型,于是数组的第三个单元存放着所传入的float 类型的值.

![1564131942514](C:\Users\Administrator.20180329-134844\AppData\Roaming\Typora\typora-user-images\1564131942514.png)

在方法体里的两个代码块中,我分别定义了两个局部变量i和s.由于这两个局部变量的生命周期没有重合之处,因此Java 编译器可以将他们编排至同一单元中.也就是说局部变量数组的第四个单元将为i/s.

存储在局部变量区的值,通常需要加载至操作数栈中,方能进行计算,得到计算结果后在存储至局部变量数组中.这些加载,存储指令是区分类型的.如下:

| 类型                          | 加载指令 | 存储指令 |
| ----------------------------- | -------- | -------- |
| int (boolean byte char short) | iload    | istore   |
| long                          | lload    | lstore   |
| float                         | fload    | fstore   |
| double                        | dload    | dstore   |
| reference                     | aload    | astore   |

### 局部变量区访问指令表

局部变量数组的加载,存储指令都需要指明所加载单元的下标.举例来说, aload 0 值得是加载第0个单元所存储的引用,在前面实例中的foo方法里指的便是加载this 指针.

Java 字节码中唯一能够直接作用于局部变量区的指令是 iinc M N(M 为非负整数,N 为整数).

该指令指的是将局部变量数组的第M个单元中的int 值增加N,常用于for 循环中自增量的更新.如下:

```java
	public void foo(){
        for(int i=100;i>=0;i--){
        }
    }
	//下方为字节码
    public void foo();
    Code:
       0: bipush        100
       2: istore_1      
       3: iload_1       
       4: iflt          13
       7: iinc          1, -1
      10: goto          3
      13: return       
```

看一个综合例子:

```java
    public static int bar(int i){
        return ((i+1)-2)*3/4;
    }
  	public static int bar(int);
    Code:
       0: iload_0       
       1: iconst_1      
       2: iadd          
       3: iconst_2      
       4: isub          
       5: iconst_3      
       6: imul          
       7: iconst_4      
       8: idiv          
       9: ireturn      
```



### Java 字节码简介

java 相关指令:

​	new (后跟目标类,生成该类的未初始化的对象)

​	instanceof(后跟目标类 判断栈顶元素是否为目标类,目标接口,是压入1 不是压入0)

​	checkcast(后跟目标类,判断栈顶元素是否为目标类/接口的实例,不是便抛出异常)	

​	athrow(将栈顶异常抛出)

​	monitorenter(栈顶对象加锁)

​	monitorexit(栈顶对象解锁)

​	getstatic 静态字段获取	

​	putstatic 静态字段设置

​	getfield   实例字段获取

​	putfield  实例字段设置

注:getstatic,putstatic,getfield,putfield 这四条指令均附带用以定位目标字段的信息.但所消耗的操作数栈元素皆不同.



方法调用指令

​	invokestatic

​	invokespecial

​	invokevirtual

​	invokeinterface

​	invokedynamic

#### 数组相关指令

新建基本数组的newarray

新建引用类型数组的 anewarray

生成多维数组 multianewarray

求数组长度 arraylength

还有如下的数组加载指令:

| 数组类型      | 加载指令 | 存储指令 |
| ------------- | -------- | -------- |
| byte(boolean) | baload   | bastore  |
| char          | caload   | castore  |
| short         | saload   | sastore  |
| int           | iastore  | iastore  |
| long          | laload   | lstore   |
| float         | faload   | fstore   |
| double        | daload   | dstore   |
| reference     | aaload   | astore   |

数组访问指令表

控制流指令,包括无条件跳转 goto,条件跳转指令  tableswitch,lookupswitch (前者针对密集的cases,后者针对稀疏的cases),返回指令 如下:

| 返回类型                     | 返回指令 |
| ---------------------------- | -------- |
| void                         | return   |
| int(byte boolean char short) | ireturn  |
| long                         | lreturn  |
| float                        | freturn  |
| double                       | dreturn  |
| reference                    | areturn  |

## 15.方法内联

方法内联指的是:在编译过程中遇到方法调用时,将目标方法的方法体纳入编译范围之中,并取代原方法调用的优化手段.

方法内联不仅可以消除调用本身带来的性能开销,还可以进一步触发更多的优化.因此它可以算是编译优化中最为重要的一环.

> 注:此处说明的编译,指的不是 编译Java 代码成字节码,而是指即时编译.

以getter/setter为例,若没有方法内联,在调用getter/setter时,程序需要保存当前方法的执行位置,创建并压入getter/setter 的栈帧,访问字段,弹出栈帧,最后在恢复当前方法的执行.而当内联了 getter/setter的方法调用后,上述操作仅剩字段访问.

在C2 中,方法内联是在解析字节码的过程中完成的.每当碰到方法调用字节码时,C2将决定是否内联该方法调用.如果需要内联.则开始解析目标方法的字节码.

在Graal 中,也会在解析字节码的过程中进行方法调用的内联.此外Graal 还拥有一个独立的优化节点,来寻找指代方法调用的IR 节点,并将之替换为目标方法的IR 图.

### 方法内联的条件

方法内联能够触发更多的优化.通常而言内联越多,生成代码的执行效率越高.然而,对于即时编译器来说.内联越多,编译时间越长,从而程序达到峰值性能的时刻也将被推迟.

此外内联越多也将导致生成的机器码越长,在JVM 中,编译生成的机器码会被部署到Code Cache 中(-XX:ReservedCodeCacheSize).越容易填满Code Cache,从而出现Code Cache 已满,即时编译器已被关闭的警告信息(CodeCache is full,Compiler has been disabled)

因此即时编译器不会无限制的进行方法内联,下方为在JVM 中即时编译器的内联规则:

1. 自动拆箱总会被内联
2. Throwable 类的方法不能被其他类中的方法所内联
3. 由-XX:CompileCommand 中的inline 指令指定的方法,以及由@ForceInline 注解的方法(仅限于JDK 内部方法),会被强制内联.
4. 由-XX:CompileCommand 中的dontinline 指令或exclude 指令(表示不编译)指定的方法,以及由@DontInline 注解的方法(仅限于JDK内部方法),则始终不会被内联
5. 若调用字节码对应的符号引用未被解析,目标方法所在类未被初始化,或者目标方法是native 方法,都将导致方法调用无法关联.
6. C2 不支持内联超过9层的调用(-XX:MaxInlineLevel),以及1层的直接递归调用
7. 即时编译器将根据方法调用指令所在程序路径的热度,目标方法的调用次数及大小.以及当前IR图的大小来觉醒方法调用能否被内联.



### 即时编译器去虚化

对于静态方法的调用,JVM 是可以确定唯一目标方法的.然而对于需要动态绑定的虚方法调用来说,即时编译器则需要先对虚方法调用进行去虚化(devirtualize),即转换为一个或多个直接调用,然后才能进行方法内联.

及时编译器的去虚化可分为**完全去虚化**以及**条件去虚化**

完全去虚化是 通过类型推导或者类层次分析(class hierarchy analysis),识别虚方法调用的唯一目标方法,从而将其转换为直接调用的一种优化手段.它的关键在于证明虚方法调用的目标方法是唯一的.

条件去虚化则是将虚方法调用转换为若干个类型测试以及直接调用的一种优化手段.它的关键在于找出需要进行比较的类型.将调用者的动态类型依次与Jvm 虚拟机所收集的类型Profile 中记录的类型相比较.若匹配,则直接调用该记录类型所对应的目标方法.

## 16.HotSpot JVM 的 intrinsic

### @HotSpotIntrinsicCandidate

在HotSpot JVM 中,所有被该注解标注的方法都是HotSpot intrinsic.对这些方法的调用,会被HotSpot JVM 替换成高效的指令序列.而原本的方法实现则会被忽略掉.(HotSpot JVM 将标注了@HotSpotIntrinsicCandidate 注解的方法额外维护一套高效实现.)



### intrinsic 与 CPU 指令

如:StringLatin1.indexOf 方法将在一个字符串(byte 数组) 中查找另一个字符串(byte 数组),并且返回命中时的索引,或者 -1 (未命中).

恰巧,在X86_64 体系架构的SSE4.2 指令集就包含一条指令 PCMPESTRI,它能够在16字节一下的字符串中,查找另一个16字节以下的字符串,并且返回命中时的索引值.因此HotSpot 虚拟机便围绕这一指令,开发出X86_64 体系架构上的高效实现.并替换原本对StringLatin1.indexOf 方法的调用.



intrinsic 与 方法内联

HotSpot JVM 中,intrinsic 的实现分为两种

1. 独立的桩程序,既可以被解释执行器利用,直接替换对原方法的调用,也可以被即时编译器利用,把它代表对原方法的调用的IR 节点,替换为对这些桩程序的调用的IR 节点,以这种形式实现的intrinsic比较少,主要包括Math 类中的一些方法.

2. 特殊的编译器IR 节点.(只能被即时编译器利用)大部分的intrinsic 都是通过这种方式实现的.

   > 注:这个替换过程是在方法内联时进行的.当即时编译器碰到方法调用节点时,将查询目标方法是不是intrinsic
   >
   > 是,则插入相应的特殊IR 节点,不是进行原本的内联操作.
   >
   > 也就是说,如果方法调用的目标方法是intrinsic,那么即时编译器会直接忽略远目标方法的字节码,甚至根本不在乎远目标方法是否有字节码.即便是native 方法,只要他被标记为intrinsic,即时编译器便能将之内联起来.并插入特殊的IR 节点.
   >
   > 实际上,大部分标记为intrinsic 的方法都是native 方法,原本对这些native 方法的调用需要经过JNI ,其性能开销十分巨大,但是经过即时编译器的intrinsic 优化后,这部分JNI 开销直接消失.性能提升明显.
   >
   > 例如 `Thread.currentThread `方法获取当前线程,这是一个native 方法,同时也是一个HotSpot intrinsic.在X86_64体系架构中,R13寄存器存放着当前线程的指针.因此,对该方法的调用将被即时编译器替换一个特殊IR 节点,并生成读取R13寄存器指令

### 已有intrinsic 简介

最新版本的HotSpot JVM 中定义了三百多个intrinsic.

在这三百多个intrinsic 中,有三成异常时Unsafe 类的方法.不过我们一般会直接使用 Unsafe类的方法,而是通过

`java.util.concurrent`包来间接使用.

> Unsafe 类中经常会被用到的便是compareAndSwap 方法,在X86_64 体系架构中,对这些方法的调用将被替换为 `lock cmpxchg` 指令,也就是原子性更新指令.

除了Unsafe 类的方法,HotSpot 虚拟机中的intrinsic 还包括下面的几种.

1. StringBuilder 与 StringBuffer 类的方法,HotSpot 虚拟机将优化利用这些方法构造字符串的方式,以尽量减少内存复制的情况.(1.8 中未查到)
2. String 类,StringLatin1类,StringUTF16类 和Arrays 类的方法,HotSpot JVM 将使用SIMD 指令(single instruction multiple data ,即用一条指令处理多个数据) 对这些方法进行优化.
3. 基本类型的包装类,Object 类,Math 类 ,System 类中各个功能性方法,反射API ,MethodHandle 类中与调用机制相关的方法,压缩,解密相关方法.这部分intrinsic 则比较简单.

## 17.逃逸分析

逃逸分析是"一种确定指针动态范围的静态分析,它可以分析在程序的哪些地方可以访问到指针"

在JVM 即时编译语境下,逃逸分析将判断新建对象是否逃逸.即时编译器判断对象是否逃逸的依据:

1. 对象是否被存入堆中(静态字段或者堆中对象的实例字段)

   一旦对象被存入堆中,其他线程便能够获得该对象的引用.即时编译器也因此无法追踪所有使用该对象的代码位置.

2. 对象是否被传入未知代码中

   由于JVM 的即时编译是以方法为单位的,对于方法中未被内联的方法调用,即时编译器会将其当做未知代码,毕竟它无法确认该方法调用会不会将调用者所传入的参数修改.通常来说,即时编译器的逃逸分析是方法方法内敛之后的,以便消除这些'未知代码'入口.

### 基于逃逸分析的优化

即时编译器可以根据逃逸分析的结果进行诸如锁消除,栈上分配,以及标量替换的优化.

#### 锁消除

通常,即时编译器可以消除对该不逃逸锁对象的加锁,解锁操作.实际上,传统编译器仅需证明锁对象不逃逸出线程,便可以进行所消除.由于JVM 即时编译的限制,上述条件被强化为证明锁对象不逃逸出当前编译的方法.

synchronized(new Object()){} 会被完全优化掉,这正是因为基于逃逸分析的锁消除.由于其他线程不能够获取该所对象,因此也无法基于该锁对象构造两个线程之间的happens-before 规则.(这种情况不常见,基本不会这么写代码)

synchronized(escapedObject){} 就不一样,由于其他线程可能会对逃逸对象escapedObject 进行加锁操作,从而造成了两个线程间的 happens-before 关系,因此即时编译器至少需要为这段代码生成一条刷新缓存的内存屏障指令.

#### 栈上分配 以及 标量替换

JVM 中的对象都是在堆上分配的,而对象的内容对于任何线程都是可见的.为此同时,JVM 需要对所分配的堆内存进行管理,并且在对象不再被引用时回收其所占据的内存.

若,逃逸分析能够证明某些新建的对象不逃逸,那么JVM 完全可以将其分配到栈上,并且在new 语句所在的方法退出时,通过弹出当前方法的栈帧来自动回收所分配的内存空间.这样一来,可以无需借助GC 来回收不在被引用的对象.

不过,由于实现起来需要更改大量假设了"对象只能对分配" 的代码,因此HotSpot JVM 并没有采用栈上分配,而是采用 **标量替换** 这一技术.

> 所谓的标量,就是仅能存储一个值的变量,比如Java 代码中的局部变量.与之想法,聚合量则可能同时存储多个值,其中一个典型的例子便是Java 对象.

标量替换这项优化技术,可以看成将原本对对象的字段的访问,替换为一个个局部变量的访问.

将对象访问直接替换成局部变量访问.原本需要在堆内存中连续分布的对象,被拆分成一个个单独在栈上分配的变量.



### 部分逃逸分析

C2 的逃逸分析与控制流无关,相对来说比较简单.Graal 则引入了一个与控制流有关的逃逸分析,名为[部分逃逸分析].

它解决了所新建的实例尽在部分程序路径中逃逸的情况.

如下代码,新建实例只会在进入if-then 分支时逃逸.(对hashCode 方法的调用是一个HotSpot intrinsic,将被替换为一个无法内联的本地方法调用)

```java
    public void test(boolean condition){
        Object obj = new Object();
        if(condition){
            obj.hashCode();
        }
    }

	//可以手工优化成
    public void test(boolean condition){
        if(condition){
            Object obj = new Object();
            obj.hashCode();
        }
    }
```

假设if语句的条件成立的可能性只有1%,那么在99%的情况下,程序没有必要新建对象.其手工优化的版本正是逃逸分析想要自动达成的效果.

部分逃逸分析将根据控制流信息,判断出新建对象仅在部分分支中逃逸,并且将对象的新建操作推延至对象逃逸的分支中.这将使得原本因对象逃逸而无法避免的新建对象操作,不在出现在执行if-else 分支的程序路径中.

综上,与C2 所使用的逃逸分析相比,Graal 所适用的部分逃逸分析能够优化更多的情况,不过它编译时间也更长一些.

## 18.字段访问相关优化

虽然在部分情况下,逃逸分析以及基于逃逸分析的优化已经十分高效了,能够将代码优化到极其简单的地步.

在现实中,Java 程序中的对象或许本身便是逃逸的,或许因为方法内联不够彻底,而被即时编译器当成是逃逸的.

这两种情况都将导致即时编译器无法进行标量替换.这时候,针对对象字段访问的优化也变得格外重要起来.

```java
	static int bar(Foo foo,int x){
        foo.a = x;
        return foo.a;
    }
```

在上述代码中,对象foo是传入参数,不属于逃逸分析的范围(JVM中的逃逸分析对象针对的是新对象).该方法会将所传入的int 型参数x 的值存储至实例字段Foo.a 中,然后再读取并返回同一字段的值.

这段代码将涉及两次内存访问操作:存储以及读取实例字段Foo.a,我们可以将其手工优化成直接读取并返回传入参数x的值.

### 字段读取优化

即时编译器会做出类似的优化.即时编译器会优化实力字段以及静态字段访问,已减少总的内存访问数目.具体来说,它将沿着控制流缓存各个字段存储节点将要存储的值,或者字段读取节点所得到的值

当即时编译器遇到对同一字段的读取节点时,如果缓存值还没有失效,那么它将会读取节点替换为该缓存值.

当即时编译器遇到对同一字段的存储节点时,它会更新所缓存的值.当即时编译器遇到可能更新字段的节点时,如方法调用节点(在即时编译器看来,方法调用会执行未知代码),或者内存屏障节点(其他线程可能异步更新了字段),那么会舍弃缓存值.

```java
static int bar(Foo o,int x){
	int y= o.a +x;
	return o.a+y;
}
```

上述代码中,实例字段Foo.a读取了两次,即时编译器会将第一次读取的值缓存起来,如下:

```java
static int bar(Foo o,int x){
    int t = o.a;
    int y = t+x;
	return t+y;
}
```



如果字段读取节点被替换成常量值,那么将进一步触发更多优化

```java
static int bar(Foo o,int x){
    o.a=1;
    if(o.a >= 0){
        return x;
    }else{
        return -x;
    }
}
```

上述代码,经过字段读取优化之后,if 条件恒为true,如此 else 分支将变成不可达代码,可以删除.优化如下:



```java
static int bar(Foo o,int x){
	o.a=1;
    return x;
}
```



下面在看一个有意思的例子.

```java
class Foo {
	boolean a;
	void bar(){
		a=true;
		while(a){
			//do something
		}
	}
	void whatever(){a=false;}
}
```

同样,即时编译器会将while 循环中读取实例字段a的操作直接替换为常量a,即下方死循环代码

```java
	void bar(){
		a=true;
		while(true){}
	}
```

注:从上文介绍Java 内存模型时,我们知道,volatile 关键字标识的字段,可以强制 读取.即时编译器将在volatile访问前后插入内存屏障,,内存屏障会阻止即时编译器将缓存使用.除了volatile ,加锁,解锁,也会阻止字段缓存优化.

### 字段读取优化

即时编译器还会对存储进行优化. 消除冗余存储

```java
	int a=0;
	void bar(){
		a=1;
		a=2;
	}
```

会直接优化成

```java
	void bar(){
		b=2
	}
```



### 死代码消除

除了字段存储优化之外,局部变量的死存储同样涉及了冗余存储,这是死代码消除的一种.

```java
int bar(int x,int y){
	int t=x*y;
	t=x+y;
	return t;
}
```

将被优化成

```java
int bar(int x,int y){
	return x+y;
}
```

## 19.循环优化

### 循环无关代码外提

将循环中值不变的表达式外移,避免重复执行表达式,提升性能

```java
	int fooManualOpt(int x,int y,int[] a){
		int sum = 0;
		int t0 = x * y;
        // 即时编译器优化后 外移
		int t1 = a.length;
		for(int i=0;i<t1;i++){
			// int t1 = a.length 优化前
			sum+=t0+a[i];
		}
	}
```

在Sea-of-Nodes IR 的帮助下,循环无关代码外移



### 循环展开

循环体中重复多次循环迭代,并减少循环次数的编译优化

在C2 中,只有计数循环(Counted Loop) 才能被展开.需满足如下:

1. 维护一个循环计数器,且基于计数器的出口只有一个(但可以有基于其他判断条件的出口)
2. 循环计数器的类型为int,short,char (即不能是byte,long,float 或者double)
3. 每个迭代循环计数器的增量为常数
4. 循环计数器的上下限是循环无关的常数

循环展开可能会增加代码的冗余度,导致所生成机器码的长度大幅上涨.不过,随着循环体的增大,可能会触发更多的优化.

循环展开有一种特殊情况,便是完全展开,当循环的数目是固定值,且非常小,即时编译器会将循环全部展开.此时,原本循环中的循环判断语句将不复存在,展开成 若干个顺序执行的循环体.

### 其他循环优化

循环判断外提

循环判断外提是指将循环中的if 语句外提至循环之前,并且在该if语句的两个分支中分别放置一份循环代码

```java
 int foo(int[] a){
 	int sum = 0;
 	for(int i=0;i<a.length;i++){
 		if(a.length>4){
 			sum+=a[i];
 		}
 	}
 }
```

上述代码,循环外提后变成如下:

```javascript
 int foo(int[] a){
 	int sum = 0;
 	if(a.length>4){
        for(int i=0;i<a.length;i++){
            sum+=a[i];
        }
 	}else{
        for(int i=0;i<a.length;i++){
        }
    }
 }
```



循环剥离

循环剥离指的是将循环的前几个迭代 或 后几个迭代剥离出循环.一般循环的前几个或后几个迭代都包含特殊处理.通过剥离前后的特殊处理,可以使原本的循环体的规律更加明显,触发进一步优化.

```java
int foo(int[] a){
	int j = 0;
	int sum = 0;
	for(int i=0;i<a.length;i++){
		sum +=a[j];
		j=i;
	}
	return sum;
}
```

剥离后会优化成

```java
int foo(int[] a){
	int j = 0;
	int sum = 0;
	if(0<a.length){
		sum += a[0];
        for(int i=1;i<a.length;i++){
            sum +=a[i-1];
        }
	}
	return sum;
}
```

## 20.向量化

```java
void foo(byte[] dst,byte[] src){
	for(int i=0;i<dst.length-4;i+=4){
        dst[i]=src[i];
        dst[i+1]=src[i+1];
        dst[i+3]=src[i+2];
        dst[i+3]=src[i+3];
    }
}
```

由于X86_64 平台不支持内存间的直接移动,上面代码中的`dst[i] = src[i]`通常会被编译成两条内存访问指令:

1. 把src[i] 的值读取至寄存器中
2. 把寄存器中的值写入dst[i] 中

因此上述代码的一个循环迭代将会执行四条内存读取指令,以及4条内存写入指令.

由于数组元素在内存中是连续的.当从src[i]的内存地址处读取32位的内容时,我们可以一次读取 src[i]至src[i+3]的值.也可以一次写入 dst[i]至dst[i+3]的值.通过上面这两个批量操作,可以使用一条内存读取指令与一条内存写入指令,完成上面代码中循环体内的工作.

### SIMD 指令

![](C:\Users\Administrator.20180329-134844\Desktop\寄存器位数对比.png)

SIMD 指令将XMM寄存器(或YMM,ZMM ) 中的值看成多个整数或者浮点数组成的向量,并批量进行计算.

### 使用SIMD指令的HotSpot Intrinsic

在Java 中,使用instrinsic 方法来执行具体的SIMD 指令.在运行过程中,HotSpot 将根据当前体系架构来决定是否对方法的调用使用高效实现,使用切换到 Intrinsic 指令方法,否则使用原本Java 实现.

如Java8 中的Arrays.equals(int[]  ,int[] )的实现将逐个比较int 数组中的元素.

对应的Instrinsic实现会将数组的多个元素加载至XMM/YMM/ZMM寄存器中,然后进行按位比较.

### 自动向量化

在循环展开优化后,将不同迭代的运算合并为向量运算.



## 21.JVM 监控诊断工具

### 命令行

#### jps

> jps -mlv 
>
> 输出进程Id,主类名, -l打印模块名与包名,-v 打印传递给JVM 的参数,-m打印传递给主类的参数
>
> 执行效果如下
>
> jps -mlv 
> 20 /app/jar/ROOT/SC.jar -Duser.timezone=GMT+8 -Xms3024m -Xmx3024m -XX:PermSize=512m -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=512m -XX:+PrintGC -XX:+PrintGCTimeStamps -XX:+PrintGCDetails -XX:+PrintGCApplicationStoppedTime -Xloggc:gc.log -XX:+UseConcMarkSweepGC -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/app/jar/logs/20190416163004-845b5fc4fb-xhld4/java_heapdump.hprof -Dfile.encoding=utf-8 -XX:+CMSClassUnloadingEnabled -XX:+CMSPermGenSweepingEnabled

#### jstat

打印目标Java 进程性能数据.

#### jinfo

查看目标Java 进程的参数

#### jmap

分析JVM 中堆对象

#### jstack

查看线程栈轨迹

