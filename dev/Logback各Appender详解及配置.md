#  Logback各Appender详解及配置

Logback将执行日志事件输出的组件称为Appender，实现的Appender必须继承 `ch.qos.logback.core.Appender 接口`

接口如下：

```java
package ch.qos.logback.core;
import ch.qos.logback.core.spi.ContextAware;
import ch.qos.logback.core.spi.FilterAttachable;
import ch.qos.logback.core.spi.LifeCycle;
public interface Appender<E> extends LifeCycle, ContextAware, FilterAttachable {
  public String getName();
  public void setName(String name);
  void doAppend(E event);
}
```



doAppende（E event） 方法的模板参数的真实类型根据logback module而定，在logback-classic中，E 为 ILoggingEvent 而在logback-access模块中，E 为 AcessEvent，doAppend可以说是logback框架最重要的部分，它负责将日志按照一定格式输出到指定设备。

Appender最终都会负责输出日志，但是他们也可能将日志格式化的工作交给Layout，或者Encoder对象。每一个Layout、Encoder只属于一个Appender。有些appender内置了固定的日志格式，所以并不需要layout/encoder声明。例如SockerAppender只是简单的序列化日志事件，然后就将它们通过网络传输。



**AppenderBase（** [`ch.qos.logback.core.AppenderBase`](http://logback.qos.ch/xref/ch/qos/logback/core/AppenderBase.html)**）**

AppenderBase是一个继承自Appender接口的抽象类，它是所有Logback目前提供的appender的父类。虽然是个抽象类，但是实际上实现了doAppender（）方法。

我们来看看它的doAppender方法的实现

```java
public **synchronized** void doAppend(E eventObject) {
  // prevent re-entry.
  if (guard) {
      return;
  }
  try {
      guard = true;
      if (!this.started) {
          if (statusRepeatCount++ < ALLOWED_REPEATS) {
              addStatus(new WarnStatus("Attempted to append to non started appender [" + name + "].",this));
          }
          return;
      }
      if (getFilterChainDecision(eventObject) == FilterReply.DENY) {
          return;
      }
      // ok, we now invoke the derived class's implementation of append
    this.append(eventObject);
  } finally {
    guard = false;
  }
}
```



可以看出doAppend方法被声明为同步级别。这使得了不同线程是用那个同一个appender的时候是安全的。当doAppend方法被一个线程访问时，后来的doAppend（）调用就必须等到该线程退出该方法才能被执行。

因为同步也并非对所有情况都适用，所以logback也提供了非同步的实现 [`ch.qos.logback.core.UnsynchronizedAppenderBase`](http://logback.qos.ch/xref/ch/qos/logback/core/UnsynchronizedAppenderBase.html)

下面讨论非同步的doAppend实现，源码查看超链接：

1. 首先，判断guard是否为真，如果为真，立刻退出，如果不为真，将在下一步被赋值为true，这样做是为了防止，在调用append（）方法之前可能采用相同的appender输出其他的日志，导致重复递归。

   **private** ThreadLocal<Boolean> guard = **new** ThreadLocal<Boolean>(); guard的类型是线程独享的对象

2. 然后，判断started是否为true，如果为false，则输出警告信息并返回。在完成配置之后，调用start（）方法检测相关属性配置，符合规范无异常则使该appender激活。如果appender有任何异常，将会在logback内部状态管理中记录警告信息。

3. 经过与之绑定的filter过滤器的一系列筛选，日志事件event可以被拒绝返回，也可以被接收，进入下一步。

4. 调用append（）

5. 设置guard=false，允许后来的doAppend（）函数调用。





下面呢！我们讲下Logback自带的Appender实现。



**OutputStreamAppender**

OutputStreamAppender  将日志事件附加到java.io.OutputStream中。这个类作为ConsoleAppender，FileAppender的父类，其中FileAppender又是``RollingFileAppender的父类。但一般来说，都不能通过实例化这个Appender。

第一种： [**ConsoleAppender**](http://logback.qos.ch/xref/ch/qos/logback/core/ConsoleAppender.html)

如同它的名字一样，这个Appender将日志输出到console，更准确的说是System.out 或者System.err。

它包含的参数如下：

| Property Name | Type                                                         | Description                                                  |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **encoder**   | [`Encoder`](http://logback.qos.ch/xref/ch/qos/logback/core/encoder/Encoder.html) | Determines the manner in which an event is written to the underlying `OutputStreamAppender`. Encoders are described in a [dedicated chapter](http://logback.qos.ch/manual/encoders.html). |
| **target**    | `String`                                                     | 指定输出目标。可选值：*System.out 或 System.err。默认值：System.out* |
| **withJansi** | `boolean`                                                    | 是否支持ANSI color codes（类似linux中的shell脚本的输出字符串颜色控制代码）。默认为false。如果设置为true。例如：[31m 代表将前景色设置成红色。在windows中，需要提供"org.fusesource.jansi:jansi:1.9"，而在linux，mac os x中默认支持。在Eclipse IDE中，你可以尝试[ANSI in Eclipse Console](http://www.mihai-nita.net/eclipse/) 插件。 |

看个简单的例子

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <!-- encoders are assigned the type ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
  <encoder>
      <pattern>%-4relative [%thread] %-5level %logger{35} - %msg %n</pattern>
  </encoder>
  </appender>
  <root level="DEBUG">
      <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

第二种： [**FileAppender**](http://logback.qos.ch/xref/ch/qos/logback/core/FileAppender.html)

将日志输出到文件当中，目标文件取决于file属性。是否追加输出，取决于append属性。

| Property Name | Type                                                         | Description                                                  |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **append**    | `boolean`                                                    | 是否以追加方式输出。默认为true。                             |
| **encoder**   | [`Encoder`](http://logback.qos.ch/xref/ch/qos/logback/core/encoder/Encoder.html) | See `OutputStreamAppender` properties.                       |
| **file**      | `String`                                                     | 指定文件名。注意在windows当中，反斜杠 \ 需要转义，或直接使用 / 也可以。例如 *c:/temp/test.log*or 或 *c:\\temp\\test.log 都可以。没有默认值，如果上层目录不存在，FileAppender会自动创建。* |
| **prudent**   | `boolean`                                                    | 是否工作在谨慎模式下。在谨慎模式下，FileAppender将会安全写入日志到指定文件，即时在不同的虚拟机jvm中有另一个相同的FileAppender实例。默认值：fales设置为true，意味着append会被自动设置成trueprudent依赖于文件排它锁。实验表明，使用文件锁，会增加3倍的日志写入消耗。比如说，当prudent模式为off，写入一条日志到文件只要10毫秒，但是prudent为真，则会接近30毫秒。prudent 模式实际上是将I/O请求序列化，因此在I/O数量较大，比如说100次/s或更多的时候，带来的延迟也会显而易见，所以应该避免。在networked file system（远程文件系统）中，这种消耗将会更大，可能导致死锁。 |



默认情况下，每一个log event都会被立即刷新到输出流。这种默认的做法对数据而言是比较安全的，可以避免因为应用程序异常退出，而导致缓冲区的日志丢失的风险。但是呢，为了加大日志吞吐量，你也可以将Encoder中的immediateFulsh属性设置成false。Encoder将在下一篇博文中说明。

下面是一个简单的例子：

```xml
<configuration>
  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>testFile.log</file>
    <append>true</append>
    <!-- encoders are assigned the type
        ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
    <encoder>
      <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="DEBUG">
    <appender-ref ref="FILE" />
  </root>
</configuration>
```

我们而已使用<timestamp>标签，获取应用启动时间，并作为新的日志文件的文件名。

```xml
<configuration>
  <!-- Insert the current time formatted as "yyyyMMdd'T'HHmmss" under
      the key "bySecond" into the logger context. This value will be
      available to all subsequent configuration elements. -->
  <timestamp key="bySecond" datePattern="yyyyMMdd'T'HHmmss"/>
  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <!-- use the previously created timestamp to create a uniquely
        named log file -->
    <file>log-${bySecond}.txt</file>
    <encoder>
      <pattern>%logger{35} - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="DEBUG">
    <appender-ref ref="FILE" />
  </root>
</configuration>
```

<timestamp>标签包含两个必要属性：key 和 datePattern 以及一个可选属性：timeReference。

- key是该属性的名字，作为variable的name
- datePattern需要符合SimpleDateFormat中的约定
- timeReference可以引用其他变量的值

```xml
<configuration>
  <timestamp key="bySecond" datePattern="yyyyMMdd'T'HHmmss" timeReference="contextBirth"/>
...
</configuration>
```

第三个： [**RollingFileAppender**](http://logback.qos.ch/xref/ch/qos/logback/core/rolling/RollingFileAppender.html)

RollingFileAppender继承自FileAppender，提供日志目标文件自动切换的功能。例如可以用日期作为日志分割的条件。

RollingFileAppender有两个重要属性，RollingPolicy负责怎么切换日志，TriggeringPolicy负责何时切换。为了使RollingFileAppender起作用，这两个属性必须设置，但是如果RollingPolicy的实现类同样实现了TriggeringPolicy接口，则也可以只设置RollingPolicy这个属性。

下面是它的参数：

| **Property Name**    | Type                                                         | Description                                                  |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **file**             | `String`                                                     | 指定文件名。注意在windows当中，反斜杠 \ 需要转义，或直接使用 / 也可以。例如 *c:/temp/test.log*or 或 *c:\\temp\\test.log 都可以。没有默认值，如果上层目录不存在，FileAppender会自动创建。* |
| **append**           | `boolean`                                                    | 是否以追加方式输出。默认为true。                             |
| **encoder**          | [`Encoder`](http://logback.qos.ch/xref/ch/qos/logback/core/encoder/Encoder.html) | See `OutputStreamAppender` properties.                       |
| **rollingPolicy**    | `RollingPolicy`                                              | 当发生日志切换时，RollingFileAppender的切换行为。例如日志文件名的修改 |
| **triggeringPolicy** | `TriggeringPolicy`                                           | 决定什么时候发生日志切换，例如日期，日志文件大小到达一定值   |
| **prudent**          | `boolean`                                                    | [`FixedWindowRollingPolicy`](http://logback.qos.ch/manual/appenders.html#FixedWindowRollingPolicy) 不支持prudent模式。`TimeBasedRollingPolicy 支持prudent模式，但是需要满足一下两条约束：`在prudent模式中，日志文件的压缩是不被允许，不被支持的。不能设置file属性。 |

我们来看一下RollingPolicy是什么：

RollingPolicy实际上就是负责日志文件的切换以及重命名的。

接口如下：

```java
package ch.qos.logback.core.rolling;
import ch.qos.logback.core.FileAppender;
import ch.qos.logback.core.spi.LifeCycle;
public interface RollingPolicy extends LifeCycle {
  /*rollover方法完成当前日志文件归档的工作 */
  public void rollover() throws RolloverFailure;
  /*getActiveFileName方法负责计算新的日志文件名*/
  public String getActiveFileName();
  /*getCompressionMode方法负责压缩模式的工作*/
  public CompressionMode getCompressionMode();
  /*设置父appender*/
  public void setParent(FileAppender appender);
}
```

RollingPolicy有几个常见的实现类：

**TimeBasedRollingPolicy**

```
TimeBasedRollingPolicy 也许是最受欢迎的日志滚动策略。它的滚动策略是基于时间的，例如根据天数，月份。
```

TimeBasedRollingPolicy继承了RollingPolicy和TriggeringPolicy接口。

它包含一个必需的属性：fileNamePattern 以及若干个可选属性。

| Property Name           | Type     | Description                                                  |
| ----------------------- | -------- | ------------------------------------------------------------ |
| **fileNamePattern**     | `String` | 这个必需的属性，决定了日志滚动时，归档日志的命名策略。它由文件名，以及一个%d转移符组成。%d{}花括号中需要包含符合SimpleDateFormat约定的时间格式，如果未指定，直接是%d,则默认相当于%d{yyyy-MM-dd}。需要注意的是，在RollingPolicy节点的父节点appender节点中，<file>节点的值可以显示声明，或忽略。如果声明file属性，你可以达到分离当前有效日志文件以及归档日志文件的目的。设置成之后，当前有效日志文件的名称永远都是file属性指定的值，当发生日志滚动时，再根据fileNamePattern的值更改存档日志的名称，然后创建一个新的有效日志文件，名为file属性指定的值。如果不指定，则当前有效日志文件名根据fileNamePattern变更。同样需要注意的是，在%d{}中，不管是“/”还是“\”都被认为是文件分隔符。 **多个%d转移符的情况：**fileNamePaatern的值允许包含多个%d的情况，但是只有一个%d作为主要的日志滚动周期的参考值。其余非主要的%d需要包含一个"aux"的参数。例如：<fileNamePattern>/var/log/%d{yyyy/MM, aux}/myapplication.%d{yyyy-MM-dd}.log</fileNamePattern>根据年月划分目录，再将相应月份按日期天数命名的归档日志存放在一起同一个月份文件夹当中。该属性值决定在每天0点的时候发生日志切换。 **时区问题：**你可以将日期转换成相应时区的时间。例如aFolder/test.%d{yyyy-MM-dd-HH, UTC}.log  //世界协调时间aFolder/test.%d{yyyy-MM-dd-HH, GMT}.log  //格林尼治时间 |
| **maxHistory**          | int      | 可选参数，声明归档日志最大保留时间。如果你是基于月份的日志滚动，则当maxHisory为6时，说明会保留6个月的日志。大于6个月的就会被删除。日志所存在的目录也会被合适的删除掉。 |
| **totalSizeCap**        | int      | 可选参数，声明归档日志的最大存储量。当超过这个值，最老的归档日志文件也会被删除。 |
| **cleanHistoryOnStart** | boolean  | 可选参数，默认为false。如果设置为true，则当appender启动时，会删除所有归档日志文件。 |
|                         |          |                                                              |



下面是一些常见的fileNamePattern的值：

| fileNamePattern                            | Rollover schedule                                | Example                                                      |
| ------------------------------------------ | ------------------------------------------------ | ------------------------------------------------------------ |
| */wombat/foo.%d*                           | 每天午夜滚动日志，未指定，默认为%d{yyyy-MM-dd}   | file属性未设置的情况下：在2016年7月17日，日志会输出到/wombat/foo.2016-07-17日志文件中，在午夜24点，当前有效日志文件会被转换成/wombat/foo.2016-07-18 file属性设置为/wombat/foo.txt的情况下：在2016年7月17日，日志会输出到/wombat/foo.txt的日志文件中，在午夜24点，日志文件被归档，并重命名为/wombat/foo.2016-07-18, 当前有效日志文件重新创建，并命名为/wombat/foo.txt |
| */wombat/%d{yyyy/MM}/foo.txt*              | 每月一更                                         | file属性未设置的情况下：在2016年7月，日志输出到到/wombat/2016/07/foo.txt。在7月31号午夜24点，日志重定向输出到/wombat/2016/08/foo.txt。 file属性设置为/wombat/foo.txt的情况下：当前有效日志文件为file指定值，在月末24点，归档日志，并重新创建file指定的文件，并将日志输出流重定向到这个新文件。 |
| */wombat/foo.%d{yyyy-ww}.log*              | 每星期一更，需要注意，每星期第一天与区域设置有关 | 与上类似                                                     |
| */wombat/foo%d{yyyy-MM-dd_HH}.log*         | 每小时一更                                       | 与上类似                                                     |
| */wombat/foo%d{yyyy-MM-dd_HH-mm}.log*      | 每分钟一更                                       | 与上类似                                                     |
| */wombat/foo%d{yyyy-MM-dd_HH-mm, UTC}.log* | 每分钟一更，并且采用UTC时间                      | 与上类似，除了时间被格式化成UTC时间                          |
| */foo/%d{yyyy-MM,\**aux**}/%d.log*         | 每日一更                                         | 包含两个%d，第一个包含aux说明是并非主要的滚动参数，主要是用来做文件夹分割。也就是讲每天的日志按月份划分存储在不同的路径下 |



TimeBaseRollingPolicy 支持自动压缩日志文件，这个功能通过设置fileNamePattern的值以 .gz 或者 .zip 结尾开启。

| fileNamePattern     | Rollover schedule                | Example                                                      |
| ------------------- | -------------------------------- | ------------------------------------------------------------ |
| */wombat/foo.%d.gz* | 每日一更，自动压缩并归档日志文件 | file属性未设置的情况下：在2016年7月17日，日志输出到 /wombat/foo.2016-07-17，在午夜24点，日志文件会被压缩并重新命名为/wombat/foo.2016-07-17.gz。之后日志便重新定向到/wombat/foo.2016-07-18 file属性被设置成/wombat/foo.txt的情况下：当前有效日志文件名永远不变，在每日0点滚动日志，并压缩日志。 |



由于多种原因，日志的滚动并不是**时钟驱动**的，而是**依赖于到来的日志事件**。如果将fileNamePattern设置成yyy-MM-dd，如果24点后，第一条日志事件请求发生在00：30，则真实的日志滚动发生事件发生在00点：30分。然而这种延迟的实现，是可以忽略的。



下面是一个简单的配置例子：

<configuration>

  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">

​    <file>logFile.log</file>

​    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">

​      <!-- daily rollover -->

​      <fileNamePattern>logFile.%d{yyyy-MM-dd}.log</fileNamePattern>



​      <!-- keep 30 days' worth of history capped at 3GB total size -->

​      <maxHistory>30</maxHistory>

​      <totalSizeCap>3GB</totalSizeCap>



​    </rollingPolicy>



​    <encoder>

​      <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>

​    </encoder>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="FILE" />

  </root>

</configuration>





**SizeAndTimeBasedRollingPolicy**

有时候你不仅想通过时间来规定滚动策略，还希望同时限制每个日志文件的大小。在TimeBasedRoolingPolicy中已经提供限制总日志文件的大小的功能，而SizeAndTimeBasedRollingPolicy提供了更为强大的，针对单个日志文件的大小限制能力。

看个小例子：

<configuration>

  <appender name="ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">

​    <file>mylog.txt</file>

​    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">

​      <!-- rollover daily -->

​      <fileNamePattern>mylog-%d{yyyy-MM-dd}.%i.txt</fileNamePattern>

​      <!-- each file should be at most 100MB, keep 60 days worth of history, but at most 20GB -->

​      <maxFileSize>100MB</maxFileSize>

​      <maxHistory>60</maxHistory>

​      <totalSizeCap>20GB</totalSizeCap>

​    </rollingPolicy>

​    <encoder>

​      <pattern>%msg%n</pattern>

​    </encoder>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="ROLLING" />

  </root>



</configuration>

需要注意的是：在该例子中，fileNamePattern中不仅包含了%d，还包含了%i，这两个都是必要的标识符。%i代表日志索引号。就是今天的日志已经拓展到第几份了，以0开始。

在1.1.7版本之前，使用的是SizeAndTimeBasedFNATP，之后就采用SizeAndTimeBasedRollingPolicy，SizeAndTimeBasedRollingPolicy继承了SizeAndTimeBasedFNATP。





**FixedWindowRollingPolicy 固定窗口的日志滚动策略**

```
FixedWindowRollingPolicy 
```

看下参数先：

| Property Name       | Type     | Description                                                  |
| ------------------- | -------- | ------------------------------------------------------------ |
| **minIndex**        | `int`    | 这个参数指定窗口索引的最小值                                 |
| **maxIndex**        | `int`    | 这个参数指定窗口索引的最大值                                 |
| **fileNamePattern** | `String` | 这个参数与之前的fileNamePattern没什么差别，唯一需要注意的是必须包含%i标识符，这个标识符的作用是指明当前窗口索引的值。例如将fileNamePattern设置成 "*MyLogFile%i* ",minIndex为1，maxIndex为3，则会创建*MyLogFile1.log*, *MyLogFile2.log* and *MyLogFile3.log*.这三个归档日志文件。 |



看个小例子：

<configuration>

  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">

​    <file>test.log</file>



​    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">

​      <fileNamePattern>tests.%i.log.zip</fileNamePattern>

​      <minIndex>1</minIndex>

​      <maxIndex>3</maxIndex>

​    </rollingPolicy>



​    <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">

​      **<maxFileSize>5MB</maxFileSize>**

​    </triggeringPolicy>

​    <encoder>

​      <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>

​    </encoder>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="FILE" />

  </root>

</configuration>



接下来我们再来看下 triggering policies的接口

```
TriggeringPolicy 负责RollingFileAppender何时发生日志滚动
```

接口说明：

package ch.qos.logback.core.rolling;



import java.io.File;

import ch.qos.logback.core.spi.LifeCycle;



public interface TriggeringPolicy<E> extends LifeCycle {



​    /*该方法决定是否发生日志滚动*/

  public boolean isTriggeringEvent(final File activeFile, final <E> event);

}



**SizeBasedTriggeringPolicy**

```
SizeBasedTriggeringPolicy 检测当前活动日志文件的大小，当到达一定容量，就开启日志滚动。
```

SizeBasedTriggeringPolicy只接收一个参数，maxFileSize，默认值：10MB

可接受B，KB，MB，GB







**SocketAppender及SSLSocketAppender**

到目前为止我们讲的appender都只能将日志输出到本地资源。与之相对的，SocketAppender就是被设计用来输出日志到远程实例中的。SocketAppender输出日志采用明文方式，SSLSocketAppender则采用加密方式传输日志。

被序列化的日志事件的类型是 `LoggingEventVO 继承ILoggingEvent接口。远程日志记录并非是侵入式的。在反序列化接收后，日志事件就可以好像在本地生成的日志一样处理了。多个SockerAppender可以向同一台日志服务器发送日志。SocketAppender并不需要关联一个Layout，因为它只是发送序列化的日志事件给远程日志服务器。SocketAppender的发送操作是基于TCP协议的。因此如果远程服务器是可到达的，则日志会被其处理，如果远程服务器宕机或不可到达，那么日志将会被丢弃。等到远程服务器复活，日志发送将会透明的重新开始。这种透明式的重连，是通过一个“连接“线程周期性的尝试连接远程服务器实现的。`

Logging events会由TCP协议实现自动缓冲。这意味着，如果网络速度比日志请求产生速度快，则网络速度并不会影响应用。但如果网络速度过慢，则网络速度则会变成限制，在极端情况下，如果远程日志服务器不可到达，则会导致应用最终阻塞。不过，如果服务器可到达，但是服务器宕机了，这种情况，应用不会阻塞，而只是丢失一些日志事件而已。

需要注意的是，即使SocketAppender没有被logger链接，它也不会被gc回收，因为他在connector thread中任然存在引用。一个connector thread 只有在网络不可达的情况下，才会退出。为了防止这个垃圾回收的问题，我们应该显示声明关闭SocketAppender。长久存活并创建/销毁大量的SocketAppender实例的应用，更应该注意这个问题。不过大多数应用可以忽略这个问题。如果JVM在SocketAppender关闭之前将其退出，又或者是被垃圾回收，这样子可能导致丢失一些还未被传输，在管道中等待的日志数据。为了防止避免日志丢失，经常可靠的办法就是调用SocketAppender的close方法，或者调用LoggerContext的stop方法，在退出应用之前。



下面我们来看看SocketAppender的属性：

| Property Name         | Type               | Description                                                  |
| --------------------- | ------------------ | ------------------------------------------------------------ |
| **includeCallerData** | `boolean`          | 是否包含调用者的信息如果为true，则以下日志输出的 ?:? 会替换成调用者的文件名跟行号，为false，则为问号。2006-11-06 17:37:30,968 DEBUG [Thread-0] [?:?] chapters.appenders.socket.SocketClient2 - Hi |
| **port**              | `int`              | 端口号                                                       |
| **reconnectionDelay** | `Duration`         | 重连延时，如果设置成“10 seconds”，就会在连接u武器失败后，等待10秒，再连接。默认值：“30 seconds”。如果设置成0，则关闭重连功能。 |
| **queueSize**         | `int`              | 设置缓冲日志数，如果设置成0，日志发送是同步的，如果设置成大于0的值，会将日志放入队列，队列长度到达指定值，在统一发送。可以加大服务吞吐量。 |
| **eventDelayLimit**   | `Duration`         | 设置日志超时丢弃时间。当设置“10 seconds”类似的值，如果日志队列已满，而服务器长时间来不及接收，当滞留时间超过10 seconds，日志就会被丢弃。默认值： 100 milliseconds |
| **remoteHost**        | `String`           | 远程日志服务器的IP                                           |
| **ssl**               | `SSLConfiguration` | 只在SSLSocketAppender包含该属性节点。提供SSL配置，详情见 [Using SSL](http://logback.qos.ch/manual/usingSSL.html). |



标准的Logback Classic包含四个可供使用的Receiver用来接收来自SocketAppender的logging evnets。

- ServerSocketReceiver，以及允许SSL的副本SSLSeverSocketReceiver。详细配置查看 [Receivers](http://logback.qos.ch/manual/receivers.html) 
- SimpleSocketServer，以及允许SSL的副本SimpleSSLSocketServer。



在服务端使用 SimpleSocketServer

SimpleSocketServer需要两个命令行参数，port 和 configFile路径。

java ch.qos.logback.classic.net.SimpleSocketServer 6000 \ src/main/java/chapters/appenders/socket/server1.xml



客户端的SocketAppender的简单配置例子：

<configuration>



  <appender name="SOCKET" class="ch.qos.logback.classic.net.SocketAppender">

​    <remoteHost>192.168.0.101</remoteHost>

​    <port>8888</port>

​    <reconnectionDelay>10000</reconnectionDelay>

​    <includeCallerData>true</includeCallerData>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="SOCKET" />

  </root>



</configuration>



在服务端使用SimpleSSLSocketServer

java -Djavax.net.ssl.keyStore=src/main/java/chapters/appenders/socket/ssl/keystore.jks \ -Djavax.net.ssl.keyStorePassword=changeit \ ch.qos.logback.classic.net.SimpleSSLSocketServer 6000 \ src/main/java/chapters/appenders/socket/ssl/server.xml



SSLSocketAppender配置

<configuration debug="true">



  <appender name="SOCKET" class="ch.qos.logback.classic.net.SSLSocketAppender">

​    <remoteHost>${host}</remoteHost>

​    <port>${port}</port>

​    <reconnectionDelay>10000</reconnectionDelay>

​    <ssl>

​      <trustStore>

​        <location>${truststore}</location>

​        <password>${password}</password>

​      </trustStore>

​    </ssl>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="SOCKET" />

  </root>



</configuration>





**ServerSocketAppender及SSLSeverSocketAppender**

前面说的SocketAppender以及SSLSockertAppender的设计理念是：让应用主动连接远程日志服务器。但有时候让应用在初始化的时候就连接日志服务器可能是不方便或者不科学的。所以Logback也提供ServerSocketAppender以及SSLServerSocketAppender。SocketAppender与ServerSocketAppender序列化Logging event使用的是同样的方法，序列化的对象都是ILoggingEvent。唯一不同的就是连接初始化的主从转换。SocketAppender作为与日志服务器建立连接的主动方，而ServerSocketAppender是被动的，它监听来自客户端的连接请求。



ServerSocketAppender属性表：

| Property Name         | Type               | Description                                                  |
| --------------------- | ------------------ | ------------------------------------------------------------ |
| **address**           | `String`           | 指定appender监听那个本地网络接口地址，如果不指定，则监听所有网络接口 |
| **includeCallerData** | `boolean`          | 如果为true，caller data 会传送给远程主机。默认不传送         |
| **port**              | `int`              | 监听那个端口                                                 |
| **ssl**               | `SSLConfiguration` | 该属性只被SSLServerSocketAppender支持，提供SSL配置           |



看个小例子：

<configuration debug="true">

  <appender name="SERVER"

​    class="ch.qos.logback.classic.net.server.ServerSocketAppender">

​    <port>${port}</port>

​    <includeCallerData>${includeCallerData}</includeCallerData>

  </appender>



  <root level="debug">

​    <appender-ref ref="SERVER" />

  </root>



</configuration>

这个配置与上一个SocketAppender的配置主要的差别是缺少了<remoteHost>属性，因为ServerSocketAppender并不是主动打开与日志服务器的连接，而是被动地等待来自远程主机的连接请求。

下面是SSLSocketAppender的配置例子：

<configuration debug="true">

  <appender name="SERVER"

​    class="ch.qos.logback.classic.net.server.SSLServerSocketAppender">

​    <port>${port}</port>

​    <includeCallerData>${includeCallerData}</includeCallerData>

​    <ssl>

​      <keyStore>

​        <location>${keystore}</location>

​        <password>${password}</password>

​      </keyStore>

​    </ssl>

  </appender>



  <root level="debug">

​    <appender-ref ref="SERVER" />

  </root>



</configuration>





说完Socket的日志输出，我们再来看看基于邮件的日志输出

**SMTPAppender**

[`SMTPAppender`](http://logback.qos.ch/xref/ch/qos/logback/classic/net/SMTPAppender.html) 可以将logging event存放在一个或多个固定大小的缓冲区中，然后在用户指定的event到来之时，将适当的大小的logging event以邮件方式发送给运维人员。

详细属性如下：

| Property Name           | Type                                                         | Description                                                  |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **smtpHost**            | `String`                                                     | SMTP server的地址，必需指定。如网易的SMTP服务器地址是： smtp.163.com |
| **smtpPort**            | `int`                                                        | SMTP server的端口地址。默认值：25                            |
| **to**                  | `String`                                                     | 指定发送到那个邮箱，可设置多个<to>属性，指定多个目的邮箱     |
| **from**                | `String`                                                     | 指定发件人名称。如果设置成“Adam Smith &lt;smith@moral.org&gt; ”，则邮件发件人将会是“Adam Smith <smith@moral.org> ” |
| **subject**             | `String`                                                     | 指定emial的标题，它需要满足PatternLayout中的格式要求。如果设置成“Log: %logger - %msg”，就案例来讲，则发送邮件时，标题为“Log: com.foo.Bar - Hello World ”。默认值："%logger{20} - %m". |
| **discriminator**       | `Discriminator`                                              | 通过Discriminator, SMTPAppender可以根据Discriminator的返回值，将到来的logging event分发到不同的缓冲区中。默认情况下，总是返回相同的值来达到使用一个缓冲区的目的。 |
| **evaluator**           | `IEvaluator`                                                 | 指定触发日志发送的条件。通过<evaluator class=... />指定EventEvaluator接口的实现类。默认情况下SMTPAppeender使用的是OnErrorEvaluator，表示当发送ERROR或更高级别的日志请求时，发送邮件。Logback提供了几个evaluators：OnErrorEvaluator[`OnMarkerEvaluator`](http://logback.qos.ch/xref/ch/qos/logback/classic/boolex/OnMarkerEvaluator.html)[`JaninoEventEvaluator`](http://logback.qos.ch/xref/ch/qos/logback/classic/boolex/JaninoEventEvaluator.html)`GEventEvaluator（功能强大）` |
| **cyclicBufferTracker** | [`CyclicBufferTracker`](http://logback.qos.ch/xref/ch/qos/logback/core/spi/CyclicBufferTracker.html) | 指定一个cyclicBufferTracker跟踪cyclic buffer。它是基于discriminator的实现的。如果你不指定，默认会创建一个[CyclicBufferTracker](http://logback.qos.ch/xref/ch/qos/logback/core/spi/CyclicBufferTracker.html) ，默认设置cyclic buffer大小为256。你也可以手动指定使用默认的CyclicBufferTracker，并且通过<bufferSize>属性修改默认的缓冲区接收多少条logging event。 |
| **username**            | `String`                                                     | 发送邮件账号，默认为null                                     |
| **password**            | `String`                                                     | 发送邮件密码，默认为null                                     |
| **STARTTLS**            | `boolean`                                                    | 如果设置为true，appender会尝试使用STARTTLS命令，如果服务端支持，则会将明文连接转换成加密连接。需要注意的是，与日志服务器连接一开始是未加密的。默认值：false |
| **SSL**                 | `boolean`                                                    | 如果设置为true，appender将会使用SSL连接到日志服务器。默认值：false |
| **charsetEncoding**     | `String`                                                     | 指定邮件信息的编码格式默认值：UTF-8                          |
| **localhost**           | `String`                                                     | 如果smtpHost没有正确配置，比如说不是完整的地址。这时候就需要localhost这个属性提供服务器的完整路径（如同java中的完全限定名 ），详情参考[com.sun.mail.smtp](http://javamail.kenai.com/nonav/javadocs/com/sun/mail/smtp/package-summary.html) 中的mail.smtp.localhost属性 |
| **asynchronousSending** | `boolean`                                                    | 这个属性决定email的发送是否是异步。默认：true，异步发送但是在某些情况下，需要以同步方式发送错误日志的邮件给管理人员，防止不能及时维护应用。 |
| **includeCallerData**   | `boolean`                                                    | 默认：false指定是否包含callerData在日志中                    |
| **sessionViaJNDI**      | `boolean`                                                    | SMTPAppender依赖javax.mail.Session来发送邮件。默认情况下，sessionViaJNDI为false。javax.mail.Session实例的创建依赖于SMTPAppender本身的配置信息。如果设置为true，则Session的创建时通过JNDI获取引用。这样做的好处可以让你的代码复用更好，让配置更简洁。需要注意的是，如果使用JNDI获取Session对象，需要保证移除mail.jar以及activation.jar这两个jar包 |
| **jndiLocation**        | `String`                                                     | 如果sessionViaJNDI设置为true，则jndiLocation指定JNDI的资源名，默认值为："java:comp/env/mail/Session" |



SMTPAppender只保留最近的256条logging events 在循环缓冲区中，当缓冲区慢，就会开始丢弃最老的logging event。因此不管什么时候，SMTPAppender一封邮件最多传递256条日志事件。SMTPAppender依赖于JavaMail API。而JavaMail API又依赖于IOC框架（依赖注入）。

原文：

 You can download the [JavaMail API](http://java.sun.com/products/javamail/) and the [JavaBeans Activation Framework](http://java.sun.com/beans/glasgow/jaf.html) from their respective websites. Make sure to place these two jar files in the classpath before trying the following examples.



看个小例子

<configuration>

  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">

​    <smtpHost>ADDRESS-OF-YOUR-SMTP-HOST</smtpHost>

​    <to>EMAIL-DESTINATION</to>

​    <to>ANOTHER_EMAIL_DESTINATION</to> <!-- additional destinations are possible -->

​    <from>SENDER-EMAIL</from>

​    <subject>TESTING: %logger{20} - %m</subject>

​    <layout class="ch.qos.logback.classic.PatternLayout">

​      <pattern>%date %-5level %logger{35} - %message%n</pattern>

​    </layout>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="EMAIL" />

  </root>

</configuration>



自定义循环缓冲区大小：考虑到256条日志的默认大小，可能并不适用，我们也可以自定义。

<configuration>

  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">

​    <smtpHost>${smtpHost}</smtpHost>

​    <to>${to}</to>

​    <from>${from}</from>

​    <subject>%logger{20} - %m</subject>

​    <layout class="ch.qos.logback.classic.html.HTMLLayout"/>



​    <cyclicBufferTracker class="ch.qos.logback.core.spi.CyclicBufferTracker">

​      <!-- send just one log entry per email -->

​      <bufferSize>1</bufferSize>

​    </cyclicBufferTracker>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="EMAIL" />

  </root>

</configuration>



触发邮件日志发送的事件：默认情况下，Evaluator指定的是OnErrorEvaluator，表明当发生level级别为error或更高的事件时，就会触发日志发送。

我们也可以编写自己的日志触发器，只要继承自EventEvaluator接口即可。主要实现evaluete方法。下面是个例子：

作用是：每隔1024个日志事件，触发日志发送请求。

package chapters.appenders.mail;



import ch.qos.logback.core.boolex.EvaluationException;

import ch.qos.logback.core.boolex.EventEvaluator;

import ch.qos.logback.core.spi.ContextAwareBase;



public class CounterBasedEvaluator extends ContextAwareBase implements EventEvaluator {



  static int LIMIT = 1024;

  int counter = 0;

  String name;



  public boolean evaluate(Object event) throws NullPointerException,

​      EvaluationException {

​    counter++;



​    if (counter == LIMIT) {

​      counter = 0;



​      return true;

​    } else {

​      return false;

​    }

  }



  public String getName() {

​    return name;

  }



  public void setName(String name) {

​    this.name = name;

  }

}



使用自定义的Evaluator

<configuration>

  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">

​    <evaluator class="chapters.appenders.mail.CounterBasedEvaluator" />

​    <smtpHost>${smtpHost}</smtpHost>

​    <to>${to}</to>

​    <from>${from}</from>

​    <subject>%logger{20} - %m</subject>



​    <layout class="ch.qos.logback.classic.html.HTMLLayout"/>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="EMAIL" />

  </root>

</configuration>



基于标签的Evaluator

前面我们在讲SMTPAppender属性的时候，讲过logback提供了几个Evaluator的实现，下面我们就看看使用例子。

在代码中指定日志事件的标签

Marker notifyAdmin = MarkerFactory.getMarker("NOTIFY_ADMIN");

logger.error(notifyAdmin,

  "This is a serious an error requiring the admin's attention",

  new Exception("Just testing"));



首先是OnmarkerEvaluator

<configuration>

  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">

​    <evaluator class="ch.qos.logback.classic.boolex.OnMarkerEvaluator">

​      <marker>NOTIFY_ADMIN</marker>

​      <!-- you specify add as many markers as you want -->

​      <marker>TRANSACTION_FAILURE</marker>

​    </evaluator>

​    <smtpHost>${smtpHost}</smtpHost>

​    <to>${to}</to>

​    <from>${from}</from>

​    <layout class="ch.qos.logback.classic.html.HTMLLayout"/>

  </appender>



  <root>

​    <level value ="debug"/>

​    <appender-ref ref="EMAIL" />

  </root>

</configuration>



然后是JaninoEventEvaluator

<configuration>

  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">

​    <evaluator class="ch.qos.logback.classic.boolex.JaninoEventEvaluator">

​      <expression>

​        (marker != null) &&

​        (marker.contains("NOTIFY_ADMIN") || marker.contains("TRANSACTION_FAILURE"))

​      </expression>

​    </evaluator>

​    ... same as above

  </appender>

</configuration>



最后看看GEventEvaluator，它使用了“ ?. ”安全调用符，只有当对象非空的时候才会调用其函数。

<configuration>

  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">

​    <evaluator class="ch.qos.logback.classic.boolex.GEventEvaluator">

​      <expression>

​        e.marker?.contains("NOTIFY_ADMIN") || e.marker?.contains("TRANSACTION_FAILURE")

​      </expression>

​    </evaluator>

​    ... same as aboveffdfdvc

  </appender>

</configuration>





下面的SMTPAppender使用SSL方式

<configuration>

  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">

​    <smtpHost>smtp.gmail.com</smtpHost>

​    <smtpPort>465</smtpPort>

​    <SSL>true</SSL>

​    <username>YOUR_USERNAME@gmail.com</username>

​    <password>YOUR_GMAIL_PASSWORD</password>



​    <to>EMAIL-DESTINATION</to>

​    <to>ANOTHER_EMAIL_DESTINATION</to> <!-- additional destinations are possible -->

​    <from>YOUR_USERNAME@gmail.com</from>

​    <subject>TESTING: %logger{20} - %m</subject>

​    <layout class="ch.qos.logback.classic.PatternLayout">

​      <pattern>%date %-5level %logger{35} - %message%n</pattern>

​    </layout>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="EMAIL" />

  </root>

</configuration>



下面的SMTPAppender使用STARTTLS

<configuration>

  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">

​    <smtpHost>smtp.gmail.com</smtpHost>

​    <smtpPort>587</smtpPort>

​    <STARTTLS>true</STARTTLS>

​    <username>YOUR_USERNAME@gmail.com</username>

​    <password>YOUR_GMAIL_xPASSWORD</password>



​    <to>EMAIL-DESTINATION</to>

​    <to>ANOTHER_EMAIL_DESTINATION</to> <!-- additional destinations are possible -->

​    <from>YOUR_USERNAME@gmail.com</from>

​    <subject>TESTING: %logger{20} - %m</subject>

​    <layout class="ch.qos.logback.classic.PatternLayout">

​      <pattern>%date %-5level %logger - %message%n</pattern>

​    </layout>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="EMAIL" />

  </root>

</configuration>





SMTPAppender 与 MDCDiscriminator

前面讲过，默认的Discriminator是根据返回值将logging event分发到不同的缓冲区中的。

下面是使用Logback Discriminator的其中一个实现 MDCDiscriminator，完成一句远程主机ip区分日志缓冲区的功能

<configuration>

  <appender name="EMAIL" class="ch.qos.logback.classic.net.SMTPAppender">

​    <smtpHost>ADDRESS-OF-YOUR-SMTP-HOST</smtpHost>

​    <to>EMAIL-DESTINATION</to>

​    <from>SENDER-EMAIL</from>



​    <discriminator class="ch.qos.logback.classic.sift.MDCBasedDiscriminator">

​      <key>req.remoteHost</key>

​      <defaultValue>default</defaultValue>

​    </discriminator>



​    <subject>${HOSTNAME} -- %X{req.remoteHost} %msg"</subject>

​    <layout class="ch.qos.logback.classic.html.HTMLLayout">

​      <pattern>%date%level%thread%X{req.remoteHost}%X{req.requestURL}%logger%msg</pattern>

​    </layout>

  </appender>



  <root>

​    <level level="DEBUG"/>

​    <appender-ref ref="EMAIL" />

  </root>

</configuration>





在一个忙绿的系统中如何管理循环缓冲区？

如果使用自定义的discriminator，有可能会创建出很多新的循环缓冲区，但是maxNumberOfBuffers（默认值：64）。无论何时，只要缓冲区数量大于这个值，就会采用“最近最少更新算法”丢弃最近最少更新的缓冲区。为了安全考虑，如果缓冲区在30分钟内无更新，同样也会被丢弃。

在一个TPS（每秒事务处理量）比较大的应用中，如果设置maxNumberOfBuffer设置的过小，会经常导致每个邮件的日志数量偏小。原因是，缓冲区可能创建，销毁，又重新创建，如此循环，导致最后的缓冲区日志量偏低。

为了防止这个摇摆不定的效果，当SMTPAppender遇到标签为" FINALIZE_SESSION "的日志事件时，就会释放与之对应的缓冲区。这样我们就可以自定义缓冲区的回收策略，不再使用最近最少更新算法。你也可以将maxNumberOfBuffers的值设置得更大些，比如512，1024等等。







**DBAppender**

 [`DBAppender`](http://logback.qos.ch/xref/ch/qos/logback/classic/db/DBAppender.html) 可以将日志事件插入到3张数据表中。它们分别是logging_event，logging_event_property，logging_event_exception。这三张数据表必须在DBAppender工作之前存在。它们的sql脚本可以在 logback-classic/src/main/java/ch/qos/logback/classic/db/script folder 这个目录下找到。这个脚本对大部分SQL数据库都是有效的，除了少部分，少数语法有差异需要调整。

下面是logback与常见数据库的支持信息：

| RDBMS                | tested version(s) | tested JDBC driver version(s) | supports  `getGeneratedKeys()` method | is a dialect  provided by logback |
| -------------------- | ----------------- | ----------------------------- | ------------------------------------- | --------------------------------- |
| DB2                  | untested          | untested                      | unknown                               | NO                                |
| H2                   | 1.2.132           | -                             | unknown                               | YES                               |
| HSQL                 | 1.8.0.7           | -                             | NO                                    | YES                               |
| Microsoft SQL Server | 2005              | 2.0.1008.2 (sqljdbc.jar)      | YES                                   | YES                               |
| MySQL                | 5.0.22            | 5.0.8 (mysql-connector.jar)   | YES                                   | YES                               |
| PostgreSQL           | 8.x               | 8.4-701.jdbc4                 | NO                                    | YES                               |
| Oracle               | 10g               | 10.2.0.1 (ojdbc14.jar)        | YES                                   | YES                               |
| SQLLite              | 3.7.4             | -                             | unknown                               | YES                               |
| Sybase SQLAnywhere   | 10.0.1            | -                             | unknown                               | YES                               |



下面来看看三张数据表的表结构信息：



logging_event

| Field                 | Type       | Description                                                  |
| --------------------- | ---------- | ------------------------------------------------------------ |
| **timestamp**         | `big int`  | 日志事件创建的时间                                           |
| **formatted_message** | `text`     | 格式化logging event的信息The message that has been added to the logging event, after formatting with `org.slf4j.impl.MessageFormatter`, in case objects were passed along with the message. |
| **logger_name**       | `varchar`  | logger的name                                                 |
| **level_string**      | `varchar`  | 日志事件等级                                                 |
| **reference_flag**    | `smallint` | 这个属性被logback用来判断logging event是否包含exception 或 MDC property这个值由`ch.qos.logback.classic.db.DBHelper计算得出。`当logging event包含MDC 或者上下文属性时，为1；当包含异常为2；前两个都有则为3。 |
| **caller_filename**   | `varchar`  | 日志事件调用者的文件名                                       |
| **caller_class**      | `varchar`  | 日志事件调用者的类名                                         |
| **caller_method**     | `varchar`  | 日志事件调用者的函数                                         |
| **caller_line**       | `char`     | 触发日志事件的代码所在行数                                   |
| **event_id**          | `int`      | 日志事件在数据库表中的id。主键                               |



logging_event_property

| Field            | Type      | Description                    |
| ---------------- | --------- | ------------------------------ |
| **event_id**     | `int`     | 日志事件在数据库表中的id。主键 |
| **mapped_key**   | `varchar` | MDC property的key              |
| **mapped_value** | `text`    | MDC property的value            |



logging_event_exception

| Field          | Type       | Description                                    |
| -------------- | ---------- | ---------------------------------------------- |
| **event_id**   | `int`      | 日志事件在数据库表中的id。主键                 |
| **i**          | `smallint` | The index of the line in the full stack trace. |
| **trace_line** | `varchar`  | The corresponding line                         |



下面是这三张表的样例输出

The *logging_event* table:



The *logging_event_exception* table:



The *logging_event_property* table:

![Logging Event Property table](file:///C:/Users/wuzhe/AppData/Local/Temp/enhtmlclip/Image(2).gif)





接下来我们看看如何配置数据库连接：实验表明，不使用数据库连接池技术的情况下，插入一条日志大概需要10毫秒，而使用连接池则大概1毫秒。所以推荐使用数据库连接池来获取数据库连接对象，常见的就有C3P0。

下面是一个使用C3P0的例子：（推荐使用）

<configuration>



  <appender name="DB" class="ch.qos.logback.classic.db.DBAppender">

​    <connectionSource

​      class="ch.qos.logback.core.db.DataSourceConnectionSource">

​      <dataSource

​        class="com.mchange.v2.c3p0.ComboPooledDataSource">

​        <driverClass>com.mysql.jdbc.Driver</driverClass>

​        <jdbcUrl>jdbc:mysql://${serverName}:${port}/${dbName}</jdbcUrl>

​        <user>${user}</user>

​        <password>${password}</password>

​      </dataSource>

​    </connectionSource>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="DB" />

  </root>

</configuration>



除了C3P0，你也可以使用logback自带的ch.qos.logback.core.db.DriverManagerConnectionSource

<configuration  debug="true">



  <appender name="DB" class="ch.qos.logback.classic.db.DBAppender">

​    <connectionSource class="ch.qos.logback.core.db.DataSourceConnectionSource">



​      <dataSource class="${dataSourceClass}">

​        <!-- Joran cannot substitute variables

​        that are not attribute values. Therefore, we cannot

​        declare the next parameter like the others.

​        -->

​        <param name="${url-key:-url}" value="${url_value}"/>

​        <serverName>${serverName}</serverName>

​        <databaseName>${databaseName}</databaseName>

​      </dataSource>



​      <user>${user}</user>

​      <password>${password}</password>

​    </connectionSource>

  </appender>



  <root level="INFO">

​    <appender-ref ref="DB" />

  </root>

</configuration>



当然，你也可以使用JNDI配置数据源，然后获取JNDI的连接池中获取连接对象

<Context docBase="/path/to/app.war" path="/myapp">

  ...

  <Resource name="jdbc/logging"

​              auth="Container"

​              type="javax.sql.DataSource"

​              username="..."

​              password="..."

​              driverClassName="org.postgresql.Driver"

​              url="jdbc:postgresql://localhost/..."

​              maxActive="8"

​              maxIdle="4"/>

  ...

</Context>





<configuration debug="true">

  <appender name="DB" class="ch.qos.logback.classic.db.DBAppender">

​    <connectionSource class="ch.qos.logback.core.db.JNDIConnectionSource">

​      <!-- please note the "java:comp/env/" prefix -->

​      <jndiLocation>java:comp/env/jdbc/logging</jndiLocation>

​    </connectionSource>

  </appender>

  <root level="INFO">

​    <appender-ref ref="DB" />

  </root>

</configuration>







下面是非连接池的配置，看看就好：

这个配置每次插入日志都会重新获取数据库连接对象。

<configuration>



  <appender name="DB" class="ch.qos.logback.classic.db.DBAppender">

​    <connectionSource class="ch.qos.logback.core.db.DriverManagerConnectionSource">

​      <driverClass>com.mysql.jdbc.Driver</driverClass>

​      <url>jdbc:mysql://host_name:3306/datebase_name</url>

​      <user>username</user>

​      <password>password</password>

​    </connectionSource>

  </appender>



  <root level="DEBUG" >

​    <appender-ref ref="DB" />

  </root>

</configuration>









**SyslogAppender**

```
SyslogAppender 是一个非常简单的协议：一个syslog sender发送一个small message给syslog receiver。这个receiver通常被叫做syslog守护进程或者syslog server。logback可以发送消息给远程的receiver
```

下面是配置项



| Property Name         | Type      | Description                                                  |
| --------------------- | --------- | ------------------------------------------------------------ |
| **syslogHost**        | `String`  | syslog server的主机名                                        |
| **port**              | `String`  | server的端口，默认值：514                                    |
| **facility**          | `String`  | facility属性用来识别message的来源必须被设定为*KERN, USER, MAIL, DAEMON, AUTH, SYSLOG, LPR, NEWS, UUCP, CRON, AUTHPRIV, FTP, NTP, AUDIT, ALERT, CLOCK, LOCAL0, LOCAL1, LOCAL2, LOCAL3, LOCAL4, LOCAL5, LOCAL6, LOCAL7*. 这些字符串的其中一个。 |
| **suffixPattern**     | `String`  | 指定发送的消息格式，默认值：*[%thread] %logger %msg 。参照的语法PatternLayout* |
| **stackTracePattern** | `String`  | The stackTracePattern property allows the customization of the string appearing just before each stack trace line. The default value for this property is "\t", i.e. the tab character. Any value accepted by `PatternLayout` is a valid value for stackTracePattern. |
| **throwableExcluded** | `boolean` | Setting throwableExcluded to `true` will cause stack trace data associated with a Throwable to be omitted. By default, throwableExcluded is set to `false` so that stack trace data is sent to the syslog server. |



一个小例子

<configuration>



  <appender name="SYSLOG" class="ch.qos.logback.classic.net.SyslogAppender">

​    <syslogHost>remote_home</syslogHost>

​    <facility>AUTH</facility>

​    <suffixPattern>[%thread] %logger %msg</suffixPattern>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="SYSLOG" />

  </root>

</configuration>

在测试这个例子的时候，你需要先让远程的syslog darmon允许接收外部request。默认是拒绝接受的。







**SiftingAppender**

人如其名，SiftingAppender提供过滤筛选日志的功能。你可以通过用户的sessions的数据来筛选日志，然后分发到不同日志文件。

我们先来看个例子再说明：

<configuration>



  <appender name="SIFT" class="ch.qos.logback.classic.sift.SiftingAppender">

​    <!-- in the absence of the class attribute, it is assumed that the

​        desired discriminator type is

​        ch.qos.logback.classic.sift.MDCBasedDiscriminator -->

​    <discriminator>

​      <key>userid</key>

​      <defaultValue>unknown</defaultValue>

​    </discriminator>

​    <sift>

​      <appender name="FILE-${userid}" class="ch.qos.logback.core.FileAppender">

​        <file>${userid}.log</file>

​        <append>false</append>

​        <layout class="ch.qos.logback.classic.PatternLayout">

​          <pattern>%d [%thread] %level %mdc %logger{35} - %msg%n</pattern>

​        </layout>

​      </appender>

​    </sift>

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="SIFT" />

  </root>

</configuration>





logger.debug("Application started");

MDC.put("userid", "Alice");

logger.debug("Alice says hello");

这个例子通过设置MDC属性，将不同用户id的日志存放在不同的日志文件中。

SiftingAppender是通过创建内置的Appender来完成这些功能的，通过使用<sift> <appender></appender> </sift>。SiftingAppender负责管理这些子Appender的生命周期。SiftingAppender会自动关闭和移除过期的appender。内置的appender如果在timeout属性指定的时间内都没有进行过日志输出，就会被认为是过期的。

在这个例子中，我们同样没有指定discriminator，则默认会使用MDCBasedDiscriminator，如果指定的key的value的值为null，则会使用<defaultValue>



SitingAppender有以下两个属性

| Property Name        | Type       | Description                                                  |
| -------------------- | ---------- | ------------------------------------------------------------ |
| **timeout**          | `Duration` | 设置内置appender未有日志输出的过期时间。默认值：30 minutes   |
| **maxAppenderCount** | `integer`  | 设置最多可以创建多少个内置的appender，默认值：Integer.MAX_VALUE |

设置一个合适的timeout和maxAppenderCount并非那么容易，太小容易导致内置appender刚出生没多久就被销毁了，而太大，又会消耗过多的资源。所以很多情况下，我们也可以手动关闭内置appender，通过产生一个标签（marker）为FINALIZE_SESSION的日志事件，当SiftingAppender看到这个标签的日志事件，就会结束与之绑定的appender，不过appender的销毁并不是实时的，而是等待几秒后，防止还有未处理的日志丢失。这就类似于TCP协议的四次挥手过程。

看个小例子

import org.slf4j.Logger;

import static ch.qos.logback.classic.ClassicConstants.FINALIZE_SESSION_MARKER;



  void job(String jobId) {



​    MDC.put("jobId", jobId);

​    logger.info("Starting job.");



​    ... do whather the job needs to do



​    // will cause the nested appender reach end-of-life. It will

​    // linger for a few seconds.

​    logger.info(FINALIZE_SESSION_MARKER, "About to end the job");



​    try {

​      .. perform clean up

​    } catch(Exception e);

​      // This log statement will be handled by the lingering appender.

​      // No new appender will be created.

​      logger.error("unexpected error while cleaning up", e);

​    }

  }







**AsyncAppender**

AsyncAppender记录ILoggingEvents的方式是异步的。它仅仅相当于一个event分配器，因此需要配合其他appender才能有所作为。

需要注意的是：AsyncAppender将event缓存在 [BlockingQueue](http://docs.oracle.com/javase/1.5.0/docs/api/java/util/concurrent/BlockingQueue.html) ，一个由AsyncAppender创建的工作线程，会一直从这个队列的头部获取events，然后将它们分配给与AsyncAppender唯一关联的Appender中。默认情况下，如果这个队列80%已经被占满，则AsyncAppender会丢弃等级为 TRACE，DEBUG，INFO这三个等级的日志事件。

在应用关闭或重新部署的时候，AsyncAppender一定要被关闭，目的是为了停止，回收再利用worker thread，和刷新缓冲队列中logging events。那如果关闭AsyncAppender呢？可以通过关闭LoggerContext来关闭所有appender，当然也包括AsyncAppender了。AsyncAppender会在maxFlushTime属性设置的时间内等待Worker thread刷新全部日志event。如果你发现缓冲的event在关闭LoggerContext的时候被丢弃，这时候你就也许需要增加等待的时间。将maxFlushTime设置成0，就是AsyncAppender一直等待直到工作线程将所有被缓冲的events全部刷新出去才执行才结束。

根据JVM退出的模式，工作线程worker thread处理被缓冲的events的工作是可以被中断的，这样就导致了剩余未处理的events被搁浅。这种现象通常的原因是当LoggerContext没有完全关闭，或者当JVM终止那些非典型的控制流（不明觉厉）。为了避免工作线程的因为这些情况而发生中断，一个shutdown hook（关闭钩子）可以被插入到JVM运行的时候，这个钩子的作用是在JVM开始shutdown刚开始的时候执行关闭 LoggerContext的任务。



下面是AsyncAppender的属性表

| Property Name           | Type      | Description                                                  |
| ----------------------- | --------- | ------------------------------------------------------------ |
| **queueSize**           | `int`     | 设置blocking queue的最大容量，默认是256条events              |
| **discardingThreshold** | `int`     | 默认，当blocking queue被占用80%以上，AsyncAppender就会丢弃level为 TRACE，DEBUG，INFO的日志事件，如果要保留所有等级的日志，需要设置成0 |
| **includeCallerData**   | `boolean` | 提取CallerData代价比较昂贵，为了提高性能，caller data默认不提供。只有一些获取代价较低的数据，如线程名称，MDC值才会被保留。如果设置为true，就会包含caller data |
| **maxFlushTime**        | `int`     | 设置最大等待刷新事件，单位为miliseconds(毫秒)。当LoggerContext关闭的时候，AsyncAppender会在这个时间内等待工作线程完成events的flush工作，超时未处理的events将会被抛弃。 |
| **neverBlock**          | `boolean` | 默认为false，如果队列被填满，为了处理所有日志，就会阻塞的应用。如果为true，为了不阻塞你的应用，也会选择抛弃一些message。 |



默认情况下，event queue最大的容量是256。如果队列被填充满那么就会阻塞你的应用，直到队列能够容纳新的logging event。所以当AsyncAppender工作在队列满的情况下，可以称作伪同步。

在以下四种情况下容易导致AsyncAppender伪同步状态的出现：

- 应用中存在大量线程
- 每秒产生大量的logging events
- 每一个logging event都存在大量的数据
- 子appender中存在很高的延迟

为了避免伪同步的出现，提高queueSizes普遍有效，但是就消耗了应用的可用内存。



说了那么多，我们来看看一个小例子吧

<configuration>

  <appender name="FILE" class="ch.qos.logback.core.FileAppender">

​    <file>myapp.log</file>

​    <encoder>

​      <pattern>%logger{35} - %msg%n</pattern>

​    </encoder>

  </appender>



  <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">

​    <appender-ref ref="FILE" />

  </appender>



  <root level="DEBUG">

​    <appender-ref ref="ASYNC" />

  </root>

</configuration>









**编写你自己的Appender**

你可以很简单的创建自己的Appender，通过继承父类AppenderBase。AppenderBase已经实现了对filters，status以及一些其他被大多数appender共享的功能的支持。我们所要做的仅仅是实现append(Object evenObject)这个方法。

下面看个例子

package chapters.appenders;



import java.io.IOException;



import ch.qos.logback.classic.encoder.PatternLayoutEncoder;

import ch.qos.logback.classic.spi.ILoggingEvent;

import ch.qos.logback.core.AppenderBase;



public class CountingConsoleAppender extends AppenderBase<ILoggingEvent> {

  static int DEFAULT_LIMIT = 10;

  int counter = 0;

  int limit = DEFAULT_LIMIT;



  PatternLayoutEncoder encoder;



  public void setLimit(int limit) {

​    this.limit = limit;

  }



  public int getLimit() {

​    return limit;

  }



  @Override

  public void start() {

​    if (this.encoder == null) {

​      addError("No encoder set for the appender named ["+ name +"].");

​      return;

​    }



​    try {

​      encoder.init(System.out);

​    } catch (IOException e) {

​    }

​    super.start();

  }



  public void append(ILoggingEvent event) {

​    if (counter >= limit) {

​      return;

​    }

​    // output the events as formatted by our layout

​    try {

​      this.encoder.doEncode(event);

​    } catch (IOException e) {

​    }



​    // prepare for next event

​    counter++;

  }



  public PatternLayoutEncoder getEncoder() {

​    return encoder;

  }



  public void setEncoder(PatternLayoutEncoder encoder) {

​    this.encoder = encoder;

  }

}

这个例子告诉我们两个点

1. 所有的属性的设置都是通过setter/getter来实现配置文件透明设置的。而start()方法则会在logback读取配置文件配置时，自动调用，负责检查相关属性是否设置有误。
2. AppenderBase.doAppend（）会调用子类的append（）的方法完成日志输出，特别的，也可以在append（）方法内调用layout格式化日志。





讲了这么多logbakc-classiic的东西，我们现在来看看logback-access

大多数在logback-classic存在的appenders，在logback-access中都有与其相似的等价物。他们大部分都是相似的，只不过ILogginEvent变成AccessEvent。下面我们主要讲他们的不同之处



在SMTPAppender中，Evaluator属性，默认使用URLEvaluator，下面看个例子

<appender name="SMTP"

  class="ch.qos.logback.access.net.SMTPAppender">

  <layout class="ch.qos.logback.access.html.HTMLLayout">

​    <pattern>%h%l%u%t%r%s%b</pattern>

  </layout>



  <Evaluator class="ch.qos.logback.access.net.URLEvaluator">

​    <URL>url1.jsp</URL>

​    <URL>directory/url2.html</URL>

  </Evaluator>

  <from>sender_email@host.com</from>

  <smtpHost>mail.domain.com</smtpHost>

  <to>recipient_email@host.com</to>

</appender>

当当前URL符合多个<URL>标签指定的路径的其中一个时，就会触发发送一封邮件。





在DBAppender中，也有不同。首先只用到了两张表，access_event以及access_event_header。它们的创建sql脚本可以在 logback-access/src/main/java/ch/qos/logback/access/db/scrip找到。

看下表结构

**access_event** 

| Field          | Type      | Description                                                  |
| -------------- | --------- | ------------------------------------------------------------ |
| **timestamp**  | `big int` | The timestamp that was valid at the access event's creation. |
| **requestURI** | `varchar` | The URI that was requested.                                  |
| **requestURL** | `varchar` | The URL that was requested. This is a string composed of the request method, the request URI and the request protocol. |
| **remoteHost** | `varchar` | The name of the remote host.                                 |
| **remoteUser** | `varchar` | The name of the remote user.                                 |
| **remoteAddr** | `varchar` | The remote IP address.                                       |
| **protocol**   | `varchar` | The request protocol, like *HTTP* or *HTTPS*.                |
| **method**     | `varchar` | The request method, usually *GET* or *POST*.                 |
| **serverName** | `varchar` | The name of the server that issued the request.              |
| **event_id**   | `int`     | The database id of the access event.                         |



 access_event_header

| Field            | Type      | Description                                                  |
| ---------------- | --------- | ------------------------------------------------------------ |
| **event_id**     | `int`     | The database id of the corresponding access event.           |
| **header_key**   | `varchar` | The header name, for example *User-Agent*.                   |
| **header_value** | `varchar` | The header value, for example *Mozilla/5.0 (Windows; U; Windows NT 5.1; fr; rv:1.8.1) Gecko/20061010 Firefox/2.0* |







而DBAppender，所有在classic模块中支持的属性，在access模块中都支持，而且access还提供多一个属性

| **Property Name** | Type      | Description                                                  |
| ----------------- | --------- | ------------------------------------------------------------ |
| **insertHeaders** | `boolean` | Tells the `DBAppender` to populate the database with the header information of all incoming requests. |



下面是个例子

<configuration>



  <appender name="DB" class="ch.qos.logback.access.db.DBAppender">

​    <connectionSource class="ch.qos.logback.core.db.DriverManagerConnectionSource">

​      <driverClass>com.mysql.jdbc.Driver</driverClass>

​      <url>jdbc:mysql://localhost:3306/logbackdb</url>

​      <user>logback</user>

​      <password>logback</password>

​    </connectionSource>

​    <insertHeaders>true</insertHeaders>

  </appender>



  <appender-ref ref="DB" />

</configuration>





在SiftingAppender虽然大部分相似，但是Discriminator默认值不同，logback-access默认使用AccessEventDiscriminator，而不是MDCBasedDiscriminator.

下面是它的配置中Discriminator部分

<Discriminator class="ch.qos.logback.access.sift.AccessEventDiscriminator">

​      <Key>id</Key>

​      <FieldName>SESSION_ATTRIBUTE</FieldName>

​      <AdditionalKey>username</AdditionalKey>

​      <defaultValue>NA</defaultValue>

</Discriminator>

<FieldName>的值可以设置成 COOKIE, REQUEST_ATTRIBUTE, SESSION_ATTRIBUTE, REMOTE_ADDRESS, LOCAL_PORT, REQUEST_URI中的其中一个。而COOKIE, REQUEST_ATTRIBUTE, SESSION_ATTRIBUTE 这设置成三个值，就必须设置<AdditionalKey>标识获取那个属性，当AdditionalKey设置的key对应的value为Null时，使用<defaultValue>设置的值作为默认值。而<key>设置的值相当于获取的value在子appender中以什么Key来获取，就相当于key value键值对中的key。

下面是个完整示例：

<configuration>

  <appender name="SIFTING" class="ch.qos.logback.access.sift.SiftingAppender">

​    <Discriminator class="ch.qos.logback.access.sift.AccessEventDiscriminator">

​      <Key>id</Key>

​      <FieldName>SESSION_ATTRIBUTE</FieldName>

​      <AdditionalKey>username</AdditionalKey>

​      <defaultValue>NA</defaultValue>

​    </Discriminator>

​    <sift>

​      <appender name="ch.qos.logback:logback-site:jar:1.1.7" class="ch.qos.logback.core.FileAppender">

​        <file>byUser/ch.qos.logback:logback-site:jar:1.1.7.log</file>

​        <layout class="ch.qos.logback.access.PatternLayout">

​          <pattern>%h %l %u %t \"%r\" %s %b</pattern>

​        </layout>

​      </appender>

​    </sift>

  </appender>

  <appender-ref ref="SIFTING" />

</configuration>