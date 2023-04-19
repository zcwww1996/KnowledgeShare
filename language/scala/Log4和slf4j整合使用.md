[TOC]
来源：https://www.jianshu.com/p/0166f4bb5fec<br/>
参考：https://www.jianshu.com/p/383c932cbf82
# 1.概念

- slf4j：按百科来说，SLF4J，即简单日志门面（Simple Logging Facade for Java），不是具体的日志解决方案，它只服务于各种各样的日志系统。实际上，SLF4J所提供的核心API是一些接口以及一个LoggerFactory的工厂类。在使用SLF4J的时候，不需要在代码中或配置文件中指定你打算使用哪个具体的日志系统，只要按照其提供的方法记录即可，最终日志的格式、记录级别、输出方式等通过具体日志系统的配置来实现，因此可以在应用中灵活切换日志系统。

- log4j：log4j是Apache的一个开源项目，可以灵活地记录日志信息，我们可以通过Log4j的配置文件灵活配置日志的记录格式、记录级别、输出格式，而不需要修改已有的日志记录代码

# 2.log4j基本用法

首先，如果使用maven构建工程，则只需要引入相应的依赖即可，如果是普通的web工程，则只需要将jar包build path即可。

```java
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>1.7.30</version>
</dependency>

```

然后，在src/main/java目录（包的根目录即classpath）新建log4j.properties文件。

## 2.1 log4j.properties路径

log4j.properties要放在哪以及怎样配置才能被解析呢？不同工程类型配置方式不同。

### 2.1.1 maven或springboot工程

这是最常见的java工程类型，把log4j.properties放在src/main/resources目录（源代码的配置文件目录）就行了。

### 2.1.2 普通java或spring工程

这类工程，写demo用的多，把log4j.properties放在src/main/java目录（包的根目录）就行了。

### 2.1.3 spring mvc工程

web工程里用spring mvc构建的比较多了，把log4j.properties放在src/main/resources的conf目录（web工程配置文件通常在resources或WEB-INF目录），编辑web.xml，添加

```xml
<context-param>
    <param-name>log4jConfigLocation</param-name>
    <param-value>classpath:/conf/log4j.properties</param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>
</listener>

```
### 2.1.4 普通web工程


```java
public class Log4jServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
  
    public void init(ServletConfig config) throws ServletException {
        String prefix = this.getClass().getClassLoader().getResource("/").getPath();
        String path = config.getInitParameter("log4j-path");
        PropertyConfigurator.configure(prefix + path);
    }
    public void doGet(HttpServletRequest req, HttpServletResponse res) throws IOException, ServletException {}
    public void doPost(HttpServletRequest req, HttpServletResponse res) throws IOException, ServletException {}
    public void destroy() {}
}

```
编辑web.xml，添加

```xml
<servlet>
    <servlet-name>log4j</servlet-name>
    <servlet-class>com.xmyself.log4j.Log4jServlet</servlet-class>
    <init-param>
        <param-name>log4j-path</param-name>
        <param-value>conf/log4j.properties</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

```

## 2.2 log4j.properties内容

log4j日志分为7个等级：**ALL、DEBUG、INFO、WARN、ERROR、FATAL、OFF**，从左到右等级由低到高，分等级是为了设置日志输出的门槛，只有等级等于或高于这个门槛的日志才有机会输出。

我们要输出日志，首先得有日志对象（logger），那这些日志对象把日志输出到哪里呢，控制台还是文件，这就要设置输出位置（appender），输出的格式与内容又是什么样的呢，这就要设置输出样式（layout），这些设置完，log4j的配置也就完了。

### 2.2.1 logger

日志实例，就是代码里实例化的Logger对象。

```properties
log4j.rootLogger=LEVEL,console,FileAppender,...
log4j.additivity.org.apache=false：表示不会在父logger的appender里输出，默认true
```


这是全局logger的配置，LEVEL用来设定日志等级，appenderName定义日志输出器，示例中的“console”就是一个日志输出器

### 2.2.2 appender

日志输出器 appender有5种选择

```properties
org.apache.log4j.ConsoleAppender（控制台）
org.apache.log4j.FileAppender（文件）
org.apache.log4j.DailyRollingFileAppender（每天产生一个日志文件）
org.apache.log4j.RollingFileAppender（文件大小到达指定尺寸的时候产生一个新的文件）
org.apache.log4j.WriterAppender（将日志信息以流格式发送到任意指定的地方
```

每种appender都有若干配置项，下面逐一介绍

#### 2.2.2.1 ConsoleAppender（常用）

```properties
Threshold=WARN：指定日志信息的最低输出级别，默认DEBUG
ImmediateFlush=true：表示所有消息都会被立即输出，设为false则不输出，默认值是true
Target=System.err：默认值是System.out
```


#### 2.2.2.2 FileAppender

```properties
Threshold=WARN：指定日志信息的最低输出级别，默认DEBUG
ImmediateFlush=true：表示所有消息都会被立即输出，设为false则不输出，默认true
Append=false：true表示消息增加到指定文件中，false则将消息覆盖指定的文件内容，默认true
File=D:/logs/logging.log4j：指定消息输出到logging.log4j文件
```


#### 2.2.2.3 DailyRollingFileAppender（常用）

```properties
hreshold=WARN：指定日志信息的最低输出级别，默认DEBUG
ImmediateFlush=true：表示所有消息都会被立即输出，设为false则不输出，默认true
Append=false：true表示消息增加到指定文件中，false则将消息覆盖指定的文件内容，默认true
File=D:/logs/logging.log4j：指定当前消息输出到logging.log4j文件
DatePattern='.'yyyy-MM：每月滚动一次日志文件，即每月产生一个新的日志文件。当前月的日志文件名为logging.log4j，前一个月的日志文件名为logging.log4j.yyyy-MM
另外，也可以指定按周、天、时、分等来滚动日志文件，对应的格式如下：
1)'.'yyyy-MM：每月
2)'.'yyyy-ww：每周
3)'.'yyyy-MM-dd：每天
4)'.'yyyy-MM-dd-a：每天两次
5)'.'yyyy-MM-dd-HH：每小时
6)'.'yyyy-MM-dd-HH-mm：每分钟
```


#### 2.2.2.4 RollingFileAppender
```properties
Threshold=WARN：指定日志信息的最低输出级别，默认DEBUG
ImmediateFlush=true：表示所有消息都会被立即输出，设为false则不输出，默认true
Append=false：true表示消息增加到指定文件中，false则将消息覆盖指定的文件内容，默认true
File=D:/logs/logging.log4j：指定消息输出到logging.log4j文件
MaxFileSize=100KB：后缀可以是KB,MB或者GB。在日志文件到达该大小时，将会自动滚动，即将原来的内容移到logging.log4j.1文件
MaxBackupIndex=2：指定可以产生的滚动文件的最大数，例如，设为2则可以产生logging.log4j.1，logging.log4j.2两个滚动文件和一个logging.log4j文件
```

#### 2.2.3 layout

指定logger输出内容及格式。

layout有4种选择。

```properties
org.apache.log4j.HTMLLayout（以HTML表格形式布局）
org.apache.log4j.PatternLayout（可以灵活地指定布局模式）
org.apache.log4j.SimpleLayout（包含日志信息的级别和信息字符串）
org.apache.log4j.TTCCLayout（包含日志产生的时间、线程、类别等信息）
```

layout也有配置项，下面具体介绍

#### 2.2.3.1 HTMLLayout

```properties
LocationInfo=true：输出java文件名称和行号，默认false
Title=My Logging： 默认值是Log4J Log Messages
```

#### 2.2.3.2 PatternLayout（最常用的配置）

```properties
ConversionPattern=%m%n：设定以怎样的格式显示消息

设置格式的参数说明如下
%p：输出日志信息的优先级，即DEBUG，INFO，WARN，ERROR，FATAL
%d：输出日志时间点的日期或时间，默认格式为ISO8601，可以指定格式如：%d{yyyy/MM/dd HH:mm:ss,SSS}
%r：输出自应用程序启动到输出该log信息耗费的毫秒数
%t：输出产生该日志事件的线程名
%l：输出日志事件的发生位置，相当于%c.%M(%F:%L)的组合，包括类全名、方法、文件名以及在代码中的行数
%c：输出日志信息所属的类目，通常就是类全名
%M：输出产生日志信息的方法名
%F：输出日志消息产生时所在的文件名
%L：输出代码中的行号
%m：输出代码中指定的具体日志信息
%n：输出一个回车换行符，Windows平台为"rn"，Unix平台为"n"
%x：输出和当前线程相关联的NDC(嵌套诊断环境)
%%：输出一个"%"字符
```

## 2.3 log4j完整配置示例
一些常用的日志输出

```properties
log4j.rootLogger=DEBUG,console,dailyFile,rollingFile,logFile
log4j.additivity.org.apache=true
```


### 2.3.1 控制台console日志输出器

```properties
# 控制台(console)
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.Threshold=DEBUG
log4j.appender.console.ImmediateFlush=true
log4j.appender.console.Target=System.err
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```


### 2.3.2 文件logFile日志输出器

```properties
# 日志文件(logFile)
log4j.appender.logFile=org.apache.log4j.FileAppender
log4j.appender.logFile.Threshold=DEBUG
log4j.appender.logFile.ImmediateFlush=true
log4j.appender.logFile.Append=true
log4j.appender.logFile.File=D:/logs/log.log4j
log4j.appender.logFile.layout=org.apache.log4j.PatternLayout
log4j.appender.logFile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```


### 2.3.3 滚动文件rollingFile日志输出器

```properties
# 滚动文件(rollingFile)
log4j.appender.rollingFile=org.apache.log4j.RollingFileAppender
log4j.appender.rollingFile.Threshold=DEBUG
log4j.appender.rollingFile.ImmediateFlush=true
log4j.appender.rollingFile.Append=true
log4j.appender.rollingFile.File=D:/logs/log.log4j
log4j.appender.rollingFile.MaxFileSize=200KB
log4j.appender.rollingFile.MaxBackupIndex=50
log4j.appender.rollingFile.layout=org.apache.log4j.PatternLayout
log4j.appender.rollingFile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```


### 2.3.4 定期滚动文件dailyFile日志输出器

```properties
# 定期滚动日志文件(dailyFile)
log4j.appender.dailyFile=org.apache.log4j.DailyRollingFileAppender
log4j.appender.dailyFile.Threshold=DEBUG
log4j.appender.dailyFile.ImmediateFlush=true
log4j.appender.dailyFile.Append=true
log4j.appender.dailyFile.File=D:/logs/log.log4j
log4j.appender.dailyFile.DatePattern='.'yyyy-MM-dd
log4j.appender.dailyFile.layout=org.apache.log4j.PatternLayout
log4j.appender.dailyFile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```

## 2.4 log4j局部日志配置

以上介绍的配置都是全局的，整个工程的代码使用同一套配置，意味着所有的日志都输出在了相同的地方，你无法直接了当的去看数据库访问日志、用户登录日志、操作日志，它们都混在一起，因此，需要为包甚至是类配置单独的日志输出，下面给出一个例子，为“com.demo.test”包指定日志输出器“test”，“com.demo.test”包下所有类的日志都将输出到/log/test.log文件。

```properties
log4j.logger.com.demo.test=DEBUG,test
log4j.appender.test=org.apache.log4j.FileAppender
log4j.appender.test.File=/log/test.log
log4j.appender.test.layout=org.apache.log4j.PatternLayout
log4j.appender.test.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```


也可以让同一个类输出不同的日志，为达到这个目的，需要在这个类中实例化两个logger。

```java
private static Log logger1 = LogFactory.getLog("myTest1");
private static Log logger2 = LogFactory.getLog("myTest2");
```


然后分别配置

```properties
log4j.logger.myTest1= DEBUG,test1
log4j.additivity.myTest1=false
log4j.appender.test1=org.apache.log4j.FileAppender
log4j.appender.test1.File=/log/test1.log
log4j.appender.test1.layout=org.apache.log4j.PatternLayout
log4j.appender.test1.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
  
log4j.logger.myTest2=DEBUG,test2
log4j.appender.test2=org.apache.log4j.FileAppender
log4j.appender.test2.File=/log/test2.log
log4j.appender.test2.layout=org.apache.log4j.PatternLayout
log4j.appender.test2.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```

# 3. slf4j与log4j联合使用(推荐)

## 3.1 添加slf4j与log4j的关联jar包
通过这个东西，将对slf4j接口的调用转换为对log4j的调用，不同的日志实现框架，这个转换工具不同

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.30</version>
</dependency>
```


当然了，slf4j-log4j12这个包肯定依赖了slf4j和log4j，所以使用slf4j+log4j的组合只要配置上面这一个依赖就够了

## 3.2 创建明logger对象
代码里声明logger要改一下，原来使用log4j是这样的

```java
import org.apache.log4j.Logger;

  object logTest {
  val log: Logger = Logger.getLogger(this.class);
   def main(args: Array[String]): Unit= {
        log.info("hello this is log4j info log")
    }
}
```


现在要改成这样

```java
import org.slf4j.{Logger, LoggerFactory}

object logTest {
  val log: Logger = LoggerFactory.getLogger(this.getClass)
  def main(args: Array[String]): Unit = {
    log.info("hello, my name is {}", "idea")
  }
}
```


依赖的Logger变了，而且，slf4j的api还能使用占位符，很方便。

# 4. slf4j与LogBack联合使用
LogBack被分为3个组件，logback-core, logback-classic 和 logback-access。

1. **logback-core**:提供了LogBack的核心功能，是另外两个组件的基础。
2. **logback-classic**:实现了Slf4j的API，所以当想配合Slf4j使用时，需要引入logback-classic。
3. **logback-access**:为了集成Servlet环境而准备的，可提供HTTP-access的日志接口。

## 4.1 添加slf4j与LogBack的关联jar包

```xml
<dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.30</version>
    </dependency>

<dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.3</version>
</dependency>
```

## 4.2 添加配置文件logback.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 定义日志文件的存储地址 -->
    <!--
        关于catalina.base解释如下：
            catalina.home指向公用信息的位置，就是bin和lib的父目录。
            catalina.base指向每个Tomcat目录私有信息的位置，就是conf、logs、temp、webapps和work的父目录。
    -->
    <property name="LOG_DIR" value="${catalina.base}/logs"/>

    <!--
        %p:输出优先级，即DEBUG,INFO,WARN,ERROR,FATAL
        %r:输出自应用启动到输出该日志讯息所耗费的毫秒数
        %t:输出产生该日志事件的线程名
        %f:输出日志讯息所属的类别的类别名
        %c:输出日志讯息所属的类的全名
        %d:输出日志时间点的日期或时间，指定格式的方式： %d{yyyy-MM-dd HH:mm:ss}
        %l:输出日志事件的发生位置，即输出日志讯息的语句在他所在类别的第几行。
        %m:输出代码中指定的讯息，如log(message)中的message
        %n:输出一个换行符号
    -->
    <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度 %msg：日志消息，%n是换行符-->
    <property name="pattern" value="%d{yyyyMMdd:HH:mm:ss.SSS} [%thread] %-5level  %msg%n"/>


    <!--
        Appender: 设置日志信息的去向,常用的有以下几个
            ch.qos.logback.core.ConsoleAppender (控制台)
            ch.qos.logback.core.rolling.RollingFileAppender (文件大小到达指定尺寸的时候产生一个新文件)
            ch.qos.logback.core.FileAppender (文件)
    -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 字符串System.out（默认）或者System.err -->
        <target>System.out</target>
        <!-- 对记录事件进行格式化 -->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>

    <appender name="SQL_INFO" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 被写入的文件名，可以是相对目录，也可以是绝对目录，如果上级目录不存在会自动创建 -->
        <file>${LOG_DIR}/sql_info.log</file>
        <!-- 当发生滚动时，决定RollingFileAppender的行为，涉及文件移动和重命名。属性class定义具体的滚动策略类 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 必要节点，包含文件名及"%d"转换符，"%d"可以包含一个java.text.SimpleDateFormat指定的时间格式，默认格式是 yyyy-MM-dd -->
            <fileNamePattern>${LOG_DIR}/sql_info_%d{yyyy-MM-dd}.log.%i.gz</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>20MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!-- 可选节点，控制保留的归档文件的最大数量，超出数量就删除旧文件。假设设置每个月滚动，如果是6，则只保存最近6个月的文件，删除之前的旧文件 -->
            <maxHistory>10</maxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${pattern}</pattern>
        </encoder>
        <!-- LevelFilter： 级别过滤器，根据日志级别进行过滤 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <!-- 用于配置符合过滤条件的操作 ACCEPT：日志会被立即处理，不再经过剩余过滤器 -->
            <onMatch>ACCEPT</onMatch>
            <!-- 用于配置不符合过滤条件的操作 DENY：日志将立即被抛弃不再经过其他过滤器 -->
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="SQL_ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_DIR}/sql_error.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_DIR}/sql_error_%d{yyyy-MM-dd}.log.%i.gz</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>20MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>10</maxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${pattern}</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="APP_INFO" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_DIR}/info.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <FileNamePattern>${LOG_DIR}/info.%d{yyyy-MM-dd}.log
            </FileNamePattern>
        </rollingPolicy>
        <encoder>
            <Pattern>[%date{yyyy-MM-dd HH:mm:ss}] [%-5level] [%thread] [%logger:%line]--%mdc{client} %msg%n</Pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="ch.qos.logback.classic.html.HTMLLayout">
                <property name="pattern" value="%d{yyyyMMdd:HH:mm:ss.SSS} [%thread] %-5level  %msg%n"/>
                <pattern>%d{yyyyMMdd:HH:mm:ss.SSS}%thread%-5level%F{32}%M%L%msg</pattern>
            </layout>
        </encoder>
        <file>${LOG_DIR}/test.html</file>
    </appender>


    <!--
        用来设置某一个包或者具体的某一个类的日志打印级别、以及指定<appender>。
        <loger>仅有一个name属性，一个可选的level和一个可选的addtivity属性
        name:
            用来指定受此logger约束的某一个包或者具体的某一个类。
        level:
            用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
            如果未设置此属性，那么当前logger将会继承上级的级别。
        additivity:
            是否向上级loger传递打印信息。默认是true。
        <logger>可以包含零个或多个<appender-ref>元素，标识这个appender将会添加到这个logger
    -->
    <logger name="java.sql" level="info" additivity="false">
        <level value="info" />
        <appender-ref ref="STDOUT"></appender-ref>
        <appender-ref ref="SQL_INFO"></appender-ref>
        <appender-ref ref="SQL_ERROR"></appender-ref>
    </logger>

    <logger name="com.souche.LogbackTest" additivity="false">
        <level value="info" />
        <appender-ref ref="STDOUT" />
        <appender-ref ref="APP_INFO" />
        <appender-ref ref="FILE"/>
    </logger>


    <!--
        也是<logger>元素，但是它是根logger。默认debug
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
        <root>可以包含零个或多个<appender-ref>元素，标识这个appender将会添加到这个logger。
    -->
    <root level="info">
        <level>info</level>
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="SQL_INFO"/>
        <appender-ref ref="SQL_ERROR"/>
        <appender-ref ref="FILE"/>
    </root>

</configuration>
```

## 4.3 实际应用
### 4.3.1 配置文件logback.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>info</level>
        </filter>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>

    </appender>

    <root level="info">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

### 4.3.2 logger对象

```java
import org.slf4j.{Logger, LoggerFactory}

object logTest {
  val log: Logger = LoggerFactory.getLogger(this.getClass)
  def main(args: Array[String]): Unit = {
    log.info("hello, my name is {}", "idea")
  }
}
```
