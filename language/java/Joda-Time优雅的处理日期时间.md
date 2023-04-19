[TOC]

参考：https://blog.csdn.net/chq88888/article/details/100591336

https://blog.csdn.net/u010454030/article/details/52486416

# 1. 简介
在Java中处理日期和时间是很常见的需求，基础的工具类就是我们熟悉的Date和Calendar，然而这些工具类的api使用并不是很方便和强大，于是就诞生了[Joda-Time](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.joda.org%2Fjoda-time%2F)这个专门处理日期时间的库。

下面是joda-time的官网和API(貌似需要翻墙)  
Home：[http://joda-time.sourceforge.net/](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fjoda-time.sourceforge.net%2F)

[![](https://static.oschina.net/uploads/space/2017/0722/100616_sNit_2505908.png)](https://pic.imgdb.cn/item/61bc59382ab3f51d91786078.png)

由于Joda-Time很优秀，在Java 8出现前的很长时间内成为Java中日期时间处理的事实标准，用来弥补JDK的不足。在Java 8中引入的`java.time`包是一组新的处理日期时间的API，遵守`JSR 310`。值得一提的是，Joda-Time的作者`Stephen Colebourne`和Oracle一起共同参与了这些API的设计和实现。

值得注意的是，Java 8中的`java.time`包中提供的API和Joda-Time并不完全相同。比如，在Joda-Time中常用的`Interval`（用来表示一对DateTime），在`JSR 310`中并不支持。因此，另一个名叫[Threeten](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.threeten.org%2F)的第三方库用来弥补Java 8的不足。Threeten翻译成中文就是310的意思，表示这个库与`JSR 310`有关。它的作者同样是Joda-Time的作者`Stephen Colebourne`。

Threeten主要提供两种发行包：**[ThreeTen-Backport](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.threeten.org%2Fthreetenbp%2F)**和**[ThreeTen-Extra](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.threeten.org%2Fthreeten-extra%2F)**。前者的目的在于对Java 6和Java 7的项目提供Java 8的date-time类的支持；后者的目的在于为Java 8的date-time类提供额外的增强功能（比如：`Interval`等）。

由于刚接触Joda-Time，并且目前的工作环境还未涉及到Java 8。因此，关于Java 8的date-time和Threeten的API，将在以后合适的时候介绍。这篇文章关注Joda-Time的使用。

总之，作为一种解决某一问题领域的工具库，我认为有以下几个方面值得关注：

*   功能是否全面，以能够满足生产需要，并用它解决这个问题领域中的绝大多数的问题
*   是否是主流工具。用的人越多，意味着该库经受了更多生产实践的验证，效率安全等方面都已被证明是可靠的
*   自己是否已经熟练掌握。会的多不如会的精，如果能够用一个工具快速熟练可靠地解决问题，在时间成本有限的情况下，就不用刻意追求学习其它可替代的库


# 2. 引入MAVEN依赖
```xml
<dependency>
    <groupId>joda-time</groupId>
    <artifactId>joda-time</artifactId>
    <version>2.9.2</version>
</dependency>
```

# 3. 核心类介绍
下面介绍5个最常用的date-time类：

*   Instant - 不可变的类，用来表示时间轴上一个瞬时的点
*   DateTime - 不可变的类，用来替换JDK的Calendar类
*   LocalDate - 不可变的类，表示一个本地的日期，而不包含时间部分（没有时区信息）
*   LocalTime - 不可变的类，表示一个本地的时间，而不包含日期部分（没有时区信息）
*   LocalDateTime - 不可变的类，表示一个本地的日期－时间（没有时区信息）

> 注意：不可变的类，表明了正如Java的String类型一样，其对象是不可变的。即，不论对它进行怎样的改变操作，返回的对象都是新对象。

Instant比较适合用来表示一个事件发生的时间戳。不用去关心它使用的日历系统或者是所在的时区。  
DateTime的主要目的是替换JDK中的Calendar类，用来处理那些时区信息比较重要的场景。  
LocalDate比较适合表示出生日期这样的类型，因为不关心这一天中的时间部分。  
LocalTime适合表示一个商店的每天开门/关门时间，因为不用关心日期部分。

# 4. DateTime类

作为Joda-Time很重要的一个类，详细地看一下它的用法。

构造一个DateTime实例

如果查看[Java Doc](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.joda.org%2Fjoda-time%2Fapidocs%2Findex.html)，会发现DateTime有很多构造方法。这是为了使用者能够很方便的由各种表示日期时间的对象构造出DateTime实例。下面介绍一些常用的构造方法：

*   DateTime()：这个无参的构造方法会创建一个在当前系统所在时区的当前时间，精确到毫秒
*   DateTime(int year, int monthOfYear, int dayOfMonth, int hourOfDay, int minuteOfHour, int secondOfMinute)：这个构造方法方便快速地构造一个指定的时间，这里精确到秒，类似地其它构造方法也可以传入毫秒。
*   DateTime(long instant)：这个构造方法创建出来的实例，是通过一个long类型的时间戳，它表示这个时间戳距`1970-01-01T00:00:00Z`的毫秒数。使用默认的时区。
*   DateTime(Object instant)：这个构造方法可以通过一个Object对象构造一个实例。这个Object对象可以是这些类型：ReadableInstant, String, Calendar和Date。其中String的格式需要是`ISO8601`格式，详见：[ISODateTimeFormat.dateTimeParser()](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.joda.org%2Fjoda-time%2Fapidocs%2Forg%2Fjoda%2Ftime%2Fformat%2FISODateTimeFormat.html%23dateTimeParser--)

下面举几个例子：

```java
DateTime dateTime1 = new DateTime();
System.out.println(dateTime1); 
DateTime dateTime2 = new DateTime(2016,2,14,0,0,0);
System.out.println(dateTime2); 
DateTime dateTime3 = new DateTime(1456473917004L);
System.out.println(dateTime3); 
DateTime dateTime4 = new DateTime(new Date());
System.out.println(dateTime4); 
DateTime dateTime5 = new DateTime("2016-02-15T00:00:00.000+08:00");
System.out.println(dateTime5); 
```

## 访问DateTime实例
https://www.cnblogs.com/heqiuyong/p/10502541.html

当你有一个DateTime实例的时候，就可以调用它的各种方法，获取需要的信息。

### with重置属性

用来设置DateTime实例到某个时间，因为DateTime是不可变对象，所以没有提供setter方法可供使用，with方法也没有改变原有的对象，而是返回了设置后的一个副本对象。下面这个例子，将2000-02-29的年份设置为1997。值得注意的是，因为1997年没有2月29日，所以自动转为了28日。

```java
DateTime dateTime2000Year = new DateTime(2000,2,29,0,0,0);
System.out.println(dateTime2000Year); // out: 2000-02-29T00:00:00.000+08:00
DateTime dateTime1997Year = dateTime2000Year.withYear(1997); 
System.out.println(dateTime1997Year); // out: 1997-02-28T00:00:00.000+08:00
```
    
### plus/minus增减日期时间
用来返回在DateTime实例上增加或减少一段时间后的实例。下面的例子：在当前的时刻加1天，得到了明天这个时刻的时间；在当前的时刻减1个月，得到了上个月这个时刻的时间。

```java
DateTime now = new DateTime();
System.out.println(now); // out: 2016-02-26T16:27:58.818+08:00
DateTime tomorrow = now.plusDays(1);
System.out.println(tomorrow); // out: 2016-02-27T16:27:58.818+08:00
DateTime lastMonth = now.minusMonths(1);
System.out.println(lastMonth); // out: 2016-01-26T16:27:58.818+08:00
```
    
注意，在增减时间的时候，想象成自己在翻日历，所有的计算都将符合历法，由Joda-Time自动完成，不会出现非法的日期（比如：3月31日加一个月后，并不会出现4月31日）。

### 返回Property的方法
Property是DateTime中的属性，保存了一些有用的信息。Property对象中的一些方法在这里一并介绍。下面的例子展示了，我们可以通过不同Property中get开头的方法获取一些有用的信息：

```java
DateTime now = new DateTime(); 
now.monthOfYear().getAsText(); 
now.monthOfYear().getAsText(Locale.KOREAN); 
now.dayOfWeek().getAsShortText(); 
now.dayOfWeek().getAsShortText(Locale.CHINESE); 
```

### round进行置0操作
有时我们需要对一个DateTime的某些属性进行置0操作。比如，我想得到当天的0点时刻。那么就需要用到Property中round开头的方法（roundFloorCopy）。如下面的例子所示：

```java
DateTime now = new DateTime(); // 2016-02-26T16:51:28.749+08:00
now.dayOfWeek().roundCeilingCopy(); // 2016-02-27T00:00:00.000+08:00
now.dayOfWeek().roundFloorCopy(); // 2016-02-26T00:00:00.000+08:00
now.minuteOfDay().roundFloorCopy(); // 2016-02-26T16:51:00.000+08:00
now.secondOfMinute().roundFloorCopy(); // 2016-02-26T16:51:28.000+08:00
```

### 判断DateTime对象大小状态的一些操作方法

```java
compareTo(DateTime d) 比较两时间大小 时间大于指定时间返回 1 时间小于指定时间返回-1 相等返回0
equals(DateTime d) 比较两时间是否相等 
isAfter(long instant) 判断时间是否大于指定时间 
isAfterNow() 判断时间是否大于当前时间 
isBefore(long instant) 判断时间是否小于指定时间 
isBeforeNow() 判断时间是否小于当前时间 
isEqual(long instant) 判断时间是否等于指定时间 
isEqualNow() 判断时间是否等于当前时间 
```

### jdk date互转 

```java
DateTime dt = new DateTime(new Date()); jdk的date转换为DateTime 
Date jdkDate = dt.toDate() 转换为jdk的date
```
### 时间格式转换
1. String转DateTime，DateTime转时间戳，DateTime转Date

```java
DateTimeFormatter yyyyMMdd = DateTimeFormat.forPattern("yyyyMMdd");
DateTime dateTime = DateTime.parse("20191122", yyyyMMdd);
// 时间戳，毫秒
long millis = dateTime.getMillis();
Date date = dateTime.toDate();
```

2. Date转DateTime，DateTime转String

```java
DateTime dateTime = new DateTime(new Date());
String dateFormat = dateTime.toString("yyyy/MM/dd");
```

3. 时间戳转DateTime，DateTime转String

```java
long millis = System.currentTimeMillis();
DateTime dateTime = new DateTime(millis); 
String dateFormat = dateTime.toString("yyyy/MM/dd");
```


# LocalDate

LocalDate只处理年月日

默认构造器

```java
LocalDate(int year, int monthOfYear, int dayOfMonth)
LocalDate(long instant)
```

方法跟DateTime方法类似，就不单独整理了，可以去api查看详细方法

其他拓展方法

```java
daysBetween(ReadableInstant start, ReadableInstant end)     获取两日期相差的天数
hoursBetween(ReadableInstant start, ReadableInstant end)    获取两日期相差的小时数
minutesBetween(ReadableInstant start, ReadableInstant end)  获取两日期相差的分钟数
monthsBetween(ReadableInstant start, ReadableInstant end)   获取两日期相差的月数
secondsBetween(ReadableInstant start, ReadableInstant end)  获取两日期相差的秒数
weeksBetween(ReadableInstant start, ReadableInstant end)    获取两日期相差的周数
yearsBetween(ReadableInstant start, ReadableInstant end)    获取两日期相差的年数
```

其它：还有许多其它方法（比如dateTime.year().isLeap()来判断是不是闰年）。它们的详细含义，请参照[Java Doc](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.joda.org%2Fjoda-time%2Fapidocs%2Findex.html)，现查现用，用需求驱动学习。

# 5. 日历系统和时区

Joda-Time默认使用的是ISO的日历系统，而ISO的日历系统是世界上公历的事实标准。然而，值得注意的是，ISO日历系统在表示1583年之前的历史时间是不精确的。  
Joda-Time默认使用的是JDK的时区设置。如果需要的话，这个默认值是可以被覆盖的。  
Joda-Time使用可插拔的机制来设计日历系统，而JDK则是使用子类的设计，比如GregorianCalendar。下面的代码，通过调用一个工厂方法获得Chronology的实现：

```java
Chronology coptic = CopticChronology.getInstance();
```

时区是作为chronology的一部分来被实现的。下面的代码获得一个Joda-Time chronology在东京的时区：

```java
DateTimeZone zone = DateTimeZone.forID("Asia/Tokyo");
Chronology gregorianJuian = GJChronology.getInstance(zone);
```

# 6. Interval和Period

Joda-Time为时间段的表示提供了支持。

*   Interval：它保存了一个开始时刻和一个结束时刻，因此能够表示一段时间，并进行这段时间的相应操作
*   Period：它保存了一段时间，比如：6个月，3天，7小时这样的概念。可以直接创建Period，或者从Interval对象构建。
*   Duration：它保存了一个精确的毫秒数。同样地，可以直接创建Duration，也可以从Interval对象构建。

虽然，这三个类都用来表示时间段，但是在用途上来说还是有一些差别。请看下面的例子：

```java
DateTime dt = new DateTime(2005, 3, 26, 12, 0, 0, 0);
DateTime plusPeriod = dt.plus(Period.days(1)); 
DateTime plusDuration = dt.plus(new Duration(24L*60L*60L*1000L));
```

因为当时那个地区执行夏令时的原因，在添加一个Period的时候会添加23个小时。而添加一个Duration，则会精确地添加24个小时，而不考虑历法。所以，Period和Duration的差别不但体现在精度上，也同样体现在语义上。因为，有时候按照有些地区的历法 **1天** _不等于_ **24小时**。

# 7. 测试

```java
import org.joda.time.DateTime;
import org.joda.time.Days;
import org.joda.time.LocalDate;
import org.joda.time.format.DateTimeFormat;
import org.joda.time.format.DateTimeFormatter;

import java.util.Calendar;
import java.util.Date;
import java.util.Locale;

public class jodaTest {


    public static void main(String\[\] args) {
        
        DateTime dateTime0 = new DateTime(2017, 7, 22, 10, 10 ,10);

        
        DateTime dateTime = new DateTime(2017, 7, 22, 10, 10 ,10,333);

        
        String dateString = dateTime.toString("dd-MM-yyyy HH:mm:ss");

        DateTimeFormatter format = DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss");
        
        DateTime dateTime2 = DateTime.parse("2017-7-22 7:22:45", format);

        
        String string\_u = dateTime2.toString("yyyy/MM/dd HH:mm:ss EE");
        System.out.println("2017/07/22 07:22:45 星期六   --->  " + string\_u);

        
        String string\_c = dateTime2.toString("yyyy年MM月dd日 HH:mm:ss EE", Locale.CHINESE);
        System.out.println("格式化带Locale，输出==> 2017年07月22日 07:22:45 星期六   --->  " + string\_c);

        DateTime dt1 = new DateTime();

        
        DateTime dt2 = DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss").parseDateTime("2012-12-26 03:27:39");

        
        LocalDate start = new LocalDate(2017, 7, 22);
        LocalDate end = new LocalDate(2018, 6, 6);
        int days = Days.daysBetween(start, end).getDays();

        

        
        DateTime dateTime1 = DateTime.parse("2017-7-22");
        dateTime1 = dateTime1.plusDays(30);
        dateTime1 = dateTime1.plusHours(3);
        dateTime1 = dateTime1.plusMinutes(3);
        dateTime1 = dateTime1.plusMonths(2);
        dateTime1 = dateTime1.plusSeconds(4);
        dateTime1 = dateTime1.plusWeeks(5);
        dateTime1 = dateTime1.plusYears(3);

        

        dateTime = dateTime.plusDays(1) 
                .plusYears(1)
                .plusMonths(1)
                .plusWeeks(1)
                .minusMillis(1)
                .minusHours(1)
                .minusSeconds(1);

        
        DateTime dt4 = new DateTime();
        org.joda.time.DateTime.Property month = dt4.monthOfYear();
        System.out.println("是否闰月:" + month.isLeap());

        
        DateTime dt5 = dateTime1.secondOfMinute().addToCopy(-3);
        dateTime1.getSecondOfMinute();
        dateTime1.getSecondOfDay();
        dateTime1.secondOfMinute();

        
        DateTime dt6 = new DateTime(new Date());
        Date date = dateTime1.toDate();
        DateTime dt7 = new DateTime(System.currentTimeMillis());
        dateTime1.getMillis();

        Calendar calendar = Calendar.getInstance();
        dateTime = new DateTime(calendar);
    }

}
```

# 8. 实战
```
import org.joda.time.DateTime;
import org.joda.time.Days;
import org.joda.time.LocalDate;
import org.joda.time.format.DateTimeFormat;
import org.joda.time.format.DateTimeFormatter;

import java.util.Scanner;

public class CalBabyJoda {

    private final static String birthday = "2016-10-23 13:23:54";

    public static void main(String\[\] args) {
        while (true) {
            Scanner s = new Scanner(System.in);
            System.out.println("########################################");
            DateTimeFormatter format1 = DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss");
            DateTimeFormatter format2 = DateTimeFormat.forPattern("yyyy-MM-dd");
            DateTime startDateTime = DateTime.parse(birthday, format1);
            System.out.println("宝宝来到这个世界已经");
            calDateToDay(startDateTime, new DateTime());
            System.out.println("如选择其它日期请输入(格式例如:2016-10-23 13:23:54或着2016-10-23)");
            System.out.println("########################################");
            String endDate = s.nextLine();
            DateTime endDateTime = null;
            try {
                endDateTime = DateTime.parse(endDate, format1);
            } catch (Exception e) {
                try {
                    endDateTime = DateTime.parse(endDate, format2);
                } catch (Exception e1) {
                    System.out.println("输入格式错误!请重新输入.");
                    continue;
                }
            }
            System.out.println("宝宝从出生到" + endDateTime.toString("yyyy-MM-dd HH:mm:ss") + "已经");
            calDateToDay(startDateTime, endDateTime);
        }
    }

    public static void calDateToDay(DateTime startDateTime, DateTime endDateTime) {

        LocalDate start = new LocalDate(startDateTime);
        LocalDate end = new LocalDate(endDateTime);
        Days days = Days.daysBetween(start, end);
        int intervalDays = days.getDays();
        int intervalHours = endDateTime.getHourOfDay() - startDateTime.getHourOfDay();
        int intervalMinutes = endDateTime.getMinuteOfHour() - startDateTime.getMinuteOfHour();
        int intervalSeconds = endDateTime.getSecondOfMinute() - startDateTime.getSecondOfMinute();


        if (intervalSeconds < 0) {
            intervalMinutes = intervalMinutes - 1;
            intervalSeconds = 60 + intervalSeconds;
        }

        if (intervalMinutes < 0) {
            intervalHours = intervalHours - 1;
            intervalMinutes = 60 + intervalMinutes;
        }

        if (intervalHours < 0) {
            intervalDays = intervalDays - 1;
            intervalHours = 24 + intervalHours;
        }

        System.out.println(intervalDays + "天" + intervalHours +
                "小时" + intervalMinutes + "分钟" + intervalSeconds + "秒");
        System.out.println("############################");
    }

}
```

http://www.jianshu.com/p/efdeda608780  
http://ylq365.iteye.com/blog/1769680