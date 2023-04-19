[TOC]

来源：https://blog.csdn.net/ac_dao_di/article/details/53750028

参考：[Google guava工具类的介绍和使用](https://blog.csdn.net/wwwdc1012/article/details/82228458)<br>
[参考代码](https://github.com/whirlys/elastic-example/tree/master/guava)<br>
[guava文档](https://github.com/google/guava/wiki)

# 1. 概述

Guava是对Java API的补充，对Java开发中常用功能进行更优雅的实现，使得编码更加轻松，代码容易理解。Guava使用了多种设计模式，同时经过了很多测试，得到了越来越多开发团队的青睐。Java最新版本的API采纳了Guava的部分功能，但依旧无法替代。本文以Getting Started With Google Guava原文为学习材料，对Guava中常用的API进行学习，尽量覆盖比较有用的API，包括字符串处理，集合类处理，文件IO处理等。

# 2. 字符串连接器Joiner
## 2.1 连接多个字符串并追加到StringBuilder

```Java
     StringBuilder stringBuilder = new StringBuilder("hello");
     // 字符串连接器，以|为分隔符，同时去掉null元素
     Joiner joiner1 = Joiner.on("|").skipNulls();
     // 构成一个字符串foo|bar|baz并添加到stringBuilder
     stringBuilder = joiner1.appendTo(stringBuilder, "foo", "bar", null, "baz");
     System.out.println(stringBuilder); // ellofoo|bar|baz
```

## 2.2 连接List元素并写到文件流

```Java
   FileWriter fileWriter = null;
    try{
     fileWriter = new FileWriter(new File("/home/gzx/Documents/tmp.txt"));
    }
    catch(Exception e){
     System.out.println(e.getMessage());
    }
    List<Date> dateList = new ArrayList<Date>();
    dateList.add(new Date());
    dateList.add(null);
    dateList.add(new Date());
    // 构造连接器：如果有null元素，替换为no string
    Joiner joiner2 = Joiner.on("#").useForNull("no string");
    try{
     // 将list的元素的tostring()写到fileWriter，是否覆盖取决于fileWriter的打开方式，默认是覆盖，若有true，则是追加
     joiner2.appendTo(fileWriter, dateList);
     // 必须添加close()，否则不会写文件
     fileWriter.close();
    }
    catch(IOException e){
     System.out.println(e.getMessage());
    }
```

最后tmp.txt的内容为：<br>
Tue Dec 20 16:51:09 CST 2016#no string#Tue Dec 20 16:51:09 CST 2016

## 2.3 将Map转化为字符串

```java
    Map<String, String> testMap = Maps.newLinkedHashMap();
    testMap.put("Cookies", "12332");
    testMap.put("Content-Length", "30000");
    testMap.put("Date", "2016.12.16");
    testMap.put("Mime", "text/html");
    // 用:分割键值对，并用#分割每个元素，返回字符串
    String returnedString = Joiner.on("#").withKeyValueSeparator(":").join(testMap);
    System.out.println(returnedString); //Cookies:12332#Content-Length:30000#Date:2016.116#Mime:text/html
```

# 3. 字符串分割器Splitter
## 3.1 将字符串分割为Iterable

```java
    // 分割符为|，并去掉得到元素的前后空白
    Splitter sp = Splitter.on("|").trimResults();
    String str = "hello | world | your | Name ";
    Iterable<String> ss = sp.split(str);
    for(String it : ss){
     System.out.println(it);
    }
```

结果为：<br>
hello<br>
world<br>
your<br>
Name
## 3.2 将字符串转化为Map

```java
   // 内部类的引用，得到分割器，将字符串解析为map
   Splitter.MapSplitter ms = Splitter.on("#").withKeyValueSeparator(':');
   Map<String, String> ret = ms.split(returnedString);
   for(String it2 : ret.keySet()){
       System.out.println(it2 + " -> " + ret.get(it2));
   }
```

结果为：<br>
Cookies -> 12332<br>
Content-Length -> 30000<br>
Date -> 2016.12.16<br>
Mime -> text/html<br>

# 4. 字符串工具类Strings

```java
    System.out.println(Strings.isNullOrEmpty("")); // true
    System.out.println(Strings.isNullOrEmpty(null)); // true
    System.out.println(Strings.isNullOrEmpty("hello")); // false
    // 将null转化为""
    System.out.println(Strings.nullToEmpty(null)); // ""
 
    // 从尾部不断补充T只到总共8个字符，如果源字符串已经达到或操作，则原样返回。类似的有padStart
    System.out.println(Strings.padEnd("hello", 8, 'T')); // helloTTT
```

# 5. 字符匹配器CharMatcher
## 5.1 空白一一替换

```java
    // 空白回车换行对应换成一个#，一对一换
    String stringWithLinebreaks = "hello world\r\r\ryou are here\n\ntake it\t\t\teasy";
    String s6 = CharMatcher.BREAKING_WHITESPACE.replaceFrom(stringWithLinebreaks,'#');
    System.out.println(s6); // hello#world###you#are#here##take#it###easy
```

## 5.2 连续空白缩成一个字符

```java
    // 将所有连在一起的空白回车换行字符换成一个#，倒塌
    String tabString = "  hello   \n\t\tworld   you\r\nare             here  ";
    String tabRet = CharMatcher.WHITESPACE.collapseFrom(tabString, '#');
    System.out.println(tabRet); // #hello#world#you#are#here#
```

## 5.3 去掉前后空白和缩成一个字符

```java
    // 在前面的基础上去掉字符串的前后空白，并将空白换成一个#
    String trimRet = CharMatcher.WHITESPACE.trimAndCollapseFrom(tabString, '#');
    System.out.println(trimRet);// hello#world#you#are#here
```

## 5.4 保留数字


```java
    String letterAndNumber = "1234abcdABCD56789";
    // 保留数字
    String number = CharMatcher.JAVA_DIGIT.retainFrom(letterAndNumber);
    System.out.println(number);// 123456789
```

6. 断言工具类Preconditions   

```java
    // 检查是否为null，null将抛出异常IllegalArgumentException，且第二个参数为错误消息。
    trimRet = null;
    //Preconditions.checkNotNull(trimRet, "label can not be null");
    int data = 10;
    Preconditions.checkArgument(data < 100, "data must be less than 100");
```

# 7. 对象工具类 Objects
## 7.1 Objects的toStringHelper和hashCode方法

```java
    // Book用Objects的相关方法简化toString(),hashCode()的实现。
    // 用ComparisonChain简化compareTo()(Comparable接口)方法的实现。
    Book book1 = new Book();
    book1.setAuthor("Tom");
    book1.setTitle("Children King");
    book1.setIsbn("11341332443");
    System.out.println(book1);
    System.out.println(book1.hashCode());
 
    Book book2 = new Book();
    book2.setAuthor("Amy");
    book2.setTitle("Children King");
    book2.setIsbn("111");
    System.out.println(book2);
    System.out.println(book2.hashCode());
 
    System.out.println(book1.compareTo(book2));
```

结果为:<br>
Book{author=Tom, title=Children King, isbn=11341332443, price=0.0}<br>
268414056<br>
Book{author=Amy, title=Children King, isbn=111, price=0.0}<br>
-1726402621<br>
1<br>

## 7.2 Objects的firstNonNull方法

```java
    // 如果第一个为空，则返回第二个，同时为null，将抛出NullPointerException异常
    String someString = null;
    String value = Objects.firstNonNull(someString, "default value");
    System.out.println(value); // deafult value
```

**Book.java**

```java
class Book implements Comparable<Book>{
    private String author;
    private String title;
    private String publisher;
    private String isbn;
    private double price;
 
    public String getAuthor() {
        return author;
    }
 
    public void setAuthor(String author) {
        this.author = author;
    }
 
    public String getTitle() {
        return title;
    }
 
    public void setTitle(String title) {
        this.title = title;
    }
 
    public String getPublisher() {
        return publisher;
    }
 
    public void setPublisher(String publisher) {
        this.publisher = publisher;
    }
 
    public String getIsbn() {
        return isbn;
    }
 
    public void setIsbn(String isbn) {
        this.isbn = isbn;
    }
 
    public double getPrice() {
        return price;
    }
 
    public void setPrice(double price) {
        this.price = price;
    }
 
    // 定义第一二关键字
    @Override
    public int compareTo(Book o) {
        return ComparisonChain.start().compare(this.title, o.title).compare(this.isbn, o.isbn).result();
    }
 
    public String toString(){
        return Objects.toStringHelper(this).omitNullValues().add("author", author).add("title", title)
                .add("publisher", publisher).add("isbn", isbn).add("price", price).toString();
    }
 
    public int hashCode(){
        return Objects.hashCode(author, title, publisher, isbn, price);
    }
}
```

# 8. 整体迭代接口FluentIterable
## 8.1 使用Predicate整体过滤

```java
        Person person1 = new Person("Wilma", 30, "F");
        Person person2 = new Person("Fred", 32, "M");
        Person person3 = new Person("Betty", 32, "F");
        Person person4 = new Person("Barney", 33, "M");
        List<Person> personList = Lists.newArrayList(person1, person2, person3, person4);
 
        // 过滤年龄大于等于32的person
        Iterable<Person> personsFilteredByAge =
                FluentIterable.from(personList).filter(new Predicate<Person>() {
                    @Override
                    public boolean apply(Person input) {
                        return input.getAge() > 31;
                    }
                });
 
        // Iterable有一个iterator方法，集合类都有一个Iterator方法
        for(Iterator<Person> it = personsFilteredByAge.iterator(); it.hasNext();){
            System.out.println(it.next());
        }
        System.out.println(Iterables.contains(personsFilteredByAge, person2));
```

结果为：<br>
Person{name='Fred', sex='M', age=32}<br>
Person{name='Betty', sex='F', age=32}<br>
Person{name='Barney', sex='M', age=33}<br>
true<br>

## 8.2 使用Function整体替换，将List<Person>转化为List<String>

结果为：

Wilma#F#30<br>
Fred#M#32<br>
Betty#F#32<br>
Barney#M#33<br>

**Person.java**


```java
class Person{
    private String name;
    private String sex;
    private int age;
    public Person(String name, int age, String sex){
        this.name = name;
        this.sex = sex;
        this.age = age;
    }
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    public String getSex() {
        return sex;
    }
 
    public void setSex(String sex) {
        this.sex = sex;
    }
 
    public int getAge() {
        return age;
    }
 
    public void setAge(int age) {
        this.age = age;
    }
 
    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", sex='" + sex + '\'' +
                ", age=" + age +
                '}';
    }
}
```

# 9.  集合运算工具类Sets
## 9.1 集合差
 
```java
    // s1 - s2
    Set<String> s1 = Sets.newHashSet("1", "2", "3", "4");
    Set<String> s2 = Sets.newHashSet("2", "3", "4", "5");
    // 得到第一个集合中有而第二个集合没有的字符串
    Sets.SetView res = Sets.difference(s1, s2);
    for(Iterator<String> it = res.iterator(); it.hasNext();){
        System.out.println(it.next()); // 1
    }
```

## 9.2 集合对称差

```java
    Sets.SetView res2 = Sets.symmetricDifference(s1, s2);
        for(Object it14 : res2){
            System.out.println(it14); // 1 5
        }
```

## 9.3 集合交

```java
    // s1和s2的交集
     Sets.SetView<String> res3 = Sets.intersection(s1, s2);
     for(String it14 : res3){
         System.out.println(it14); // 2 3 4
     }
```

## 9.4 集合并

```java
     // 合并s1和s2
     Sets.SetView<String> res4 = Sets.union(s1, s2);
     for(String it14 : res4){
         System.out.println(it14); // 1 2 3 4 5
     }
```

# 10. Function和Predicate
## 10.1 利用Functions将Map转化为Function

```java
Map<String, Person> mp = Maps.newHashMap();
    mp.put(person1.getName(), person1);
    mp.put(person2.getName(), person2);
    mp.put(person3.getName(), person3);
    mp.put(person4.getName(), person4);
    // 将map转化为Function，Function的功能是将一个类型转化为另一个类型
    Function<String, Person> lookup = Functions.forMap(mp);
    // 如果键值不存在，则会抛出异常。lookup内部已经有元素
    Person tmp = lookup.apply("Betty");
    System.out.println(tmp == person3); // true
```

## 10.2 Predicate单个判断

```java
    Predicate<Person> agePre = new Predicate<Person>(){
        @Override
        public boolean apply(Person person) {
            return person.getAge() < 32;
        }
    };
    Predicate<Person> namePre = new Predicate<Person>(){
        @Override
        public boolean apply(Person person) {
            return person.getName().equals("Betty");
        }
    };
    // 判断是否符合条件
    System.out.println(agePre.apply(person2)); // false
    System.out.println(agePre.apply(person3)); // false
    System.out.println(namePre.apply(person2)); // false
    System.out.println(namePre.apply(person3)); // true
```

## 10.3 Predicates的and运算

```java
    // 利用Predicates工具类，同时满足两个条件成一个predicate
    Predicate<Person> both = Predicates.and(agePre, namePre);
    System.out.println(both.apply(person1)); // false
    System.out.println(both.apply(person3)); // false
```

## 10.4 Predicates的or运算

```java
    // 至少一个满足组成一个Predicate
    Predicate<Person> orPre = Predicates.or(agePre, namePre);
    System.out.println(orPre.apply(person2)); // false
```

## 10.5 Predicates的compose运算

```java
    // 通过键name获得值Person，然后检查Person的age < 32，即agepre.apply(lookup.apply(name)) == true?
    // lookup内部已经有集合
    Predicate<String> two = Predicates.compose(agePre, lookup);
    System.out.println(two.apply("Wilma")); // true
```

# 11. Map工具类Maps

```java
    // 将List<Person> 转化为Map<String, Person>，其中键值对是person.name -> Person
    Map<String, Person> myMp = Maps.uniqueIndex(personList.iterator(), new Function<Person, String>(){
        // name作为person的键
        @Override
        public String apply(Person person) {
            return person.getName();
        }
    });
    for(String name : myMp.keySet()){
        System.out.println(myMp.get(name));
    }
```

结果为：<br>
Person{name='Wilma', sex='F', age=30}<br>
Person{name='Fred', sex='M', age=32}<br>
Person{name='Betty', sex='F', age=32}<br>
Person{name='Barney', sex='M', age=33}<br>
# 12. 一键多值类Multimap
## 12.1 数组存储多值类ArrayListMultimap

```java
    // 用ArrayList保存，一键多值，值不会被覆盖
    ArrayListMultimap<String, String> multimap = ArrayListMultimap.create();
    multimap.put("foo", "1");
    multimap.put("foo", "2");
    multimap.put("foo", "3");
    multimap.put("bar", "a");
    multimap.put("bar", "a");
    multimap.put("bar", "b");
    for(String it20 : multimap.keySet()){
        // 返回类型List<String>
        System.out.println(it20 + " : " + multimap.get(it20));
    }
    // 返回所有ArrayList的元素个数的和
    System.out.println(multimap.size());
```

结果为：<br>
bar : [a, a, b]<br>
foo : [1, 2, 3]<br>
6<br>
## 12.2 HashTable存储多值类 HashMultimap

```java
    //这里采用HashTable保存
    HashMultimap<String, String> hashMultimap = HashMultimap.create();
    hashMultimap.put("foo", "1");
    hashMultimap.put("foo", "2");
    hashMultimap.put("foo", "3");
    // 重复的键值对值保留一个
    hashMultimap.put("bar", "a");
    hashMultimap.put("bar", "a");
    hashMultimap.put("bar", "b");
    for(String it20 : hashMultimap.keySet()){
     // 返回类型List<String>
     System.out.println(it20 + " : " + hashMultimap.get(it20));
    }
    // 5
    System.out.println(hashMultimap.size());
```

结果为：<br>
bar : [a, b]<br>
foo : [1, 2, 3]<br>
5<br>

# 13. 多键类Table
## 13.1 两个键操作

```java
  // 两个键row key和column key，其实就是map中map, map<Integer, map<Integer, String> > mp
  HashBasedTable<Integer, Integer, String> table = HashBasedTable.create();
  table.put(1, 1, "book");
  table.put(1, 2, "turkey");
  table.put(2, 2, "apple");
  System.out.println(table.get(1, 1)); // book
  System.out.println(table.contains(2, 3)); // false
  System.out.println(table.containsRow(2)); // true
  table.remove(2, 2);
  System.out.println(table.get(2, 2)); // null
```

## 13.2 获取一个Map

```java
    // 获取单独的一个map
    Map<Integer, String> row = table.row(1);
    Map<Integer, String> column = table.column(2);
    System.out.println(row.get(1)); // book
    System.out.println(column.get(1)); // turkey
```

# 14. 可以通过value获取key的HashBiMap 
## 14.1 value不可以有相同的key

```java
    BiMap<String, String> biMap = HashBiMap.create();
    // value可以作为Key，即value不可以有多个对应的值
    biMap.put("hello", "world");
    biMap.put("123", "tell");
    biMap.put("123", "none"); // 覆盖tell
    // biMap.put("abc", "world"); 失败
    // 下面是强制替换第一对
    biMap.forcePut("abc", "world");
    System.out.println(biMap.size()); // 2
    System.out.println(biMap.get("hello"));// null
    System.out.println(biMap.get("abc")); // world
    System.out.println(biMap.get("123")); // none
```

## 14.2 键值对互换得到新的BiMap

```java
    // 键值对互换
    BiMap<String, String> inverseMap = biMap.inverse();
    System.out.println(inverseMap.get("world")); // abc
    System.out.println(inverseMap.get("tell")); // null
    System.out.println(inverseMap.get(null)); // null
```

# 15. 不可变集合类ImmutableListMultimap

```java
    // 不可变的集合，都有一个Builder内部类。不可以修改和添加
    Multimap<Integer, String> map = new ImmutableListMultimap.Builder<Integer, String>().put(1, "hello")
            .putAll(2, "abc", "log", "in").putAll(3, "get", "up").build();
    System.out.println(map.get(2)); // [abc, log, in]
```

# 16. 区间工具类Range

```java
    // 闭区间
    Range<Integer> closedRange = Range.closed(30, 33);
    System.out.println(closedRange.contains(30)); // true
    System.out.println(closedRange.contains(33)); // true
 
    // 开区间
    Range<Integer> openRange = Range.open(30, 33);
    System.out.println(openRange.contains(30)); // false
    System.out.println(openRange.contains(33)); // false
 
    Function<Person, Integer> ageFunction = new Function<Person, Integer>(){
        @Override
        public Integer apply(Person person) {
            return person.getAge();
        }
    };
    // Range实现了Predicate接口，这里的第一个参数是Predicate，第二个参数是Function
    // ageFunction必须返回整数
    Predicate<Person> agePredicate = Predicates.compose(closedRange, ageFunction);
    System.out.println(agePredicate.apply(person1)); // person1.age == 30 true
```

# 17. 比较器工具类 Ordering
## 17.1 逆置比较器

```java
    // 自定义比较器，嵌入式的比较器，匿名类。注意这里有两个person参数，与Comparable的区别
    Comparator<Person> ageCmp = new Comparator<Person>(){
        // Ints是Guava提供的，递增
        @Override
        public int compare(Person o1, Person o2) {
            return Ints.compare(o1.getAge(), o2.getAge());
        }
    };
    List<Person> list = Lists.newArrayList(person1, person2, person3, person4);
    // 将比较器转化为Ordering，得到比较器ageCmp的相反比较器，递减
    Collections.sort(list, Ordering.from(ageCmp).reverse());
    for(Iterator<Person> iter = list.iterator(); iter.hasNext(); ){
            System.out.println(iter.next());
```

结果为：<br>
Person{name='Barney', sex='M', age=33}<br>
Person{name='Fred', sex='M', age=32}<br>
Person{name='Betty', sex='F', age=32}<br>
Person{name='Wilma', sex='F', age=30}<br>
## 17.2 组合多个比较器

```java
   // 按照名字排序
    Comparator<Person> nameCmp = new Comparator<Person>(){
        @Override // 两个对象，而Comparable是this和一个对象
        public int compare(Person o1, Person o2) {
            return o1.getName().compareTo(o2.getName());
        }
    };
    // 组合两个比较器，得到第一二排序关键字
    // 年龄相同时按照名字排序
    Ordering order = Ordering.from(ageCmp).compound(nameCmp);
    Collections.sort(list, order);
    for(Iterator<Person> iter = list.iterator(); iter.hasNext(); ){
        System.out.println(iter.next());
    }
```

结果为：

Person{name='Wilma', sex='F', age=30}<br>
Person{name='Betty', sex='F', age=32}<br>
Person{name='Fred', sex='M', age=32}<br>
Person{name='Barney', sex='M', age=33}<br>

## 17.3 直接获取最小几个和最大几个

```java
    Ordering order2 = Ordering.from(nameCmp);
    // 最小的两个，无序
    System.out.println("least 2...");
    List<Person> least = order2.leastOf(personList, 2);
    for(int i = 0; i < 2; i++){
        System.out.println(least.get(i));
    }
    // 最大的三个，无序
    System.out.println("greatest 3....");
    List<Person> great = order2.greatestOf(personList, 3);
    for(int i = 0; i < 3; i++){
        System.out.println(great.get(i));
    }
```

结果为:<br>
least 2...
Person{name='Barney', sex='M', age=33}<br>
Person{name='Betty', sex='F', age=32}<br>
greatest 3....<br>
Person{name='Wilma', sex='F', age=30}<br>
Person{name='Fred', sex='M', age=32}<br>
Person{name='Betty', sex='F', age=32}<br>
# 18. 文件工具类Files
## 18.1 复制移动重命名文件

```java
    // 文件操作：复制，移动，重命名
    File originFile = new File("/home/gzx/Documents/Program/Java/abc.java");
    File copyFile = new File("/home/gzx/Documents/test.java");
    File mvFile = new File("/home/gzx/Documents/abc.java");
    try {
        Files.copy(originFile, copyFile);
        Files.move(copyFile, mvFile); // 重命名
    }
    catch(IOException e){
        e.printStackTrace();
    }
```

## 18.2 获取文件哈希码

```java
  try {
        // File,HashFunction
        HashCode hashCode = Files.hash(originFile, Hashing.md5());
        System.out.println(originFile.getName() + " : " + hashCode);
    } catch (IOException e) {
        e.printStackTrace();
    }
```

结果为：<br>
abc.java : 66721c8573de09bd17bafac125e63e98
## 18.3 读取文件流，将文件行转化为List

```java
   // 读文件流
    int lineNumber = 1;
    try {
        // 读出所有的行到list中，去掉\n
        List<String> list2 = Files.readLines(mvFile, Charsets.UTF_8);
        for(Iterator<String> it = list2.iterator(); it.hasNext();){
            System.out.println("line " + lineNumber + ":" + it.next());
            lineNumber++;
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
```

结果为:

line 1:public class test{<br>
line 2: static String str;<br>
line 3: public static void main(String[] args){<br>
line 4:<br>
line 5: System.out.println(str);<br>
line 6: }<br>
line 7:}<br>

## 18.4 将文件行进行处理，再得到List

```java
   // LineProcessor处理每一行，得到返回值
    /*
        内容：
        Linux命令行大全,人民邮电出版社
        Linux内核完全注释,机械工业出版社
        Linux命令行和shell脚本编程大全,人民邮电出版社
     */
    File bookFile = new File("/home/gzx/Documents/book.txt");
    try {
        // 只取书名
        List<String> list3 = Files.readLines(bookFile, Charsets.UTF_8, new TitleLineProcessor());
        for(Iterator<String> it = list3.iterator(); it.hasNext();){
            System.out.println(it.next());
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
```

结果为:

Linux命令行大全<br>
Linux内核完全注释<br>
Linux命令行和shell脚本编程大全<br>

## 18.5 写文件流

```java
   // 写文件流
    File writeFile = new File("/home/gzx/Documents/write.txt");
    try {
        // 不必打开或关闭文件流，会自动写盘
        Files.write("hello world!", writeFile, Charsets.UTF_8); // 重新写
        Files.append("你的名字", writeFile, Charsets.UTF_8); // 追加
    } catch (IOException e) {
        e.printStackTrace();
    }
```

write.txt的内容为:

hello world!你的名字

**TitleLineProcessor.java**


```java
   class TitleLineProcessor implements LineProcessor<List<String>>{
    private final static int INDEX = 0;
    private final static Splitter splitter = Splitter.on(",");
    private List<String> titles = new ArrayList<String>();
    // 每一行都会调用这个函数，进而追加成一个list
    @Override
    public boolean processLine(String s) throws IOException {
        // 获取第一项，并追加到titles
        titles.add(Iterables.get(splitter.split(s), INDEX));
        return true;
    }
 
    // 最终的结果
    @Override
    public List<String> getResult() {
        return titles;
    }
}
```

# 19. 读输入字节流ByteSource和写输出字节流ByteSink

```java
    // source是源的意思，封装输入流
    ByteSource byteSource = Files.asByteSource(writeFile);
    try {
        byte[] contents1 = byteSource.read();
        byte[] contents2 = Files.toByteArray(writeFile); // 两个方法的作用相同
        for(int i = 0; i < contents1.length; i++){
            assert(contents1[i] == contents2[i]);
            System.out.print(contents1[i] + " ");
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
 
    // sink是目的地的意思，封装输出流，流会自动关闭
    File tmpFile = new File("/home/gzx/Documents/hello.txt"); // acd
    ByteSink byteSink = Files.asByteSink(tmpFile);
    try {
        byteSink.write(new byte[]{'a', 'c', 'd', '\n'});
    } catch (IOException e) {
        e.printStackTrace();
    }
```

# 20. 编码工具类BaseEncoding

```java
    File pdfFile = new File("/home/gzx/Documents/google.pdf");
    BaseEncoding baseEncoding = BaseEncoding.base64();
    try {
        byte[] content = Files.toByteArray(pdfFile);
        String encoded = baseEncoding.encode(content); // 将不可打印的字符串转化为可以打印的字符串A-Za-z0-9/+=，pdf不是纯文本文件
        System.out.println("encoded:\n" + encoded);
        System.out.println(Pattern.matches("[A-Za-z0-9/+=]+", encoded));
        // 获得对应的加密字符串，可以解密，可逆的，得到原来的字节
        byte[] decoded = baseEncoding.decode(encoded);
        for(int i = 0; i < content.length; i++){
            assert(content[i] == decoded[i]);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
```

# 21. 提醒处理null的类Optional

```java
    Optional<Person> optional = Optional.fromNullable(person1); // 允许参数为null
    System.out.println(optional.isPresent()); // true
    System.out.println(optional.get() == person1); // 如果是person1 == null，get将抛出IllegalStateException, true
 
    Optional<Person> optional2 = Optional.of(person1); // 不允许参数为null。如果person1 == null, 将抛出NullPointerException
    System.out.println(optional2.isPresent()); // true
```

