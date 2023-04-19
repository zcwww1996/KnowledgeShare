[TOC]
# 1、hdfs内容回顾

hdfs基本概念：hadoop提供的一个分布式文件系统

功能：帮用户存储海量文件

hdfs的特性：

1. hdfs是一个分布式系统，用户存储的文件被切块后存储在N台DATANODE服务器上；
2. 每一个文件的块信息及具体存储位置由NAMENODE记录
3. 文件的切块默认大小是：128M
4. 文件的默认副本数量：3

**<font style="background-color: yellow;color: red;">知识补充</font>**

> 文件的切块大小和副本数量不是由HDFS的服务端决定的，是由客户端决定的，具体来说，由客户端的这两个参数决定：
> 
> 块大小： dfs.blocksize = 128m
> 
> 副本数： dfs.replication = 3
> 
>  参数可以配置在客户端的什么地方？
> 
> 可以配置在客户端的hadoop安装目录的配置文件hdfs-site.xml中；
> 
> 可以在客户端的程序中设置；

**<font color="red">Hadoop压缩格式中“是否可切分”字段说明</font>**

https://blog.csdn.net/lin_wj1995/article/details/78967486

https://www.bbsmax.com/A/o75N69re5W/

|压缩格式|工具|算法|文件扩展名|压缩率|速度|native|是否可切分|Hadoop编码/解码器|
|---|---|---|---|---|---|---|---|---|
|DEFLATE|无|DEFLATE|.deflate|||是|否|org.apache.hadoop.io.compress.DefalutCodec|
|Gzip|gzip|DEFLATE|.gz|很高|比较快|是|否|org.apache.hadoop.io.compress.GzipCodec|
|bzip2|bzip2|bzip2|.bz2|最高|慢|否|<font color="red">是</font>|org.apache.hadoop.io.compress.Bzip2Codec|
|LZO|lzop|LZO|.lzo|比较高|很快|是|<font color="red">是(索引)</font>|com.hadoop.compression.lzo.LzoCodec|
|LZ4|无|LZ4|.lz4||||否|org.apache.hadoop.io.compress.Lz4Codec|
|Snappy|无|Snappy|.snappy|比较高|很快|是|否|org.apache.hadoop.io.compress.SnappyCodec|

**LZO为了支持split需要建切分点索引，还需要指定inputformat为lzo格式**

# 2、HDFS读写流程
## 2.1 HDFS读取数据流程
[![hdfs读数据.png](https://i0.wp.com/i.loli.net/2019/11/15/PCGABgwdEaHnoy2.png "hdfs读数据")](https://s2.ax1x.com/2020/01/03/lauZCj.png)

读数据详细过程：

1) 跟namenode通信查询元数据（block所在的datanode节点），找到文件块所在的datanode服务器

2) 挑选一台datanode（就近原则，然后随机）服务器，请求建立socket流

3) datanode开始发送数据（从磁盘里面读取数据放入流，以packet为单位来做校验）

4) 客户端以packet(一个包是64KB)为单位接收，先在本地缓存，然后写入目标文件，后面的block块就相当于是append到前面的block块最后合成最终需要的文件。

> **HDFS 在读取文件的时候,如果其中一个块突然损坏了怎么办?**
>
> 客户端读取完DataNode 上的块之后会进行checksum 验证，也就是把客户端读取到本地的块与HDFS 上的原始块进行校验。<br>
> 如果发现校验结果不一致，客户端会通知NameNode，然后再**从下一个拥有该 block 副本的DataNode继续读**。
## 2.2 HDFS写数据流程

[![hdfs写数据.png](https://i0.wp.com/i.loli.net/2019/11/15/hlcrdBgpuyqEzOK.png "hdfs写数据")](https://s2.ax1x.com/2020/01/03/lauE5Q.png)

写数据详细步骤：

1）跟namenode通信请求上传文件，namenode检查目标文件是否已存在，父目录是否存在

2）namenode返回是否可以上传

3）client会先对文件进行切分，比如一个blok块128m，文件有300m就会被切分成3个块，一个128M、一个128M、一个44M请求第一个 block该传输到哪些datanode服务器上

4）namenode返回datanode的服务器

5）client请求一台datanode上传数据（本质上是一个RPC调用，建立pipeline），第一个datanode收到请求会继续调用第二个datanode，然后第二个调用第三个datanode，将整个pipeline建立完成，逐级返回客户端

6）client开始往A上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位（一个packet为64kb），当然在写入的时候datanode会进行数据校验，它并不是通过一个packet进行一次校验而是以chunk为单位进行校验（512byte），第一台datanode收到一个packet就会传给第二台，第二台传给第三台；第一台每传一个packet会放入一个应答队列等待应答

7）当一个block传输完成之后，client再次请求namenode上传第二个block的服务器

> **HDFS 在上传文件的时候,如果其中一个DataNode 突然挂掉了怎么办？**
> 
> 客户端上传文件时与DataNode 建立pipeline 管道，管道正向是客户端向DataNode 发送的数据包，管道反向是DataNode 向客户端发送ack 确认，也就是正确接收到数据包之后发送一个已确认接收到的应答。<br>
> 当DataNode 突然挂掉了，客户端接收不到这个DataNode 发送的ack 确认，客户端会通知 NameNode，NameNode 检查该块的副本与规定的不符，**NameNode会通知DataNode 去复制副本，并将挂掉的DataNode作下线处理，不再让它参与文件上传与下载**。

# 3、HDFS数据可靠性
## 3.1 DataNode工作机制
[![DataNode工作机制.png](https://s1.ax1x.com/2023/02/20/pSO5139.png)](https://img.imgdb.cn/item/60546790524f85ce291c9ead.png)

1) 一个数据块在DataNode上以文件形式存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳。
2) DataNode启动后向NameNode注册，通过后，周期性（1小时）的向NameNode上报所有的块信息。
3) 心跳是每3秒一次，心跳返回结果带有NameNode给该DataNode的命令如复制块数据到另一台机器，或删除某个数据块。如果超过10分钟没有收到某个DataNode的心跳，则认为该节点不可用。
4) 集群运行中可以安全加入和退出一些机器。

**DataNode节点保证数据完整性的方法**
1) 当DataNode读取Block的时候，它会计算CheckSum
2) 如果计算后的CheckSum，与Block创建时值(第一次上传是会计算checksum值)不一样，说明Block已经损坏
3) Client读取其他DataNode上的Block
4) DataNode在其文件创建后周期验证CheckSum

[![image](https://s1.ax1x.com/2023/02/20/pSO5l9J.png)](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8zMzAxODUwLTY4YjAwN2E1ZTk0NmNlNjMucG5n)

## 3.2 HDFS是通过什么机制保证数据可靠性的？
主要有以下6点：

1.**安全模式：**<br>
HDFS刚启动时，namenode进入安全模式，处于安全模式的namenode不能做任何的文件操作，甚至内部的副本创建也是不允许的，namenode此时需要和各个datanode通信，获得datanode存储的数据块信息，并对数据块信息进行检查，只有通过了namenode的检查，一个数据块才被认为是安全的。当认为安全的数据块所占比例达到了某个阈值，namenode才会启动。

2.**SecondaryNamenode：**<br>
Hadoop中使用SecondaryNameNode来备份namenode的元数据，以便在namenode失效时能从SecondaryNameNode恢复出namenode上的元数据。SecondaryNameNode充当namenode的一个副本，它本身并不处理任何请求，因为处理这些请求都是NameNode的责任。

namenode中保存了整个文件系统的元数据，而SecondaryNameNode的作用就是周期性（周期长短也可配）保存NameNode的元数据。这些源数据中包括文件镜像数据FSImage和编辑日志EditLog。FSImage相当于HDFS的检查点，namenode启动时候会读取FSImage的内容到内存，并将其与EditLog日志中的所有修改信息合并生成新的FSImage；在namenode

运行过程中，所有关于HDFS的修改都将写入EditLog。这样，如果namenode失效，可以通过SecondaryNameNode中保存的FSImage和EditLog数据恢复出namenode最近的状态，尽量减少损失。

3.**心跳机制和副本重新创建**<br>
为了保证namenode和各个datanode的联系，HDFS采用了心跳机制。位于整个HDFS核心的namenode，通过周期性的活动来检查datanode的活性，像跳动的心脏一样。Namenode周期性向各个datanode发送心跳包，而收到心跳包的datanode要进行回复。因为心跳包是定时发送的，所以namenode就把要执行的命令也通过心跳包发送给datanode，而datanode收到心跳包，一方面回复namenode，另一方面就开始了用户或者应用的数据传输。

如果侦测到datanode失效，namenode之前保存在这个datanode上的数据就变成不可用数据。如果有的副本存储在失效的datanode上，则需要重新创建这个副本，放到另外可用的地方。

4.**数据一致性：**<br>
一般来讲，datanode与应用交互的大部分情况都是通过网络进行的，而网络数据传输带来的一大问题就是数据是否原样到达。为了保证数据的一致性，HDFS采用了数据校验和(checkSum)机制。创建文件时，HDFS会为这个文件生成一个校验和，校验和文件和文件本身保存在同一空间中。传输数据时会将数据与校验数据和一同传输，应用收到数据后可以进行校验，如果两个校验的结果不同，则文件肯定出错了，这个数据块就变成无效的。如果判定无效，则需要从其他datanode上读取副本。

5.**租约：**<br>
在linux中，为了防止多个进程向同一个文件写数据的情况，采用了文件加锁的机制。而在HDFS中，同样需要一个机制来防止同一个文件被多个人写入数据。这种机制就是租约(Lease)，每当写入数据之前，一个客户端必须获得namenode发放的一个租约。Namenode保证同一个文件只发放一个允许写的租约。那么就可以有效防止多人写入的情况。

6.**回滚：**<br>
HDFS安装或升级时，会将当前的版本信息保存起来，如果升级一段时间内运行正常，可以认为这次升级没有问题，重新保存版本信息，否则，根据保存的旧版本信息，将HDFS恢复至之前的版本。

# 4、 HDFS block数据块大小的设置规则
## 4.1 默认值
从2.7.3版本开始block size的默认大小为128M，之前版本的默认值是64M
## 4.2 如何修改block块的大小？

可以通过修改hdfs-site.xml文件中的dfs.blocksize对应的值。

注意：在修改HDFS的数据块大小时，首先停掉集群hadoop的运行进程，修改完毕后重新启动。

## 4.3 block块大小设置规则
首先我们先来了解几个概念：

1) 寻址时间：HDFS中找到目标文件block块所花费的时间。
2) 原理：文件块越大，寻址时间越短，但磁盘传输时间越长；文件块越小，寻址时间越长，但磁盘传输时间越短。
3) 普通文件系统的数据块大小一般为4KB

### 4.3.1 block不能设置过大，也不要能设置过小

**一、如果块设置过大**

1) 从磁盘传输数据的时间会明显大于寻址时间，导致程序在处理这块数据时，变得非常慢；
2) mapreduce中的map任务通常一次只处理一个块中的数据，如果块过大运行速度也会很慢。
3) 在数据读写计算的时候,需要进行网络传输.如果block过大会导致网络传输时间增长,程序卡顿/超时/无响应. 任务执行的过程中拉取其他节点的block或者失败重试的成本会过高.
4) namenode监管容易判断数据节点死亡.导致集群频繁产生/移除副本, 占用cpu,网络,内存资源.

> 由于数据块在硬盘上非连续存储，普通硬盘因为需要移动磁头，所以随机寻址较慢，读越多的数据块就增大了总的硬盘寻道时间。当硬盘寻道时间比io时间还要长的多时，那么硬盘寻道时间就成了系统的一个瓶颈。

**二、如果块设置过小**

1) 存放大量小文件会占用NameNode中大量内存来存储元数据，而NameNode的物理内存是有限的；
2) 文件块过小，寻址时间增大，导致程序一直在找block的开始位置。
3) 操作系统对目录中的小文件处理存在性能问题.比如同一个目录下文件数量操作100万,执行"fs -l "之类的命令会卡死.
4) 会频繁的进行文件传输,严重占用网络/CPU资源.

> **namenode内存大小由谁决定?**<br>
>  由集群中的块的数量决定；<br>
>  换算规则：默认情况下。每个block大小对应元数据为**150字节**。<br>
>  那么，如集群中存在1亿个块文件，元数据大小为1亿×150/(1024×1024×1024)=14G

### 4.3.2 block设置多大合适呢？

1) HDFS中平均寻址时间大概为10ms；
2) 经过前任的大量测试发现，寻址时间为传输时间的1%时，为最佳状态，所以最佳传输时间为：
10ms/0.01=1000s=1s
3) 目前磁盘的传输速度普遍为100MB/s， 网卡普遍为千兆网卡传输速率普遍也是100MB/s，最佳block大小计算：100MB/s×1s=100MB<br>
普通文件系统的数据块大小一般为4KB,所以我们设置block大小为128MB.
4) 实际中，磁盘传输速率为200MB/s时，一般设定block大小为256MB;磁盘传输速率为400MB/s时，一般设定block大小为512MB.


# 5、HDFS-API
## 5.1 HdfsClientDemo

```java
package top.ganhoo.hadoop.hdfs.demo;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.URI;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map.Entry;
import java.util.Set;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.BlockLocation;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.LocatedFileStatus;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.fs.RemoteIterator;
import org.junit.Before;
import org.junit.Test;

public class HdfsClientDemo {
	FileSystem fs = null;


    /*参数优先级排序：
    （1）客户端代码中设置的值
    （2）ClassPath下的用户自定义配置文件
    （3）然后是服务器的自定义配置（xxx-site.xml）
    （4）服务器的默认配置（xxx-default.xml）
   */

	@Before
	public void init() throws Exception {
		// 封装一些客户端参数
		Configuration conf = new Configuration();
		conf.set("dfs.blocksize", "64m");
		conf.setInt("dfs.replication", 1);

		// 先构造一个访问HDFS的工具对象，其内部封装了HDFS的URI、客户端参数、客户端身份等信息
		fs = FileSystem.get(new URI("hdfs://hdp26-01:9000"), conf, "root");
	}

	// 上传文件
	@Test
	public void testUpload() throws Exception {

		// 调用工具对象上的方法进行文件操作：上传
		fs.copyFromLocalFile(new Path("e:/红蜘蛛软件.zip"), new Path("/aaa/"));

		fs.close();
	}

	// 上传文件
	@Test
	public void testUpload2() throws Exception {
		// 封装一些客户端参数
		Configuration conf = new Configuration();
		conf.set("dfs.blocksize", "64m");
		conf.setInt("dfs.replication", 1);
		// 设置HDFS集群的URI
		conf.set("fs.defaultFS", "hdfs://hdp26-01:9000");
		// 设置当前系统环境中的hdfs客户端的用户名
		System.setProperty("HADOOP_USER_NAME", "root");

		// 先构造一个访问HDFS的工具对象，其内部封装了HDFS的URI、客户端参数、客户端身份等信息
		FileSystem fs = FileSystem.get(conf);

		// 调用工具对象上的方法进行文件操作：上传
		// 是否删除原数据，Override（覆盖？），原路径，目标路径
		fs.copyFromLocalFile(false,new Path("D:\\install-pkgs\\apache-flume-1.7.0-bin.tar.gz"), new Path("/"));

		fs.close();
	}
	
	
	// 下载文件
	@Test
	public void testDownload() throws Exception {
		// 本方法在往本地平台文件系统中写文件时，直接使用jdk的原生文件api机制
		// fs.copyToLocalFile(false,new Path("/start-dfs.sh"), new Path("d:/"),true);

		// 使用windows平台相关的工具进行本地文件操作
		// 1、需要在windows中放置一个hadoop安装包，并配置到HADOOP_HOME环境变量中
		// 2、需要在hadoop安装包中，替换windows平台下编译出来的bin目录
		
		// boolean useRawLocalFileSystem 是否开启crc文件校验
		fs.copyToLocalFile(false,new Path("/start-dfs.sh"), new Path("d:/"));

		fs.close();
	}

	// 创建目录
	@Test
	public void testMkDir() throws Exception {

		boolean mkdirs = fs.mkdirs(new Path("/eclipse/hdfs/"));
		System.out.println(mkdirs ? "创建完成" : "创建失败");

		fs.close();
	}

	// 删除
	@Test
	public void testRemove() throws Exception {
        // 第一个参数指的是删除文件对象，第二参数是指递归删除，一般用作删除目录
        // 如果要删除的路径是一个文件，参数recursive 是true还是false都行
        // 如果要删除的路径是一个目录，参数recursive如果填false，而下级目录还有文件，就会报错
		boolean delete = fs.delete(new Path("/eclipse"), true);
		System.out.println(delete ? "删了" : "没删");
		fs.close();
	}

	// 移动+改名
	@Test
	public void testRename() throws Exception {
		fs.rename(new Path("/spark.tgz"), new Path("/aaa/iloveyou.zip"));
		fs.close();
	}

	// 查询目录信息：只显示文件
	@Test
	public void testList() throws Exception {
		// 迭代器：就是一个用来取数据的工具，而且取数据的方式负责Iterator接口规范：hasNext()方法判断是否还有数据；next()方法可以拿一个数据
		RemoteIterator<LocatedFileStatus> iter = fs.listFiles(new Path("/"), false); // false那就只查看/下的文件，不查看/下的子目录中的文件信息

		while (iter.hasNext()) {
			LocatedFileStatus fileStatus = iter.next();

			System.out.println(fileStatus);

			System.out.println(fileStatus.getGroup()); // 文件所属组
			System.out.println(fileStatus.getOwner()); // 文件所有者
			System.out.println(fileStatus.getAccessTime()); // 文件最近访问时间
			System.out.println(fileStatus.getBlockSize()); // 文件的切块大小
			System.out.println(fileStatus.getLen()); // 文件总大小
			System.out.println(fileStatus.getModificationTime()); // 文件最后修改时间
			System.out.println(fileStatus.getPath()); // 文件的全路径
			System.out.println(fileStatus.getPermission()); // 文件的权限信息
			System.out.println(fileStatus.getReplication()); // 文件的副本数量
			System.out.println(fileStatus.isDirectory()); // 是否是目录
			System.out.println(fileStatus.isSymlink()); // 是否是链接
			System.out.println(fileStatus.isFile()); // 是否是文件

			BlockLocation[] blockLocations = fileStatus.getBlockLocations(); // 取文件的块信息
			for (BlockLocation blockLocation : blockLocations) {
				System.out.println("该块在文件中的起始偏移量：" + blockLocation.getOffset());
				System.out.println("该块的大小：" + blockLocation.getLength());
				System.out.println("该块在哪些datanode上：" + Arrays.toString(blockLocation.getHosts()));
			}

			System.out.println("----------------------风骚的分割线-----------------------------------");

		}

	}

	// 查询目录信息：包含文件夹
	@Test
	public void testList2() throws Exception {
		FileStatus[] listStatus = fs.listStatus(new Path("/"));
		for (FileStatus fileStatus : listStatus) {
			System.out.println(fileStatus.getPath());
		}

		fs.close();
	}
	
	 // 查询文件/目录大小
    @Test
    public void GetDirSize() throws Exception {
        Path filePath = new Path("/user/test/ce.txt.gz");
        Path dirPath = new Path("/hbase");
        // 会根据集群的配置输出，例如我这里输出3G
        System.out.println("CONFIGURATION SIZE OF THE HDFS DIRECTORY : " + fs.getContentSummary(filePath).getSpaceConsumed());
        // 显示实际的输出，例如这里显示 1G
        System.out.println("ACTUAL SIZE OF THE HDFS DIRECTORY : " + fs.getContentSummary(filePath).getLength());
        fs.close();
    }



	// 读文件内容
	// 加入hdfs中有一个文本文件，需要读取文件内容 并统计出其中每一个单词出现的总次数:wordcount
	@Test
	public void testReadFile() throws Exception {

		HashMap<String, Integer> map = new HashMap<>();

		FSDataInputStream in = fs.open(new Path("/words.txt"));
		BufferedReader br = new BufferedReader(new InputStreamReader(in));
		String line = null;
		while ((line = br.readLine()) != null) {

			String[] words = line.split(" ");
			for (String w : words) {
				if (map.containsKey(w)) {
					map.put(w, map.get(w) + 1);
				} else {
					map.put(w, 1);
				}
			}
		}

		System.out.println(map);

		// 将统计结果写入hdfs的文件

		// -- TODO -- 请加强该代码，实现写出去的结果文件中，单词统计按照单词次数倒序排序
		FSDataOutputStream out = fs.create(new Path("/wordcount.output"));
		Set<Entry<String, Integer>> entrySet = map.entrySet();
		for (Entry<String, Integer> entry : entrySet) {
			out.write((entry.getKey() + ":" + entry.getValue() + "\n").getBytes());
		}
		out.close();
		fs.close();

	}

	@Test
	public void testRandomRead() throws Exception {
		FSDataInputStream in = fs.open(new Path("/words.txt"));

		in.seek(10);

		byte[] b = new byte[10];

		in.read(b);

		System.out.println(new String(b));

	}

	// 往文件中写内容
	@Test
	public void testWriteDataToFile() throws Exception {

		// 写数据到文件中，需要一个输出流
		FSDataOutputStream out = fs.create(new Path("/wordcount.result"));

		out.write("i love you 迪丽热巴\n".getBytes("utf-8"));

		out.close();
		fs.close();
	}
}
```


Scala版

```Scala
import java.io._
import java.net.URI
import java.util._

import org.apache.commons.lang3.StringUtils
import org.apache.hadoop.conf.Configuration
import org.apache.hadoop.fs._
import org.apache.hadoop.io.IOUtils

/**
  * <Description> 通过scala操作HDFS<br>
  *
  * @author Sunny<br>
  * @taskId: <br>
  * @version 1.0<br>
  * @createDate 2018/06/11 9:29 <br>
  * @see com.spark.sunny.hdfs <br>
  */
object HDFSUtil {
  val hdfsUrl = "hdfs://iotsparkmaster:9000"
  var realUrl = ""

  /**
    * make a new dir in the hdfs
    *
    * @param dir the dir may like '/tmp/testdir'
    * @return boolean true-success, false-failed
    */
  def mkdir(dir : String) : Boolean = {
    var result = false
    if (StringUtils.isNoneBlank(dir)) {
      realUrl = hdfsUrl + dir
      val config = new Configuration()
      val fs = FileSystem.get(URI.create(realUrl), config)
      if (!fs.exists(new Path(realUrl))) {
        fs.mkdirs(new Path(realUrl))
      }
      fs.close()
      result = true
    }
    result
  }

  /**
    * delete a dir in the hdfs.
    * if dir not exists, it will throw FileNotFoundException
    *
    * @param dir the dir may like '/tmp/testdir'
    * @return boolean true-success, false-failed
    *
    */
  def deleteDir(dir : String) : Boolean = {
    var result = false
    if (StringUtils.isNoneBlank(dir)) {
      realUrl = hdfsUrl + dir
      val config = new Configuration()
      val fs = FileSystem.get(URI.create(realUrl), config)
      fs.delete(new Path(realUrl), true)
      fs.close()
      result = true
    }
    result
  }

  /**
    * list files/directories/links names under a directory, not include embed
    * objects
    *
    * @param dir a folder path may like '/tmp/testdir'
    * @return List<String> list of file names
    */
  def listAll(dir : String) : List[String] = {
    val names : List[String] = new ArrayList[String]()
    if (StringUtils.isNoneBlank(dir)) {
      realUrl = hdfsUrl + dir
      val config = new Configuration()
      val fs = FileSystem.get(URI.create(realUrl), config)
      val stats = fs.listStatus(new Path(realUrl))
      for (i <- 0 to stats.length - 1) {
        if (stats(i).isFile) {
          names.add(stats(i).getPath.toString)
        } else if (stats(i).isDirectory) {
          names.add(stats(i).getPath.toString)
        } else if (stats(i).isSymlink) {
          names.add(stats(i).getPath.toString)
        }
      }
    }
    names
  }
  
  
  
  /**
   * 过滤出含时间目录的集合
   * 路径内包含yyyy/MM/dd格式
   *
   * @param dir    文件目录
   * @param suffix 后缀
   * @return 获取指定目录格式的下一级文件目录集合
   */
  def listAll(dir: String, suffix: String): List[String] = {
    val paths = ListBuffer[String]()
    val realUrl = dir
    // val fs = FileSystem.get(new URI("hdfs://10.244.12.215:8020"), conf, "zhangchao");
    val stats: Array[FileStatus] = fs.listStatus(new Path(realUrl))
    for (i <- 0 until stats.length) {
      val r1: Regex = new Regex("\\d{4}\\/\\d{2}\\/\\d{2}\\" + suffix)
      val strPath = stats(i).getPath.toString + suffix
      if (stats(i).isDirectory && r1.pattern.matcher(strPath).find) {
        paths += strPath
      }
    }
    paths.toList
  }

  /**
     * upload the local file to the hds,
     * notice that the path is full like /tmp/test.txt
     * if local file not exists, it will throw a FileNotFoundException
     *
     * @param localFile local file path, may like F:/test.txt or /usr/local/test.txt
     *
     * @param hdfsFile hdfs file path, may like /tmp/dir
     * @return boolean true-success, false-failed
     *
     **/
  def uploadLocalFile2HDFS(localFile : String, hdfsFile : String) : Boolean = {
    var result = false
    if (StringUtils.isNoneBlank(localFile) && StringUtils.isNoneBlank(hdfsFile)) {
      realUrl = hdfsUrl + hdfsFile
      val config = new Configuration()
      val hdfs = FileSystem.get(URI.create(hdfsUrl), config)
      val src = new Path(localFile)
      val dst = new Path(realUrl)
      hdfs.copyFromLocalFile(src, dst)
      hdfs.close()
      result = true
    }
     result
  }

  /**
    * create a new file in the hdfs. notice that the toCreateFilePath is the full path
    *  and write the content to the hdfs file.

    * create a new file in the hdfs.
    * if dir not exists, it will create one
    *
    * @param newFile new file path, a full path name, may like '/tmp/test.txt'
    * @param content file content
    * @return boolean true-success, false-failed
    **/
  def createNewHDFSFile(newFile : String, content : String) : Boolean = {
    var result = false
    if (StringUtils.isNoneBlank(newFile) && null != content) {
      realUrl = hdfsUrl + newFile
      val config = new Configuration()
      val hdfs = FileSystem.get(URI.create(realUrl), config)
      val os = hdfs.create(new Path(realUrl))
      os.write(content.getBytes("UTF-8"))
      os.close()
      hdfs.close()
      result = true
    }
    result
  }

  /**
    * delete the hdfs file
    *
    * @param hdfsFile a full path name, may like '/tmp/test.txt'
    * @return boolean true-success, false-failed
    */
  def deleteHDFSFile(hdfsFile : String) : Boolean = {
    var result = false
    if (StringUtils.isNoneBlank(hdfsFile)) {
      realUrl = hdfsUrl + hdfsFile
      val config = new Configuration()
      val hdfs = FileSystem.get(URI.create(realUrl), config)
      val path = new Path(realUrl)
      val isDeleted = hdfs.delete(path, true)
      hdfs.close()
      result = isDeleted
    }
    result
  }

  /**
    * read the hdfs file content
    *
    * @param hdfsFile a full path name, may like '/tmp/test.txt'
    * @return byte[] file content
    */
  def readHDFSFile(hdfsFile : String) : Array[Byte] = {
    var result =  new Array[Byte](0)
    if (StringUtils.isNoneBlank(hdfsFile)) {
      realUrl = hdfsUrl + hdfsFile
      val config = new Configuration()
      val hdfs = FileSystem.get(URI.create(realUrl), config)
      val path = new Path(realUrl)
      if (hdfs.exists(path)) {
        val inputStream = hdfs.open(path)
        val stat = hdfs.getFileStatus(path)
        val length = stat.getLen.toInt
        val buffer = new Array[Byte](length)
        inputStream.readFully(buffer)
        inputStream.close()
        hdfs.close()
        result = buffer
      }
    }
    result
  }

  /**
    * append something to file dst
    *
    * @param hdfsFile a full path name, may like '/tmp/test.txt'
    * @param content string
    * @return boolean true-success, false-failed
    */
  def append(hdfsFile : String, content : String) : Boolean = {
    var result = false
    if (StringUtils.isNoneBlank(hdfsFile) && null != content) {
      realUrl = hdfsUrl + hdfsFile
      val config = new Configuration()
      config.set("dfs.client.block.write.replace-datanode-on-failure.policy", "NEVER")
      config.set("dfs.client.block.write.replace-datanode-on-failure.enable", "true")
      val hdfs = FileSystem.get(URI.create(realUrl), config)
      val path = new Path(realUrl)
      if (hdfs.exists(path)) {
        val inputStream = new ByteArrayInputStream(content.getBytes())
        val outputStream = hdfs.append(path)
        // 注意导包 import org.apache.hadoop.io.IOUtils
        IOUtils.copyBytes(inputStream, outputStream, 4096, true)
        outputStream.close()
        inputStream.close()
        hdfs.close()
        result = true
      }
    } else {
      HDFSUtil.createNewHDFSFile(hdfsFile, content);
      result = true
    }
    result
  }
  
  
    /**
   * hdfs复制文件
   *
   * @param src 源文件
   * @param dst 目标文件
   */
  def copyFile(src: String, dst: String): Unit = {
    // 方法1 需要导入org.apache.hadoop.fs下的包
    FileContext.getFileContext().util().copy(new Path(src), new Path(dst))
    // 方法2 支持不同源文件系统，需要导入org.apache.hadoop.fs下的包
    FileUtil.copy(fs, new Path(src), fs, new Path(dst), false, conf)
  }

}
```

## 5.2 读取hdfs目录下所有文件
java版

```java
public class ReadHDFS {
    // 必需的FileSystem类
    private static FileSystem fs = null;
    private static String DEFAULT_FS = "hdfs://linux01:9000/";

    public static void main(String[] args) {
        ArrayList<String> arrayList = new ArrayList<>();

        Configuration conf = new Configuration();
        // 单机伪分布式
        conf.set("fs.defaultFS", DEFAULT_FS);

        try {
            fs = FileSystem.get(conf);
            FileStatus[] fileStatuses = fs.listStatus(new Path("hdfs://linux01:9000/user/test"));
            for (FileStatus fileStatus : fileStatuses) {
                String path = fileStatus.getPath().toString();
                FSDataInputStream hdfsInStream = fs.open(new Path(path));
                InputStreamReader isr = new InputStreamReader(hdfsInStream, "utf-8");
                BufferedReader br = new BufferedReader(isr);
                String line = null;

                while ((line = br.readLine()) != null) {
                    arrayList.add(line);
                }
                br.close();
            }

            System.out.println(arrayList);
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}
```

scala版

```java
object ReadHDFS {
  private val  DEFAULT_FS = "hdfs://linux01:9000/"
  def main(args: Array[String]): Unit = {
    val hdfsPath = DEFAULT_FS+"user/test"
    val arrBuffer = new ArrayBuffer[String]()
    val conf = new Configuration()
    conf.set("fs.defaultFS", DEFAULT_FS)
    val fs = FileSystem.get(conf)
    val input_files = fs.listStatus(new Path(hdfsPath))
    val details: List[String] = input_files.map(m => m.getPath.toString).toList

    for (i <- 0 until details.length) {
      val file = details(i)
      var bufferedReader: BufferedReader = null
      val reader: FSDataInputStream = fs.open(new Path(file))
      try {
        bufferedReader = new BufferedReader(new InputStreamReader(reader))
        var line: String = bufferedReader.readLine()

        while (line != null) {
          arrBuffer += line
          line = bufferedReader.readLine()
        }

      }
      catch {
        case e: Exception => // todo: handle error
          e.printStackTrace()
      } finally {
        if (bufferedReader != null) {
          bufferedReader.close()
        }
      }
    }
    val array = arrBuffer.toArray

   array.foreach(println)

  }
}
```
