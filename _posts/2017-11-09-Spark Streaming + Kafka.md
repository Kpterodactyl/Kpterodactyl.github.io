---
layout:     post
title:      Spark Steaming + Kafka 分析处理消息队列
subtitle:   使用spark steaming 进行实时分析数据
date:       2017-11-09  09:41:08 +0800
author:     Memory
header-img: img/post-bg-universe.jpg
catalog: Scala
tags:
    - Scala
    - Spark
    - Spark-streaming
    - Kafka
---

Kafka是一种分布式的流平台，是基于发布/订阅的消息系统。Kafka可以以一个集群的形式运行在一台或多台服务器上，其中的服务器称为Broker；Kafka将消息分类存储，这个类别称为Topics；每条记录由key, value和时间戳组成；每个Consumer必须属于一个特定的Consumer Group（有默认的GroupId),配置位置在安装目录下的`config/consumer.properties`文件中。若不设置的话有默认值，但最好设置一下，我在之前的尝试中出现过groupId无效的错误，在cousumer获取消息时最好也指定对应的groupId，可以把groupId写到一个文件groupfile中，在Kafka 10 之后Consumer的语法如下
{% highlight ruby %}
bin/kafka-console-consumer.sh --bootstrap-server master:9092 --topic getdata --from-beginning --consumer-property groupfile
{% endhighlight %} 
kafka的拓扑结构如下
![](https://i.imgur.com/dbIn9YU.png)
在这篇文章中展示的是把CSV文件由Kafka Producer API 放入Broker中由Spark steaming作为Consumer读取消息队列。所需环境：包含hdfs、 yarn、 hadoop、 spark、 zookeeper、 kafka的集群

## 功能介绍 ##
将实现由Spark Streaming从Kafka或HDFS text上读取文件，并计算CSV文件中特定列的平均值。首先需要在hdfs上创建好相应的文件目录并修改权限，需要的指令为 `hdfs dfs -mkdir -p + 路径`和`hdfs dfs -chown -R + 路径`。 运行时在spark-submit指定运行方式[kafka|text]

## kafka的使用 ##
那么如何将CSV文件放入kafka的消息队列呢，有以下几种方式，首先可以通过指令的方式
{% highlight ruby %}
cat ./The_CSV_To_Process.csv | bin/kafka-console-producer.sh --broker-list master:9092 --topic getdata
或者
bin/kafka-console-producer.sh --broker-list master:9092 --topic getdata < my_input_file.csv
{% endhighlight %} 
或者写一个Kafka Producer API `KafkaFileProducer` 首先要指明其topic、需要上传的文件名、producer有两种方式，一种是同步，一种是异步，如果设置成异步的模式，可以运行生产者以batch的形式push数据，这样会极大的提高broker的性能，但是这样会增加丢失数据的风险。以batch的方式推送数据可以极大的提高处理效率，kafka producer可以将消息在内存中累计到一定数量后作为一个batch发送请求。batch的数量大小可以通过producer的参数`batch.num.messages`控制。通过增加batch的大小，可以减少网络请求和磁盘IO的次数。
{% highlight ruby %}
private static final String topicName = "getdata";
public static final String fileName = "datafile.csv";
private final KafkaProducer<String, String> producer;
private final Boolean isAsync;
public KafkaFileProducer(String topic, Boolean isAsync) {
     Properties props = new Properties();
     props.put("bootstrap.servers", "master:9092");
     props.put("client.id", "MyProducer");
     props.put("key.serializer",
                "org.apache.kafka.common.serialization.StringSerializer");
     props.put("value.serializer",
                "org.apache.kafka.common.serialization.StringSerializer");
     producer = new KafkaProducer<String, String>(props);
     this.isAsync = isAsync;
}
{% endhighlight %} 
Producer类提供了`send`方法发送一个或多个topices   
#### Async producer is preferred when you want a higher throughput. In the previous releases like 0.8, an async producer does not have a callback for send() to register error handlers. This is available only in the current release of 0.9.
{% highlight ruby %}
public void send(KeyedMessaget<k,v> message) 
- sends the data to a single topic,par-titioned by key using either sync or async producer.
public void send(List<KeyedMessage<k,v>>messages)
- sends data to multiple topics.
{% endhighlight %} 
`ProducerRecord`是发送给Kafka cluster.ProducerRecord类的键/值对，用于使用以下形式创建包含分区，键和值对的记录  `public ProducerRecord (string topic, int partition, k key, v value)`  
－ Topic − user defined topic name that will appended to record.

－ Partition − partition count

－ Key − The key that will be included in the record.

－ Value − Record contents

{% highlight ruby %}
public void sendMessage(String key, String value) {
        long startTime = System.currentTimeMillis();
        if (isAsync) { // Send asynchronously
            producer.send(
                    new ProducerRecord<String, String>(topicName, key),
                    (Callback) new DemoCallBack(startTime, key, value));
        } else { // Send synchronously
            try {
                producer.send(
                        new ProducerRecord<String, String>(topicName, key, value))
                        .get();
                //System.out.println("Sent message: (" + key + ", " + value + ")");
                System.out.println(key + ", " + value);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
    }
{% endhighlight %} 
生成对应的jar，通过`java -jar MyKafkaProject.jar MyKafkaProject.MyKafkaProducer.KafkaFileProducer`运行 可以看待大量的数据从csv中读取出来了       
![](file:///Users/mac/Desktop/14.png)     
在consumer中查看一下消息`bin/kafka-console-consumer.sh --bootstrap-server master:9092 --topic getdata --from-beginning`
![](file:///Users/mac/Desktop/15.png)     
## Spark-streaming的使用 ##  
Spark Streaming 是Spark核心API的一个扩展，可以实现高吞吐量的、具备容错机制的实时流数据的处理。Spark Streaming的工作方式是将数据的实时数据流分成预定义间隔（N秒）的批处理（称microbatches），然后将每批数据视为RDD。然后我们可以使用map，reduce，reduceByKey，join等操作来处理这些RDD。这些RDD操作的结果是分批返回的。我们通常将这些结果存储到数据存储中以进一步分析，并生成报告和仪表盘或发送基于事件的警报。根据使用情况和数据处理要求，确定Spark Streaming的时间间隔非常重要。它的N值太低，那么在分析过程中，微量的批次将没有足够的数据给出有意义的结果。

与Spark Streaming相比，其他流处理框架可以处理每个事件的数据流，而不是微批处理。使用微批处理方法，我们可以在同一个应用程序中使用其他Spark库（如Core，Machine Learning等）和Spark Streaming API。Spark Streaming的数据源可以为Kafka、Flume、Twitter、ZeroMQ、Amazon’s Kinesis、TCP sockets。这是[Spark Streaming Scala API](https://spark.apache.org/docs/1.3.0/api/scala/index.html#org.apache.spark.sql.package)的链接           
![](file:///Users/mac/Desktop/16.jpg)  
{% highlight ruby %}
//通过一个枚举类型来定义可选的两种运行方式
object Mode extends Enumeration {
    type mode = Value
    val KAFKA, TEXT = Value
  }  
{% endhighlight %} 
{% highlight ruby %}
//必要的StreamingContext和kafkaParams，一定不要忘记group.id
val conf = new SparkConf().setAppName(this.getClass.toString)
val ssc = new StreamingContext(conf, Seconds(1))
//lazy val streamingContext = new StreamingContext(conf, Seconds(3))
val hdfsPath = "/test/dataset"

val kafkaParams: mutable.Map[String, String] = new mutable.HashMap[String, String]()
kafkaParams.update("bootstrap.servers", "master:9092")
kafkaParams.update("key.deserializer", "org.apache.kafka.common.serialization" +
      ".StringDeserializer")
kafkaParams.update("value.deserializer", "org.apache.kafka.common.serialization" +
      ".StringDeserializer")
kafkaParams.update("group.id","group1")

val topics = Array("getdata")
{% endhighlight %}    
{% highlight ruby %}
//通过KafkaUtils.createDirectStream创建DStream
    val stream: InputDStream[ConsumerRecord[String, String]] =
      KafkaUtils.createDirectStream[String, String](
        ssc,
        LocationStrategies.PreferConsistent,
        ConsumerStrategies.Subscribe[String, String](topics, kafkaParams))
    val incomeCsv: DStream[(String, String)] = stream.map(
      (item: ConsumerRecord[String, String]) =>
        (item.key, item.value))
{% endhighlight %}  
{% highlight ruby %}
//由HDFS上的ssc.fileStream创建DStream
val incomeCsv: DStream[(String, String)] = ssc.fileStream[LongWritable, Text, TextInputFormat](hdfsPath).map(kv =>
      (kv._1.toString, kv._2.toString))
{% endhighlight %}
{% highlight ruby %}
//按行读取DStream
def parse(incomeCsv: DStream[(String, String)]): DStream[(String, Int)] = {
    val builder = StringBuilder.newBuilder
    val parsedCsv: DStream[List[String]] = incomeCsv.map(entry => {
      val x = entry._2
      var result = List[String]()
      var withinQuotes = false
      x.foreach(c => {
        if (c.equals(',') && !withinQuotes) {
          result = result :+ builder.toString
          builder.clear()
        } else if (c.equals('\"')) {
          builder.append(c)
          withinQuotes = !withinQuotes
        } else {
          builder.append(c)
        }
      })
      result :+ builder.toString
    })
    parsedCsv.map(record => (record(1).substring(0, 3), record(3).toInt))
}
{% endhighlight %}



 
 







