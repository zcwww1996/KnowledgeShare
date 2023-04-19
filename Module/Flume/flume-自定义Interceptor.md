来源： https://blog.csdn.net/xiao_jun_0820/article/details/38333171

还是针对自定义source中的那个需求，我们现在换一种实现方式，采用拦截器来实现。

需求：读一个文件的时候，将文件的名字和文件生成的日期作为event的header传到hdfs上时，不同的event存到不同的目录下，如一个文件是a.log.2014-07-25在hdfs上是存到/a/2014-07-25目录下，a.log.2014-07-26存到/a/2014-07-26目录下，就是每个文件对应自己的目录。

先回想一下，spooldir source可以将文件名作为header中的key:basename写入到event的header当中去。试想一下，如果有一个拦截器可以拦截这个event,然后抽取header中这个key的值，将其拆分成3段，每一段都放入到header中，这样就可以实现那个需求了。

遗憾的是，flume没有提供可以拦截header的拦截器。不过有一个抽取body内容的拦截器：RegexExtractorInterceptor，看起来也很强大，以下是一个官方文档的示例：


```properties
If the Flume event body contained 1:2:3.4foobar5 and the following configuration was used


a1.sources.r1.interceptors.i1.regex = (\\d):(\\d):(\\d)
a1.sources.r1.interceptors.i1.serializers = s1 s2 s3
a1.sources.r1.interceptors.i1.serializers.s1.name = one
a1.sources.r1.interceptors.i1.serializers.s2.name = two
a1.sources.r1.interceptors.i1.serializers.s3.name = three
The extracted event will contain the same body but the following headers will have been added one=>1, two=>2, three=>3
```


大概意思就是，通过这样的配置，event body中如果有1:2:3.4foobar5 这样的内容，这会通过正则的规则抽取具体部分的内容，然后设置到header当中去。



于是决定打这个拦截器的主义，觉得只要把代码稍微改改，从拦截body改为拦截header中的具体key，就OK了。翻开源码，哎呀，很工整，改起来没难度，以下是我新增的一个拦截器：RegexExtractorExtInterceptor：

```java
package com.besttone.flume;

import java.util.List;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.commons.lang.StringUtils;
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.interceptor.Interceptor;
import org.apache.flume.interceptor.RegexExtractorInterceptorPassThroughSerializer;
import org.apache.flume.interceptor.RegexExtractorInterceptorSerializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.google.common.base.Charsets;
import com.google.common.base.Preconditions;
import com.google.common.base.Throwables;
import com.google.common.collect.Lists;

/**
 * Interceptor that extracts matches using a specified regular expression and
 * appends the matches to the event headers using the specified serializers</p>
 * Note that all regular expression matching occurs through Java's built in
 * java.util.regex package</p>. Properties:
 * <p>
 * regex: The regex to use
 * <p>
 * serializers: Specifies the group the serializer will be applied to, and the
 * name of the header that will be added. If no serializer is specified for a
 * group the default {@link RegexExtractorInterceptorPassThroughSerializer} will
 * be used
 * <p>
 * Sample config:
 * <p>
 * agent.sources.r1.channels = c1
 * <p>
 * agent.sources.r1.type = SEQ
 * <p>
 * agent.sources.r1.interceptors = i1
 * <p>
 * agent.sources.r1.interceptors.i1.type = REGEX_EXTRACTOR
 * <p>
 * agent.sources.r1.interceptors.i1.regex = (WARNING)|(ERROR)|(FATAL)
 * <p>
 * agent.sources.r1.interceptors.i1.serializers = s1 s2
 * agent.sources.r1.interceptors.i1.serializers.s1.type =
 * com.blah.SomeSerializer agent.sources.r1.interceptors.i1.serializers.s1.name
 * = warning agent.sources.r1.interceptors.i1.serializers.s2.type =
 * org.apache.flume.interceptor.RegexExtractorInterceptorTimestampSerializer
 * agent.sources.r1.interceptors.i1.serializers.s2.name = error
 * agent.sources.r1.interceptors.i1.serializers.s2.dateFormat = yyyy-MM-dd
 * </code>
 * </p>
 * 
 * <pre>
 * Example 1:
 * </p>
 * EventBody: 1:2:3.4foobar5</p> Configuration:
 * agent.sources.r1.interceptors.i1.regex = (\\d):(\\d):(\\d)
 * </p>
 * agent.sources.r1.interceptors.i1.serializers = s1 s2 s3
 * agent.sources.r1.interceptors.i1.serializers.s1.name = one
 * agent.sources.r1.interceptors.i1.serializers.s2.name = two
 * agent.sources.r1.interceptors.i1.serializers.s3.name = three
 * </p>
 * results in an event with the the following
 * 
 * body: 1:2:3.4foobar5 headers: one=>1, two=>2, three=3
 * 
 * Example 2:
 * 
 * EventBody: 1:2:3.4foobar5
 * 
 * Configuration: agent.sources.r1.interceptors.i1.regex = (\\d):(\\d):(\\d)
 * <p>
 * agent.sources.r1.interceptors.i1.serializers = s1 s2
 * agent.sources.r1.interceptors.i1.serializers.s1.name = one
 * agent.sources.r1.interceptors.i1.serializers.s2.name = two
 * <p>
 * 
 * results in an event with the the following
 * 
 * body: 1:2:3.4foobar5 headers: one=>1, two=>2
 * </pre>
 */
public class RegexExtractorExtInterceptor implements Interceptor {

	static final String REGEX = "regex";
	static final String SERIALIZERS = "serializers";

	// 增加代码开始

	static final String EXTRACTOR_HEADER = "extractorHeader";
	static final boolean DEFAULT_EXTRACTOR_HEADER = false;
	static final String EXTRACTOR_HEADER_KEY = "extractorHeaderKey";

	// 增加代码结束

	private static final Logger logger = LoggerFactory
			.getLogger(RegexExtractorExtInterceptor.class);

	private final Pattern regex;
	private final List<NameAndSerializer> serializers;

	// 增加代码开始

	private final boolean extractorHeader;
	private final String extractorHeaderKey;

	// 增加代码结束

	private RegexExtractorExtInterceptor(Pattern regex,
			List<NameAndSerializer> serializers, boolean extractorHeader,
			String extractorHeaderKey) {
		this.regex = regex;
		this.serializers = serializers;
		this.extractorHeader = extractorHeader;
		this.extractorHeaderKey = extractorHeaderKey;
	}

	@Override
	public void initialize() {
		// NO-OP...
	}

	@Override
	public void close() {
		// NO-OP...
	}

	@Override
	public Event intercept(Event event) {
		String tmpStr;
		if(extractorHeader)
		{
			tmpStr = event.getHeaders().get(extractorHeaderKey);
		}
		else
		{
			tmpStr=new String(event.getBody(),
					Charsets.UTF_8);
		}
		
		Matcher matcher = regex.matcher(tmpStr);
		Map<String, String> headers = event.getHeaders();
		if (matcher.find()) {
			for (int group = 0, count = matcher.groupCount(); group < count; group++) {
				int groupIndex = group + 1;
				if (groupIndex > serializers.size()) {
					if (logger.isDebugEnabled()) {
						logger.debug(
								"Skipping group {} to {} due to missing serializer",
								group, count);
					}
					break;
				}
				NameAndSerializer serializer = serializers.get(group);
				if (logger.isDebugEnabled()) {
					logger.debug("Serializing {} using {}",
							serializer.headerName, serializer.serializer);
				}
				headers.put(serializer.headerName, serializer.serializer
						.serialize(matcher.group(groupIndex)));
			}
		}
		return event;
	}

	@Override
	public List<Event> intercept(List<Event> events) {
		List<Event> intercepted = Lists.newArrayListWithCapacity(events.size());
		for (Event event : events) {
			Event interceptedEvent = intercept(event);
			if (interceptedEvent != null) {
				intercepted.add(interceptedEvent);
			}
		}
		return intercepted;
	}

	public static class Builder implements Interceptor.Builder {

		private Pattern regex;
		private List<NameAndSerializer> serializerList;

		// 增加代码开始

		private boolean extractorHeader;
		private String extractorHeaderKey;

		// 增加代码结束

		private final RegexExtractorInterceptorSerializer defaultSerializer = new RegexExtractorInterceptorPassThroughSerializer();

		@Override
		public void configure(Context context) {
			String regexString = context.getString(REGEX);
			Preconditions.checkArgument(!StringUtils.isEmpty(regexString),
					"Must supply a valid regex string");

			regex = Pattern.compile(regexString);
			regex.pattern();
			regex.matcher("").groupCount();
			configureSerializers(context);

			// 增加代码开始
			extractorHeader = context.getBoolean(EXTRACTOR_HEADER,
					DEFAULT_EXTRACTOR_HEADER);

			if (extractorHeader) {
				extractorHeaderKey = context.getString(EXTRACTOR_HEADER_KEY);
				Preconditions.checkArgument(
						!StringUtils.isEmpty(extractorHeaderKey),
						"必须指定要抽取内容的header key");
			}
			// 增加代码结束
		}

		private void configureSerializers(Context context) {
			String serializerListStr = context.getString(SERIALIZERS);
			Preconditions.checkArgument(
					!StringUtils.isEmpty(serializerListStr),
					"Must supply at least one name and serializer");

			String[] serializerNames = serializerListStr.split("\\s+");

			Context serializerContexts = new Context(
					context.getSubProperties(SERIALIZERS + "."));

			serializerList = Lists
					.newArrayListWithCapacity(serializerNames.length);
			for (String serializerName : serializerNames) {
				Context serializerContext = new Context(
						serializerContexts.getSubProperties(serializerName
								+ "."));
				String type = serializerContext.getString("type", "DEFAULT");
				String name = serializerContext.getString("name");
				Preconditions.checkArgument(!StringUtils.isEmpty(name),
						"Supplied name cannot be empty.");

				if ("DEFAULT".equals(type)) {
					serializerList.add(new NameAndSerializer(name,
							defaultSerializer));
				} else {
					serializerList.add(new NameAndSerializer(name,
							getCustomSerializer(type, serializerContext)));
				}
			}
		}

		private RegexExtractorInterceptorSerializer getCustomSerializer(
				String clazzName, Context context) {
			try {
				RegexExtractorInterceptorSerializer serializer = (RegexExtractorInterceptorSerializer) Class
						.forName(clazzName).newInstance();
				serializer.configure(context);
				return serializer;
			} catch (Exception e) {
				logger.error("Could not instantiate event serializer.", e);
				Throwables.propagate(e);
			}
			return defaultSerializer;
		}

		@Override
		public Interceptor build() {
			Preconditions.checkArgument(regex != null,
					"Regex pattern was misconfigured");
			Preconditions.checkArgument(serializerList.size() > 0,
					"Must supply a valid group match id list");
			return new RegexExtractorExtInterceptor(regex, serializerList,
					extractorHeader, extractorHeaderKey);
		}
	}

	static class NameAndSerializer {
		private final String headerName;
		private final RegexExtractorInterceptorSerializer serializer;

		public NameAndSerializer(String headerName,
				RegexExtractorInterceptorSerializer serializer) {
			this.headerName = headerName;
			this.serializer = serializer;
		}
	}
}

```

简单说明一下改动的内容：
增加了两个配置参数：

extractorHeader   是否抽取的是header部分，默认为false,即和原始的拦截器功能一致，抽取的是event body的内容

extractorHeaderKey 抽取的header的指定的key的内容，当extractorHeader为true时，必须指定该参数。

按照第八讲的方法，我们将该类打成jar包，作为flume的插件放到了/var/lib/flume-ng/plugins.d/RegexExtractorExtInterceptor/lib目录下，重新启动flume，将该拦截器加载到classpath中。

最终的flume.conf如下：

```properties
tier1.sources=source1
tier1.channels=channel1
tier1.sinks=sink1
tier1.sources.source1.type=spooldir
tier1.sources.source1.spoolDir=/opt/logs
tier1.sources.source1.fileHeader=true
tier1.sources.source1.basenameHeader=true
tier1.sources.source1.interceptors=i1
tier1.sources.source1.interceptors.i1.type=com.besttone.flume.RegexExtractorExtInterceptor$Builder
tier1.sources.source1.interceptors.i1.regex=(.*)\\.(.*)\\.(.*)
tier1.sources.source1.interceptors.i1.extractorHeader=true
tier1.sources.source1.interceptors.i1.extractorHeaderKey=basename
tier1.sources.source1.interceptors.i1.serializers=s1 s2 s3
tier1.sources.source1.interceptors.i1.serializers.s1.name=one
tier1.sources.source1.interceptors.i1.serializers.s2.name=two
tier1.sources.source1.interceptors.i1.serializers.s3.name=three
tier1.sources.source1.channels=channel1
tier1.sinks.sink1.type=hdfs
tier1.sinks.sink1.channel=channel1
tier1.sinks.sink1.hdfs.path=hdfs://master68:8020/flume/events/%{one}/%{three}
tier1.sinks.sink1.hdfs.round=true
tier1.sinks.sink1.hdfs.roundValue=10
tier1.sinks.sink1.hdfs.roundUnit=minute
tier1.sinks.sink1.hdfs.fileType=DataStream
tier1.sinks.sink1.hdfs.writeFormat=Text
tier1.sinks.sink1.hdfs.rollInterval=0
tier1.sinks.sink1.hdfs.rollSize=10240
tier1.sinks.sink1.hdfs.rollCount=0
tier1.sinks.sink1.hdfs.idleTimeout=60
tier1.channels.channel1.type=memory
tier1.channels.channel1.capacity=10000
tier1.channels.channel1.transactionCapacity=1000
tier1.channels.channel1.keep-alive=30

```

我把source type改回了内置的spooldir，而不是上一讲自定义的source,然后添加了一个拦截器i1,type是自定义的拦截器：com.besttone.flume.RegexExtractorExtInterceptor$Builder,正则表达式按“.”分隔抽取三部分，分别放到header中的key:one,two,three当中去，即a.log.2014-07-31,通过拦截器后，在header当中就会增加三个key: one=a,two=log,three=2014-07-31。这时候我们在tier1.sinks.sink1.hdfs.path=hdfs://master68:8020/flume/events/%{one}/%{three}。

就实现了和前面一模一样的需求。

也可以看到，自定义拦截器的改动成本非常小，比自定义source小多了，我们这就增加了一个类，就实现了该功能。