---
layout: post
title: Optimizing GC time by reducing Humongous allocations
---

Recently I was debugging into latency problems in a service and figured quite a few GC related issues on the way. This is a summary of the findings. 

**Background**

This was a Java 8 service running with ParallelGC, primarily consuming messages from an SQS, processing, and sending the response back to another SQS. The processing
involved a bunch of serialization/deserialization along with quite a lot of network calls.

Upon enabling G1GC for the same service, the GC activities spiked up by at least 15% and processing latency increased proportionally. The below is a gist of
finding out issues related to GC and resolving those.

**Step 1 - Enable GC logs**

Enabling GC logs provides more details on whats going on in the service.

~~~
-XX:+PrintGCDetails 
-XX:+PrintGCDateStamps
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintGCApplicationConcurrentTime
-XX:+PrintTenuringDistribution
-XX:+PrintAdaptiveSizePolicy
-XX:+UseStringDeduplication
-XX:+PrintStringDeduplicationStatistics
~~~

**Step 2 - Analyze the logs**

After enabling detailed GC activity logging, we could see that a lot of GC pauses were ocuring due to `Humongous Object` allocations. [Humongous Objects](https://docs.oracle.com/javase/10/gctuning/garbage-first-garbage-collector.htm#JSGCT-GUID-D74F3CC7-CC9F-45B5-B03D-510AEEAC2DAC) mean, memory
chunks of size more than 50% of the G1 block size. The G1 block size depends on the heap memory size and for us it was 4M. So, essentially, any memory allocation more than
2M was considered as `Humongous` and too many of those was creating high GC activities and pause times.

The logs were analyzed using [GCEasy](https://gceasy.io/). A really good handy tool to quickly analyze the GC logs.

Sample logs:

~~~
2021-11-26T09:20:57.159+0000: 87437.995: [GC pause (G1 Humongous Allocation) (young) (initial-mark)
2021-11-26T09:21:01.763+0000: 87442.598: [GC pause (G1 Humongous Allocation) (young)
2021-11-26T09:21:04.983+0000: 87445.819: [GC pause (G1 Humongous Allocation) (young)
~~~

**Step 3 - Tracing the humongous allocations**

Now that we know that humnogous object allocations are causing so much of trouble, the next step is to identify where exactly the allocations are happening. There are a few blog posts like [this](https://www.pingtimeout.fr/posts/2020-01-23-trace-humongous-allocations-with-bpf/) which explain a way to trace them. However I tried with [JavaFlightRecorder](https://docs.oracle.com/javacomponents/jmc-5-4/jfr-runtime-guide/about.htm) which worked quite smooth. 

Recording with Java Flight Recorder:

~~~
jdk1.8/bin/jcmd 2092 JFR.start settings=profile
2092:
Started recording 10. No limit specified, using maxsize=250MB as default.

Use jcmd 2092 JFR.dump name=10 filename=FILEPATH to copy recording data to file.
~~~

Once the output of JFR is available, we could analyze with [JDK Mission Control](https://www.oracle.com/java/technologies/jdk-mission-control.html).

The humongous allocations will be visible under `Event Browser -> Allocation Outside TLAB`.

![ScreenShot](https://cafeaffe.minbox.in/img/JDKMissionControl_HumongousAllocation.png)

**Step 4 - Removing humongous allocations**

As seen above, the humongous allocations were created during certain `serialization` process. On going through the code, it was seen that the code flow actually
serialized a java object to bytes, compressed it to a gzip bytes, encoded within Base64 and then uploaded to S3.

The serialization flow does this:

1. ObjectMapper writes to a byte array: This allocates the most of the memory as seen above in the JFR recording.
2. The byte array is gzipped creating another byte array, shorter in size
3. The resulting byte array is Base64 encoded which creates another byte array of a bit larger size.
4. The byte array is then converted to InputStream and is uploaded to S3.

Sample code:

~~~
public byte[] serialize(T obj) throws IOException, ParseException {
    byte[] bytes = this.mapper.writeValueAsBytes(obj);

    ByteArrayOutputStream output = new ByteArrayOutputStream(1024);

    GZIPOutputStream gzip = new GZIPOutputStream(output) {
        {
            def.setLevel(Deflater.BEST_COMPRESSION);
        }
    };
    
    gzip.write(bytes);
    gzip.close();

    return output.toByteArray();

}
/* ... Else where ... */

byte[] serialized = Base64.encodeBase64(serialize(object));

ObjectMetadata objectMetadata = new ObjectMetadata();
objectMetadata.setContentLength(serializedArtifacts.length);

PutObjectRequest putObjectRequest = new PutObjectRequest(s3Bucket,
    "some-s3-key",
    new ByteArrayInputStream(serialized),
    objectMetadata);

s3Client.putObject(putObjectRequest);

~~~


The above flow actually created 3 new instances of `byte[]` in each execution. Could we do better? Can we remove certain byte array creations?

Turns out we can.

The new flow looks like:

1. Object mapper can write to an outputStream instead of a byte array.
2. OutputStream can be wrapped inside Base64.encoder which will not require additional array creation for base64 conversion.
3. Gzip can take the “wrapped” output stream
4. ObjectMapper can directly write to the Gzipped stream. So no need to create a byte array.
5. S3 takes an InputStream. So we can directly convert the above output stream to an inputStream.

Sample code after the changes:

Serilization:

~~~
public InputStream serializeToInputStream(T obj) throws IOException {
    ByteArrayOutputStream output = new ByteArrayOutputStream(1024);

    OutputStream stream = java.util.Base64.getEncoder().wrap(output);
    GZIPOutputStream gzip = new GZIPOutputStream(stream)
    {
        {
            def.setLevel(Deflater.BEST_COMPRESSION);
        }
    };

    this.mapper.writeValue(gzip, obj);

    PipedInputStream in = new PipedInputStream();
    final PipedOutputStream out = new PipedOutputStream(in);
    new Thread(() -> {
        try {
            output.writeTo(out);
        } catch (IOException e) {
            throw new RuntimeException("Failed to convert to InputStream", e);
        } finally {
            // close the PipedOutputStream here because we're done writing data
            // once this thread has completed its run
            try {
                out.close();
            } catch (IOException e) {
                // ignore
            }
        }
    }).start();
    return in;
}
~~~

And then use the above `InputStream` to upload objects to S3:

~~~
InputStream inputStream = serializer.serializeToInputStream(object);

PutObjectRequest putObjectRequest = new PutObjectRequest(s3Bucket,
               "some-s3-key",
                inputStream,
                objectMetadata);
 
s3Client.putObject(putObjectRequest);
~~~

With the above changes, I ran the JFR once again, and the humongous object allocations were gone! With a substantial (~15%) improvement in GC activities and reduction of FullGC times.

**Conclusion**

* We should avoid creation of un-necessary byte arrays if we can work with IO streams. The catch here is that, S3 upload might fail in mid-stream and that needs to be taken care of in the code.
* We should avoid creation of intermediate strings through ObjectMapper. Not covered here in this post, but I did find quite a few instances where we were doing `objectMapper.writeValueAsString()` and then getting a byte array out of it. The intermediate strings take almost 3 times more memory than byte arrays and provides no great use.
* JFR and JMC are great tools to get better insights into application behavior.

**Additional Resources**

 - https://www.infoq.com/articles/tuning-tips-G1-GC/
 - https://www.oracle.com/technical-resources/articles/java/g1gc.html
 - https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector-tuning.htm#JSGCT-GUID-43ADE54E-2054-465C-8376-81CE92B6C1A4

