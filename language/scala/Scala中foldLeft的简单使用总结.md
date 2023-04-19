[TOC]

参考：

- [scala - 从合并两个Map说开去 - foldLeft 和 foldRight 还有模式匹配](https://www.cnblogs.com/tugeler/p/5134862.html)
- [Scala的foldLeft /：和foldRight :\的原理理解以及区别对照](https://blog.csdn.net/qq_41733481/article/details/90370649)
# 源码分析
def seq: TraversableOnce[A]
[![image](https://img-blog.csdnimg.cn/20181124120058694.png)](https://gd-hbimg.huaban.com/45ac1230560bf73c540718ff5c9aedc9c312b05b2a5b2-Z1Cdq0)

上面两段代码是`scala.collection.TraversableOnce`特质的foldLeft方法源代码,实现了TraversableOnce trait的seq就是可迭代的集合；

```
//将初始值z赋给result变量
    var result = z
    //遍历seq集合，将result变量与seq集合取出的每个对象作为参数传递给op函数，每迭代一次op函数的返回值都赋给result变量
    this.seq foreach (x => result = op(result, x))  
    //最终获取result作为返回值
    result
```

# 示例

List(1,2,3).foldLeft(<font color="red">0</font>)(<font color="blue">(sum,i)=>sum+i</font>)  // 红色部分是初始值，蓝色部分是操作函数

```scala
val list = List(1,2,3)

//0为初始值，sum表示返回结果对象（迭代值），i表示list集合中的每个值
val rs = list.foldLeft(0)((sum,i)=>{
      sum+i
    })

// 可以写成
(List(1,2,3) foldLeft 0)((sum,i)=>sum+i)
// 语法糖
(0 /: List(1,2,3))(_+_)  
```

斜杠是睡在冒号上的，斜杠在左就是左折叠，斜杠方为初始值，冒号方为列表
![image](https://gd-hbimg.huaban.com/e50d0a48211a40bbd4ed136aff70e317bcec1aae3146-n75Kry)

如果接受了这个设定。。。foldRight也可以脑补出来。。 一定是 `((1 to 5) :\ 100)((i,sum)=> sum-i)` .......

**左折叠的口诀 =>   列表从左，初始在左。**

**右折叠的口诀 =>   列表从右，初始在右。**

另外。一个例子说明  foldLeft 和 foldRight：

[![fold](https://s1.ax1x.com/2023/02/06/pSc0I39.png)](https://gd-hbimg.huaban.com/c25b4302c4dde2bfc68a0bead6a91a4933990174302f-EKFc7k)
运行过程为：

```
b=0+a，即0+1=1
b=1+a，即1+2=3
b=3+a，即3+3=6
此处的a为循环取出集合中的值
最终结果: rs=6
```
