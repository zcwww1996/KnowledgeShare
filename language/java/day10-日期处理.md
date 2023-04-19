[TOC]

# 1. Calender

```java
public class CalenderDemo {

    Calendar calendar = Calendar.getInstance();

    @Test
    public void getTest() {
        ……
    }
    
}
```

## 1.1 获取今天或者之后多少天的日期

```Java
    @Test
    // 1，获取今天或者之后多少天的日期
    public void getTest() throws ParseException {
        calendar.setTime(new Date());
        /*获取今天的日期*/
        System.out.println(calendar.getTime());
        System.out.println("今天的日期是：" + calendar.get(Calendar.DAY_OF_MONTH));

        /*获取十天之后的日期*/
        calendar.clear();//避免继承当前系统的时间

        // SimpleDateFormat线程不安全，推荐org.apache.commons.lang3的FastDateFormat
        // SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        FastDateFormat fds = FastDateFormat.getInstance("yyyy-MM-dd HH:mm:ss");
        // 两种设置时间的方法，方法1由于年份减1900，月份减1（已过时）
        // 方法2使用了FastDateFormat
        // calendar.setTime(new Date(2019-1900,6-1,3,11,11,00));

         calendar.setTime(fds.parse("2019-06-03 11:01:00"));
        // 两种方法，add/set
        calendar.add(Calendar.DAY_OF_MONTH, 10);
        // calendar.set(Calendar.DAY_OF_MONTH, calendar.get(Calendar.DAY_OF_MONTH) + 10);

        System.out.println("十天之后的日期是：" + fds.format(calendar.getTime()));
        System.out.println("十天之后是：" + calendar.get(Calendar.DAY_OF_MONTH) + "日");
    }
```

## 1.2 计算某一个月的天数是多少
```java
    public void maxDay(int year, int month) {
        calendar.clear();
        calendar.set(Calendar.YEAR, year);
        calendar.set(Calendar.MONTH, month - 1);//默认1月为0月
        int day = calendar.getActualMaximum(Calendar.DAY_OF_MONTH);
        System.out.println(year + "年" + month + "月" + "的最大天数是：" + day);
    }

    @Test
    public void maxDayTest() {
        maxDay(2018, 9);
    }
```

## 1.3 计算某一天是该年或该月的第几个星期
```java
    public void weekNum(int year, int month, int day) {
        calendar.clear();
        calendar.set(Calendar.YEAR, year);
        calendar.set(Calendar.MONTH, month - 1);
        calendar.set(Calendar.DAY_OF_MONTH, day);
        /*计算某一天是该年的第几个星期*/
        int weekOfYear = calendar.get(Calendar.WEEK_OF_YEAR);
        System.out.println(year + "年" + month + "月" + day + "日是这年中的第" + weekOfYear + "个星期");
        /*计算某一天是该月的第几个星期*/
        int weekOfMonth = calendar.get(Calendar.WEEK_OF_MONTH);
        System.out.println(year + "年" + month + "月" + day + "日是这个月中的第" + weekOfMonth + "个星期");
        String[] weekDays = {"星期日", "星期一", "星期二", "星期三", "星期四", "星期五", "星期六"};
        int dayOfWeek = calendar.get(Calendar.DAY_OF_WEEK);
        //第一天是从星期日开始的，所以要-1
        System.out.println(dayOfWeek + weekDays[dayOfWeek - 1]);
    }

    @Test
    public void weekNumTest() {
        weekNum(2018, 5, 6);
    }
```

## 1.4 计算一年中的第几星期是几号
```java
    public void dayNum(int year, int week) {
        calendar.clear();
        SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        calendar.set(Calendar.YEAR, year);
        calendar.set(Calendar.WEEK_OF_YEAR, week);
        calendar.set(Calendar.DAY_OF_WEEK, Calendar.MONDAY);//设置一周内的周一这天
        String time = df.format(calendar.getTime());
        System.out.println(year + "年第" + week + "个星期是" + time);
    }

    @Test
    public void dayNumTest() {
        dayNum(2018, 19);
    }
```

## 1.5 查询显示当前的后几天，前几天等
```java
    /*roll()方法和add()方法用法一样，不过roll()方法是在本月循环，
    比如，七月三十一号加五天，add()方法结果是八月五号；
    roll()方法是七月五号，roll()方法用到的少，一般add()使用较多。*/
	// roll()方法将delta添加到字段f而不更改更大的字段。 
	// 这相当于调用add(f, delta),比f更大的字段不变，数值只在f字段内回滚
    public void add(int year, int month, int day, int num) {
        calendar.clear();
        SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        calendar.set(Calendar.YEAR, year);
        calendar.set(Calendar.MONTH, month - 1);
        calendar.set(Calendar.DAY_OF_MONTH, day);
        Date date = calendar.getTime();

        //   calendar.add(Calendar.YEAR, -1); // 年份减1
        //   calendar.add(Calendar.MONTH, 1);// 月份加1
        //   calendar.add(Calendar.DATE, -1);// 日期减1
        calendar.add(Calendar.DATE, num); //正加负减
        date = calendar.getTime();
        System.out.println(year + "-" + month + "-" + day + "往后推" + num + "天是" + df.format(date));
    }

    /*使用场景比如，发找回密码邮件，设置一天后过期*/
    @Test
    public void addTest() {
        add(2018, 3, 25, 10);
    }
```

## 1.6 计算两个任意时间中间相隔的天数
```java
    public int getDaysBetween(Calendar day1, Calendar day2) {
        if (day1.after(day2)) {
            Calendar swap = day1;
            day1 = day2;
            day2 = swap;
        }
        int days = day2.get(Calendar.DAY_OF_YEAR) - day1.get(Calendar.DAY_OF_YEAR);
        int y2 = day2.get(Calendar.YEAR);
        if (day1.get(Calendar.YEAR) != y2) {
            day1 = (Calendar) day1.clone();
            do {
                days += day1.getActualMaximum(Calendar.DAY_OF_YEAR);//得到当年的实际天数
                day1.add(Calendar.YEAR, 1);
            } while (day1.get(Calendar.YEAR) != y2);
        }
        return days;
    }

    @Test
    public void getDaysBetweenTest() {
        Calendar calendar1 = Calendar.getInstance();
        Calendar calendar2 = Calendar.getInstance();
        calendar1.set(2017, 04, 01);
        calendar2.set(2018, 04, 03);
        int days = getDaysBetween(calendar1, calendar2);
        System.out.println("相隔" + days + "天");
    }
```

# 2. DateTimeFormat

参考：https://blog.csdn.net/chenleixing/article/details/44408875

和SimpleDateFormat不同的是，DateTimeFormatter不但是不变对象，它还是<font color="red">**线程安全的**</font>。<br>
因为SimpleDateFormat不是线程安全的，使用的时候，只能在方法内部创建新的局部变量。
而DateTimeFormatter可以只创建一个实例，到处引用。

创建DateTimeFormatter时，我们仍然通过传入格式化字符串实现：


```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
```

格式化字符串的使用方式与SimpleDateFormat完全一致。

另一种创建DateTimeFormatter的方法是，传入格式化字符串时，同时指定Locale：

```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("E, yyyy-MMMM-dd HH:mm", Locale.US);
```

这种方式可以按照Locale默认习惯格式化

## 2.1 使用步骤

使用java类库

`import java.time.format.DateTimeFormatter;`

### 2.1.1 LocalDate转换

代码如下（示例）：

```java
        String dateStr= "2016年10月25日";
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy年MM月dd日");
        
        LocalDate date= LocalDate.parse(dateStr, formatter);
        System.out.println(date);
        String format1 = date.format(formatter);
        System.out.println(format1);
        
    // 2016-10-25
	// 2016年10月25日

```

### 2.1.2 LocalDateTime的转换

代码如下（示例）：

```java
     	String dateTimeStr= "2016-10-25 12:00:00";
        DateTimeFormatter formatter02 = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        LocalDateTime localDateTime=LocalDateTime.parse(dateTimeStr,formatter02);
        System.out.println(localDateTime);
        String format = localDateTime.format(formatter02);
        System.out.println(format);
        
	//	2016-10-25T12:00
	//	2016-10-25 12:00:00
```

### 2.1.3 Text <---> Date的转换

```Java
//Text-->Date
String dateStr= "2018-06-20 11:25:56";
DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
LocalDateTime localDateTime = LocalDateTime.parse(dateStr, dateTimeFormatter);
ZoneId zone = ZoneId.systemDefault();
Instant instant = localDateTime.atZone(zone).toInstant();
Date date = Date.from(instant);
println("转换出的date数据: " + date)

//Date-->Text
Date date = new Date();
Instant instant = date.toInstant();
ZoneId zone = ZoneId.systemDefault();
LocalDateTime localDateTime = LocalDateTime.ofInstant(instant, zone);
DateTimeFormatter format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String nowStr = localDateTime.format(format);
System.out.println(nowStr);

//LocalDateTime获取毫秒
System.out.println(localDateTime.toInstant(ZoneOffset.ofHours(8)).toEpochMilli());
System.out.println(localDateTime.toInstant(ZoneOffset.UTC).toEpochMilli());
```


### 2.1.4 返回n月前的时间

例子：返回20211231的上一月的时间20211130

```scala
      val strDateTime = "20211231"
      val format = "yyyyMMdd"
      val millis = DateTimeFormat.forPattern(format).parseDateTime(strDateTime)
      val time: DateTime = millis.minusMonths(1)
      val str = time.toString(format)
      println(str)
```

### 2.1.5 超高时间精度

```Scala
def dateToTimestamp(){
    val formatter = DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss.SSSSSS")
    val startTs=formatter.parseDateTime("2021-06-14 08:40:55.742646").getMillis
    assertEquals(startTs,1623631255742L)
  }
```

