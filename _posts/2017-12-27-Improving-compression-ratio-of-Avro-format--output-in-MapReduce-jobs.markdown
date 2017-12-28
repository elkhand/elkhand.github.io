---
layout: post
comments: true
title:  "Improving compression ratio of Avro format output in MapReduce jobs"
excerpt: "syncByteInterval parameter will help you achieve best compression while writing into Avro format in your MapReduce jobs"
date:   2017-12-27 20:00:00 -0700
categories: Avro Compression Hadoop MapReduce Scale 
mathjax: true
---

### What is Avro ?

Avro is one of the most widely used data serializaiton formats in Hadoop eco-system. Avro is row based format, and provides schema with each avro file, which enables reading the avro file without storing schema separately. Avro also supports schema evolution and projection. 

One of the main reasons we use Avro format, it helps us to store metadata in addition to data itself(log data) in structured format and in the same file.

**_Ref_**: https://avro.apache.org/docs/current/

### What can be done for achieving better compression?

You have set up Deflate level to 9 (or 6), if you are using Defalte codec for compression, or just using Bzip2. 

As Avro is row based, thus changing the order fields does not help for compression. 

## syncByteInterval parameter for achieving much higher compression ratio

**So by setting syncByteInterval into `(>=4M)` value, you can achieve best compression ratio.**

### What is syncByteInterval attribute ?

syncIntervalBytes determines the size of the data buffer used before flushing the data to HDFS and therefore this setting can affect your compression ratio when using codec.

**_Ref_**: https://books.google.com/books?id=u1bTBgAAQBAJ&pg=PA45&lpg=PA45&dq=Avro+avro.mapred.sync.interval&source=bl&ots=WgDmDr-166&sig=ybjFN12m5526gnWZtXIBRBbHyeM&hl=en&sa=X&ved=0ahUKEwirgpHygKDUAhXJiFQKHQ60BsY4ChDoAQg-MAY#v=onepage&q=Avro%20avro.mapred.sync.interval&f=false 

By default Avro implementations does not set syncByteInterval to high enough value. Avro 1.7.7 version sets syncByteInterval into 64 KB only, but setting it to higher value you can achieve better compression than using default values. (Though it depends on compression codec used)

In Avro 1.7.7 version Avro is using default sync interval of 64kb:


```java
  public static final int SYNC_SIZE = 16;
  public static final int DEFAULT_SYNC_INTERVAL = 4000*SYNC_SIZE; 
```

**_Ref_**: http://grepcode.com/file/repo1.maven.org/maven2/org.apache.avro/avro/1.7.7/org/apache/avro/file/DataFileConstants.java/


## Setting syncByterInterval via Configuration object in MapReduce job

Here you can see how syncByteInterval is set in `AvroOutputFormat class`.

```java
public class AvroOutputFormat <T>
  extends FileOutputFormat<AvroWrapper<T>, NullWritable> {
  ...
    /** The configuration key for Avro sync interval. */
  public static final String SYNC_INTERVAL_KEY = "avro.mapred.sync.interval";
  ...

  /** Set the sync interval to be used by the underlying {@link DataFileWriter}.*/
  public static void setSyncInterval(JobConf job, int syncIntervalInBytes) {
    job.setInt(SYNC_INTERVAL_KEY, syncIntervalInBytes);
  }
  
  static <T> void configureDataFileWriter(DataFileWriter<T> writer,
      JobConf job) throws UnsupportedEncodingException {
    ...
    writer.setSyncInterval(job.getInt(SYNC_INTERVAL_KEY, DEFAULT_SYNC_INTERVAL));
  }
  ...
}
```

**_Ref_**: http://grepcode.com/file_/repo1.maven.org/maven2/org.apache.avro/avro-mapred/1.7.7/org/apache/avro/mapred/AvroOutputFormat.java/?v=source 


So you can set in configuration object:
```java
conf.set("avro.mapred.sync.interval",134217728); // 128 MB
```

### Comparing different syncByteInterval values affecting Deflate 9 and Bzip2 compressions for data stored in Avro format

Same input data was compressed with different syncByteInterval values and results were compared for Deflate 9 and Bzip2 compression codecs:

|syncIntervalInBytes |Deflate 9 Output (GB) |	Space Saved with Defalte 9 (%) |	Bzip2 Output (GB) |Space Saved with Bzip2 (%) |
|      ------------- |:       -------------:|                            -----:|                -----:|                     -----:|
|               16KB |                  1.3 |                      43.47826087 |                  1.3 | 	          43.47826087 |
|                4 M |                  1.2 |                      47.82608696 |               0.8446 |           **63.27826087** |
|                8 M |                  1.2 |                      47.82608696 |                 0.84 |               63.47826087 |
|               16 M |                  1.2 |                      47.82608696 |               0.8372 |                      63.6 |
|              128 M |                  1.2 |                      47.82608696 |               0.8341 |               63.73478261 |


**Space Saved = [100- (100 * output / input)]**

As you can see, for deflate codec increasing syncByteInterval does not help much for achieving  better compression.

But for Bzip2 compression with setting syncByteInterval to higher value `(>= 4M)` you can achieve 20% better compression. syncIntervalBytes determines the size of the data buffer used before flushing the data to HDFS, thus compression algorithm will perform better when applied to more data you have in buffer. 

With syncByteInterval set to value `(>= 4M)`, you can achieve almost same compression ratio of Avro fileif you have stored same data in text format. So you do not get any space overhead.

Another key point to keep in mind is, bzip2 compression is relatively slower than Deflate codec. So if the next steps of your pipeline cannot wait long, you go with Deflate codec ( with deflate level 6, as deflate level 9 is much slower and gives only ~1% better compression than 6%).

### Conclusion
By default Avro implementations does not set syncByteInterval to high enough value. But by setting it to higher value you can achieve better compression than using default provided values.
