[TOC]
# 1. 如何给Mapper和Reducer传递参数

[![MapperReducer传参](https://c-ssl.duitang.com/uploads/item/201911/25/20191125100658_4Vuvk.thumb.700_0.jpeg "MapperReducer传参")](https://c-ssl.duitang.com/uploads/item/201911/25/20191125100658_4Vuvk.thumb.700_0.jpeg "MapperReducer传参")

通过**conf**参数对象，向maptask和reducetask传递参数

## 1.1 从main方法的参数中获取参数传递给conf

```java
if(args.length>0) {
		int topn  =  Integer.parseInt(args[0]);
			conf.setInt("rate.top.n", topn);
		}
```
## 1.2 在core-site.xml配置文件中获取参数传递给conf
不用写代码，只是需要写配置文件，`new Configuration()`会自动加载配置中的参数
方式3： 从自定义xml配置文件中加载参数后，放入conf

```java
conf.addResource("userdefine-conf.xml");
```

方式4： 从properties文件中加载参数后，放入conf

```java
Properties properties = new Properties();
properties.load(UserRateTopn.class.getClassLoader().getResourceAsStream("conf.properties"));
String property = properties.getProperty("rate_top_n");
```

# 2. mapreduce任务并行度
## 2.1 mapreduce框架如何设置reduce task的并行度
是由我们在Driver的代码中设置的：
job.setNumReduceTasks(3)


## 2.2 mapreduce框架如何设置map task的并行度
mapreduce框架会对输入目录中文件进行任务分配计算，计算的规则如下：

[![mapTask并行度](https://c-ssl.duitang.com/uploads/item/201911/25/20191125100658_dGNxt.thumb.700_0.jpeg "mapTask并行度")](https://c-ssl.duitang.com/uploads/item/201911/25/20191125100658_dGNxt.thumb.700_0.jpeg "mapTask并行度")

生成了几个split切片，就会有几个map task来对这些切片范围的数据进行处理；

split切片的规格大小（按多大范围划分任务）：

[![split切片大小](https://c-ssl.duitang.com/uploads/item/201911/25/20191125100658_GjYek.thumb.700_0.jpeg "split切片大小")](https://c-ssl.duitang.com/uploads/item/201911/25/20191125100658_GjYek.thumb.700_0.jpeg "split切片大小")

上图中的两个参数可以调控任务切分规格：

```shell
mapreduce.input.fileinputformat.split.maxsize
mapreduce.input.fileinputformat.split.minsize

如果将maxsize< blocksize  ==》 切片就按maxsize
如果将minsize> blocksize  ==》 切片就按minsize
```

# 3. MapReduce中Combiner的作用和用法
**(1).** 每个map可能会产生大量的输出，Combiner的作用就是在map端对输出**先做一次合并**，以减少传输到reducer的数据量。

**(2).** Combiner最基本是实现**本地key的归并**，Combiner具有类似本地的reduce功能。

如果不用Combiner，那么，所有的结果都是reduce完成，效率会相对低下。
 
使用Combiner，先完成的map会在本地聚合，提升速度。

**注意**：Combiner的输出是Reducer的输入，如果Combiner是可插拔的，添加Combiner绝不能改变最终的计算结果。所以Combiner只应该用于那种Reduce的输入key/value与输出key/value类型完全一致，且不影响最终结果的场景。比如累加，最大值等。

[![Combiner作用](https://c-ssl.duitang.com/uploads/item/201911/25/20191125100658_jkcyv.thumb.700_0.png "Combiner作用")](https://c-ssl.duitang.com/uploads/item/201911/25/20191125100658_jkcyv.thumb.700_0.png "Combiner作用")

# 4. mapreduce编程案例
## 4.1 mapreduce编程案例：文本索引创建
**涉及知识点**：

- setup方法

context.getInputSplit(); 获取当前所处理的切片信息


```java
/**
 * 学了两个新知识： setup()方法 -- 每个maptask程序会调用一次setup方法 context.getInputSplit() --
 * 获取当前这个maptask所处理的文件任务切片信息
 * 
 * @author hunter.ganhoo
 *
 */
public class IndexStepOne {

	public static class IndexStepOneMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

		String fileName = null;
		Text k = new Text();
		IntWritable v = new IntWritable(1);

		/**
		 * 这个方法，每个maptask只调用一次
		 */
		@Override
		protected void setup(Context context) throws IOException, InterruptedException {
			// InputSplit是用于描述map任务切片
			// maptask可以读文件（文本文件、图片文件、视频），也可以读mysql数据库，也可以读redis数据库，也可以读mongodb
			// 不同的数据形式，进行任务划分时，肯定有不同的属性
			FileSplit split = (FileSplit) context.getInputSplit();
			fileName = split.getPath().getName();
		}

		@Override
		protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

			String[] words = value.toString().split(" ");
			for (String word : words) {
				k.set(word + "-->" + fileName);
				context.write(k, v);

			}
		}
	}

	public static class IndexStepOneReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
		IntWritable v = new IntWritable();

		@Override
		protected void reduce(Text key, Iterable<IntWritable> values,
				Reducer<Text, IntWritable, Text, IntWritable>.Context context)
				throws IOException, InterruptedException {

			int count = 0;
			for (IntWritable value : values) {
				count += value.get();
			}

			v.set(count);
			context.write(key, v);
		}

	}

	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

		job.setJarByClass(IndexStepOne.class);

		job.setMapperClass(IndexStepOneMapper.class);
		job.setReducerClass(IndexStepOneReducer.class);

		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);

		FileInputFormat.setInputPaths(job, new Path("F:\\mrdata\\index\\input"));
		FileOutputFormat.setOutputPath(job, new Path("F:\\mrdata\\index\\out-1"));

		boolean res = job.waitForCompletion(true);
		System.exit(res ? 0 : -1);

	}

}
```


```java
public class IndexStepTwo {

	public static class IndexStepTwoMapper extends Mapper<LongWritable, Text, Text, Text> {
		// hello-->a.txt 4
		@Override
		protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, Text>.Context context)
				throws IOException, InterruptedException {
			String[] split = value.toString().split("-->");
			context.write(new Text(split[0]), new Text(split[1].replaceAll("\t", "-->")));
		}
	}

	public static class IndexStepTwoReducer extends Reducer<Text, Text, Text, Text> {

		@Override
		protected void reduce(Text key, Iterable<Text> values, Context context)
				throws IOException, InterruptedException {

			StringBuilder sb = new StringBuilder();
			for (Text value : values) {
				sb.append(value).append(" ");
			}

			context.write(key, new Text(sb.toString()));
		}
	}

	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

		job.setJarByClass(IndexStepTwo.class);

		job.setMapperClass(IndexStepTwoMapper.class);
		job.setReducerClass(IndexStepTwoReducer.class);

		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(Text.class);

		FileInputFormat.setInputPaths(job, new Path("F:\\mrdata\\index\\out-1"));
		FileOutputFormat.setOutputPath(job, new Path("F:\\mrdata\\index\\out-2"));

		boolean res = job.waitForCompletion(true);
		System.exit(res ? 0 : -1);
	}
}
```

## 4.2 mapreduce编程案例：join算法
JavaBean

```java
public class JoinBean implements Writable {
	// order001,u006,dingding,19,jiji
	private String orderid;
	private String uid;
	private String uname;
	private int age;
	private String lover;

	private String tableName;

	public void set(String orderid, String uid, String uname, int age, String lover, String tableName) {
		this.orderid = orderid;
		this.uid = uid;
		this.uname = uname;
		this.age = age;
		this.lover = lover;
		this.tableName = tableName;
	}

	public String getOrderid() {
		return orderid;
	}

	public void setOrderid(String orderid) {
		this.orderid = orderid;
	}

	public String getUid() {
		return uid;
	}

	public void setUid(String uid) {
		this.uid = uid;
	}

	public String getUname() {
		return uname;
	}

	public void setUname(String uname) {
		this.uname = uname;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	public String getLover() {
		return lover;
	}

	public void setLover(String lover) {
		this.lover = lover;
	}

	public String getTableName() {
		return tableName;
	}

	public void setTableName(String tableName) {
		this.tableName = tableName;
	}

	@Override
	public void readFields(DataInput in) throws IOException {
		this.orderid = in.readUTF();
		this.uid = in.readUTF();
		this.uname = in.readUTF();
		this.age = in.readInt();
		this.lover = in.readUTF();
		this.tableName = in.readUTF();

	}

	@Override
	public void write(DataOutput out) throws IOException {
		out.writeUTF(this.orderid);
		out.writeUTF(this.uid);
		out.writeUTF(this.uname);
		out.writeInt(this.age);
		out.writeUTF(this.lover);
		out.writeUTF(this.tableName);
	}

	@Override
	public String toString() {
		return "[orderid=" + orderid + ", uid=" + uid + ", uname=" + uname + ", age=" + age + ", lover=" + lover + "]";
	}
}
```

join

```java
public class OrderUserJoin {

	public static class OrderUserJoinMapper extends Mapper<LongWritable, Text, Text, JoinBean> {
		String tableName = null;
		JoinBean bean = new JoinBean();

		@Override
		protected void setup(Mapper<LongWritable, Text, Text, JoinBean>.Context context)
				throws IOException, InterruptedException {
			FileSplit inputSplit = (FileSplit) context.getInputSplit();
			tableName = inputSplit.getPath().getName();
		}

		@Override
		protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

			String[] split = value.toString().split(",");
			if (tableName.startsWith("order")) {
				bean.set(split[0], split[1], "N", -1, "N", "t_order"); // 填充缺失的数据是为了防止在序列化时出现空指针异常
			} else {
				bean.set("N", split[0], split[1], Integer.parseInt(split[2]), split[3], "t_user");
			}
			context.write(new Text(bean.getUid()), bean);
		}
	}

	public static class OrderUserJoinReducer extends Reducer<Text, JoinBean, JoinBean, NullWritable> {

		/*
		 * u006 <order001,u006,N,N,N,t_order> u006 <N,u006,dingding,19,jiji,t_user> u006
		 * <order001,u006,N,N,N,t_order> u006 <order001,u006,N,N,N,t_order>
		 */
		@Override
		protected void reduce(Text uid, Iterable<JoinBean> beans,
				Reducer<Text, JoinBean, JoinBean, NullWritable>.Context context)
				throws IOException, InterruptedException {

			JoinBean userBean = new JoinBean();
			ArrayList<JoinBean> orderBeanList = new ArrayList<>();

			// 分离两类数据，分别缓存
			for (JoinBean joinBean : beans) {

				if (joinBean.getTableName().equals("t_user")) {
					userBean.set(joinBean.getOrderid(), joinBean.getUid(), joinBean.getUname(), joinBean.getAge(),
							joinBean.getLover(), joinBean.getTableName());
					;
				} else {
					JoinBean newBean = new JoinBean();
					newBean.set(joinBean.getOrderid(), joinBean.getUid(), joinBean.getUname(), joinBean.getAge(),
							joinBean.getLover(), joinBean.getTableName());
					orderBeanList.add(newBean);
				}

			}

			// 将user数据拼接到order数据中，输出
			for (JoinBean orderBean : orderBeanList) {
				orderBean.setUname(userBean.getUname());
				orderBean.setAge(userBean.getAge());
				orderBean.setLover(userBean.getLover());
				context.write(orderBean, NullWritable.get());
			}
		}
	}

	public static void main(String[] args) throws Exception {

		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

		job.setJarByClass(OrderUserJoin.class);

		job.setMapperClass(OrderUserJoinMapper.class);
		job.setReducerClass(OrderUserJoinReducer.class);

		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(JoinBean.class);

		job.setOutputKeyClass(JoinBean.class);
		job.setOutputValueClass(NullWritable.class);

		FileInputFormat.setInputPaths(job, new Path("F:\\mrdata\\join\\input"));
		FileOutputFormat.setOutputPath(job, new Path("F:\\mrdata\\join\\join"));

		boolean res = job.waitForCompletion(true);
		System.exit(res ? 0 : -1);
	}
}
```

## 4.3 mapreduce编程案例：共同好友分析
 1 CommonFriendStepOne
 
```java
/**
 * 统计出每一个人，都是哪些人的共同好友
 * @author hunter.ganhoo
 *
 */
public class CommonFriendStepOne {
	
	public static class CommonFriendStepOneMapper extends Mapper<LongWritable, Text, Text, Text>{
		
		@Override
		protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, Text>.Context context)
				throws IOException, InterruptedException {
			//A:B,C,D,F,E,O 
			String[] split = value.toString().split(":");
			String user = split[0];
			String[] friends = split[1].split(",");
			for (String f : friends) {
				context.write(new Text(f), new Text(user));
			}
		}
	}
	
	public static class CommonFriendStepOneReducer extends Reducer<Text, Text, Text, Text>{
		
		
		@Override
		protected void reduce(Text f, Iterable<Text> users, Reducer<Text, Text, Text, Text>.Context context)
				throws IOException, InterruptedException {
			
			StringBuilder sb = new StringBuilder();
			for (Text u : users) {
				sb.append(u).append(",");
			}
			context.write(f, new Text(sb.toString()));
		}
	}
	
	
	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

		job.setJarByClass(CommonFriendStepOne.class);

		job.setMapperClass(CommonFriendStepOneMapper.class);
		job.setReducerClass(CommonFriendStepOneReducer.class);

		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(Text.class);

		FileInputFormat.setInputPaths(job, new Path("F:\\mrdata\\friends\\input"));
		FileOutputFormat.setOutputPath(job, new Path("F:\\mrdata\\friends\\out-1"));

		boolean res = job.waitForCompletion(true);
		System.exit(res ? 0 : -1);
	}
}
```

2 CommonFriendStepTwo

```java
**
 * 
 * @author hunter.ganhoo
 *
 */
public class CommonFriendsStepTwo {

	public static class CommonFriendsStepTwoMapper extends Mapper<LongWritable, Text, Text, Text> {

		@Override
		protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, Text>.Context context)
				throws IOException, InterruptedException {
			// A I,K,C,B,G,F,H,O,D,

			String[] split = value.toString().split("\t");
			String f = split[0];
			String[] users = split[1].split(",");
			Arrays.sort(users);

			for (int i = 0; i < users.length - 2; i++) {

				for (int j = i + 1; j < users.length - 1; j++) {

					context.write(new Text(users[i] + "-" + users[j]), new Text(f));
				}
			}
		}
	}

	public static class CommonFriendsStepTwoReducer extends Reducer<Text, Text, Text, Text> {

		@Override
		protected void reduce(Text userpair, Iterable<Text> friends, Reducer<Text, Text, Text, Text>.Context context)
				throws IOException, InterruptedException {

			StringBuilder sb = new StringBuilder();
			for (Text f : friends) {
				sb.append(f).append(",");
			}
			context.write(userpair, new Text(sb.substring(0, sb.length() - 1)));
		}

	}

	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

		job.setJarByClass(CommonFriendsStepTwo.class);

		job.setMapperClass(CommonFriendsStepTwoMapper.class);
		job.setReducerClass(CommonFriendsStepTwoReducer.class);

		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(Text.class);

		FileInputFormat.setInputPaths(job, new Path("F:\\mrdata\\friends\\out-1"));
		FileOutputFormat.setOutputPath(job, new Path("F:\\mrdata\\friends\\out-2"));

		boolean res = job.waitForCompletion(true);
		System.exit(res ? 0 : -1);
	}
}
```

## 4.4 mapreduce编程案例：数据倾斜

### 4.4.1 数据倾斜概念：

如果数据分布不均匀，则可能导致不同的reduce task负载不同，可能有些reduce task的负载比其他reduce task多很多倍。就会导致程序运算的整体效率降低；

 

### 4.4.2 如何解决数据倾斜带来的效率下降问题

1、<font style="background-color: yellow;color: red;">**最核心办法**</font>：

map所产生的key拼上一个随机前缀或后缀，以便于<font style="color: red;">将key打散</font>，分布均匀

2、使用combiner：

map task可以使用combiner将map()方法所产生的key-value进行局部的数据聚合；以减少需要交给reduce task的数据量；

注意：<font style="background-color: #80FFFF;">combiner要谨慎使用，只能用于幂等运算中，否则可能产生逻辑运算错误；</font>


```java
public class SkewWcStepOne {

	public static class SkewWcStepOneMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
		Random random = new Random();

		@Override
		protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
			String[] words = value.toString().split(" ");
			for (String word : words) {
				if (word.length() < 1)
					return;
				context.write(new Text(word + "-" + random.nextInt(3)), new IntWritable(1));

			}
		}
	}

	public static class SkewWcStepOneReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

		@Override
		protected void reduce(Text key, Iterable<IntWritable> values,
				Reducer<Text, IntWritable, Text, IntWritable>.Context context)
				throws IOException, InterruptedException {

			int count = 0;
			for (IntWritable value : values) {
				count += value.get();

			}
			context.write(key, new IntWritable(count));

		}

	}

	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

		job.setJarByClass(SkewWcStepOne.class);

		job.setMapperClass(SkewWcStepOneMapper.class);
		job.setReducerClass(SkewWcStepOneReducer.class);

		// 告诉maptask，需要做局部聚合，并使用如下combiner类
		job.setCombinerClass(WcCombiner.class);

		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);

		FileInputFormat.setInputPaths(job, new Path("F:\\mrdata\\skew\\input"));
		FileOutputFormat.setOutputPath(job, new Path("F:\\mrdata\\skew\\out-1"));

		job.setNumReduceTasks(3);

		boolean res = job.waitForCompletion(true);
		System.exit(res ? 0 : -1);
	}
}
```


```java
public class SkewWcStepTwo {

	public static class SkewWcStepTwoMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

		@Override
		protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
			// a-0 58
			String[] split = value.toString().split("\t");
			context.write(new Text(split[0].split("-")[0]), new IntWritable(Integer.parseInt(split[1])));

		}
	}

	public static class SkewWcStepTwoReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

		@Override
		protected void reduce(Text key, Iterable<IntWritable> values,
				Reducer<Text, IntWritable, Text, IntWritable>.Context context)
				throws IOException, InterruptedException {

			int count = 0;
			for (IntWritable value : values) {
				count += value.get();
			}
			context.write(key, new IntWritable(count));

		}

	}

	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

		job.setJarByClass(SkewWcStepTwo.class);

		job.setMapperClass(SkewWcStepTwoMapper.class);
		job.setReducerClass(SkewWcStepTwoReducer.class);

		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);

		FileInputFormat.setInputPaths(job, new Path("F:\\mrdata\\skew\\out-1"));
		FileOutputFormat.setOutputPath(job, new Path("F:\\mrdata\\skew\\out-2"));

		job.setNumReduceTasks(3);

		boolean res = job.waitForCompletion(true);
		System.exit(res ? 0 : -1);
	}
}
```


```java
public class WcCombiner extends Reducer<Text, IntWritable, Text, IntWritable> {

	@Override
	protected void reduce(Text key, Iterable<IntWritable> values,
			Reducer<Text, IntWritable, Text, IntWritable>.Context context) throws IOException, InterruptedException {

		int count = 0;
		for (IntWritable value : values) {
			count += value.get();
		}
		context.write(key, new IntWritable(count));

	}
}
```
