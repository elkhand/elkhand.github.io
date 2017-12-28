---
layout: post
comments: true
title:  "Solution for Reducer Container is running beyond physical memory limits"
excerpt: "This will help you running your map-reduce jobs at scale of 100 TBs and avoid reducer containers being killed"
date:   2017-12-27 20:00:00 -0700
categories: Hadoop MapReduce Scale
mathjax: true
---


## Scenario: Container is running beyond physical memory limits

Reducer containers are getting killed with log msg:

"Container [pid=115232,containerID=container_e21_1512491152219_26472_01_000253] is **running beyond physical memory limits**. Current usage: 5.0 GB of 5 GB physical memory used; 6.7 GB of 10.5 GB virtual memory used. Killing container...""
```
mapreduce.reduce.memory.mb : 5120
mapreduce.reduce.java.opts : 4608
```

### TL;DR  (Old Conclusion with logical reasoning)
Keep enough memory buffer between JVM memory and container memory allocation.
Recommended value is usually JVM heap memory being 80% of the total reduce/map container memory.

### At first, want to mention that this is <u>not</u> OOM exception causing this failure. 

OOM usually occurs because of memory leak in your code, when you try to allocate more memory than you have for your Java objects in JVM heap.

### Issues with decompressors used in reducer

In almost all mapreduce jobs we enable compression for map output, and for the job output. 

**SnappyDecompressor, Bzip2Decompressor, ZlibDecompressor all use direct buffers.**

Usually SnappyCodec is used map output compression. So I choose SnappyDecompressor as an example:

SnappyCodec uses SnappyDecompressor, which uses direct byte buffers:
```
	compressedDirectBuf = ByteBuffer.allocateDirect(directBufferSize);
	uncompressedDirectBuf = ByteBuffer.allocateDirect(directBufferSize);
```

**_Ref_**: http://grepcode.com/file/repo1.maven.org/maven2/org.apache.hadoop/hadoop-common/2.7.1/org/apache/hadoop/io/compress/snappy/SnappyDecompressor.java#SnappyDecompressor


####  Some more insights of direct byte buffer:
"Direct buffers are intended for interaction with channels and native I/O routines. They make a best effort to store the byte elements in a memory area that a channel can use for direct, or raw, access by using native code to tell the operating system to drain or fill the memory area directly.

Direct buffers are optimal for I/O, but they may be more expensive to create than nondirect byte buffers. The memory used by direct buffers is allocated by calling through to native, operating system-specific code, bypassing the standard JVM heap. **The memory-storage areas of direct buffers are not subject to garbage collection because they are outside the standard JVM heap.**"

**_Ref_**: https://stackoverflow.com/questions/5670862/bytebuffer-allocate-vs-bytebuffer-allocatedirect

**_Ref_**: https://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html

Thus when Snappy is used for intermediate compression, Snappy uses off-heap memory, and when it does not have enough space to allocate it causes your reduce tasks fail, which ends up your job failing.

### Solution
One solution will be keeping enough memory between JVM memory allocation and container memory allocation so that enough off-heap memory will be there for decompression. Decreasing your JVM memory allocation will make your job run slower (as you can create/hold fewer objects in JVM heap), but will not cause your job failure.

Recommended value is usually JVM heap memory being 80% of the total reduce/map container memory.

**_Ref_**: http://crazyadmins.com/tag/tuning-yarn-to-get-maximum-performance/


