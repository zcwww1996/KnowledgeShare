[TOC]

参考：https://blog.csdn.net/weixin_44865574/article/details/105240113

# 1. 影响JOIN操作的因素

## 1.1 数据集的大小

参与JOIN的数据集的大小会直接影响Join操作的执行效率。同样，也会影响JOIN机制的选择和JOIN的执行效率。

## 1.2 JOIN的条件

JOIN的条件会涉及字段之间的逻辑比较。根据JOIN的条件，JOIN可分为两大类：**等值连接**和**非等值连接**。等值连接会涉及一个或多个需要同时满足的相等条件。在两个输入数据集的属性之间应用每个等值条件。当使用其他运算符(运算连接符**不为 =** ,包括>、>=、<=、<、!>、!<和<>)时，称之为非等值连接。

## 1.3 JOIN的类型

在输入数据集的记录之间应用连接条件之后，JOIN类型会影响JOIN操作的结果。主要有以下几种JOIN类型：

- 内连接(_Inner Join_)：仅从输入数据集中输出匹配连接条件的记录。
- 外连接(_Outer Join_)：又分为左外连接、右外链接和全外连接。
- 半连接(_Semi Join_)：右表只用于过滤左表的数据而不出现在结果集中。
- 交叉连接(_Cross Join_)：交叉联接返回左表中的所有行，左表中的每一行与右表中的所有行组合。交叉联接也称作笛卡尔积。

# 2. Spark中JOIN执行的5种策略

Spark提供了5种JOIN机制来执行具体的JOIN操作。该5种JOIN机制如下所示：

- **Shuffle Hash Join**
- **Broadcast Hash Join**
- **Sort Merge Join**
- Cartesian Join
- Broadcast Nested Loop Join

## 2.1 Shuffle Hash Join

### 2.1.1 简介

当要JOIN的表数据量比较大时，可以选择Shuffle Hash Join。这样可以将大表进行**按照JOIN的key进行重分区**，保证每个相同的JOIN key都发送到同一个分区中。如下图示：

[![Shuffle Hash Join.png](https://img-blog.csdn.net/20181003185922498)](https://i0.wp.com/i.loli.net/2021/10/12/qeFBaHsObhGK7v4.png)


如上图所示：Shuffle Hash Join的基本步骤主要有以下两点：

*   首先，对于两张参与JOIN的表，分别按照join key进行重分区，该过程会涉及Shuffle，其目的是将相同join key的数据发送到同一个分区，方便分区内进行join。
*   其次，对于每个Shuffle之后的分区，会将小表的分区数据构建成一个Hash table，然后根据join key与大表的分区数据记录进行匹配。

### 2.1.2 条件与特点

*  仅支持等值连接，join key不需要排序
*  支持除了全外连接(full outer joins)之外的所有join类型
*  需要对小表构建Hash map，属于内存密集型的操作，如果构建Hash表的一侧数据比较大，可能会造成OOM
*  将参数 *spark.sql.join.prefersortmergeJoin (default true)* 置为false
*  一侧的表要明显小于另外一侧，小的一侧将被广播（根据经验值，明显小于的定义为大表的1/3）
*  在一定程度上减少了driver广播一侧表的压力，也减少了executor端取整张被广播表的内存消耗

## 2.2 Broadcast Hash Join

### 2.2.1 简介

也称之为**Map端JOIN**。当有一张表较小时，我们通常选择Broadcast Hash Join，这样可以避免Shuffle带来的开销，从而提高性能。

比如事实表与维表进行JOIN时，由于维表的数据通常会很小( < 10MB)，所以可以使用Broadcast Hash Join将维表进行Broadcast。这样可以避免数据的Shuffle(在Spark中Shuffle操作是很耗时的)，从而提高JOIN的效率。

在进行 Broadcast Join 之前，Spark 需要把处于 Executor 端的数据先发送到 Driver 端，然后 Driver 端再把数据广播到 Executor 端。如果我们需要广播的数据比较多，会造成 Driver 端出现 OOM。具体如下图示：

[![Broadcast Hash Join.png](https://img-blog.csdn.net/20181003185905898)](https://i0.wp.com/i.loli.net/2021/10/12/OhwDtisAcX1BFIq.png)


Broadcast Hash Join主要包括两个阶段：

*   **Broadcast**阶段：小表被缓存在executor中
*   **Hash Join**阶段：在每个 executor中执行Hash Join

[使用实例](https://www.javaroad.cn/questions/89067)

```Scala
//spark版本 < 2.2
largeDF.join(broadcast(smallDF), Seq("foo"))
//spark版本 >= 2.2 广播提示
largeDF.join(smallDF.hint("broadcast"), Seq("foo"))
```

```sql
SELECT /*+ MAPJOIN(small) */ *  FROM large JOIN small ON large.foo = small.foo

SELECT /*+  BROADCASTJOIN(small) */ * FROM large JOIN small ON large.foo = small.foo

SELECT /*+ BROADCAST(small) */ *  FROM large JOIN small ON larger.foo = small.foo
```




### 2.2.2 条件与特点

*   仅支持等值连接，join key不需要排序
*   支持除了全外连接(full outer joins)之外的所有join类型
*   Broadcast Hash Join相比其他的JOIN机制而言，效率更高。但是，Broadcast Hash Join属于网络密集型的操作(数据冗余传输)，除此之外，需要在Driver端缓存数据，所以当小表的数据量较大时，会出现OOM的情况
*   被广播的小表的数据量要小于**spark.sql.autoBroadcastJoinThreshold**值，默认是10MB(10485760)
*   被广播表的大小阈值**不能超过8GB**，spark2.4源码如下：**BroadcastExchangeExec.scala**

```scala
longMetric("dataSize") += dataSize
          if (dataSize >= (8L << 30)) {
            throw new SparkException(
              s"Cannot broadcast the table that is larger than 8GB: ${dataSize >> 30} GB")
          } 
```

*   基表不能被broadcast，比如左连接时，只能将右表进行广播。形如：`fact_table.join(broadcast(dimension_table)`，可以不使用**broadcast**提示，当满足条件时会自动转为该JOIN方式。

**问题汇总**：

> [SparkSql 2.2.x 中 Broadcast Join的陷阱(hint不生效)](https://blog.csdn.net/MrLevo520/article/details/104024814)
> 
> 1. spark 2.2.2的版本当小表join小表(两表都符合广播条件)，hint 指定广播表会失效，默认广播右表；若不hint，则默认广播右表
> 2. spark 2.4.3的版本可以指定(inner join)广播表(即使超过广播阈值,但小心OOM风险)；若不hint，则在符合广播阈值的条件下，使用较小的表进行广播
> 3. spark不支持full outer join；对于right outer join 只能广播左表；对于left outer join，left semi join，left anti join，internal join等只能广播右表，inner join 可以指定广播


## 2.3 Sort Merge Join

### 2.3.1 简介

该JOIN机制是Spark默认的，可以通过参数**spark.sql.join.preferSortMergeJoin**进行配置，默认是true，即优先使用Sort Merge Join。一般在两张大表进行JOIN时，使用该方式。Sort Merge Join可以减少集群中的数据传输，该方式不会先加载所有数据的到内存，然后进行hashjoin，但是在JOIN之前需要对join key进行排序。具体图示：

[![Sort Merge Join.png](https://img-blog.csdn.net/20181003185932690)](https://i0.wp.com/i.loli.net/2021/10/12/E9NpySxh8dJeQg2.png)

Sort Merge Join主要包括三个阶段：

*   **Shuffle Phase** : 两张大表根据Join key进行Shuffle重分区
*   **Sort Phase**: 每个分区内的数据进行排序
*   **Merge Phase**: 对来自不同表的排序好的分区数据进行JOIN，通过遍历元素，连接具有相同Join key值的行来合并数据集

### 2.3.2 条件与特点

*   仅支持等值连接
*   支持所有join类型
*   Join Keys是排序的
*   参数 **spark.sql.join.prefersortmergeJoin (默认true)** 设定为true

## 2.4 Cartesian Join

### 2.4.1 简介

如果 Spark 中两张参与 Join 的表没指定join key（ON 条件）那么会产生 Cartesian product join，这个 Join 得到的结果其实就是两张行数的乘积。

### 2.4.2 条件

*   仅支持内连接
*   支持等值和不等值连接
*   开启参数spark.sql.crossJoin.enabled=true

## 2.5 Broadcast Nested Loop Join

### 2.5.1 简介

该方式是在没有合适的JOIN机制可供选择时，最终会选择该种join策略。优先级为：_Broadcast Hash Join > Sort Merge Join > Shuffle Hash Join > cartesian Join > Broadcast Nested Loop Join_.

在Cartesian 与Broadcast Nested Loop Join之间，如果是内连接，或者非等值连接，则优先选择Broadcast Nested Loop策略，当时非等值连接并且一张表可以被广播时，会选择Cartesian Join。

### 2.5.2 条件与特点

*   支持等值和非等值连接
*   支持所有的JOIN类型，主要优化点如下：
    *   当右外连接时要广播左表
    *   当左外连接时要广播右表
    *   当内连接时，要广播左右两张表

# 3. Spark是如何选择JOIN策略的


## 3.1 等值连接的情况

### 3.1.1 有join提示(hints)的情况，按照下面的顺序

*   1.Broadcast Hint：如果join类型支持，则选择broadcast hash join
*   2.Sort merge hint：如果join key是排序的，则选择 sort-merge join
*   3.shuffle hash hint：如果join类型支持， 选择 shuffle hash join
*   4.shuffle replicate NL hint： 如果是内连接，选择笛卡尔积方式

### 3.1.2 没有join提示(hints)的情况，则逐个对照下面的规则

*   1.如果join类型支持，并且其中一张表能够被广播(**spark.sql.autoBroadcastJoinThreshold**值，默认是10MB)，则选择 broadcast hash join
*   2.如果参数**spark.sql.join.preferSortMergeJoin设定为false**，且一张表足够小(可以构建一个hash map) ，则选择shuffle hash join
*   3.如果join keys 是排序的，则选择sort-merge join
*   4.如果是内连接，选择 cartesian join
*   5.如果可能会发生OOM或者没有可以选择的执行策略，则最终选择broadcast nested loop join

## 3.2 非等值连接情况

### 3.2.1 有join提示(hints)，按照下面的顺序

*   1.broadcast hint：选择broadcast nested loop join.
*   2.shuffle replicate NL hint: 如果是内连接，则选择cartesian product join

### 3.2.2 没有join提示(hints)，则逐个对照下面的规则

*   1.如果一张表足够小(可以被广播)，则选择 broadcast nested loop join
*   2.如果是内连接，则选择cartesian product join
*   3.如果可能会发生OOM或者没有可以选择的执行策略，则最终选择broadcast nested loop join

## 3.3 join策略选择的源码片段

```scala
 object JoinSelection extends Strategy
    with PredicateHelper
    with JoinSelectionHelper {
    def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {

      case j @ ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, nonEquiCond, left, right, hint) =>
        def createBroadcastHashJoin(onlyLookingAtHint: Boolean) = {
          getBroadcastBuildSide(left, right, joinType, hint, onlyLookingAtHint, conf).map {
            buildSide =>
              Seq(joins.BroadcastHashJoinExec(
                leftKeys,
                rightKeys,
                joinType,
                buildSide,
                nonEquiCond,
                planLater(left),
                planLater(right)))
          }
        }

        def createShuffleHashJoin(onlyLookingAtHint: Boolean) = {
          getShuffleHashJoinBuildSide(left, right, joinType, hint, onlyLookingAtHint, conf).map {
            buildSide =>
              Seq(joins.ShuffledHashJoinExec(
                leftKeys,
                rightKeys,
                joinType,
                buildSide,
                nonEquiCond,
                planLater(left),
                planLater(right)))
          }
        }

        def createSortMergeJoin() = {
          if (RowOrdering.isOrderable(leftKeys)) {
            Some(Seq(joins.SortMergeJoinExec(
              leftKeys, rightKeys, joinType, nonEquiCond, planLater(left), planLater(right))))
          } else {
            None
          }
        }

        def createCartesianProduct() = {
          if (joinType.isInstanceOf[InnerLike]) {
            Some(Seq(joins.CartesianProductExec(planLater(left), planLater(right), j.condition)))
          } else {
            None
          }
        }

        def createJoinWithoutHint() = {
          createBroadcastHashJoin(false)
            .orElse {
              if (!conf.preferSortMergeJoin) {
                createShuffleHashJoin(false)
              } else {
                None
              }
            }
            .orElse(createSortMergeJoin())
            .orElse(createCartesianProduct())
            .getOrElse {
              val buildSide = getSmallerSide(left, right)
              Seq(joins.BroadcastNestedLoopJoinExec(
                planLater(left), planLater(right), buildSide, joinType, nonEquiCond))
            }
        }

        createBroadcastHashJoin(true)
          .orElse { if (hintToSortMergeJoin(hint)) createSortMergeJoin() else None }
          .orElse(createShuffleHashJoin(true))
          .orElse { if (hintToShuffleReplicateNL(hint)) createCartesianProduct() else None }
          .getOrElse(createJoinWithoutHint())

    
          if (canBuildLeft(joinType)) BuildLeft else BuildRight
        }

        def createBroadcastNLJoin(buildLeft: Boolean, buildRight: Boolean) = {
          val maybeBuildSide = if (buildLeft && buildRight) {
            Some(desiredBuildSide)
          } else if (buildLeft) {
            Some(BuildLeft)
          } else if (buildRight) {
            Some(BuildRight)
          } else {
            None
          }

          maybeBuildSide.map { buildSide =>
            Seq(joins.BroadcastNestedLoopJoinExec(
              planLater(left), planLater(right), buildSide, joinType, condition))
          }
        }

        def createCartesianProduct() = {
          if (joinType.isInstanceOf[InnerLike]) {
            Some(Seq(joins.CartesianProductExec(planLater(left), planLater(right), condition)))
          } else {
            None
          }
        }

        def createJoinWithoutHint() = {
          createBroadcastNLJoin(canBroadcastBySize(left, conf), canBroadcastBySize(right, conf))
            .orElse(createCartesianProduct())
            .getOrElse {
              Seq(joins.BroadcastNestedLoopJoinExec(
                planLater(left), planLater(right), desiredBuildSide, joinType, condition))
            }
        }

        createBroadcastNLJoin(hintToBroadcastLeft(hint), hintToBroadcastRight(hint))
          .orElse { if (hintToShuffleReplicateNL(hint)) createCartesianProduct() else None }
          .getOrElse(createJoinWithoutHint())
      case _ => Nil
    }
  } 
```

# 4. demo代码
```scala
import org.apache.spark.sql.{DataFrame, SparkSession}


object SparkSQLJoinDemo {
  def main(args: Array[String]): Unit = {
    //程序的入口
    val spark = SparkSession.builder().appName("SparkSQLJoinDemo").master("local[*]").getOrCreate()
    //导入spark对象的隐式转换
    import spark.implicits._



    //spark.sql.autoBroadcastJoinThreshold = -1
    //不限定小表的大小
    spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
    // 每个分区的平均大小不超过spark.sql.autoBroadcastJoinThreshold设定的值
    spark.conf.set("spark.sql.join.preferSortMergeJoin", true)

    println(spark.conf.get("spark.sql.autoBroadcastJoinThreshold"))


    //定义两个集合，转换成dataframe
    val df1: DataFrame = Seq(
      ("0", "a"),
      ("1", "b"),
      ("2", "c")
    ).toDF("id", "name")


    val df2: DataFrame = Seq(
      ("0", "d"),
      ("1", "e"),
      ("2", "f")
    ).toDF("aid", "aname")

    //重新分区
    df2.repartition()

    //df1.cache().count()

    //进行连接
    val result = df1.join(df2,$"id" === $"aid")

    //查看执行计划
    result.explain()

    //展示结果
    result.show()

    //释放资源
    spark.stop()

  }
}


```

