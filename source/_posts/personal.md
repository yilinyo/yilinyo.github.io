---
title: "personal"
date: "2026-03-31 10:01:02"
---
1. **目标冲刺2月中旬年后实习或则说3、4月春招 （大概50天）每天必须保证8小时学习时间（5小时必须是高效学习），分10个阶段吧，每5天一个阶段**

1. 考研结束，我的优势是**408计算机基础知识，学习能力、效率应该比较强**，应该加强能够将理论和抽象知识结合相互转换

1. 需要尽快回忆以前的知识，尤其是以前的java基础、项目、框架，（大概两到三天）

1. **八股文每天都要看，每2个阶段都要复盘模拟面试**

1. 新的项目也要跟上，这里优先级先分 自己毕业设计 **Java实现数据库、鱼皮API项目、鱼皮的在线OJ项目 ，我为什么要做这三个项目？第一个熟悉Java以及数据库底层实现，第二个是复习微服务，最后一个是一直想做的项目非硬性要求，没时间可以暂且搁置，这里希望做项目能够不仅仅是跟着视频敲一遍，要尽量自己思考并且给自己提问这个项目可能会被面试官怎么问**

1. **算法每天要写 2-4道，50天至少写120+道算法，要复盘**

1. **再加强一下多线程、redis、rabbitmq**

1. **看书，睡前或则休息时间看下技术书吧，最近在看数据库和JVM的书**

**1/9**

**1.**IO复习字节流 和 字符流 ，一般音频、图片、视频类文件就字节流读取比较快，而文字类文件采用字符流不容易产生乱码，字节流缓存区大小设置合理也能正确处理乱码问题。

2. synchronized 的实现机制反编译字节码   javap -c   monitorenter 和monitorexit

底层实现以前是通过操作系统同步互斥信号量，但后来改进使用偏向锁、轻量锁、重量锁，减少那个线程上下文切换，因为Java线程是通过虚拟机JVM 对操作系统内核级线程的1：1映射，切换开销较大

3. 抽象类和接口的区别，抽象类一般是提供模板，接口一般是一种方法规范，接口里一有public abstract 方法，java8后有默认方法。也有静态方法.抽象类里有protect方法也有public方法,有抽象方法也有非抽象方法，接口成员变量默认public static final，抽象类不一定。一个类可以实现多个接口但只能继承一个。抽象方法一定不可能被final static 修饰因为要被重写。、

4.JVM内存区域

线程安全（独立） 程序计数器、虚拟机栈（栈帧：局部变量表、信息、返回值、操作数栈））、本地方法栈

线程共享 ：堆（对象）、方法区（类信息、常量池）

5.事务隔离级别、并发问题（脏读、不可重复读、幻读）

6.线程构建

![](images/WEBRESOURCE4257bec3bbe30a3d5ec2144d79a017ceimage.png)

**1/10**

**1.常见排序的复习 快排、冒泡、以及简单插入排序**

**快排 平均nlogn，效率快慢取决于那个中枢元素的划分是否均等，**

**冒泡|插入有序情况比较合适，可以快速排序完很接近O（n）**

**2.协程**

由于协程切换是在线程内完成的，涉及到的资源比较少。不像内核级线程（进程）切换那样，上下文的内容比较多，切换代价较大。协程本身是非常轻巧的，可以简单理解为只是切换了寄存器和协程栈的内容。这样代价就非常小。

同步（Synchronised）和异步（Asynchronized）的概念描述的是应用程序与内核的交互方式，同步是指应用程序发起 I/O 请求后需要等待或者轮询内核 I/O 操作完成后才能继续执行；而异步是指应用程序发起 I/O 请求后仍继续执行，当内核 I/O 操作完成后会通知应用程序，或者调用应用程序注册的回调函数。阻塞和非阻塞

阻塞和非阻塞的概念描述的是应用程序调用内核 IO 操作的方式，阻塞是指 I/O 操作需要彻底完成后才返回到用户空间；而非阻塞是指 I/O 操作被调用后立即返回给用户一个状态值，无需等到 I/O 操作彻底完成。

3.HashMap和HashTable的区别

 	线程安全问题，key能不能为null，效率问题，默认值（不赋值map默认值16，table默认值11）扩容机制（map是二倍扩容，table是2n+1），结构（map是数组+链表+红黑树）那个Table（数组链表）

4.ps -ef | grep java   ps展示进程 |管道命令 链接后面的 grep查找方法实现进程查找的过滤

5.线程不能多次start 会报错

6.join 

![](images/WEBRESOURCEb2c2333487f260b358a8c17480057d3dimage.png)

使用join实现对另一个线程的同步，等待调用线程的结束才继续执行

7.算法循环不变量

![](images/WEBRESOURCE1489559df2ab4bc8a8eeec0826f9e888image.png)

二分查找的左闭右开的（或则左闭右闭）的循环不变量，排序算法的循环不变量

8. 

volatile 关键字修饰的只能是字段属性，它采取的是每次取值不直接从jmm内存模型的本地内存模型（一种对本地寄存器缓存的抽象）而是直接在内存获取值，这能够规避多个线程在自己各自工作空间访问同一共享变量造成一些读写不一致。保证了可见性，另外也禁止了指令重排，保证有序性。

并发三大特性就是可见性、原子性、有序性

原子性，**volatile 不能保证原子性**

原子性指的是一个或多个操作执行过程中不被打断的特性。被synchronized修饰的代码是具有原子性的，要么全部都能执行成功，要么都不成功。

**synchronized 保证了 有序性、可见性、原子性**

前面我们提到过，synchronized无论是修饰代码块还是修饰方法，本质上都是获取监视器锁monitor。获取了锁的线程就进入了临界区，锁释放之前别的线程都无法获得处理器资源，保证了不会发生时间片轮转，因此也就保证了原子性。

可见性

所谓可见性，就是指一个线程改变了共享变量之后，其他线程能够立即知道这个变量被修改。我们知道在Java内存模型中，不同线程拥有自己的本地内存，而本地内存是主内存的副本。如果线程修改了本地内存而没有去更新主内存，那么就无法保证可见性。

synchronized在修改了本地内存中的变量后，解锁前会将本地内存修改的内容刷新到主内存中，确保了共享变量的值是最新的，也就保证了可见性。

有序性

有序性是指程序按照代码先后顺序执行。

synchronized是能够保证有序性的。根据as-if-serial语义，无论编译器和处理器怎么优化或指令重排，单线程下的运行结果一定是正确的。而synchronized保证了单线程独占CPU，也就保证了有序性。

9.数据库的 四大特性ACID 原子性、一致性、隔离性、持久性、

原子性、 要么都成功 要么都失败 （体现）

一致性、前后数据合法性一致 （准则）

隔离性、多个事务 之间不互相影响 （前提）

持久性、一次事务提交就会永久有保存（目的）

**1/11 **

**1. ArraryList 的扩容机制 从10 -->1.5倍内部实现是由Arrays.copyOf方法完成新的数组复制**

**2. **interrupt  是动词，是一个操作进行打断打断方式

**打断阻塞进程**

打断 sleep，wait，join 的线程

这几个方法都会让线程进入阻塞状态

打断 sleep 的线程, 会清空打断状态，会致打断标记设置为假

**打断非阻塞进程**

会致打断标记设置为真




**Isinterrupt 返回是否打断  IsInterrupted 返回是否发打断并重置标志为false**

3.双指针 重中间向边界遍历一般可以可以用三个并行的while循环，如归并有序段

双指针向中间遍历 或合并一般用left <right 向中间靠拢，如二分、快排

4.**threadLocal 变量是用来操作每个线程Thread类中操作Threadlocalmap的工具类，一个threadlocal作为该线程中操作threadlocalmap的key值，然后存value，多个threadLocal 变量存多个变量。**

**threadlocal springmvc httpsession、httprequest等都存在其中，或则说如果我们如果要跨很多方法层传递信息数据，也可以直接存在这个ThreadLocalMap这个线程独享变量中，免去繁琐的参数传递**

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCE1792e496e265b14fdaa022a1a3c691ffimage.png)

5.类的加载 ， 加载，链接（验证（验证元素据、字节码是否安全）、准备（默认数据的一个先前准备、final修饰的一个赋值）、解析（将一些可以确定的符号引用转换为直接引用））、初始化（显示赋值、代码块等等）要把实例化的赋值区分开来，这不属于类的加载范畴

6.

COUNT(*)  对所有列包括null值进行统计

COUNT(1)  对所有列不包括null值进行统计

COUNT(列名)  对该列不包括null值进行统计

COUNT(DISTINC 列名)  对该列不包括null值且不同项进行统计

1.任何情况下SELECT COUNT(*) FROM tablename是最优选择;

2.尽量减少SELECT COUNT(*) FROM tablename WHERE COL = ‘value’ 这种查询;

3.杜绝SELECT COUNT(COL) FROM tablename WHERE COL2 = ‘value’ 的出现。

**在 MyISAM 引擎 中，每个表的总行数都会在内存和磁盘文件中进行保存，内存中的 count 变量值通过读取文件中的 count 值来进行初始化。当执行 count() 语句的时候，会直接将内存中保存的数值返回，所以执行非常快。而在InnoDB 引擎中，当执行 count() 的时候，它需要一行一行的进行统计和计数，并将最终的统计结果返回。**[**深入理解count()**](https://zhuanlan.zhihu.com/p/572387666)

7[having 和where区别](https://zhuanlan.zhihu.com/p/169737345)

having子句与where都是设定条件筛选的语句，有相似之处也有区别。

having与where的区别:

having是在分组后对数据进行过滤  HAVING子句中不能使用除了分组字段和聚合函数之外的其他字段.

where是在分组前对数据进行过滤

having后面可以使用聚合函数

where后面不可以使用聚合

在查询过程中执行顺序：**from>where>group（含聚合）>having>order>select。**


1/12

explain sql执行计划分析 

**type关键字** 的效率排序  system > const >eq_ref>ref>**range>index>all**

**Exindex condition （索引下推） using where（全部在回表过滤）**

![](images/WEBRESOURCE037b6129c598896c7b169b97f8e4977fimage.png)

![](images/WEBRESOURCE6966c8aad65833f15064ac49a08f2997image.png)

![](images/WEBRESOURCE049e588e6bb3a4820675496fd93182acimage.png)

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCE29f348aabf1b69aa68adc44c32e58c95image.png)

效率低

![](images/WEBRESOURCEb52aca8b9dd522b0b24183e43afcc93e截图.png)

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCEd6d6e8d0b253eec2aab354496cf1fcf8image.png)

1/13

Springboot 常见注解

[常见注解](https://zhuanlan.zhihu.com/p/395766827)

1、自动配置类 以及配置属性 及组件扫描

**@ComponentScan** 做的是扫描注解所在类的包及其子包 有注解@Component 的要备注IOC管理的bean

Springboot 的启动类

@SpringBootApplication 就有实现@ComponentScan ，只有主类包下的组件才会被加载进去，

当然也可以指定包名参数来指定扫描类

@ConfigurationProperties("yilinapi.client")

绑定配置文件的key-value，**批量绑定(@values一般是单个绑定）**结合以下依赖实现配置文件自动补全

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

springboot 的自动配置[springboot自动装配原理](https://blog.csdn.net/dongdong199033/article/details/131426590)

[EnableAutoConfiguration作用原理](https://cloud.tencent.com/developer/article/1435004)

2、Bean相关

@Bean 的用法


@Bean是一个方法级别上的注解，主要用在@Configuration注解的类里，也可以用在@Component注解的类里。添加的bean的id为方法名,你也可以使用name属性来指定

Bean的Scope   **@Scope**


Scope 描述的是Spring 容器如何新建Bean 的实例的。Spring 的Scope 有以下几种，通过@Scope 注解来实现。





1. Singleton ：一个Spring 容器中只有一个Bean 的实例，此为Spring 的默认配置，全容器共享一个实例。


2. Prototype ：每次调用新建一个Bean 的实例。


3. Request: Web 项目中，给每一个http request 新建一个Bean 实例。


4. Session: Web 项目中，给每一个http session 新建一个Bean 实例。


5. Global Session ：这个只在portal 应用中有用，给每一个global http session 新建一个Bean


实例。


6. 在Spring Batch 中还有一个Scope 是使用@StepScope








如果希望在启动的时候加载以后不在加载，使用单例模式，就是





@Scope("Singleton")  或者 @Scope,因为默认情况下就是 单例的

```java
@Configuration
public class MyConfiguration {

    @Bean
    @Scope("prototype")
    public Encryptor encryptor() {
        // ...
    }

}
```

1/14

1. 清空表表数据 最快的方式

TRUNCATE TABLE 这种直接把相关数据文件删了，然后重新分配页

DELEET 是遍历所有数据 一行一行删除比较慢

2。 double 初始化 赋值时 double = 1.2d d可以省略

float 初始化赋值 float =1.2f 必须加f

3.,Monitor对象本质是操作系统的管程，管程 同时只能运行一个线程进入用Obj对象头的markworf前20个bit指向唯一对应monitor实现上锁关联，没有获得锁的线程会在Monitor的等待队列中Entrylist阻塞

[java synchronize - 线程同步机制 - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000017134827)

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCE08d2cd3411615d9ee4128e4ad9ea3fe6image.png)

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCEf48ac8705a2fc2adefe478b2685da3d8image.png)

4. sdk 的 签名认证 accessKey表示 类似识别用户名 accessSecret

1/15

1. maven项目打包配置要配置主类 [maven配置打包模板](note://WEB3b33ba56f9cd18b254004021ccec8a9f)

2. 如果打成jar包读取rsourese下的文件应该用getResourceAsStream（）

```
JsonUtils.class.getClassLoader().getResourceAsStream(filepath)
```

文件字符流

```java
InputStreamReader isr = new InputStreamReader( JsonUtils.class.getClassLoader().getResourceAsStream(filepath));
int c;
while ((c = isr.read()) != -1)
    sb.append((char) c);

//这里要注意转char read读取是结果是int ascall码
```

3. 

假设表中有 id（主键）、name（二级索引）、age（年龄）

Order By 会索引失效吗？

** 会**，当 select ** from t order by name; 虽然name构建了二级索引，但是此时不会走索引，因为sql执行顺序是from ->where ->order by 这里查* 所以走了全表然后才根据 name来排序 using 会是file sort

要改成 select ** from t  where name =`jack`·order by name; 此时就不会索引失效 ，因为记住order by在后面执行所以一定要选中索引树 才能充分发挥索引有序性的特点减少排序消耗*

*select name，id from t where name like `%ack`； 一定不走索引吗，****不一定，虽然不满足最左匹配因为查询的字段恰好是二级索引的全部数据，相比全表扫描聚簇索引要付出代价更小（因为聚簇索引还有age、事务相关信息），会优化成走索引***

1/16

微服务架构 将业务功能拆分为 一个一个小模块，低耦合高内聚 ，减少高并发某个接口会影响其他接口

- 由于服务拆分，每个服务代码量大大减少，参与开发的后台人员在1~3名，协作成本大大降低

- 每个服务都是独立部署，当有某个服务有代码变更时，只需要打包部署该服务即可

- 每个服务独立部署，并且做好服务隔离，使用自己的服务器资源，不会影响到其它服务。

1/17

软件开发流程

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCE774be1605931d2a5a5daf62ae6175007IMG_3950.PNG)

动态代理

JMM

Java内存模型”（Java Memory Model简称JMM）来屏蔽各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果。Java内存模型是一种抽象的概念，并不真实存在，它描述的是一组规则或规范，通过这组规范定义了程序中各个变量（包括实例字段，静态字段和构成数组对象的元素）的访问方式。JMM是围绕原子性，有序性、可见性展开。

Java内存模型规定了所有的变量都存储在主内存（Main Memory）中（此处的主内存与介绍物理硬件时提到的主内存名字一样，两者也可以类比，但物理上它仅是虚拟机内存的一部分）。JVM运行程序的实体是线程，而每个线程创建时JVM都会为其创建一个工作内存(Working Memory，可与前面讲的处理器高速缓存类比)，用于存储线程私有的数据，线程的工作内存中保存了被该线程使用的变量的主内存副本，线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的数据。不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成。

**Java 内存模型其实是一种规范，定义了很多东西：**





所有的变量都存储在主内存（Main Memory）中。


每个线程都有一个私有的本地内存（Local Memory），本地内存中存储了该线程以读/写共享变量的拷贝副本。


线程对变量的所有操作都必须在本地内存中进行，而不能直接读写主内存。


不同的线程之间无法直接访问对方本地内存中的变量。

线程间的通信一般有两种方式进行，一是通过消息传递，二是共享内存。Java 线程间的通信采用的是共享内存方式，JMM 为共享变量提供了线程间的保障。

1/18

微服务架

网关的作

配置文件的共享、配置文件的热部署、动态路由

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCEd8011e6b91c39a3713af9a5116df1267image.png)

1/22

1. Mybatis-plus Mybatis 进阶 3天

1. redis进阶 3天

1. JVM

分布式事务 seata XA模式(等整个事务都执行完才一起提交，TC若发现有异常通知所有的事务回滚，串行化锁定资源时间性能慢）、TA模式（每个分支事务均会提交但会保存一个快照，若失败就用快照恢复，空间换时间）

[day05-服务保护和分布式事务 - Lark云文档 (feishu.cn)](https://b11et3un53m.feishu.cn/wiki/QfVrw3sZvihmnPkmALYcUHIDnff)

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCEab68f181f5753009619901f3405f08a3image.png)

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCE5895b922f7e9836137bd13dcb632724cimage.png)

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCE0cd77a8290a3f10ce5b741ee4ed0849cimage.png)

1/23

Redis 共有 5 种基本数据类型：String（字符串）、List（列表）、Set（集合）、Hash（散列）、Zset（有序集合）。

key一般都是字符串，value 是上面说的5大类型，

String类型应用，可以做token缓存、java类对象序列化缓存、Seesion

List可以存一些最新信息的集合

Hash可以存用户的基本信息等等

Set可以存不重复的不同用户量，点赞数等等，也可以利用集合操作统计共同好友（并集）、利用集合操作推荐可能认识的好友（差集）

Zset 设置权重可以做排行榜信息

**持久化**

- 快照（snapshotting，RDB） 保存数据快照到磁盘

- 只追加文件（append-only file, AOF）保存操作指令到磁盘

缓存问题

缓存穿透，无效的key即在缓存中没有，数据库也没有

解决方案

1.将无效key也放到缓存直接无效返回

2.布隆过滤器

缓存雪崩

缓存在同库造成了巨大的压力。

解决方案

缓存预热、设置随机失效时间

2/5

mq技术？发送者不同步调用 接口，而是通过将消息异步转发到消息中间件然后立即返回

解除了同步阻塞，可以消息群订阅复用 

同步调用的方式存在下列问题：

- 拓展性差

- 性能下降

- 级联失败

- 

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCE57d293534e1767beac63e276ac149c51image.png)

- publisher

：生产者，也就是发送消息的一方

- consumer

：消费者，也就是消费消息的一方

- queue

：队列，存储消息。生产者投递的消息会暂存在消息队列中，等待消费者处理

- exchange

：交换机，负责消息路由。生产者发送的消息由交换机决定投递到哪个队列。

- virtual host

：虚拟主机，起到数据隔离的作用。每个虚拟主机相互独立，有各自的exchange、queue

队列被消费者绑定后就直接消费了，只能被消费一次

绑定队列直接（不用交换机）

```java
@RabbitListener(queues = "simple.queue")
public void listenSimpleQueue(String msg){
    e.queue的消息：【" + msg +"】");
}
```

指定发送到队列

```
rabbitTemplate.convertAndSend(queueName, msg);
```

指定发送到交换机

```
rabbitTemplate.convertAndSend(exchangeName, null, msg); //第二个参数是routingkey
```

交换机绑定不同队列实现路由

将消息转发到不同队列 实现

- Fanout

![](/Users/yilin/Documents/ydynotes/博客/images/WEBRESOURCE3afee5337d1399b064d71a7345a33481image.png)

：广播，将消息交给所有绑定到交换机的队列。我们最早在控制台使用的正是Fanout交换机

- Direct

：订阅，基于RoutingKey（自定义路由key）发送给订阅了消息的队列

- Topic

：通配符订阅，与Direct类似，只不过RoutingKey可以使用通配符

#

匹配一个或多个词

*

匹配不多不少恰好1个词

- Headers

：头匹配，基于MQ的消息头匹配，用的较少。

申明队列和交换机

1.控制台声明（不推荐）

代码声明 一般在消费者声明

2.Configurition 声明 定义 Bean

3.注解声明最简单

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue1"),
    exchange = @Exchange(name = "hmall.direct", type = ExchangeTypes.DIRECT),
    key = {"red", "blue"}
))
public void listenDirectQueue1(String msg){
    System.out.println("消费者1接收到direct.queue1的消息：【" + msg + "】");
}

@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue2"),
    exchange = @Exchange(name = "hmall.direct", type = ExchangeTypes.DIRECT),
    key = {"red", "yellow"}
))
public void listenDirectQueue2(String msg){
    System.out.println("消费者2接收到direct.queue2的消息：【" + msg + "】");
}

```

2/7

rabbitmq可靠性

1.生产者可靠性

连接重连机制

ack机制

2.消费者可靠性

 mq持久化

mq ack机制类似事务，若抛出异常就重试就 设置ack nack reject 或者路由到一个错误交换机的错误队列

rabbitmq幂等性

多次消费同一消息结果要一样

1.令牌机制，为消息添加唯一uuid

消息转换器

[Spring事务传播行为详解（有场景） - 明叶师兄。 - 博客园 (cnblogs.com)](https://www.cnblogs.com/renxiuxing/p/15395798.html)

2/20

sql 语句子查询

in 是先查子表，存起来，然后A一个个去查，时间复杂度o(nm)。

exists是先查外表，再去看一个个存不存在，时间复杂度o(nb+树查询时间)

那么你可能会问，这样看exists肯定会比in快啊。等等，别着急，in查到的子表存到内存里了，exists去b+树中查还是查数据库，是基于磁盘的...

所以，如何选型呢？一般来讲用这种方式：

子表数据量比外表数据量少，使用in。

子表数据量比外表数据量大，使用exists。

子表与外表数据量大小差不多，用in与exists的效率相差不大。

jdk11 

1、垃圾回收器 默认改为G1垃圾回收器 减少用户线程中断，可以高效垃圾回收

2、var型字段 可推断

jdk21 虚拟线程 分代垃圾回收ZGC

### minorGC 和 Full GC区别

- 新生代 GC（Minor GC）:指发生新生代的的垃圾收集动作，Minor GC 非常频繁，回收速度一般也比较快。

- 老年代 GC（Major GC/Full GC）:指发生在老年代的 GC，出现了 Major GC 经常会伴随至少一次的 Minor GC（并非绝对），Major GC 的速度一般会比 Minor GC 的慢 10 倍以上。

### GC触发条件

Minor GC触发条件：Eden区满时

Full GC触发条件：

（1）调用System.gc时，系统建议执行Full GC，但是不必然执行

（2）老年代空间不足

（3）方法去空间不足

（4）通过Minor GC后进入老年代的平均大小大于老年代的可用内存

（5）由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小。