[TOC]
# 1. mapreduce程序的运行模式(local、hdfs+yarn)
mapreduce程序除了可以在yarn上以分布式形式运行之外，也可以在一个jvm中以单进程模式本地运行；

mapreduce程序究竟让LocalJobRunner在**本地运行，还是在Yarn中以多进程分布式的方式运行**，由如下参数决定：

- mapreduce.framework.name = yarn--> 在yarn中分布式运行
- mapreduce.framework.name = local--> 在本地由LocalJobRunner多线程方式运行

mapreduce程序访问的数据输入、输出目录，究竟是**本地磁盘目录，还是HDFS中的目录**，由如下参数决定：

- fs.defaultFS = file:///  --> 访问的是本地磁盘文件系统
- fs.defaultFS = hdfs://hdp26-01:9000   --> 访问的是HDFS中的目录

这两个参数，在哪里设置呢？

- 方式1:在main方法中, conf.set("","")；
- 方式2:在Driver启动的机器的hadoop安装目录中，配到配置文件中；
- 方式3:在mr项目工程的src目录中，放入配置文件，也会加载；
 
参数配置位置的优先级：

<font style="background-color:LightBlue;">**代码中set > hadoop机器上的配置文件 > 项目工程中的配置文件 > 默认值**</font>



## 1.1 纯本地运行

conf里面不设任何参数，数据输入输出目录用本地目录；

windows中配HADOOP_HOME  和 PATH

**好处：方便debug**

## 1.2 半本地运行
<font style="color: red;">mr在LocalJobRunner中运行，但访问的文件是HDFS的</font>

- conf中设置: conf.set("fs.defaultFS","hdfs://hdp26-01:9000")
- 设置客户端身份为root:   System.setProperty("HADOOP_USER_NAME","root")
- 输入输出目录用：HDFS中的目录

## 1.3 从本地运行Driver提交job到YARN运行
conf中设置: 

```shell
conf.set(“fs.defaultFS”,”hdfs://hdp26-01:9000”)
conf.set(“mapreduce.framework.name”,”yarn”);
conf.set(“yarn.resourcemanager.hostname”,”hdp26-01”);
conf.set("mapreduce.app-submission.cross-platform", "true");
```

job.setJar(“d:/wc.jar”);   并且，将工程打包到d:盘，命名为wc.jar

## 1.4 在hadoop集群的机器上用hadoop jar x.jar运行
代码中可以不用设置任何参数；
但是，**机器上配置文件必须有上述那些参数的配置**

```shell
core-site.xml   --> fs.defaultFS = hdfs://hdp26-01:9000
mapred-site.xml -->mapreduce.framework.name = yarn
yarn-site.xml   -->yarn.resourcemanager.hostname = hdp26-01
```

# 2. 高级自定义模式
将bean作为key传递，自定义分区、分组

1 需求

有如下订单数据：
> order001,u001,小米6,1999.9,2
> 
> order001,u001,雀巢咖啡,99.0,2
> 
> order001,u001,安慕希,250.0,2
> 
> order001,u001,经典红双喜,200.0,4
> 
> order001,u001,防水电脑包,400.0,2
> 
> order002,u002,小米手环,199.0,3
> 
> order002,u002,榴莲,15.0,10
> 
> order002,u002,苹果,4.5,20
> 
> order002,u002,肥皂,10.0,40
> 
> order003,u001,小米6,1999.9,2
> 
> order003,u001,雀巢咖啡,99.0,2
> 
> order003,u001,安慕希,250.0,2
> 
> order003,u001,经典红双喜,200.0,4
> 
> order003,u001,防水电脑包,400.0,2

需要统计出每一个订单中成交金额最高的两件商品；

## 2.1 普通模式


```java
public class ItemAmountTopn {

	public static class ItemAmountTopnMapper extends Mapper<LongWritable, Text, Text, OrderBean> {
		@Override
		protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, OrderBean>.Context context)
				throws IOException, InterruptedException {

			String[] split = value.toString().split(",");
			OrderBean orderBean = new OrderBean(split[0], split[1], split[2], Float.parseFloat(split[3]),
					Integer.parseInt(split[4]));
			context.write(new Text(orderBean.getOrderId()), orderBean);

		}

	}

	public static class ItemAmountTopnReducer extends Reducer<Text, OrderBean, OrderBean, NullWritable> {

		/**
		 * order001,u001,小米6,1999.9,2 order001,u001,防水电脑包,400.0,2
		 * order001,u001,经典红双喜,200.0,4 order001,u001,雀巢咖啡,99.0,2
		 * order001,u001,安慕希,250.0,2
		 *
		 *
		 */
		@Override
		protected void reduce(Text orderId, Iterable<OrderBean> orderBeans, Context context)
				throws IOException, InterruptedException {

			ArrayList<OrderBean> beanList = new ArrayList<>();

			// 从迭代器中把这一组bean取出来，放入一个集合list
			for (OrderBean orderBean : orderBeans) {
				// 严重注意：不要把迭代器中的orderBean添加到list；因为每次迭代返回的orderBean其实是同一个对象，只是改了数据
				OrderBean newBean = new OrderBean(orderBean.getOrderId(), orderBean.getUserId(),
						orderBean.getItemName(), orderBean.getItemPrice(), orderBean.getNumber());
				beanList.add(newBean);
			}

			// 对list集合进行排序：按照每一个对象的成交金额倒序排序
			Collections.sort(beanList, new Comparator<OrderBean>() {

				@Override
				public int compare(OrderBean o1, OrderBean o2) {
					Float o1Amount = o1.getItemPrice() * o1.getNumber();
					Float o2Amount = o2.getItemPrice() * o2.getNumber();
					return o2Amount.compareTo(o1Amount) == 0 ? o1.getItemName().compareTo(o2.getItemName())
							: o2Amount.compareTo(o1Amount);
				}
			});

			for (int i = 0; i < 2; i++) {
				context.write(beanList.get(i), NullWritable.get());
			}

		}

	}

	public static void main(String[] args) throws Exception {

		Configuration conf = new Configuration();

		Job job = Job.getInstance(conf);

		job.setJarByClass(ItemAmountTopn.class);

		job.setMapperClass(ItemAmountTopnMapper.class);
		job.setReducerClass(ItemAmountTopnReducer.class);

		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(OrderBean.class);

		job.setOutputKeyClass(OrderBean.class);
		job.setOutputValueClass(NullWritable.class);

		FileInputFormat.setInputPaths(job, new Path("F:\\mrdata\\order\\input"));
		FileOutputFormat.setOutputPath(job, new Path("F:\\mrdata\\order\\output"));

		boolean res = job.waitForCompletion(true);

		System.exit(res ? 0 : -1);
	}
}
```

## 2.2 高级模式 ★★★

注意：

在mapreduce编程中，需要注意以下几个重要环节：

**1、** map阶段把数据分给不同的reduce时，会<font color="red">**分区**</font>，默认是按key的hashcode来分，有可能打乱你的业务逻辑；

你要控制，将需要分到一个reducetask的数据，保证能分到一个reducetask；

靠：<font style="background-color: Khaki;">Partitioner</font>的计算逻辑；

**2、** map阶段的数据发给reduce时，还会<font color="red">**排序**</font>，默认是按key.compareTo()比大小后排序；所以，你要根据自己的业务需求，控制key上的：<font style="background-color: Khaki;">compareTo()</font>方法，按你的需求比大小；

**3、** reduce task拿到数据后，在处理之前，首先会对数据<font color="red">**分组**</font>，每组调一次reduce方法；为了让它将正确的数据分成正确的组，需要控制分组比较规则：<font style="background-color: Khaki;">GroupingComparator</font>；


```java
package top.ganhoo.mr.ordertopn;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

import org.apache.hadoop.io.Writable;
import org.apache.hadoop.io.WritableComparable;

public class OrderBean implements WritableComparable<OrderBean> {

	// order001,u001,小米6,1999.9,2
	private String orderId;
	private String userId;
	private String itemName;
	private float itemPrice;
	private int number;

	public OrderBean() {
	}

	public void set(String orderId, String userId, String itemName, float itemPrice, int number) {
		this.orderId = orderId;
		this.userId = userId;
		this.itemName = itemName;
		this.itemPrice = itemPrice;
		this.number = number;
	}

	public OrderBean(String orderId, String userId, String itemName, float itemPrice, int number) {
		super();
		this.orderId = orderId;
		this.userId = userId;
		this.itemName = itemName;
		this.itemPrice = itemPrice;
		this.number = number;
	}

	public String getOrderId() {
		return orderId;
	}

	public void setOrderId(String orderId) {
		this.orderId = orderId;
	}

	public String getUserId() {
		return userId;
	}

	public void setUserId(String userId) {
		this.userId = userId;
	}

	public String getItemName() {
		return itemName;
	}

	public void setItemName(String itemName) {
		this.itemName = itemName;
	}

	public float getItemPrice() {
		return itemPrice;
	}

	public void setItemPrice(float itemPrice) {
		this.itemPrice = itemPrice;
	}

	public int getNumber() {
		return number;
	}

	public void setNumber(int number) {
		this.number = number;
	}

	@Override
	public void readFields(DataInput in) throws IOException {
		this.orderId = in.readUTF();
		this.userId = in.readUTF();
		this.itemName = in.readUTF();
		this.itemPrice = in.readFloat();
		this.number = in.readInt();
	}

	@Override
	public void write(DataOutput out) throws IOException {
		out.writeUTF(this.orderId);
		out.writeUTF(this.userId);
		out.writeUTF(this.itemName);
		out.writeFloat(this.itemPrice);
		out.writeInt(this.number);
	}

	@Override
	public String toString() {
		return this.orderId + "," + this.userId + "," + this.itemName + "," + this.itemPrice + "," + this.number;
	}

	/**
	 * 比大小
	 */
	@Override
	public int compareTo(OrderBean o) {
		Float amountSelf = this.getItemPrice() * this.getNumber();
		Float amountOther = o.getItemPrice() * o.getNumber();

		// 先比较订单id，保证相同id的订单排列在一起。再比较金额，保证相同id的订单中，金额大的排前面
		return this.getOrderId().compareTo(o.getOrderId()) == 0 ? amountOther.compareTo(amountSelf)
				: this.getOrderId().compareTo(o.getOrderId());
	}

}

```



```java
package top.ganhoo.mr.ordertopn;

import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class ItemAmountTopnEffecient {

	public static class ItemAmountTopnEffecientMapper extends Mapper<LongWritable, Text, OrderBean, NullWritable> {

		@Override
		protected void map(LongWritable key, Text value,
				Mapper<LongWritable, Text, OrderBean, NullWritable>.Context context)
				throws IOException, InterruptedException {

			String[] split = value.toString().split(",");
			OrderBean orderBean = new OrderBean(split[0], split[1], split[2], Float.parseFloat(split[3]),
					Integer.parseInt(split[4]));

			// 以自定义的bean作为key
			context.write(orderBean, NullWritable.get());

		}
	}

	public static class ItemAmountTopnEffecientReducer
			extends Reducer<OrderBean, NullWritable, OrderBean, NullWritable> {

		@Override
		protected void reduce(OrderBean key, Iterable<NullWritable> values,
				Reducer<OrderBean, NullWritable, OrderBean, NullWritable>.Context context)
				throws IOException, InterruptedException {
			int i = 0;
			for (NullWritable value : values) {
				context.write(key, NullWritable.get());
				if (++i == 2)
					return;
			}

		}

	}

	public static void main(String[] args) throws Exception {

		Configuration conf = new Configuration();

		Job job = Job.getInstance(conf);

		job.setJarByClass(ItemAmountTopnEffecient.class);

		job.setMapperClass(ItemAmountTopnEffecientMapper.class);
		job.setReducerClass(ItemAmountTopnEffecientReducer.class);

		job.setMapOutputKeyClass(OrderBean.class);
		job.setMapOutputValueClass(NullWritable.class);

		job.setOutputKeyClass(OrderBean.class);
		job.setOutputValueClass(NullWritable.class);

		// 设置本job的map分发数据到reduce时的分发规则逻辑类
		job.setPartitionerClass(OrderIdPartitioner.class);

		// 设置本job的reducetask对数据进行分组时的比较器逻辑类
		job.setGroupingComparatorClass(OrderIdGroupingComparator.class);

		FileInputFormat.setInputPaths(job, new Path("F:\\mrdata\\order\\input"));
		FileOutputFormat.setOutputPath(job, new Path("F:\\mrdata\\order\\output3"));

		job.setNumReduceTasks(2);

		boolean res = job.waitForCompletion(true);

		System.exit(res ? 0 : -1);

	}

}
```



```java
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.mapreduce.Partitioner;

/**
 * 根据orderid去分发数据的逻辑组件
 * @author hunter.ganhoo
 *
 */
public class OrderIdPartitioner extends Partitioner<OrderBean, NullWritable>{

	@Override
	public int getPartition(OrderBean key, NullWritable value, int numReduceTasks) {
		
		int partition = (key.getOrderId().hashCode() & Integer.MAX_VALUE) % numReduceTasks;
		
		return partition;
	}

}
```



```java
public class OrderIdGroupingComparator extends WritableComparator {

	public OrderIdGroupingComparator() {
		super(OrderBean.class, true); // 注册key的类型
	}

	/**
	 * reduce task会将收到的是二进制序列化的key数据，根据上面注册的类型，反序列化成两个对象，交给compare方法来比较
	 */
	@Override
	public int compare(WritableComparable a, WritableComparable b) {
		OrderBean bean1 = (OrderBean) a;
		OrderBean bean2 = (OrderBean) b;

		return bean1.getOrderId().compareTo(bean2.getOrderId());
	}

}
```
