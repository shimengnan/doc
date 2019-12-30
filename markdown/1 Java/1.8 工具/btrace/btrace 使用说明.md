# Btrace 使用
## Btrace 使用限制
> Btrace 脚本就是一个普通的使用@Btrace注解的 Java 类,注意 拦截方法必须是用 public static void 修饰的.

> 为了保证对目标程序不造成影像, Btrace 脚本对其可以执行的动作做了很多限制,如下:
    
    1. 不能创建对象,不能使用数组,不能使用循环
    2. 不能抛出或捕获异常
    3. 不能使用`synchronized`关键字
    4. 属性方法必须使用`static`修饰
    5. 不能对 目标对象的静态实例赋值,不能获取非静态对象
    7. 不能使用内部,外部类
    8. 不能实现接口
> 注意:可以使用 -u 运行在 unsafe mode 来规避限制.

## @OnMethod 注解
> Btrace使用@OnMethod注解定义需要分析的方法入口

``` java
    @OnMethod(
        clazz = "btrace.test.BtraceCase",
        method="add",
        location=@Location(value= Kind.RETURN)
    )
```
> @OnMethod 注解中,有 `class` ,method,location 等参数.`class` 表明需要监控的类,method 表明需要监控的方法.配置方式如下:
1. 使用完全限定名:`clazz="btrace.test.BtraceCase",method="add"`
2. 使用正则表达式:`clazz="/javax\\.swing\\..*/",method="/.*/"`
3. 使用接口:`clazz="+com.test.Filter",method="doFilter"`
4. 使用注解:`clazz="@javax.jws.WebService",method="@javax.jws.WebMethod"`
5. 若需要分析构造方法,需要执行`method="<init>"`

> @Location注解用来定义Btrace对方法的拦截位置,默认为`Kind.ENTRY`

1. `Kind.ENTRY`: 在进入方法时 调用Btrace 脚本
2. `Kind.RETURN`:在方法执行结束,调用Btrace脚本
``` java
    @OnMethod(
        clazz = "btrace.test.BtraceCase",
        method="add",
        location=@Location(value= Kind.RETURN)
    )
    public static void printMethodRunTime(@Self Object self,int a,int b ,@Return int result,@Duration long time){
        println("parameter:a="+a+",b="+b+",cost time:"+ time/1000000);
        counter++;
    }
```
3. `Kind.CALL`:分析方法中调用其他方法的执行情况,比如在execute 方法中,想获取add 方法的执行耗时,必须把 where 设置成`Where.AFTER`
``` java
    @OnMethod(
            clazz = "btrace.test.BtraceCase",
            method="execute",
            location=@Location(value= Kind.CALL,method="add",where=Where.AFTER)
    )
```
4. `Kind.LINE`:通过设置line ,监控代码是否执行到指定的位置
``` java
    @OnMethod(
            clazz = "btrace.test.BtraceCase",
            method="add",
            location=@Location(value= Kind.LINE,line=21)
    )
```
5. `Kind.ERROR`,`Kind.THROW`,`Kind.CATCH` 用于对某些异常的跟踪,包括异常抛出,异常捕获

## 使用实战
1. 统计执行超过 1ms 接口
``` java
    @OnMethod(
            clazz = "+btrace.test.BtraceCaseService",
            method="test",
            location=@Location(value= Kind.RETURN)
    )
    public static void printMethodRunTime(@ProbeClassName String className,@ProbeMethodName String methodName,@Duration long time){
        if(time/1000000 >0)
            println(className+"."+methodName+":"+ time/1000000);
    }
```
2. 分析调用System.GC 的方法,并打印堆栈信息
``` java
    @OnMethod(clazz = "java.lang.System",method = "gc",location = @Location(Kind.ENTRY))
    public static void onSystemGC(){
        println("SYSTEM GC");
        jstack();
    }
```
3. 统计方法的调用次数,每隔一秒钟打印一次
``` java
    @BTrace
    public class BtraceScript4 {
        @Export static long counter;
        @OnMethod(
            clazz = "btrace.test.BtraceCase",
            method="test",
            location=@Location(value= Kind.RETURN)
        )
        public static void run(){
            counter++;
        }
        @OnTimer(1000)
        public static void counter(){
            println("execute count:"+counter);
        }
    }
```
4. 获取执行方法的静态实例属性
``` java
    @BTrace
    public class BtraceScript5 {
        static Field field = BTraceUtils.field(classForName("btrace.test.BtraceCase",contextClassLoader()),"size");
        @OnMethod(
            clazz = "btrace.test.BtraceCase",
            method="test"
        )
        public static void run(@Self Object self){
            int size = BTraceUtils.getInt(field);
            BTraceUtils.println(size);
        }
    }
```

## 其余异常情况
使用idea 启动的服务,使用btrace 不能正产侵入,怀疑是 idea 做了类似的操作.