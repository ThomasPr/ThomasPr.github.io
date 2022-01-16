---
title:  "Release memory back to the OS with Java 11"
date:   2021-05-02 16:19:03 +0200
excerpt: "Java 11 is by default very reluctant to release unnecessary memory back to the operation system. The Shenandoah GC is more aggressive and available in Java 11."
header:
  og_image: /assets/images/2021-05-02/os2.png
---

I'm responsible for a Java application running in OpenShift. This application has to process a huge amount of data occassionally, but most of the time the application idles and waits for new input.

The application takes a huge amount of memory during processing the data. But when the processing job has been completed, the memory can be released and returned to the operating system. Unfortunately, this doesn't happen. The JVM seems to keep the memory forever.

![OpenShift Metrics](/assets/images/2021-05-02/os1.png)


## What's going on?

>  Currently the G1 garbage collector may not return committed Java heap memory to the operating system in a timely manner. G1 only returns memory from the Java heap at either a full GC or during a concurrent cycle. Since G1 tries hard to completely avoid full GCs, and only triggers a concurrent cycle based on Java heap occupancy and allocation activity, it will not return Java heap memory in many cases unless forced to do so externally. -- [JEP 346](https://openjdk.java.net/jeps/346)

The motivation of JEP 346 describes perfectly what I observed from my application. When it should release memory, there's no need anymore to run the garbage collector at all. Therefore, it will keep the memory until the next huge processing phase starts and requires the memory again.


## How can this be fixed?

[JEP 346](https://openjdk.java.net/jeps/346) has exactly this issue in mind: Promptly Return Unused Committed Memory from G1. JEP 346 has been implemented in Java 12. Unfortunately, I have to use Java 11 and cannot benefit from this improvment.

But there are other garbage collectors than the default G1. [Ruslan Synytsky](https://jelastic.com/blog/tuning-garbage-collector-java-memory-usage-optimization/) has a well-written blog post about the memory consumption of different garbage collectors.

Based on his observations there's a huge difference in the behaviour of the memory consumption. It seems that the Shenandoah GC might be a really good option for my application. Next to ZGC, Shenandoah is one of the newest garbage collectors and available as production ready in Java 15. But Shenandoah has been backported to OpenJDK 11 and is available since version 11.0.9. Therefore I'm able to use it. Let's give it a try:

    java -XX:+UseShenandoahGC -jar app.jar

![OpenShift Metrics](/assets/images/2021-05-02/os2.png)

It works! As you can see, some time after finishing the hard processing work, Shenandoah returns most of the memory back to the operating system.

Since reducing the heap is an expensive operation, Shenandoah takes a delay of 5 minutes (300,000 ms) by default to release any memory back to the operating system. You can set Shenandoah to be more aggressive, but the command line options for tuning Shenandoah are still marked as experimental. However, I'm fine with the default 5 minute delay.

```sh
java \
  -XX:+UseShenandoahGC \
  -XX:+UnlockExperimentalVMOptions \
  -XX:ShenandoahUncommitDelay=1000 \
  -XX:ShenandoahGuaranteedGCInterval=10000 \
  -jar app.jar
```

Comments are welcome on [Twitter](https://twitter.com/TheThomasPr/status/1388916641178718208) or [LinkedIn](https://www.linkedin.com/posts/thomas-preissler_release-memory-back-to-the-os-with-java-11-activity-6857659740864987136-oBwF).

## Links

* [JEP 346: Promptly Return Unused Committed Memory from G1](https://openjdk.java.net/jeps/346)
* [Garbage Collector Tuning as the First Step to Java Memory Usage Optimization](https://jelastic.com/blog/tuning-garbage-collector-java-memory-usage-optimization/)
* [Shenandoah GC in production: experience report](http://clojure-goes-fast.com/blog/shenandoah-in-production/)
* [Stackoverflow: Does GC release back memory to OS?](https://stackoverflow.com/questions/30458195/does-gc-release-back-memory-to-os)
* [Stackoverflow: Does G1 GC release back memory to the OS even if Xms = Xmx?](https://stackoverflow.com/questions/59362760/does-g1gc-release-back-memory-to-the-os-even-if-xms-xmx)
