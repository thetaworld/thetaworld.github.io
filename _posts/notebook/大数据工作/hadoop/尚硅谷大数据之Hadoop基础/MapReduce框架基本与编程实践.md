# 自定义输入格式（InputFormat）基本原理

Hadoop可以处理多种不同类型的数据格式，从文本型数据到数据库中的数据都可以。

基于InputFormat的实现类如图所示

![img](https://img-blog.csdn.net/20160126185925582?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##  输入分片和记录（InputSplitand Record）



一个Input split是整个输入文件的一块，每个Input split被单独的MapTask处理，每个split被切分成多个记录，Map处理每条记录（k/v）。Input split在JAVA类中是InputSplit类。

InputSplit类的结构：



```java
public abstract class InputSplit {

  /**
   * 获取每个split的长度，每个输入分片按照大小排序

   * @return 输入分片的字节长度
   * @throws IOException
   * @throws InterruptedException
   */
  public abstract long getLength() throws IOException, InterruptedException;

  /**

   * 获取含有该数据的所有主机节点
   * @return a new array of the node nodes.
   * @throws IOException
   * @throws InterruptedException
   */
  public abstract String[] getLocations() throws IOException, InterruptedException
}
```

一个MapReduce开发者，不需要关心InputSplits，他们通过InputFormat来创建，那么InputFormat类的结构：

```java
public abstract class InputFormat<K, V> {

  /** 
   * 获取job的分片列表，每个InputSplit被分片给单独Mapper处理
   * 
   * 这个split是输入数据的逻辑表示，不代表物理上块。每个InputSplit文件包括
   * 文件路径，开始位置，偏移量。另InputFormat也创建RecordReader来，主要读取InputSplit
   */
  public abstract List<InputSplit> getSplits(JobContext context) throws IOException, InterruptedException;
  /**
   * 为InputSplit创建一个record reader.框架会在split用之前执行RecordReader的 initialize方法.

   * @param split the split to be read
   * @param context the information about the task
   * @return a new record reader
   * @throws IOException
   * @throws InterruptedException
   */
  public abstract RecordReader<K,V> createRecordReader(InputSplit split, TaskAttemptContext context) throws IOException,InterruptedException;
}
```

运行一个job的客户端调用getSplit方法计算出splits，然后发送给application master，它用这个存储目录安排maptasks在集群上来处理这些数据。

MapTask通过InputFormat类中createRecordReader( )方法为InputSplit获取一个RecordReader

然后通过initialize方法获取记录键值对，每个k/v通过map函数来处理，具体可以参考Mapper类中的run方法。

```java
public void run(Context context) throws IOException, InterruptedException {
    //开始：每个maptask仅调用一次
	setup(context);
    try {
	  //每个记录<k,v>都调用
      while (context.nextKeyValue()) {
        map(context.getCurrentKey(), context.getCurrentValue(), context);
      }
    } finally {
	   //每个maptask仅调用一次
      cleanup(context);
    }
  }
```

最后，需要重点说明一点，Mapper‘s的run方法是个公共的方法，需要用户自定义来实现。而MultithreadedMapper是运行mappers的一种实现，MultithreadedMapper中内部类MapRunner具体如下：

```java
public class MultithreadedMapper<K1, V1, K2, V2> extends Mapper<K1, V1, K2, V2> {
  private static final Log LOG = LogFactory.getLog(MultithreadedMapper.class);
  public static String NUM_THREADS = "mapreduce.mapper.multithreadedmapper.threads";
  public static String MAP_CLASS = "mapreduce.mapper.multithreadedmapper.mapclass";
  private Class<? extends Mapper<K1,V1,K2,V2>> mapClass;
  private Context outer;
  private List<MapRunner> runners;
  /**
   * 用一个线程池来运行MapRunner线程，默认开启10个线程
   */
  @Override
  public void run(Context context) throws IOException, InterruptedException {
    outer = context;
    int numberOfThreads = getNumberOfThreads(context);
    mapClass = getMapperClass(context);
    if (LOG.isDebugEnabled()) {
      LOG.debug("Configuring multithread runner to use " + numberOfThreads + " threads");
    }
    runners =  new ArrayList<MapRunner>(numberOfThreads);
    for(int i=0; i < numberOfThreads; ++i) {
      MapRunner thread = new MapRunner(context);
      thread.start();
      runners.add(i, thread);
    }
    for(int i=0; i < numberOfThreads; ++i) {
      MapRunner thread = runners.get(i);
      thread.join();
      Throwable th = thread.throwable;
      if (th != null) {
        if (th instanceof IOException) {
          throw (IOException) th;
        } else if (th instanceof InterruptedException) {
          throw (InterruptedException) th;
        } else {
          throw new RuntimeException(th);
        }
      }
    }
  }
  private class MapRunner extends Thread {
    private Mapper<K1,V1,K2,V2> mapper;
    private Context subcontext;
    private Throwable throwable;
    private RecordReader<K1,V1> reader = new SubMapRecordReader();
    MapRunner(Context context) throws IOException, InterruptedException {

	  //通过反射获取用户自定义的mapper

      mapper = ReflectionUtils.newInstance(mapClass, 
                context.getConfiguration());
      MapContext<K1, V1, K2, V2> mapContext = 
        new MapContextImpl<K1, V1, K2, V2>(outer.getConfiguration(),  outer.getTaskAttemptID(),reader,new SubMapRecordWriter(),  context.getOutputCommitter(),new SubMapStatusReporter(),
outer.getInputSplit());
      subcontext = new WrappedMapper<K1, V1, K2, V2>().getMapContext(mapContext);

	  //调用InputFormat中通过createRecordReader中的RecordReader方法
      reader.initialize(context.getInputSplit(), context);
    }
    @Override
    public void run() {
      try {
		//这里直接调用Mapper中的run方法
        mapper.run(subcontext);
        reader.close();
      } catch (Throwable ie) {
        throwable = ie;
      }
    }
  }
}
```

# MapReduce 排序

排序是 MapReduce 框架中最重要的操作之一。Map Task 和 Reduce Task 均会对数据（按照 key）进行排序。该操作属于 Hadoop 的默认行为。任何应用程序中的数据均会被排序，而不管逻辑上是否需要。默认排序是按照==**字典顺序排序（从小到大），且实现该排序的方法是快速排序。**==

对于 Map Task，它会将处理的结果暂时放到一个缓冲区中，当缓冲区使用率达到一定阈值后，再对缓冲区中的数据进行一次排序，并将这些有序数据写到磁盘上，而当数据处理完毕后，它会对磁盘上所有文件进行一次合并，以将这些文件合并成一个大的有序文件。

对于 Reduce Task，它从每个 Map Task 上远程拷贝相应的数据文件，如果文件大小超过一定阈值，则放到磁盘上，否则放到内存中。如果磁盘上文件数目达到一定阈值，则进行一次合并以生成一个更大文件；如果内存中文件大小或者数目超过一定阈值，则进行一次合并后将数据写到磁盘上。当所有数据拷贝完毕后，Reduce Task 统一对内存和磁盘上的所有数据进行一次合并。

每个阶段的默认排序：

- 排序的分类：

1. 部分排序：
   MapReduce 根据输入记录的键对数据集排序。保证输出的每个文件内部排序。
2. 全排序：
   如何用 Hadoop 产生一个全局排序的文件？最简单的方法是使用一个分区。但该方法在处理大型文件时效率极低，因为一台机器必须处理所有输出文件，从而完全丧失了MapReduce 所提供的并行架构。替代方案：首先创建一系列排好序的文件；其次，串联这些文件；最后，生成一个全局排序的文件。主要思路是使用一个分区来描述输出的全局排序。例如：可以为上述文件创建3 个分区，在第一分区中，记录的单词首字母 a-g，第二分区记录单词首字母 h-n, 第三分区记录单词首字母 o-z。
3. 辅助排序：（GroupingComparator 分组）
   Mapreduce 框架在记录到达 reducer 之前按键对记录排序，但键所对应的值并没有被排序。甚至在不同的执行轮次中，这些值的排序也不固定，因为它们来自不同的 map 任务且这些 map 任务在不同轮次中完成时间各不相同。一般来说，大多数 MapReduce 程序会避免让 reduce 函数依赖于值的排序。但是，有时也需要通过特定的方法对键进行排序和分组等以实现对值的排序。
4. 二次排序：
   在自定义排序过程中，如果 compareTo 中的判断条件为两个即为二次排序。

# [GroupingComparator](https://www.cnblogs.com/DarrenChan/p/6773277.html)案例

- GroupingComparator是在reduce阶段分组来使用的
  - 由于reduce阶段，如果key相同的一组，只取第一个key作为key，迭代所有的values。
  - 如果reduce的key是自定义的bean，我们只需要bean里面的某个属性相同就认为这样的key是相同的，这是我们就需要之定义GroupCoparator来“欺骗”reduce了。
- 我们需要理清楚的还有map阶段你的几个自定义：
  - parttioner中的getPartition（）这个是map阶段自定义分区，bean中定义CopmareTo()是在溢出和merge时用来来排序的。

> 需求：
>
> Order_0000001,Pdt_01,222.8
> Order_0000001,Pdt_05,25.8
> Order_0000002,Pdt_05,325.8
> Order_0000002,Pdt_03,522.8
> Order_0000002,Pdt_04,122.4
> Order_0000003,Pdt_01,222.8
>
> 按照订单的编号分组，计算出每组的商品价格最大值。
>
> 分析：
>
> 我们可以把订单编号当做key，然后按照在reduce端去找出每组的最大值。在这里，我想介绍另外一种方法，顺便介绍GroupingComparator。
>
> 我们可以自定义一个类型，然后通过GroupingComparator来让其被看成一组（到达reduce端），如果我们对类型进行从大到小的排序，根据MapReduce的规则，同一组的内容到达reduce端，是取==**第一个内容的key作为reduce的key**==的，我们不妨利用这个规则，写一个OrderBean的类型，只要让其orderid相同，就被分到同一组，这样一来，到达reduce时，相同id的所有bean已经被看成一组，且金额最大的那个一排在第一位，就是我们想要的结果。

代码：

OrderBean.java:

```java
package com.darrenchan.mr.groupingcomparator;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

import org.apache.hadoop.io.DoubleWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.WritableComparable;

public class OrderBean implements WritableComparable<OrderBean>{

    private Text itemid;
    private DoubleWritable amount;

    public OrderBean() {
    }

    public OrderBean(Text itemid, DoubleWritable amount) {
        set(itemid, amount);
    }

    public void set(Text itemid, DoubleWritable amount) {
        this.itemid = itemid;
        this.amount = amount;
    }

    public Text getItemid() {
        return itemid;
    }

    public DoubleWritable getAmount() {
        return amount;
    }

    @Override
    public int compareTo(OrderBean o) {
//        int cmp = this.itemid.compareTo(o.getItemid());
//        if (cmp == 0) {
        int    cmp = -this.amount.compareTo(o.getAmount());
//        }
        return cmp;
    }

    @Override
    public void write(DataOutput out) throws IOException {
        out.writeUTF(itemid.toString());
        out.writeDouble(amount.get());
    }

    @Override
    public void readFields(DataInput in) throws IOException {
        String readUTF = in.readUTF();
        double readDouble = in.readDouble();
        
        this.itemid = new Text(readUTF);
        this.amount= new DoubleWritable(readDouble);
    }


    @Override
    public String toString() {
        return itemid.toString() + "\t" + amount.get();
    }

}
```

ItemidGroupingComparator.java:

```java
package com.darrenchan.mr.groupingcomparator;

import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;

/**
 * 利用reduce端的GroupingComparator来实现将一组bean看成相同的key
 *
 */
public class ItemidGroupingComparator extends WritableComparator {

    //传入作为key的bean的class类型，以及制定需要让框架做反射获取实例对象
    protected ItemidGroupingComparator() {
        super(OrderBean.class, true);
    }

    @Override
    public int compare(WritableComparable a, WritableComparable b) {
        OrderBean abean = (OrderBean) a;
        OrderBean bbean = (OrderBean) b;
        
        //比较两个bean时，指定只比较bean中的orderid
        return abean.getItemid().compareTo(bbean.getItemid());
    }

}
```

ItemIdPartitioner.java:

```java
package com.darrenchan.mr.groupingcomparator;

import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.mapreduce.Partitioner;


public class ItemIdPartitioner extends Partitioner<OrderBean, NullWritable>{

    @Override
    public int getPartition(OrderBean bean, NullWritable value, int numReduceTasks) {
        //相同id的订单bean，会发往相同的partition
        //而且，产生的分区数，是会跟用户设置的reduce task数保持一致
        return (bean.getItemid().hashCode() & Integer.MAX_VALUE) % numReduceTasks;
        
    }

}
```

SecondarySort.java:

```java
package com.darrenchan.mr.groupingcomparator;

import java.io.IOException;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.DoubleWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import com.sun.xml.bind.v2.schemagen.xmlschema.List;

/**
 * 
 */
public class SecondarySort {
    
    static class SecondarySortMapper extends Mapper<LongWritable, Text, OrderBean, NullWritable>{
        
        OrderBean bean = new OrderBean();
        
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

            String line = value.toString();
            String[] fields = StringUtils.split(line, ",");
            
            bean.set(new Text(fields[0]), new DoubleWritable(Double.parseDouble(fields[2])));
            
            context.write(bean, NullWritable.get());
            
        }
        
    }
    
    static class SecondarySortReducer extends Reducer<OrderBean, NullWritable, OrderBean, NullWritable>{
        
        //到达reduce时，相同id的所有bean已经被看成一组，且金额最大的那个一排在第一位
        @Override
        protected void reduce(OrderBean key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
            context.write(key, NullWritable.get());
        }
    }
    
    
    public static void main(String[] args) throws Exception {
        
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        
        job.setJarByClass(SecondarySort.class);
        
        job.setMapperClass(SecondarySortMapper.class);
        job.setReducerClass(SecondarySortReducer.class);
        
        
        job.setOutputKeyClass(OrderBean.class);
        job.setOutputValueClass(NullWritable.class);
        
        FileInputFormat.setInputPaths(job, new Path("/grouping/srcdata"));
        FileOutputFormat.setOutputPath(job, new Path("/grouping/output"));
        
        //在此设置自定义的Groupingcomparator类 
        job.setGroupingComparatorClass(ItemidGroupingComparator.class);
        //在此设置自定义的partitioner类
        job.setPartitionerClass(ItemIdPartitioner.class);
        
        job.setNumReduceTasks(3);
        
        job.waitForCompletion(true);
        
    }

}
```



# Join多种应用场景与使用



##  Reduce Join 多种应用场景与使用

在关系型数据库中 Join 是非常常见的操作，各种优化手段已经到了极致。在海量数据的环境下，不可避免的也会碰到这种类型的需求， 例如在数据分析时需要连接从不同的数据源中获取到数据。不同于传统的单机模式，在分布式存储下采用 MapReduce 编程模型，也有相应的处理措施和优化方法。

我们先简要地描述待解决的问题。假设有两个数据集：气象站数据库和天气记录数据库，并考虑如何合二为一。一个典型的查询是：输出气象站的历史信息，同时各行记录也包含气象站的元数据信息。



![实例](http://hadoop2.dajiangtai.com/content/222/1.png)



 

在 Reudce 端进行连接是 MapReduce 框架实现 join 操作最常见的方式，其具体的实现原理如下：

Map 端的主要工作：为来自不同表 (文件) 的 key/value 对打标签以区别不同来源的记录。然后用连接字段（两张表中相同的列）作为 key，其余部分和==**新加的标志**==作为 value，最后进行输出。

reduce 端的主要工作：在 reduce 端以连接字段作为 key 的分组已经完成，我们只需要在每一个分组当中将那些来源于不同文件的记录 (在 map 阶段已经打标志) 分开，最后进行合并就 ok 了

 

### 实现方式一：二次排序



![img](https://images2017.cnblogs.com/blog/1014746/201708/1014746-20170813154404007-626453746.png)



#### 适用场景：其中一个表的连接字段 key 唯一

#### 思路概述

 二次排序的意思在 map 阶段只是对于不同表进行打标签排序，决定了在 reduce 阶段输出后两张中的先后顺序；而 reduce 的作用就是根本 map 输出的数据连接两张表。

#### 代码实现

自定义 TextPair 作为两个文件的 Mapper 输出 key。

TextPair.java

```java
package com.hadoop.reducejoin.test;

import org.apache.hadoop.io.WritableComparable;
import java.io.*;
import org.apache.hadoop.io.*;

import com.hadoop.mapreduce.test.IntPair;

public class TextPair implements WritableComparable<TextPair> {
    private    Text first;//Text 类型的实例变量 first
    private    Text second;//Text 类型的实例变量 second
    
    public TextPair() {
        set(new Text(),new Text());
    }
    
    public TextPair(String first, String second) {
        set(new Text(first),new Text(second));
    }
    
    public TextPair(Text first, Text second) {
        set(first, second);
    }
    
    public void set(Text first, Text second) {
        this.first = first;
        this.second = second;
    }
    
    public Text getFirst() {
        return first;
    }
    
    public Text getSecond() {
        return second;
    }
    
    //将对象转换为字节流并写入到输出流out中
    public void write(DataOutput out)throws IOException {
        first.write(out);
        second.write(out);
    }
    
    //从输入流in中读取字节流反序列化为对象
    public void readFields(DataInput in)throws IOException {
        first.readFields(in);
        second.readFields(in);
    }
    
    @Override
    public int hashCode() {
        return first.hashCode() *163+second.hashCode();
    }
    
    
    @Override
    public boolean equals(Object o) {
        if(o instanceof TextPair) {
            TextPair tp = (TextPair) o;
            return first.equals(tp.first) && second.equals(tp.second);
        }
            return false;
    }
    
    @Override
    public String toString() {
        return first +"\t"+ second;
    }
    

    @Override
    public int compareTo(TextPair o) {
        // TODO Auto-generated method stub
        if(!first.equals(o.first)){
            return first.compareTo(o.first);
        }
        else if(!second.equals(o.second)){
            return second.compareTo(o.second);
        }else{
            return 0;
        }
    }
}
```

ReduceJoinBySecondarySort

```java
package com.hadoop.reducejoin.test;

import java.io.IOException;
import java.util.Iterator;


import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Partitioner;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.MultipleInputs;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;
/*
 * 通过二次排序实现reduce join
 * 适用场景：其中一个表的连接字段key唯一
 */
public class ReduceJoinBySecondarySort extends Configured implements Tool{  
//    JoinStationMapper 处理来自气象站数据
    public static class JoinStationMapper extends Mapper< LongWritable,Text,TextPair,Text>{
        protected void map(LongWritable key,Text value,Context context) throws IOException,InterruptedException{
            String line = value.toString();
            String[] arr = line.split("\\s+");//解析天气记录数据
            if(arr.length==2){//满足这种数据格式
                //key=气象站id  value=气象站名称
                context.write(new TextPair(arr[0],"0"),new Text(arr[1]));
            }
        }
    }
    
    public static class JoinRecordMapper extends Mapper< LongWritable,Text,TextPair,Text>{
        
        protected void map(LongWritable key,Text value,Context context) throws IOException,InterruptedException{
            String line = value.toString();
            String[] arr = line.split("\\s+");//解析天气记录数据
            if(arr.length==3){
                //key=气象站id  value=天气记录数据
                context.write(new TextPair(arr[0],"1"),new Text(arr[1]+"\t"+arr[2]));
            }  
        }
    }
    
    public static class KeyPartitioner  extends Partitioner< TextPair,Text>{
        public int getPartition(TextPair key,Text value,int numPartitions){
//            &是位与运算
            return (key.getFirst().hashCode()&Integer.MAX_VALUE)% numPartitions;
        }
    }
    
    public static class GroupingComparator extends WritableComparator{
         protected GroupingComparator(){
             super(TextPair.class, true);
         }
         @Override
         //Compare two WritableComparables.
         public int compare(WritableComparable w1, WritableComparable w2){
             TextPair ip1 = (TextPair) w1;
             TextPair ip2 = (TextPair) w2;
             Text l = ip1.getFirst();
             Text r = ip2.getFirst();
             return l.compareTo(r);
         }
    }
    
    public static class JoinReducer extends Reducer< TextPair,Text,Text,Text>{

        protected void reduce(TextPair key, Iterable< Text> values,Context context) throws IOException,InterruptedException{
            Iterator< Text> iter = values.iterator();
            Text stationName = new Text(iter.next());//气象站名称
            while(iter.hasNext()){
                Text record = iter.next();//天气记录的每条数据
                Text outValue = new Text(stationName.toString()+"\t"+record.toString());
                context.write(key.getFirst(),outValue);
            }
        }        
    }
    
    
    
    public int run(String[] args) throws Exception{
        Configuration conf = new Configuration();// 读取配置文件
        
        Path mypath = new Path(args[2]);
        FileSystem hdfs = mypath.getFileSystem(conf);// 创建输出路径
        if (hdfs.isDirectory(mypath)) {
            hdfs.delete(mypath, true);
        }
        Job job = Job.getInstance(conf, "join");// 新建一个任务
        job.setJarByClass(ReduceJoinBySecondarySort.class);// 主类
        
        Path recordInputPath = new Path(args[0]);//天气记录数据源
        Path stationInputPath = new Path(args[1]);//气象站数据源
        Path outputPath = new Path(args[2]);//输出路径
//        两个输入类型
        MultipleInputs.addInputPath(job,recordInputPath,TextInputFormat.class,JoinRecordMapper.class);//读取天气记录Mapper
        MultipleInputs.addInputPath(job,stationInputPath,TextInputFormat.class,JoinStationMapper.class);//读取气象站Mapper
        FileOutputFormat.setOutputPath(job,outputPath);
        job.setReducerClass(JoinReducer.class);// Reducer
        
        job.setPartitionerClass(KeyPartitioner.class);//自定义分区
        job.setGroupingComparatorClass(GroupingComparator.class);//自定义分组
        
        job.setMapOutputKeyClass(TextPair.class);
        job.setMapOutputValueClass(Text.class);
        
        
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);
        
        return job.waitForCompletion(true)?0:1;
    }
        
    public static void main(String[] args) throws Exception{
        String[] args0 = {"hdfs://sparks:9000/middle/reduceJoin/records.txt"
                ,"hdfs://sparks:9000/middle/reduceJoin/station.txt"
                ,"hdfs://sparks:9000/middle/reduceJoin/secondSort-out"
        };
        int exitCode = ToolRunner.run(new ReduceJoinBySecondarySort(),args0);
        System.exit(exitCode);
    }
}
```



 

### 实现方式二：笛卡尔积



![img](https://images2017.cnblogs.com/blog/1014746/201708/1014746-20170813155427023-81057644.png)



#### 适用场景：两个表的连接字段 key 都不唯一 (包含一对多，多对多的关系)

#### 思路概述

 在 map 阶段将来自不同表或文件的 key、value 打标签对区别不同的来源，在 reduce 阶段对于来自不同表或文件的相同 key 的数据分开，然后做笛卡尔积。这样就实现了连接表。

 

#### 代码实现

ReduceJoinByCartesianProduct

```java
package com.hadoop.reducejoin.test;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;

/*
 * 两个大表
 * 通过笛卡尔积实现 reduce join
 * 适用场景：两个表的连接字段key都不唯一(包含一对多，多对多的关系)
 */
public class ReduceJoinByCartesianProduct {
    /**
    为来自不同表(文件)的key/value对打标签以区别不同来源的记录。
    然后用连接字段作为key，其余部分和新加的标志作为value，最后进行输出。
    */
    public static class ReduceJoinByCartesianProductMapper extends Mapper<Object,Text,Text,Text>{
        private Text joinKey=new Text();
        private Text combineValue=new Text();
        
        @Override
        protected void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            String pathName=((FileSplit)context.getInputSplit()).getPath().toString();
            //如果数据来自于records，加一个records的标记
            if(pathName.endsWith("records.txt")){
                String line = value.toString();
                String[] valueItems = line.split("\\s+");
                //过滤掉脏数据
                if(valueItems.length!=3){
                    return;
                }
                joinKey.set(valueItems[0]);
                combineValue.set("records.txt" + valueItems[1] + "\t" + valueItems[2]);
            }else if(pathName.endsWith("station.txt")){
                //如果数据来自于station，加一个station的标记
                String line = value.toString();
                String[] valueItems = line.split("\\s+");
                //过滤掉脏数据
                if(valueItems.length!=2){
                    return;
                }
                joinKey.set(valueItems[0]);
                combineValue.set("station.txt" + valueItems[1]);
            }
            context.write(joinKey,combineValue);
        }
    }
    /*
     * reduce 端做笛卡尔积
     */
     public static class ReduceJoinByCartesianProductReducer extends Reducer<Text,Text,Text,Text>{
            private List<String> leftTable=new ArrayList<String>();
            private List<String> rightTable=new ArrayList<String>();
            private Text result=new Text();
            @Override
            protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
                //一定要清空数据
                leftTable.clear();
                rightTable.clear();
                //相同key的记录会分组到一起，我们需要把相同key下来自于不同表的数据分开，然后做笛卡尔积
                for(Text value : values){
                    String val=value.toString();
                    if(val.startsWith("station.txt")){
                        leftTable.add(val.replaceFirst("station.txt",""));
                    }else if(val.startsWith("records.txt")){
                        rightTable.add(val.replaceFirst("records.txt",""));
                    }
                }
                //笛卡尔积
                for(String leftPart:leftTable){
                    for(String rightPart:rightTable){
                        result.set(leftPart+"\t"+rightPart);
                        context.write(key, result);
                    }
                }
            }
        }
     
     public static void main(String[] arg0) throws Exception{
            Configuration conf = new Configuration();
            String[] args = {"hdfs://sparks:9000/middle/reduceJoin/records.txt"
                    ,"hdfs://sparks:9000/middle/reduceJoin/station.txt"
                    ,"hdfs://sparks:9000/middle/reduceJoin/JoinByCartesian-out"
            };
            String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
            if (otherArgs.length < 2) {
                System.err.println("Usage: reducejoin <in> [<in>...] <out>");
                System.exit(2);
            }
            
            //输出路径
            Path mypath = new Path(otherArgs[otherArgs.length - 1]);
            FileSystem hdfs = mypath.getFileSystem(conf);// 创建输出路径
            if (hdfs.isDirectory(mypath)) {
                hdfs.delete(mypath, true);
            }
            Job job = Job.getInstance(conf, "ReduceJoinByCartesianProduct");
            job.setJarByClass(ReduceJoinByCartesianProduct.class);
            job.setMapperClass(ReduceJoinByCartesianProductMapper.class);
            job.setReducerClass(ReduceJoinByCartesianProductReducer.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);
            //添加输入路径
            for (int i = 0; i < otherArgs.length - 1; ++i) {
                FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
            }
            //添加输出路径
            FileOutputFormat.setOutputPath(job,
                    new Path(otherArgs[otherArgs.length - 1]));
            System.exit(job.waitForCompletion(true) ? 0 : 1);
        }
}
```

### 实现方式三：分布式缓存



![img](https://images2017.cnblogs.com/blog/1014746/201708/1014746-20170813161244570-1544548209.png)



#### 适用场景：一个大表和一个小表连接

#### 分布式知识点补充

当 MapReduce 处理大型数据集间的 join 操作时，此时如果一个数据集很大而另外一个集合很小，以至于可以分发到集群中的每个节点之中。 这种情况下，我们就用到了 Hadoop 的分布式缓存机制，它能够在任务运行过程中及时地将文件和存档复制到任务节点以供使用。为了节约网络宽带，在每一个作业中， 各个文件通常只需要复制到一个节点一次。

1、用法

Hadoop 命令行选项中，有三个命令可以实现文件复制分发到任务的各个节点。

1) 用户可以使用 -files 选项指定待分发的文件，文件内包含以逗号隔开的 URL 列表。文件可以存放在本地文件系统、HDFS、或其它 Hadoop 可读文件系统之中。 如果尚未指定文件系统，则这些文件被默认是本地的。即使默认文件系统并非本地文件系统，这也是成立的。

2) 用户可以使用 -archives 选项向自己的任务中复制存档文件，比如 JAR 文件、ZIP 文件、tar 文件和 gzipped tar 文件，这些文件会被解档到任务节点。

3) 用户可以使用 -libjars 选项把 JAR 文件添加到 mapper 和 reducer 任务的类路径中。如果作业 JAR 文件并非包含很多库 JAR 文件，这点会很有用。

2、工作机制

当用户启动一个作业，Hadoop 会把由 -files、-archives、和 -libjars 等选项所指定的文件复制到分布式文件系统之中。接着，在任务运行之前， tasktracker 将文件从分布式文件系统复制到本地磁盘（缓存）使任务能够访问文件。此时，这些文件就被视为 “本地化” 了。从任务的角度来看， 这些文件就已经在那儿了，它并不关心这些文件是否来自 HDFS 。此外，有 -libjars 指定的文件会在任务启动前添加到任务的类路径（classpath）中。

3、分布式缓存 API

由于可以通过 Hadoop 命令行间接使用分布式缓存，大多数应用不需要使用分布式缓存 API。然而，一些应用程序需要用到分布式缓存的更高级的特性，这就需要直接使用 API 了。 API 包括两部分：将数据放到缓存中的方法，以及从缓存中读取数据的方法。

1) 首先掌握数据放到缓存中的方法，以下列举 Job 中可将数据放入到缓存中的相关方法：

```java
public void addCacheFile(URI uri);
public void addCacheArchive(URI uri);//以上两组方法将文件或存档添加到分布式缓存
public void setCacheFiles(URI[] files);
public void setCacheArchives(URI[] archives);//以上两组方法将一次性向分布式缓存中添加一组文件或存档
public void addFileToClassPath(Path file);
public void addArchiveToClassPath(Path archive);//以上两组方法将文件或存档添加到 MapReduce 任务的类路径
public void createSymlink();
```

　　

在缓存中可以存放两类对象：文件（files）和存档（achives）。文件被直接放置在任务节点上，而存档则会被解档之后再将具体文件放置在任务节点上。

2) 其次掌握在 map 或者 reduce 任务中，使用 API 从缓存中读取数据。

```java
public Path[] getLocalCacheFiles() throws IOException;
public Path[] getLocalCacheArchives() throws IOException;
public Path[] getFileClassPaths();
public Path[] getArchiveClassPaths();
```

　　

我们可以使用 getLocalCacheFiles() 和 getLocalCacheArchives() 方法获取缓存中的文件或者存档的引用。 当处理存档时，将会返回一个包含解档文件的目的目录。相应的，用户可以通过 getFileClassPaths() 和 getArchivesClassPaths() 方法获取被添加到任务的类路径下的文件和文档。

 

#### 思路概述

 小表作为缓存分发至各个节点，在 reduce 阶段，通过读取缓存中的小表过滤大表中一些不需要的数据和字段。

 

#### 代码实现

```java
package com.hadoop.reducejoin.test;

import java.io.BufferedReader;
import java.io.FileNotFoundException;
import java.io.FileReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.URI;
import java.util.Hashtable;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.Reducer.Context;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;
import org.apache.hadoop.util.bloom.Key;

/*
 * 通过分布式缓存实现Reduce Join
 * 适用场景：其中一个表比较小，能放入内存
 */
public class ReduceJoinByDistributedCache  extends Configured implements Tool {
    
    //直接输出大表数据records.txt 
    public static class ReduceJoinByDistributedCacheMapper extends
            Mapper< LongWritable, Text, Text, Text> {
        private Text combineValue=new Text();
        public void map(LongWritable key, Text value, Context context)
                throws IOException, InterruptedException {
            String line = value.toString();
            String[] arr = line.split("\\s+");
            if (arr.length == 3) {
                combineValue.set(arr[1] + "\t" + arr[2]);
                context.write(new Text(arr[0]), combineValue);
            }
        }
    }
    
    //在reduce 端通过缓存文件实现join操作
    public static class ReduceJoinByDistributedCacheReducer extends
            Reducer< Text, Text, Text, Text> {
        //定义Hashtable存放缓存数据
        private Hashtable< String, String> table = new Hashtable< String, String>();
        /**
         * 获取分布式缓存文件
         */
        protected void setup(Context context) throws IOException,
                InterruptedException {
            BufferedReader br;
            String infoAddr = null;
            // 返回缓存文件路径
            Path[] cacheFilesPaths = context.getLocalCacheFiles();
            for (Path path : cacheFilesPaths) {
                String pathStr = path.toString();
                br = new BufferedReader(new FileReader(pathStr));
                while (null != (infoAddr = br.readLine())) {
                    // 按行读取并解析气象站数据
                    String line = infoAddr.toString();
                    String[] records = line.split("\\s+");
                    if (null != records)    // key为stationID，value为stationName
                        table.put(records[0], records[1]);
                }
            }
        }
        public void reduce(Text key, Iterable< Text> values, Context context)
                throws IOException, InterruptedException {
            //天气记录根据stationId 获取stationName
            String stationName = table.get(key.toString());
            for (Text value : values) {
                value.set(stationName + "\t" + value.toString());
                context.write(key, value);
            }
        }
    }
    public int run(String[] arg0) throws Exception {
        Configuration conf = new Configuration();
        String[] args = {"hdfs://sparks:9000/middle/reduceJoin/station.txt"
                ,"hdfs://sparks:9000/middle/reduceJoin/records.txt"
                ,"hdfs://sparks:9000/middle/reduceJoin/DistributedCache-out"
        };
        String[] otherArgs = new GenericOptionsParser(conf, args)
                .getRemainingArgs();
        if (otherArgs.length < 2) {
            System.err.println("Usage: cache <in> [<in>...] <out>");
            System.exit(2);
        }

        //输出路径
        Path mypath = new Path(otherArgs[otherArgs.length - 1]);
        FileSystem hdfs = mypath.getFileSystem(conf);// 创建输出路径
        if (hdfs.isDirectory(mypath)) {
            hdfs.delete(mypath, true);
        }
        Job job = Job.getInstance(conf, "ReduceJoinByDistributedCache");

        //添加缓存文件
        job.addCacheFile(new Path(otherArgs[0]).toUri());//station.txt
        
        job.setJarByClass(ReduceJoinByDistributedCache.class);
        job.setMapperClass(ReduceJoinByDistributedCacheMapper.class);
        job.setReducerClass(ReduceJoinByDistributedCacheReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);
        //添加输入路径
        for (int i = 1; i < otherArgs.length - 1; ++i) {
            FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
        }
        //添加输出路径
        FileOutputFormat.setOutputPath(job, new Path(
                otherArgs[otherArgs.length - 1]));
        return job.waitForCompletion(true) ? 0 : 1;
    }
    public static void main(String[] args) throws Exception {
        int ec = ToolRunner.run(new Configuration(),new ReduceJoinByDistributedCache(), args);
        System.exit(ec);
    }
}
```

ReduceJoinByDistributedCache

 

### Reduce Join 的不足

这里主要分析一下 reduce join 的一些不足。之所以会存在 reduce join 这种方式，是因为整体数据被分割了，每个 map task 只处理一部分数据而不能够获取到所有需要的 join 字段，因此我们可以充分利用 mapreduce 框架的特性，让他按照 join key 进行分区，将所有 join key 相同的记录集中起来进行处理，所以 reduce join 这种方式就出现了。

这种方式的缺点很明显就是会造成 map 和 reduce 端也就是 shuffle 阶段出现大量的数据传输，效率很低。

##  Map Join 多种应用场景与使用

### Map Join 实现方式一：分布式缓存

● 使用场景：一张表十分小、一张表很大。

● 用法:

在提交作业的时候先将小表文件放到该作业的 DistributedCache 中，然后从 DistributeCache 中取出该小表进行 join (比如放到 Hash Map 等等容器中)。然后扫描大表，看大表中的每条记录的 join key /value 值是否能够在内存中找到相同 join key 的记录，如果有则直接输出结果。

DistributedCache 是分布式缓存的一种实现，它在整个 MapReduce 框架中起着相当重要的作用，他可以支撑我们写一些相当复杂高效的分布式程序。说回到这里，JobTracker 在作业启动之前会获取到 DistributedCache 的资源 uri 列表，并将对应的文件分发到各个涉及到该作业的任务的 TaskTracker 上。另外，关于 DistributedCache 和作业的关系，比如权限、存储路径区分、public 和 private 等属性。



![img](https://images2017.cnblogs.com/blog/1014746/201708/1014746-20170813162202617-325857597.png)



#### 代码实现

```java
package com.hadoop.reducejoin.test;

import java.io.BufferedReader;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.URI;
import java.util.Hashtable;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.Reducer.Context;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;


/*
 * 通过分布式缓存实现 map join
 * 适用场景：一个小表，一个大表
 */
public class MapJoinByDistributedCache extends Configured implements Tool {

    /*
     * 直接在map 端进行join合并
     */
    public static class MapJoinMapper extends
            Mapper<LongWritable, Text, Text, Text> {
        private Hashtable<String, String> table = new Hashtable<String, String>();// 定义Hashtable存放缓存数据

        /**
         * 获取分布式缓存文件
         */
        @SuppressWarnings("deprecation")
        protected void setup(Context context) throws IOException,
                InterruptedException {
            Path[] localPaths = (Path[]) context.getLocalCacheFiles();// 返回本地文件路径
            if (localPaths.length == 0) {
                throw new FileNotFoundException(
                        "Distributed cache file not found.");
            }
            FileSystem fs = FileSystem.getLocal(context.getConfiguration());// 获取本地
                                                                            // FileSystem
                                                                            // 实例
            FSDataInputStream in = null;

            in = fs.open(new Path(localPaths[0].toString()));// 打开输入流
            BufferedReader br = new BufferedReader(new InputStreamReader(in));// 创建BufferedReader读取器
            String infoAddr = null;
            while (null != (infoAddr = br.readLine())) {// 按行读取并解析气象站数据
                String[] records = infoAddr.split("\t");
                table.put(records[0], records[1]);// key为stationID，value为stationName
            }
        }

        public void map(LongWritable key, Text value, Context context)
                throws IOException, InterruptedException {
            String line = value.toString();
            String[] valueItems = line.split("\\s+");
//            使用下面一行将没有数据, StringUtils不能接正则,只能接分隔符
//            String[] valueItems = StringUtils.split(value.toString(), "\\s+");
            String stationName = table.get(valueItems[0]);// 天气记录根据stationId
                                                            // 获取stationName
            if (null != stationName)
                context.write(new Text(stationName), value);
        }

    }



    public int run(String[] args) throws Exception {
        // TODO Auto-generated method stub
        Configuration conf = new Configuration();

        Path out = new Path(args[2]);
        FileSystem hdfs = out.getFileSystem(conf);// 创建输出路径
        if (hdfs.isDirectory(out)) {
            hdfs.delete(out, true);
        }
        Job job = Job.getInstance();// 获取一个job实例
        job.setJarByClass(MapJoinByDistributedCache.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[2]));
        // 添加分布式缓存文件 station.txt
        job.addCacheFile(new URI(args[1]));
        job.setMapperClass(MapJoinMapper.class);

        job.setOutputKeyClass(Text.class);// 输出key类型
        job.setOutputValueClass(Text.class);// 输出value类型
        return job.waitForCompletion(true) ? 0 : 1;
    }

    public static void main(String[] args0) throws Exception {
        String[] args = { "hdfs://sparks:9000/middle/reduceJoin/records.txt",
                "hdfs://sparks:9000/middle/reduceJoin/station.txt",
                "hdfs://sparks:9000/middle/reduceJoin/MapJoinByDistributedCache-out" };

        int ec = ToolRunner.run(new Configuration(),
                new MapJoinByDistributedCache(), args);
        System.exit(ec);
    }
}
```

MapJoinByDistributedCache

 

### Map Join 实现方式二：数据库 join

● 使用场景：一张表在数据库、一张表很大。

另外还有一种比较变态的 Map Join 方式，就是结合 HBase 来做 Map Join 操作。这种方式完全可以突破内存的控制，使你毫无忌惮的使用 Map Join，而且效率也非常不错。



![img](http://hadoop2.dajiangtai.com/content/222/6.png)



## Semi Join 多种应用场景与使用

### Map Join 实现方式一

● 使用场景：**一个大表（整张表内存放不下，但表中的 key 内存放得下），一个超大表**

● 实现方式：分布式缓存

● 用法:

SemiJoin 就是所谓的半连接，其实仔细一看就是 reduce join 的一个变种，就是在 map 端过滤掉一些数据，在网络中只传输参与连接的数据不参与连接的数据不必在网络中进行传输，从而减少了 shuffle 的网络传输量，使整体效率得到提高，其他思想和 reduce join 是一模一样的。**说得更加接地气一点就是将小表中参与 join 的 key 单独抽出来通过 DistributedCach 分发到相关节点**，然后将其取出放到内存中 (可以放到 HashSet 中)，在 map 阶段扫描连接表，将 join key 不在内存 HashSet 中的记录过滤掉，让那些参与 join 的记录通过 shuffle 传输到 reduce 端进行 join 操作，其他的和 reduce join 都是一样的。



![img](https://images2017.cnblogs.com/blog/1014746/201708/1014746-20170813163334351-445953845.png)



#### 代码实现

SemiJoin

```java
package com.hadoop.reducejoin.test;

import java.io.BufferedReader;
import java.io.FileNotFoundException;
import java.io.FileReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;
import org.apache.hadoop.util.bloom.Key;

/*
 * 一个大表，一个小表（也很大，内存中放不下）
 * map 阶段：Semi Join解决小表整个记录内存放不下的场景，那么就取出来一小部分关键字段放入内存，过滤大表
 * 提前过滤，提前提取出小表中的连接字段放入内存中，在map阶段就仅留下大表中那些小表中存在的连接字段key
 * reduce 阶段：reduce side join
 */
public class SemiJoin {
    /**
     * 为来自不同表(文件)的key/value对打标签以区别不同来源的记录。
     * 然后用连接字段作为key，其余部分和新加的标志作为value，最后进行输出。
     */
    public static class SemiJoinMapper extends
            Mapper<Object, Text, Text, Text> {
        // 定义Set集合保存小表中的key
        private Set<String> joinKeys = new HashSet<String>();
        private Text joinKey = new Text();
        private Text combineValue = new Text();

        /**
         * 获取分布式缓存文件
         */
        protected void setup(Context context) throws IOException,
                InterruptedException {
            BufferedReader br;
            String infoAddr = null;
            // 返回缓存文件路径
            Path[] cacheFilesPaths = context.getLocalCacheFiles();
            for (Path path : cacheFilesPaths) {
                String pathStr = path.toString();
                br = new BufferedReader(new FileReader(pathStr));
                while (null != (infoAddr = br.readLine())) {
                    // 按行读取并解析气象站数据
                    String[] records = StringUtils.split(infoAddr.toString(),
                            "\t");
                    if (null != records)// key为stationID
                        joinKeys.add(records[0]);
                }
            }
        }

        @Override
        protected void map(Object key, Text value, Context context)
                throws IOException, InterruptedException {
            String pathName = ((FileSplit) context.getInputSplit()).getPath()
                    .toString();
            // 如果数据来自于records，加一个records的标记
            if (pathName.endsWith("records-semi.txt")) {
                String[] valueItems = StringUtils.split(value.toString(),
                        "\t");
                // 过滤掉脏数据
                if (valueItems.length != 3) {
                    return;
                }
//                提前过滤，提前提取出小表中的连接字段，在map阶段就仅留下大表中那些小表中存在的连接字段key
                if (joinKeys.contains(valueItems[0])) {
                    joinKey.set(valueItems[0]);
                    combineValue.set("records-semi.txt" + valueItems[1] + "\t"
                            + valueItems[2]);
                    context.write(joinKey, combineValue);
                }

            } else if (pathName.endsWith("station.txt")) {
                // 如果数据来自于station，加一个station的标记
                String[] valueItems = StringUtils.split(value.toString(),
                        "\t");
                // 过滤掉脏数据
                if (valueItems.length != 2) {
                    return;
                }
                joinKey.set(valueItems[0]);
                combineValue.set("station.txt" + valueItems[1]);
                context.write(joinKey, combineValue);

            }
        }
    }
    /*
     * reduce 端做笛卡尔积
     */
    public static class SemiJoinReducer extends
            Reducer<Text, Text, Text, Text> {
        private List<String> leftTable = new ArrayList<String>();
        private List<String> rightTable = new ArrayList<String>();
        private Text result = new Text();

        @Override
        protected void reduce(Text key, Iterable<Text> values, Context context)
                throws IOException, InterruptedException {
            // 一定要清空数据
            leftTable.clear();
            rightTable.clear();
            // 相同key的记录会分组到一起，我们需要把相同key下来自于不同表的数据分开，然后做笛卡尔积
            for (Text value : values) {
                String val = value.toString();
                System.out.println("value=" + val);
                if (val.startsWith("station.txt")) {
                    leftTable.add(val.replaceFirst("station.txt", ""));
                } else if (val.startsWith("records-semi.txt")) {
                    rightTable.add(val.replaceFirst("records-semi.txt", ""));
                }
            }
            // 笛卡尔积
            for (String leftPart : leftTable) {
                for (String rightPart : rightTable) {
                    result.set(leftPart + "\t" + rightPart);
                    context.write(key, result);
                }
            }
        }
    }

    public static void main(String[] arg0) throws Exception {
        Configuration conf = new Configuration();
        String[] args = { "hdfs://sparks:9000/middle/reduceJoin/station.txt",
                "hdfs://sparks:9000/middle/reduceJoin/station.txt",
                "hdfs://sparks:9000/middle/reduceJoin/records-semi.txt",
                "hdfs://sparks:9000/middle/reduceJoin/SemiJoin-out" };
        String[] otherArgs = new GenericOptionsParser(conf, args)
                .getRemainingArgs();
        if (otherArgs.length < 2) {
            System.err.println("Usage: semijoin <in> [<in>...] <out>");
            System.exit(2);
        }

        //输出路径
        Path mypath = new Path(otherArgs[otherArgs.length - 1]);
        FileSystem hdfs = mypath.getFileSystem(conf);// 创建输出路径
        if (hdfs.isDirectory(mypath)) {
            hdfs.delete(mypath, true);
        }
        Job job = Job.getInstance(conf, "SemiJoin");

        //添加缓存文件
        job.addCacheFile(new Path(otherArgs[0]).toUri());
        job.setJarByClass(SemiJoin.class);
        job.setMapperClass(SemiJoinMapper.class);
        job.setReducerClass(SemiJoinReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);
        //添加输入路径
        for (int i = 1; i < otherArgs.length - 1; ++i) {
            FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
        }
        //添加输出路径
        FileOutputFormat.setOutputPath(job, new Path(
                otherArgs[otherArgs.length - 1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

 

### Reduce join + BloomFilter

● 使用场景：**一个大表（表中的 key 内存仍然放不下），一个超大表**

在某些情况下，SemiJoin 抽取出来的小表的 key 集合在内存中仍然存放不下，这时候可以使用 BloomFiler 以节省空间。

BloomFilter 最常见的作用是：判断某个元素是否在一个集合里面。它最重要的两个方法是：add() 和 membershipTest ()。

因而可将小表中的 key 保存到 BloomFilter 中，在 map 阶段过滤大表，可能有一些不在小表中的记录没有过滤掉（但是在小表中的记录一定不会过滤掉），这没关系，只不过增加了少量的网络 IO 而已。

● BloomFilter 参数计算方式：

n：小表中的记录数。

m：位数组大小，一般 m 是 n 的倍数，倍数越大误判率就越小，但是也有内存限制，不能太大，这个值需要反复测试得出。

k：hash 个数，最优 hash 个数值为：k = ln2 * (m/n)

 

#### 代码实现

BloomFilteringDriver

```java
package com.hadoop.reducejoin.test;

import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;
import org.apache.hadoop.util.bloom.BloomFilter;
import org.apache.hadoop.util.bloom.Key;
import org.apache.hadoop.util.hash.Hash;
/*
 * 一个大表，一个小表
 * map 阶段：BloomFilter 解决小表的key集合在内存中仍然存放不下的场景，过滤大表
 * reduce 阶段：reduce side join 
 */
public class BloomFilteringDriver {
    /**
     * 为来自不同表(文件)的key/value对打标签以区别不同来源的记录。
     * 然后用连接字段作为key，其余部分和新加的标志作为value，最后进行输出。
     */
    public static class BloomFilteringMapper extends
            Mapper<Object, Text, Text, Text> {
        // 第一个参数是vector的大小，这个值尽量给的大，可以避免hash对象的时候出现索引重复
        // 第二个参数是散列函数的个数
        // 第三个是hash的类型，虽然是int型，但是只有默认两个值
        // 哈希函数个数k、位数组大小m及字符串数量n之间存在相互关系
        //n 为小表记录数,给定允许的错误率E，可以确定合适的位数组大小，即m >= log2(e) * (n * log2(1/E))
        // 给定m和n，可以确定最优hash个数，即k = ln2 * (m/n)，此时错误率最小
        private BloomFilter filter = new BloomFilter(10000, 6, Hash.MURMUR_HASH);
        private Text joinKey = new Text();
        private Text combineValue = new Text();

        /**
         * 获取分布式缓存文件
         */
        @SuppressWarnings("deprecation")
        protected void setup(Context context) throws IOException,
                InterruptedException {
            BufferedReader br;
            String infoAddr = null;
            // 返回缓存文件路径
            Path[] cacheFilesPaths = context.getLocalCacheFiles();
            for (Path path : cacheFilesPaths) {
                String pathStr = path.toString();
                br = new BufferedReader(new FileReader(pathStr));
                while (null != (infoAddr = br.readLine())) {
                    // 按行读取并解析气象站数据
                    String[] records = StringUtils.split(infoAddr.toString(),
                            "\t");
                    if (null != records)// key为stationID
                        filter.add(new Key(records[0].getBytes()));
                }
            }

        }

        @Override
        protected void map(Object key, Text value, Context context)
                throws IOException, InterruptedException {
            String pathName = ((FileSplit) context.getInputSplit()).getPath()
                    .toString();
            // 如果数据来自于records，加一个records的标记
            if (pathName.endsWith("records-semi.txt")) {
                String[] valueItems = StringUtils.split(value.toString(),
                        "\t");
                // 过滤掉脏数据
                if (valueItems.length != 3) {
                    return;
                }
                //通过filter 过滤大表中的数据
                if (filter.membershipTest(new Key(valueItems[0].getBytes()))) {
                    joinKey.set(valueItems[0]);
                    combineValue.set("records-semi.txt" + valueItems[1] + "\t"
                            + valueItems[2]);
                    context.write(joinKey, combineValue);
                }

            } else if (pathName.endsWith("station.txt")) {
                // 如果数据来自于station，加一个station的标记
                String[] valueItems = StringUtils.split(value.toString(),
                        "\t");
                // 过滤掉脏数据
                if (valueItems.length != 2) {
                    return;
                }
                joinKey.set(valueItems[0]);
                combineValue.set("station.txt" + valueItems[1]);
                context.write(joinKey, combineValue);
            }

        }
    }
    /*
     * reduce 端做笛卡尔积
     */
    public static class BloomFilteringReducer extends
            Reducer<Text, Text, Text, Text> {
        private List<String> leftTable = new ArrayList<String>();
        private List<String> rightTable = new ArrayList<String>();
        private Text result = new Text();

        @Override
        protected void reduce(Text key, Iterable<Text> values, Context context)
                throws IOException, InterruptedException {
            // 一定要清空数据
            leftTable.clear();
            rightTable.clear();
            // 相同key的记录会分组到一起，我们需要把相同key下来自于不同表的数据分开，然后做笛卡尔积
            for (Text value : values) {
                String val = value.toString();
                System.out.println("value=" + val);
                if (val.startsWith("station.txt")) {
                    leftTable.add(val.replaceFirst("station.txt", ""));
                } else if (val.startsWith("records-semi.txt")) {
                    rightTable.add(val.replaceFirst("records-semi.txt", ""));
                }
            }
            // 笛卡尔积
            for (String leftPart : leftTable) {
                for (String rightPart : rightTable) {
                    result.set(leftPart + "\t" + rightPart);
                    context.write(key, result);
                }
            }
        }
    }

    public static void main(String[] arg0) throws Exception {
        Configuration conf = new Configuration();
        String[] args = { "hdfs://sparks:9000/middle/reduceJoin/station.txt",
                "hdfs://sparks:9000/middle/reduceJoin/station.txt",
                "hdfs://sparks:9000/middle/reduceJoin/records-semi.txt",
                "hdfs://sparks:9000/middle/reduceJoin/BloomFilte-out" };
        String[] otherArgs = new GenericOptionsParser(conf, args)
                .getRemainingArgs();
        if (otherArgs.length < 2) {
            System.err.println("Usage: BloomFilter <in> [<in>...] <out>");
            System.exit(2);
        }

        //输出路径
        Path mypath = new Path(otherArgs[otherArgs.length - 1]);
        FileSystem hdfs = mypath.getFileSystem(conf);// 创建输出路径
        if (hdfs.isDirectory(mypath)) {
            hdfs.delete(mypath, true);
        }
        Job job = Job.getInstance(conf, "bloomfilter");

        //添加缓存文件
        job.addCacheFile(new Path(otherArgs[0]).toUri());
        job.setJarByClass(BloomFilteringDriver.class);
        job.setMapperClass(BloomFilteringMapper.class);
        job.setReducerClass(BloomFilteringReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);
        //添加输入文件
        for (int i = 1; i < otherArgs.length - 1; ++i) {
            FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
        }
        //设置输出路径
        FileOutputFormat.setOutputPath(job, new Path(
                otherArgs[otherArgs.length - 1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

 

## 总结

三种 join 方式适用于不同的场景，其处理效率上相差很大，其主要导致因素是网络传输。Map join 效率最高，其次是 SemiJoin，最低的是 reduce join。另外，写分布式大数据处理程序的时最好要对整体要处理的数据分布情况作一个了解，这可以提高我们代码的效率，使数据的倾斜度降到最低，使我们的代码倾向性更好。

[MapReduce编程之Reduce Join多种应用场景与使用](https://www.cnblogs.com/LiCheng-/p/7353825.html)

[MapReduce编程之Semi Join多种应用场景与使用](https://www.cnblogs.com/LiCheng-/p/7353890.html)

[MapReduce编程之Map Join多种应用场景与使用](https://www.cnblogs.com/LiCheng-/p/7353860.html)



# MR综合案例

- [「 Hadoop」windows 下开发 WordCount 程序并提交 job 远程调试](http://www.ptbird.cn/windows-hadoop-wordcount.html)
- [「 Hadoop」使用 FileSystem API 操作 HDFS 文件](http://www.ptbird.cn/hdfs-filesystem-api.html)
- [Windows 下使用 IntelliJ IDEA 远程编写和调试 hadoop 集群](http://www.ptbird.cn/intellij-idea-for-hadoop-programming.html)



## 一、需求说明



### 1、数据文件说明

hdfs 中有一些存储温度的数据文件，以文本形式存储，示例如下：

**日期和时间中间是空格，为整体，表示检测站点监测的时间，后面是检测的温度，中间通过制表符 t 相隔。**



![img](http://static.ptbird.cn/usr/uploads/2017/02/3586987878.png)





### 2、需求

1. 计算在 1949-1955 年中, 每年的温度降序排序且每年单独一个文件输出存储

需要进行自定义分区、自定义分组、自定义排序。



## 二、解决



### 1、思路

1. 按照年份升序排序再按照每年的温度降序排序
2. 按照年份进行分组, 每一年份对应一个 reduce task



### 2、自定义 mapper 输出类型 KeyPair

可以看出，每一行温度姑且称为一个数据，每个数据中有两部分，一部分是时间，另一部分是温度。

因此 map 输出必须使用自定义的格式输出，并且输出之后需要自定义进行排序和分组等操作，默认的那些都不管用了。

**定义 KeyPair**

自定义的输出类型因为要将 map 的输出放到 reduce 中去运行，因此需要实现 hadoop 的 WritableComparable 的接口，并且该接口的模板变量也得是 KeyPair，就像是 LongWritable 一个意思（查看 LongWritable 的定义就可以知道）

**实现 WritableComparable 的接口，就必须重写 write/readFileds/compareTo 三个方法，依次作用于序列化 / 反序列化 / 比较**

同时需要重写 toString 和 hashCode 避免 equals 的问题。

KeyPair 定义如下

值得注意的是：**在进行序列化输出的时候也就是 write，里面用了将标准格式的时间（文件中显示的格式时间）进行的时间的转换，用了 DataInput 和 DataOutput**

```java
import org.apache.hadoop.io.WritableComparable;
import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

/**
 * Project   : hadooptest2
 * Package   : com.mapreducetest.temp
 * User      : Postbird @ http://www.ptbird.cn
 * TIME      : 2017-01-19 21:53
 */

/**
 * 为温度和年份封装成对象
 * year表示年份 而temp为温度
 */
public class KeyPair implements WritableComparable<KeyPair>{
    //年份
    private int year;
    //温度
    private int temp;

    public void setYear(int year) {
        this.year = year;
    }

    public void setTemp(int temp) {
        this.temp = temp;
    }

    public int getYear() {
        return year;
    }

    public int getTemp() {
        return temp;
    }
    @Override
    public int compareTo(KeyPair o) {
        //传过来的对象和当前的year比较 相等为0 不相等为1
        int result=Integer.compare(year,o.getYear());
        if(result != 0){
            //两个year不相等
            return 0;
        }
        //如果年份相等 比较温度
        return Integer.compare(temp,o.getTemp());
    }

    @Override
    //序列化
    public void write(DataOutput dataOutput) throws IOException {
       dataOutput.writeInt(year);
       dataOutput.writeInt(temp);
    }

    @Override
    //反序列化
    public void readFields(DataInput dataInput) throws IOException {
        this.year=dataInput.readInt();
        this.temp=dataInput.readInt();
    }

    @Override
    public String toString() {
        return year+"\t"+temp;
    }

    @Override
    public int hashCode() {
        return new Integer(year+temp).hashCode();
    }
}
```



### 3、自定义分组

将同一年监测的温度放到一起，因此需要对年份进行比较。

因此比较输入的数据中的年份即可，注意此时比较的都是 KeyPair 的类型，Map 出来的输出也是这个类型。

因为继承了 WritableComparator，因此需要重写 compare 方法，比较的是 KeyPair（KeyPair 实现了 WritableComparable 接口），实际比较的使他们的年份, 年份相同则得到 0

```java
/**
 * Project   : hadooptest2
 * Package   : com.mapreducetest.temp
 * User      : Postbird @ http://www.ptbird.cn
 * TIME      : 2017-01-19 22:08
 */

import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;

/**
 *  为温度分组 比较年份即可
 */
public class GroupTemp extends WritableComparator{

    public GroupTemp() {
        super(KeyPair.class,true);
    }
    @Override
    public int compare(WritableComparable a, WritableComparable b) {
        //年份相同返回的是0
        KeyPair o1=(KeyPair)a;
        KeyPair o2=(KeyPair)b;
        return Integer.compare(o1.getYear(),o2.getYear());
    }
}
```



### 4、自定义分区

自定义分区的目的是在根据年份分好了组之后，将不同的年份创建不同的 reduce task 任务，因此需要对年份处理。

```java
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Partitioner;

/**
 * Project   : hadooptest2
 * Package   : com.mapreducetest.temp
 * User      : Postbird @ http://www.ptbird.cn
 * TIME      : 2017-01-19 22:17
 */

//自定义分区
//每一个年份生成一个reduce任务
public class FirstPartition extends Partitioner<KeyPair,Text>{
    @Override
    public int getPartition(KeyPair key, Text value, int num) {
        //按照年份进行分区 年份相同,返回的是同一个值
        return (key.getYear()*127)%num;
    }
}
```



### 5、自定义排序

最终还是比较的是温度的排序，因此这部分也是非常重要的。

根据上面的需求，需要对年份进行生序排序，而对温度进行降序排序，首选比较条件是年份.

```java
/**
 * Project   : hadooptest2
 * Package   : com.mapreducetest.temp
 * User      : Postbird @ http://www.ptbird.cn
 * TIME      : 2017-01-19 22:08
 */

import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;

/**
 *  为温度排序的封装类
 */
public class SortTemp extends WritableComparator{

    public SortTemp() {
        super(KeyPair.class,true);
    }
    //自定义排序
    @Override
    public int compare(WritableComparable a, WritableComparable b) {
        //按照年份升序排序 按照温度降序排序
        KeyPair o1=(KeyPair)a;
        KeyPair o2=(KeyPair)b;
        int result=Integer.compare(o1.getYear(),o2.getYear());
        //比较年份 如果年份不相等
        if(result != 0){
            return result;
        }
        //两个年份相等 对温度进行降序排序，注意 - 号
        return -Integer.compare(o1.getTemp(),o2.getTemp());
    }
}
```



### 6、MapReduce 程序的编写

几个值得注意的点：

1. 数据文件中前面的时间是字符串，但是我们的 KeyPair 的 set 却不是字符串，因此需要进行字符串转日期的 format 操作，使用的是 SimpleDateFormat，格式自然是 "yyyy-MM-dd HH:mm:ss" 了。
2. 输入每行数据之后，通过正则匹配 "t" 的制表符，然后将温度和时间分开，将时间 format 并得到年份，将第二部分字符串去掉 “℃” 的符号得到数字，然后创建 KeyPair 类型的数据，在输出即可。

1. 每个年份都生成一个 reduce task 依据就是自定义分区中对年份进行了比较处理，为了简单就把 map 的输出结果在 reduce 中再输出一次，三个 reduce task，就会生成三个输出文件。
2. 因为使用了自定义的排序，分组，分区，因此就需要进行指定相关的 class，同时也需要执行 reduce task 的数量。
3. **其实最后客户端还是八股文的固定形式而已，只不过多了自定义的指定，没有别的。**

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;

import java.io.IOException;
import java.net.URI;
import java.text.SimpleDateFormat;
import java.util.Calendar;
import java.util.Date;

/**
 * Project   : hadooptest2
 * Package   : com.mapreducetest.temp
 * User      : Postbird @ http://www.ptbird.cn
 * TIME      : 2017-01-19 22:28
 */
public class RunTempJob {
    //字符串转日期format
    public static SimpleDateFormat SDF=new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    /**
     * Mapper
     * 输出的Key是自定义的KeyPair
     */
    static class TempMapper extends Mapper<LongWritable,Text,KeyPair,Text>{
        protected void map(LongWritable key,Text value,Context context) throws IOException,InterruptedException{
            String line=value.toString();
            //1949-10-01 14:21:02    34℃
            // 前面是空格 时间和温度通过\t分割
            String[] ss=line.split("\t");
//            System.err.println(ss.length);
            if(ss.length==2){
                try{
                    //获得日期
                    Date date=SDF.parse(ss[0]);
                    Calendar c=Calendar.getInstance();
                    c.setTime(date);
                    int year=c.get(1);//得到年份
                    //字符串截取得到温度，去掉℃
                    String temp = ss[1].substring(0,ss[1].indexOf("℃"));
                    //创建输出key 类型为KeyPair
                    KeyPair kp=new KeyPair();
                    kp.setYear(year);
                    kp.setTemp(Integer.parseInt(temp));
                    //输出
                    context.write(kp,value);
                }catch(Exception ex){
                    ex.printStackTrace();
                }
            }
        }
    }
    /**
     *  Reduce 区域
     *  Map的输出是Reduce的输出
     */
    static class TempReducer extends Reducer<KeyPair,Text,KeyPair,Text> {
        @Override
        protected void reduce(KeyPair kp, Iterable<Text> values, Context context) throws IOException, InterruptedException {
            for (Text value:values){
                context.write(kp,value);
            }
        }
    }

    //client
    public static void main(String args[]) throws IOException, InterruptedException{
        //获取配置
        Configuration conf=new Configuration();

        //修改命令行的配置
        String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
        if (otherArgs.length != 2) {
            System.err.println("Usage: temp <in> <out>");
            System.exit(2);
        }
        //创建Job
        Job job=new Job(conf,"temp");
        //1.设置job运行的类
        job.setJarByClass(RunTempJob.class);
        //2.设置map和reduce的类
        job.setMapperClass(RunTempJob.TempMapper.class);
        job.setReducerClass(RunTempJob.TempReducer.class);
        //3.设置map的输出的key和value 的类型
        job.setMapOutputKeyClass(KeyPair.class);
        job.setMapOutputValueClass(Text.class);
        //4.设置输入文件的目录和输出文件的目录
        FileInputFormat.addInputPath(job,new Path(otherArgs[0]));
        FileOutputFormat.setOutputPath(job,new Path(otherArgs[1]));
        //5.设置Reduce task的数量 每个年份对应一个reduce task
        job.setNumReduceTasks(3);//3个年份
        //5.设置partition sort Group的class
        job.setPartitionerClass(FirstPartition.class);
        job.setSortComparatorClass(SortTemp.class);
        job.setGroupingComparatorClass(GroupTemp.class);
        //6.提交job 等待运行结束并在客户端显示运行信息
        boolean isSuccess= false;
        try {
            isSuccess = job.waitForCompletion(true);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        //7.结束程序
        System.exit(isSuccess ?0:1);
    }
}
```

三、生成效果：

HDFS 中三个 reduce task 会生成三个输出。



![img](http://static.ptbird.cn/usr/uploads/2017/02/1039351296.png)



每个输出文件都是每年中的温度的排序结果：



![img](http://static.ptbird.cn/usr/uploads/2017/02/3588994659.png)



**可以看出，1951 是 map（也可以说是 KeyPair）输出的年份，46 是温度，而后面是将 text 又输出了一次，每一年都是根据需求降序排序的。）**