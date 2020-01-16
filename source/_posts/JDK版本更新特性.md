---
title: JDK版本更新特性
date: 2020-01-16 14:30:27
tags:
---
# Java 8 

JDK8 发布于2014年3月，Oracle 在 2019 年 1 月停止免费商用更新。

## Lambda 和函数式接口

lambda表达式是Java8 中最重大的改变，其允许将函数当成参数传递给某些方法，或者将程序当作处理对象。最简单的Lambda表达式可由逗号分隔的参数列表、->符号和语句块组成。
```java
Arrays.asList( "a", "b", "d" ).forEach( e -> System.out.println( e ) );

```
lambda表达式中复杂语句块需要使用花括号包裹；可以引用类成员和局部变量，这些被引用的对象会被隐式转换为`final`的；只有一行的语句可以省略`return`;参数和返回值类型都自动推断。

lambda表达式配合方法引用可以实现更加简洁高效的代码。方法引用包含4中类型：
1. 构造器引用，Class::new
2. 静态方法引用，Class::static_method
3. 成员方法引用，Class::method
4. 实例对象的方法引用，instance::method

为了让现有的功能更好的和lambda函数兼容，产生了函数式接口的概念。函数式接口指的是只有一个方法的接口。这样的接口可以转化为lambda表达式。为了增加函数式接口的健壮性，可以通过在接口声明上增加`@FunctionalInterface`注解表明函数是一个函数式接口。默认方法和静态方法不会导致函数式接口被破坏。

## 接口静态方法和默认方法

默认方法适用于接口增加新方法，但是无法改动已存在的实现的情况，实现类不必实现新增加的默认方法。该特性在官方库中的应用是：给`java.util.Collection`接口添加新方法，如`stream()`、`parallelStream()`、`forEach()`和`removeIf()`等等。由于JVM上的默认方法的实现在字节码层面提供了支持，因此效率非常高。

```java
private interface Defaulable {
    // 接口现在可以添加默认方法，实现者可以覆写这些方法，也可以不覆写
    default String notRequired() { 
        return "Default implementation"; 
    }
    // 接口现在可以添加静态方法
    static Defaulable create( Supplier< Defaulable > supplier ) {
        return supplier.get();
    }        
}
```

## 重复注解

JDK 1.5 引入注解，但是不允许在同一个地方多次使用同一个注解。JDK 8 中增加了`@Repeatable`注解，来使目标注解变得可以重复使用。
```java
    @Target( ElementType.TYPE )
    @Retention( RetentionPolicy.RUNTIME )
    @Repeatable( Filters.class )
    public @interface Filter {
        String value();
    };
```
除此之外JDK 8 还增强了类型推断和注解的使用范围。

## 参数名称

在编译器编译时候指定参数`-parameters`可以保留源码中参数的名称，随后通过反射可以获取到这个名称。否则参数名称会被`arg0`,`arg1`之类无意义的标识符代替。

```java
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;

public class ParameterNames {
    public static void main(String[] args) throws Exception {
        Method method = ParameterNames.class.getMethod( "main", String[].class );
        for( final Parameter parameter: method.getParameters() ) {
            System.out.println( "Parameter: " + parameter.getName() );
        }
    }
}
```

## 工具类

`Optional`类增强了对空值的处理，可以避免大多数的非空判断以增加代码的简洁性。

`java.util.stream`类是对集合操作的一个极大增强，也可以应用于数据流中。Steam之上的操作可分为中间操作和终止操作。所有的中间操作并不会立刻执行，所有的操作只有在遇到终止操作的时候才会被触发。

Date/Time API 吸收了Joda-Time的精华，将日期相关类设置为不可变对象，当需要修改时会返回新的对象。新的java.time包包含了所有关于日期、时间、时区、Instant（跟日期类似但是精确到纳秒）、duration（持续时间）和时钟操作的类。

## 其他

Nashorn JavaScript引擎，Nashorn JavaScript引擎是`javax.script.ScriptEngine`的另一个实现版本。使得Java代码可以调用JS代码。

新增了Base64，成为官方库文件。

parallexXxx系列方法，增加多核机器数组排序速度。

`java.util.concurrent.ConcurrentHashMap`针对lambda进行优化，内部使用红黑树替代了之前的链表实现。`java.util.concurrent.locks.StampedLock`优化了原来的读写锁，增加了读写锁之间状态的转换，用于支持基于容量的锁。`java.util.concurrent.atomic`包中新增了一些类。

## JVM

使用Metaspace（JEP 122）代替持久代（PermGen space）。在JVM参数方面，使用-XX:MetaSpaceSize和-XX:MaxMetaspaceSize代替原来的-XX:PermSize和-XX:MaxPermSize。

# Java 9

Java 9 正式发布于 2017年9月21日。其中最重要的改动是 Java 平台模块系统的引入。

## 模块化

在引入了模块系统之后，JDK 被重新组织成 94 个模块。Java 应用可以通过新增的 jlink 工具，创建出只包含所依赖的 JDK 模块的自定义运行时镜像。这样可以极大的减少 Java 运行时环境的大小。模块化的控制通过在一个工件的根目录建立一个module-info.java，由其编译而来。工件的格式可以是传统的 JAR 文件或是 Java 9 新增的 JMOD 文件。

模块文件的规则包括：
1. 模块导出的包：使用 exports 可以声明模块对其他模块所导出的包。包中的 public 和 protected 类型，以及这些类型的 public 和 protected 成员可以被其他模块所访问。没有声明为导出的包相当于模块中的私有成员，不能被其他模块使用。
2. 模块的依赖关系：使用 requires 可以声明模块对其他模块的依赖关系。使用 requires transitive 可 以把一个模块依赖声明为传递的。传递的模块依赖可以被依赖当前模块的其他模块所读取。 如果一个模块所导出的类型中包含了来自它所依赖的模块的类型，那么对该模块的依赖应该声明为传递的。
3. 服务的提供和使用：如果一个模块中包含了可以被 ServiceLocator 发现的服务接口的实现 ，需要使用 provides with 语句来声明具体的实现类 ；如果一个模块需要使用服务接口，可以使用 uses 语句来声明。

## JShell

jshell 是 Java 9 新增的一个实用工具。jshell 为 Java 增加了类似 NodeJS 和 Python 中的读取-求值-打印循环(Read-Evaluation-Print Loop)。在 jshell 中 可以直接 输入表达式并查看其执行结果。jsell中每个表达式的结果会被自动保存下来，以数字编号作为引用，类似 $1 和$2 这样的名称。可以在后续的表达式中引用之前语句的运行结果。在 jshell 中 ，除了表达式之外，还可以创建 Java 类和方法。jshell 也有基本的代码完成功能。

## 集合工具类

Java 9 增加 了 `List.of()`、`Set.of()`、`Map.of()` 和 `Map.ofEntries()`等工厂方法来创建不可变集合。 

`Stream` 中增加了新的方法 `ofNullable`、`dropWhile`、`takeWhile` 和 `iterate`。

`Collectors` 中增加了新的方法 `filtering` 和 `flatMapping`。

`Optional`类中新增了 `ifPresentOrElse`、`or` 和 `stream` 等方法。

## 平台日志管理

新增了`System.LoggerFinder`用来管理JDK使用的日志记录器实现，JVM在运行期间通过`LoggerFinder`在系统范围内寻找日志记录器的实现，默认情况下使用`java.logging`模块下的`java.util.logging`来实现。应用程序可以使用系统内置的日志记录器实现。也可以自定义`LoggerFinder`来修改JDK自带的日志记录器。

## 反应式流

反应式流指的式一个带有非阻塞被压得异步流处理规范，在Java平台上流行的反应式库有RxJava和Reactor。反应式流规范的核心接口已经添加到了 Java9 中的`java.util.concurrent.Flow` 类中。其中包含了 `Flow.Publisher`、`Flow.Subscriber`、`Flow.Subscription` 和 `Flow.Processor` 等 4 个核心接口。Java 9 还提供了 `SubmissionPublisher` 作为 `Flow.Publisher` 的一个实现。

## 反射句柄

新增了`java.lang.invoke.VarHandle`类用来指代类中变量，并且提供了包括对变量进行读取、写入、原子更新、数值原子更新和比特位原子操作等操作。`VarHandle`可以通过`java.lang.invoke.MethodHandles.Lookup`中的静态工厂方法来创建 。

`java.lang.invoke.MethodHandles`中新增了不同的静态方法来创建方法句柄。包括：
1. arrayConstructor：创建指定类型的数组。
2. arrayLength：获取指定类型的数组的大小。
3. varHandleInvoker 和 varHandleExactInvoker：调用 VarHandle 中的访问模式方法。
4. zero：返回一个类型的默认值。
5. empty：返 回 MethodType 的返回值类型的默认值。
6. loop、countedLoop、iteratedLoop、whileLoop 和 doWhileLoop：创建不同类型的循环，包括 for 循环、while 循环 和 do-while 循环。
7. tryFinally：把对方法句柄的调用封装在 try-finally 语句中。


## 其他

`CompletableFuture`类增加了新的方法。`completeAsync` 使用一个异步任务来获取结果并完成该 `CompletableFuture`。`orTimeout` 在 `CompletableFuture` 没有在给定的超时时间之前完成时抛出 `TimeoutException` 异常来终止任务。`completeOnTimeout` 与 `orTimeout` 类似，只不过它在超时时使用给定的值来完成 `CompletableFuture`。

`java.io.InputStream`增加了新的方法。`readAllBytes`读取 InputStream 中的所有剩余字节。`readNBytes`从 `InputStream` 中读取指定数量的字节到数组中。`transferTo`读取 `InputStream` 中的全部字节并写入到指定的 `OutputStream` 中 。

优化了Nashorn 引擎，支持了ECMAScript 6规范。Nashorn 还提供了 API 把 ECMAScript 源代码解析成抽象语法树，可以用来对 ECMAScript 源代码进行分析。

增加了`java.net.http`包，用于支持HTTP/1和HTTP/2以及websocket访问。

Java 9 移除了在 Java 8 中 被废弃的垃圾回收器配置组合，同时 把 G1 设为默认的垃圾回收器实现。另外，CMS 垃圾回收器已经被声明为废弃。Java 9 也增加了很多可以通过 jcmd 调用的诊断命令。

Java 9 允许在接口中使用私有方法。 在try-with-resources 语句中可以使用 effectively-final 变量。 

Java 9 把对 Unicode 的支持升级到了 8.0。

ResourceBundle 加载属性文件的默认编码从 ISO-8859-1 改成了 UTF-8，不再需要使用 native2ascii 命令来对属性文件进行额外处理。

注解`@Deprecated`增加了 `since` 和 `forRemoval` 两 个属性，可以分别指定一个程序元素被废弃的版本，以及是否会在今后的版本中被删除。

# Java 10

Java 10 发布于2018年3月21日。其中最备受广大开发者关注的莫过于局部变量类型推断。自Java 5引入泛型之后，Java 7 泛型实现了部分的类型推断，在声明变量时可以直接使用<>而不指定泛型。Java 8中lambda入参和返回值实际上也由类型推断来确定其类型。

## 局部变量类型推断

局部变量类型推断，允许开发人员省略一些不必要的局部变量类型初始化声明。降低语法冗余度的同时保持静态类型的安全性。其主要实现是引入了保留字var。但是这种var变量的类型推断同时也有局限，仅限于有初始化器的变量类型，增强for循环中的索引变量以及传统for循环中的局部变量。不能推断入参和返回值的类型，不能推断属性，和其他任何类型的变量声明。最为关键的一点，不使用类型推断会提供极大的程序可读性。

```java
var list = new ArrayList<String>(); // ArrayList<String>
var stream = list.stream(); // Stream<String>
```
## 垃圾收集器

目前的GC实现分布于不同的模块中，为了使所有的GC条理清晰易于拓展，需要整合并清理GC。Java 10 引入了纯净的GC接口，并将各种GC通用的部分重构并入了工具类部分，以更好的实现不同的GC隔离。

G1 收集器是Java 9 中hotspot的默认垃圾收集器，其通过一种尝试并行的策略降低full gc 出现的概率，但是尝试失败后触发垃圾收集器退回进行full gc。在之前的Java版本中G1进行垃圾回收时是基于单线程的标记压缩算法。Java 10 中引入了多线程并行GC，同时使用于年轻代回收和混合回收相同的并行工作线程数量，从而较少full gc 发生的概率，以提升性能和吞吐量。Java 10 中采用了并行的标记压缩算法。GC线程的数量可以通过`-XX：ParallelGCThreads`参数来进行调节。

## 应用程序类数据 (AppCDS) 共享

JDK 5 引入了数据共享机制，允许将JDK必须包通过共享文档的形式共享给不同的虚拟机实例，以使得后者在启动时不再重新加载，进而节约内存空间提升启动速度。Java 10 在此基础上允许引用类的共享。

在启动时记录加载类的过程，写入到文本文件中，再次启动时直接读取此启动文本并加载。如果应用环境没有大的变化，启动速度就会得到提升。整个过程类似于PC的休眠，将PC的运行环境写入到磁盘，再次使用时直接读取文件。

对于大型企业级应用以及微服务应用来说，其在启动时都会载入成千上万的类到应用程序类加载器。使用AppCDS会让这些服务快速启动并改善系统的响应时间。

## Graal GIT编译器

Java 10 新增了基于Java的GIT编译器Graal，使用 Java 9 中引入的JVM编译器接口（JVMCI）。并作为了linux/X64平台的实验性GIT编译器。

Graal是一个以Java为主要编程语言，主要面向Java字节码的编译器，与C++实现的C1、C2相比，其模块化更为明显，更容易维护。Graal可以作为动态编译器，也可以作为静态编译器实现AOT编译。Graal性能能与C++编译器匹敌，甚至有希望超过C++。

## 其他

`-XX:ThreadLocalHandshakes`参数开启了线程的握手功能，能允许在不运行全局安全点的时候实现线程回调。此功能能够对JVM的单个线程进行控制。提升JVM功能的性能开销。

扩展了Unicode语言标签。通过`java.time.format.DateTimeFormatter::localizedBy`方法，可以采用某种数字样式，区域定义或者时区来获得时间信息所需的语言地域本地环境信息。

当操作系统通过文件系统提供了使用非DRAM内存的方法的时候，JVM可以将堆内存分配在不同的备用设备上。通过` -XX：AllocateHeapAt = <path>`参数进行控制。

Java 9 命令`keytool -cacerts`命令可以查看根证书，但是Java 9 中没有添加任何证书。在Java 10 中根据Oracle 开源出Oracle JavaSE中的cacerts信息，在 OpenJDK 中提供80条默认的根证书颁发机构。

从Java 10开始，JDK发布频率调整为每半年发布一个大版本，每个季度发布一个中间特性版本。Java 10 中将重新编写之前 JDK 版本中引入的版本号方案，将使用基于时间模型定义的版本号格式来定义新版本。使用格式为

> \$FEATURE.\$INTERIM.\$UPDATE.\$PATCH

$FEATURE，每次版本发布加 1，不考虑具体的版本内容。

$INTERIM，中间版本号，在大版本中间发布的，包含问题修复和增强的版本，不会引入非兼容性修改。

# Java 11

Java 11 发布于2018年9月25日。是Java 8后又一个长期支持的Java版本。

## 标准HTTP Client升级

Java 11 对Java 9 引入的Http Client API 进行了优化，现在完全支持异步非阻塞。新版本的HTTP Client 通过`CompleteableFutures`实现的阻塞响应功能，并且可以联合使用触发相应的动作。并且吸收了响应式流的概念,其中广泛使用了Java Flow API。

提供了对 HTTP/2 等业界前沿标准的支持，同时也向下兼容 HTTP/1.1，精简而又友好的 API 接口，与主流开源 API（如：Apache HttpClient、Jetty、OkHttp 等）类似甚至拥有更高的性能。

```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
      .uri(URI.create("http://openjdk.java.net/"))
      .build();
client.sendAsync(request, BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println)
      .join();
```

## No-op垃圾收集器 Epsilon
Epsilon 垃圾回收器的目标是开发一个控制内存分配，但是不执行任何实际的垃圾回收工作（no-op）。它提供一个完全消极的 GC 实现，分配有限的内存资源，最大限度的降低内存占用和内存吞吐延迟时间。一旦堆内存耗尽，JVM就会退出。通过`-XX:+UseEpsilonGC`开启。

这种垃圾收集器适用于：

1. 性能测试。no-op则不会因为GC而引起性能损耗。
2. 内存压力测试。内存耗尽时候JVM会退出，可以帮助确定内存的阈值。
3. VM 接口测试。这是一个最简单的VM GC实现。
4. 短暂的job任务。短生命周期的任务退出时会直接销毁栈信息，此时不必进行常规GC。
5. 延迟改进。对于内存控制严格，几乎不存在垃圾回收的应用。常规GC会增加消耗，但是无意义。
6. 吞吐改进。传统GC分代工作，会存在锁。对于无需分配内存的应用，避免这个锁提升了性能。

## 可伸缩低延迟垃圾收集器ZGC

ZGC 是一个可伸缩的、低延迟的垃圾收集器，主要为了满足如下目标进行设计：

1. GC 停顿时间不超过 10ms
2. 既能处理几百 MB 的小堆，也能处理几个 TB 的大堆
3. 应用吞吐能力不会下降超过 15%（与 G1 回收算法相比）
4. 方便在此基础上引入新的 GC 特性和利用 colord
5. 针以及 Load barriers 优化奠定基础
6. 当前只支持 Linux/x64 位平台

10ms的停顿延迟实际上是堆ZGC的保守估计，即使是10ms也已经是传统GC不能达到的极限。根据 SPECjbb 2015 的基准测试，128G 的大堆下ZGC最大停顿时间才 1.68ms，远低于 10ms，和 G1 算法相比，改进非常明显。

![ZGC性能对比](./image001.png)

ZGC目前处于实验阶段，只支持Linux/x64平台，可以在编译虚拟机时候通过参数`--with-jvm-features=zgc`来在虚拟机中启用ZGC。并在虚拟机参数中增加`-XX：+ UnlockExperimentalVMOptions -XX：+ UseZGC -Xmx10g`启用ZGC。

## 增强 Java 启动器

增强Java启动器指的是Java 11 能够直接运行单个的Java文件，源码在内存中编译，然后由解释器执行。要求所有的相关类定义在一个Java文件中。通过命令

> java HelloWorld.java

可以直接运行源文件，这个命令相当于

>javac HelloWorld.java
>java -cp . hello.World

这个功能搭配JShell运行，对调试语言功能或者初学者非常友好。

## lambda与局部类型推断

在Java 10中还存在一条限制，局部类型推断不能用于lambda表达式。但是在Java 11中已经可以使用局部类型推断。这使得变量声明的方式可以统一。并且可以对lambda和隐式变量添加例如`@Nonnull`的注解。

其他方面，lambda表达式本身可以通过类型推断类确定变量类型，和使用`var`保留字的功能有覆盖。意义不大。

## 内存分析

Java 11 中提供一种低开销的 Java 堆分配采样方法，能够得到堆分配的 Java 对象信息，并且能够通过 JVMTI 访问堆信息。引入的新工具能够达到：
1. 长期开启而不会占用太多的内存消耗。
2. 只通过预定义的接口访问。
3. 能够对所有的堆区域进行采样。
4. 能给出Java对象信息。

对于用于来说在遇到高CPU，高内存占用的使用可以通过一些额外的工具类来进行堆内存分析，例如 VisualVM tools,新工具相对于这类第三方工具能够提供内存中对象的分配的具体信息，能够显示出发生高内存分配的确切位置，更容易定位问题。

## 其他

在内部类的处理方式上做了改进。在字节码方面引入了两个新的属性：一个叫做 `NestMembers` 的属性，用于标识其它已知的静态内嵌成员；另外一个是每个内嵌成员都包含的`NestHost`属性，用于标识出它的内嵌宿主类。

将商用特性“飞行记录器”开源，飞行记录器是一种低开销的事件信息收集框架，用于对JVM进行故障检测分析，其数据来源于应用程序，VM，OS。通过`-XX:StartFlightRecording`启用。

字节码层面增加了新的常量`CONSTANT_Dynamic`。其在初始化时候委托给 bootstrap 方法进行初始化创建对上层软件没有很大的影响，但是可以降低开发新形式的可实现类文件约束带来的成本和干扰。

# Java 12

Java 12 发布于2019年3月12日。

## 低停顿垃圾收集器 Shenandoah

Shenandoah是一个实验阶段的收集器，通过交换 CPU 并发周期和空间以改善停顿时间，使得垃圾回收器执行线程能够在 Java 线程运行时进行堆压缩，并且标记和整理能够同时进行。研发团队对外宣称，Shenandoah 垃圾回收器的暂停时间与堆大小无关。不过实际使用性能将取决于实际工作堆的大小和工作负载。

![Shenandoah](./Shenandoah.png)

## 增强switch

这是一个预览版的功能，并没有包含在Java SE标准里，预览指的是如果这个功能的反响并不好，则可能会被删除掉。本功能是第一个预览功能。

```java
int dayNumber = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY                -> 7;
    case THURSDAY, SATURDAY     -> 8;
    case WEDNESDAY              -> 9;
    default                      -> throw new IllegalStateException("Huh? " + day);
}
```

Java 12中取消了`break`,同时也不会造成穿透。并提供了将多个`case`合并到一行的新特性。标签右侧只能是表达式，代码块或者`throw`语句。为了保持兼容性，原先的":"依然可以使用，此时穿透会存在，并且不能和"->"混用。

## 改善G1收集器

G1收集器确定了回收集合后，一旦垃圾回收开始就必须处理完会收集合内的所有对象，一旦对象过于复杂或者对象过多，有可能在设定的暂停时间内无法完成垃圾回收工作。G1 垃圾收集器仅在进行完整 GC (Full GC) 或并发处理周期时才能将 Java 堆返回内存。由于 G1 回收器尽可能避免完整 GC，并且只触发基于 Java 堆占用和分配活动的并发周期，因此在许多情况下 G 1 垃圾回收器不能回收 Java 堆内存，除非有外部强制执行。

Java 12对G1收集器进行了改进，将回收集合拆分为必需和可选两部分，使 G1 垃圾回收器能中止垃圾回收过程。其中必需处理的部分包括 G1 垃圾收集器不能递增处理的 GC 回收集的部分（如：年轻代），同时也可以包含老年代以提高处理效率。将 GC 回收集拆分为必需和可选部分时，需要为可选 GC 回收集部分维护一些其他数据，这会产生轻微的 CPU 开销，但小于 1 ％的变化，同时在 G1 回收器处理 GC 回收集期间，本机内存使用率也可能会增加，使用上述情况只适用于包含可选 GC 回收部分的 GC 混合回收集合。

在G1收集器完成必须部分的垃圾回收后，根据剩余时间来决定是否终止回收工作，如果还有时间则进行可选部分的一个子集的垃圾回收直到可选部分完成。如果时间不足，则可以终止回收工作。

Java 12 中同时也对 G1 垃圾回收器进行了改进，使其能够在空闲时自动将 Java 堆内存返还给操作系统。为了尽可能的回收内存空间，G1收集器会记录堆内存的使用情况，将长期不使用的内存返还给操作系统。这种情况适用于长期空闲的应用程序。当应用程序没有处于活动状态，G1收集器根据以下两个条件决定是否开始回收内存空间：

1. 自上次垃圾回收已经过了`G1PeriodicGCInterval`毫秒（》0），并且此时没有正在进行的垃圾回收。
2. 通过`getloadavg()`方法查询系统的平均负载，如果一分钟之内系统的平均负载小于`G1PeriodicGCSystemLoadThreshold`且`G1PeriodicGCInterval`不为0.

如果两个条件都不满足，则再等待一个周期。如果`G1PeriodicGCInvokesConcurrent`设置了值，G1将继续上一个或者启动一个新的并发周期，如果没有设置，则G1会执行一个完整的GC。再每一个GC的末尾，G1将会调整堆内存大小，可能会将空闲内存返还给操作系统。

## 其他

Java 12 中添加一套新的基本的微基准测试套件，使开发人员可以轻松运行现有的微基准测试并创建新的基准测试。

在`java.lang.constant`包中增加了类，用来对关键类文件的名义描述进行建模。

Java 12 中将只保留一套 AArch64 实现，删除所有与 arm64 实现相关的代码，只保留 32 位 ARM 端口和 64 位 aarch64 的端口。删除此套实现将允许所有开发人员将目标集中在剩下的这个 64 位 ARM 实现上，消除维护两套端口所需的重复工作。

针对64位平台下的 JDK 构建过程进行了增强改进，使其默认生成类数据共享（CDS）归档，以进一步达到改进应用程序的启动时间的目的，同时也避免了需要手动运行：`-Xshare:dump` 的需要，修改后的 JDK 将在 lib/server 目录中保留构建时生成的 CDS 存档。

# Java 13

Java 13 发布于2019年9月13日。

## 动态AppCDS

Java 13 对Java 10 中引入的AppCDS进行了增强，允许在 Java 应用程序执行结束时动态进行类归档。在Java 10中，CDS必须要创建需要进行类归档的类列表，在Java 13中不再必须创建，可以使用`XX:ArchiveClassesAtExit`来控制应用程序在退出时候创建存档，或者使用`-XX:SharedArchiveFile`使用动态存档功能。

## 增强ZGC

ZGC 是Java 11中引入基于Linux的实验性质GC，使得应用程序的吞吐能力下降不超过15%。但是ZGC并不会将空闲内存返回给操作系统。ZGC堆由一系列的ZPage组成，在GC之后，被回收的ZPage会被返回到ZPageCache中，并按照LRU原则，将最长时间没有使用的ZPage返换给操作系统，并保证最小堆内存不低于设置。如果将最大最小堆内存设置为一样，则不返还内存空间。

除此之外Java 13 调整了ZGC使之最大支持的堆内存大小为16TB，并添加了新的参数`-XX:SoftMaxHeapSize`来软限制堆的最大大小，这个值应该不超过最大堆内存大小。它指的是，ZGC应该尽量将堆限制在`-XX:SoftMaxHeapSize`之下，但是也保留超出的能力。`-XX：-ZUncommit`显式关闭返还内存的功能。`-XX：ZUncommitDelay = <seconds>`（默认值为 300 秒）指定ZPage的最大空闲时间，超过此时间没有使用的空间将被返还。

## 重构Socket API

Java 13 为 Socket API 带来了新的底层实现方法，并且在 Java 13 中是默认使用新的 Socket 实现，引入 `NioSocketImpl` 的实现用以替换 SocketImpl 的 `PlainSocketImpl` 实现，此实现与 NIO实现共享相同的内部基础结构，并且与现有的缓冲区高速缓存机制集成在一起，因此不需要使用线程堆栈。使其易于发现并在排除问题同时增加可维护性。使用 `java.lang.ref.Cleaner` 机制来关闭套接字，以及在轮询时套接字处于非阻塞模式时处理超时操作等方面。

## switch 语句增强

在Java 12 的预览功能基础上进一步增强，仍然是一个预览功能。引入了`yield`关键字用来返回数据。也就是说在需要返回值的时候式用`yield`关键字，在不需要返回值的时候使用`break`。`yield`只会跳出当前switch语句块，并且需要由`default`语句。

```java
private static String getText(int number) {
    return switch (number) {
        case 1, 2:
            yield "one or two";
        case 3:
            yield "three";
        case 4, 5, 6:
            yield "four or five or six";
        default:
            yield "unknown";
    };
}
```

## 文本块

之前的Java中多行文本只能使用“+”拼接，并且需要处理转义字符，造成阅读的不便。Java 13 开始增加了文本块功能，借鉴python的多行文本，以“"""”开头，以“"""”结束的文本可以跨越多行，并且行中的所有特殊字符都被当作字符串处理。多行文本仍然是`String`的实例。

文本块是一个预览功能，可以通过以下命令启用：

> javac --enable-preview --release 13 Example.java
> java --enable-preview Example


## 其他

增加了`FileSystems.newFileSystem(Pth,Map)`方法，以便更轻松地使用将文件内容视为文件系统的文件系统提供程序。

`java.util`中的I18N将Unicode支持升级到12.1。