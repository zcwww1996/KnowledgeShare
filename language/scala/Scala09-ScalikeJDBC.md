[TOC]
# 1. 什么是 ScalikeJDBC 
 ScalikeJDBC 是一款给 Scala 开发者使用的简洁 DB 访问类库，它是基于 SQL 的，使用者只需要关注 SQL 逻辑的编写，所有的数据库操作都交给 ScalikeJDBC。这个类库内置包含了JDBC API，并且给用户提供了简单易用并且非常灵活的 API。

并且，QueryDSL(通用查询查询框架)使你的代码类型安全的并且可重复使用。我们可以在生产环境大胆地使用这款 DB 访问类库。 

# 2. 项目中使用 ScalikeJDBC
## 2.1. 添加依赖

```xml
<!-- scalikejdbc_2.11 -->
<dependency>
  <groupId>org.scalikejdbc</groupId>
  <artifactId>scalikejdbc_2.11</artifactId>
  <version>2.5.0</version>
</dependency>

<!-- scalikejdbc-config_2.11 -->
<dependency>
  <groupId>org.scalikejdbc</groupId>
  <artifactId>scalikejdbc-config_2.11</artifactId>
  <version>2.5.0</version> 
</dependency>

<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>5.1.38</version>
</dependency>
```

## 2.2 数据库 CURD 
### 2.2.1 配置数据库信息

```bash
# MySQL example

db.default.driver="com.mysql.jdbc.Driver"

db.default.url="jdbc:mysql://linux01:3306/bbs?characterEncoding= utf-8"

db.default.user="root" db.default.password="123456"
```

### 2.2.2 加载数据配置信息

```scala
//加载数据库配置信息

//默认加载 db.default
DBs.setup()
```
### 2.2.3 查询数据库并封装数据

```scala
// 加载数据库配置信息

// 默认加载 db.default.*

DBs.setup()

// 查询数据并返回单个列, 并将列数据封装到集合中 
    val list: List[String] = DB readOnly { implicit session => 
    SQL("select content from post").map(rs =>  rs.string("content")).list().apply()

}

for (s <- list ) {
  println(s)
}

// 用户实体

case class Users(id: String, name: String, nickName: String)

/**
 * 查询数据库,并将数据封装成对象,并返回一个集合
 */

// 初始化数据库链接

DBs.setup('sheep)

val userses: List[Users] = NamedDB('sheep) readOnly  { implicit session =>
 
  SQL("SELECT * from users").map(rs => Users(rs.string("id"),  rs.string("name"), rs.string("nickname"))).list().apply()

}

for (usr <- userses ) {
  println(usr.nickName)
}
```

### 2.2.4 插入数据

#### 2.2.4.1 AutoCommit

```scala
val insertResult: Int = DB.autoCommit { implicit session =>
    SQL("insert into users(name, nickname)  values(?,?)").bind("test01", "test01").update().apply()
}

println(insertResult)
```

#### 2.2.4.2 事务插入

```scala
val tx: Int = DB.localTx { implicit session =>

  SQL("INSERT INTO users(name, nickname, sex) VALUES  (?,?,?)").bind("犊子", "000", 1).update().apply()

  // var s = 1 / 0

  SQL("INSERT INTO users(name, nickname, sex)  values(?,?,?)").bind("王八犊子", "xxx", 0).update().apply()

}

println(s"tx = ${tx}")
```

### 2.2.5 更新数据

```scala
DB.localTx{ implicit session =>

  SQL("UPDATE users SET pwd = ?").bind("88999").update().apply()

}
```

# 3. 利用mysql存储kafaka偏移量
mysql配置文件 application.conf

```bash
# JDBC settings
db.default.driver="com.mysql.jdbc.Driver"
db.default.url="jdbc:mysql://linux01:3306/bigdata?characterEncoding=utf-8"
db.default.user="root"
db.default.password="pwd"
```
## 3.1 读取MySQL中的偏移量

```scala
//加载mysql配置文件
  DBs.setup()
      
    def fromOffset(): Map[TopicAndPartition, Long] = {
    DB.readOnly { implicit session =>
      SQL("SELECT * FROM kafka_offset WHERE topic=? AND groupId =?").bind(topic.head, consumer).map(
        t =>
          (TopicAndPartition(t.string("topic"), t.int("partitionId")), t.long("offset"))
      ).list().apply()
    }.toMap
  }
```
## 3.2 将偏移量写入MySQL（replace into 插入|更新）

```scala
  /**
    * @param ranges 偏移量范围（主题，分区,起始偏移量，结束偏移量）
    *"replace into 如果数据库原来没有，就等于insert into
    *              如果数据库有数值，就等于update
    */

//加载mysql配置文件
  DBs.setup()

  def offset2Mysql(ranges: Array[OffsetRange]): Unit = {
    DB.localTx { implicit session =>
      ranges.foreach(offset => {
        SQL("replace into kafka_offset(topic,partitionId,offset,groupId) values(?,?,?,?)").
          bind(offset.topic, offset.partition, offset.untilOffset, consumer).update().apply()
      })

    }
  }
```
