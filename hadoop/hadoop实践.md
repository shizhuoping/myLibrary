# 初识hadoop

## 背景

随着数据增长的速度越来越快，处理大数据的方法也应运而生。有一句话叫“大数据胜过好算法”，意思是对于某些应用场合，不论算法有多牛，基于小数据的算法效果不如基于大数据的一般算法。

多年来硬盘存储容量在不断提升，但是硬盘数据的读取速度却没有与时俱进，读完整个硬盘的速度反而增长了几十倍甚至更多，但是如果我们把数据分散在100个硬盘，然后并行读取，就可以使读取速度大大提升，但是这其中有很多问题需要解决：

1、硬件故障问题：为了避免数据丢失，最常见的做法是保存数据的副本，一旦某个节点发生故障就可以使用其他副本。

2、大多数分析任务需要以某种方式结合来自不同节点的数据共同分析。

hadoop针对这些问题，为我们提供了一个可靠的共享存储和分析系统。

## 和关系型数据库的比较

为什么不能用数据库来对大量硬盘上的大规模数据进行批量分析呢？这个问题的答案在于：传输速率的提升远大于寻址时间的提升，相对于快速读写磁盘的数据库，利用快速的传输更合算。

关系型数据库和MapReduce比较：

![QQ图片20200423133810](.\QQ图片20200423133810.png)

MapReduce对非结构化数据或半结构化数据非常有效，它只有在处理数据的时候才对数据进行解释。此外，它还有一个特点：数据本地化，也就是尽量在计算节点上存储数据，因为在集群计算的时候网络带宽是珍贵的资源，这种特性可以尽量不让网络带宽拖慢计算速度。

# 气象数据案例

## 需求分析

分析每年全球气温的最高纪录，气温数据是分行写的，一行代表一次观测：

~~~
0029029070999991901010106004+64333+023450FM-12+000599999V0202701N015919999999N0000001N9-00781+99999102001ADDGF108991999999999999999999
~~~

这里的1901代表年份，气温数据是-0078（摄氏度的10倍），这个数据如果是正数还要加上正号。温度后面的数据1代表质量等级，如果是0、1、4、5、9其中之一就代表数据是合格的，如果气温数据是9999代表检测失败，数据不合格。

## mapper类

~~~java
package hadoop6;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;

public class MaxTemperatureOfYearMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

    int unVaildTem = 9999;
    Text resultYear = new Text();
    IntWritable resultTemperature = new IntWritable();

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

        String text = value.toString();
        String year = text.substring(15, 19);
        String isNegative = text.substring(87, 88);
        int temperature = new Integer(text.substring(88, 92));
        String isVaild = text.substring(92, 93);

        if("-".equals(isNegative)){
            temperature *= -1;
        }
        if(Math.abs(temperature) != unVaildTem && isVaild.matches("[01459]")){
            resultYear.set(year);
            resultTemperature.set(temperature);
            context.write(resultYear, resultTemperature);
        }
    }
}
~~~

## reducer类

~~~java
package hadoop6;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;

public class MaxTemperatureOfYearReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

    IntWritable maxTemperature = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        int max = Integer.MIN_VALUE;
        for (IntWritable temperature : values) {
            int temperatureVal = temperature.get();
            max = Math.max(temperatureVal, max);
        }
        maxTemperature.set(max);
        context.write(key, maxTemperature);
    }
}
~~~

## Driver类

~~~java
package hadoop6;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class MaxTemperatureOfYearDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration);

        job.setJarByClass(MaxTemperatureOfYearDriver.class);
        job.setMapperClass(MaxTemperatureOfYearMapper.class);
        job.setReducerClass(MaxTemperatureOfYearReducer.class);

        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);

        FileInputFormat.setInputPaths(job,new Path("1901"));
        FileOutputFormat.setOutputPath(job, new Path("output"));

        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
~~~

