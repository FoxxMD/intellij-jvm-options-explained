# Intellij Jvm Options Explained

Intellij is run on the Java Virtual Machine (VM). The VM has many options/flags that can be used to tune its performance which, by extension, is IntelliJ's performance. This guide can help you get more performance out of your Jetbrains IDE or help you fix problems occuring due to the VMs default configuration. I wrote this guide from the perspective of a person curious about what all these options mean but has no idea what they do.

# TL;DR

Don't care about the nitty gritty? Just want to copy and paste a slew of options? Look no further!

This should be a good lowest common denominator set of settings to help increase performance in Intellij.

Replace brackets with values so:
```
-Xmx[memoryValue]
```
becomes
```
-Xmx512m
```
___ 

```
-server
-ea
-Xmx[memoryValue, If total memory < 2GB then at least 1/4 total memory. If > 2GB then 1-4 GB. (See note below)]
-Xms[memoryValue, at least 1/2 Xmx. Can be = to Xmx]
-XX:+UseG1GC
-XX:-UseParNewGC
-XX:-UseConcMarkSweepGC
-XX:ReservedCodeCacheSize=[between 128m and 256m, depending on how much free physical memory you have available]
-XX:-OmitStackTraceInFastThrow
-Dsun.io.useCanonCaches=false
```

**Note:** To tune **-Xmx** for your project pay attention to the [**Memory Indicator**](https://www.jetbrains.com/help/idea/2016.2/status-bar.html).
* If you have free memory increase **-Xmx** until Memory Indicator usage is < 85%. 
* If you are memory constrained and can't reach that target **-Xmx** should be at least 1/2 to 3/4 total physical memory. 

# How to Edit VM Options

The easiest way to edit these options for your Jetbrains IDE is to use the methods described in this [article](https://intellij-support.jetbrains.com/hc/en-us/articles/206544869-Configuring-JVM-options-and-platform-properties):

> The recommended way of changing the JVM options in the recent product versions is from the **Help | Edit Custom VM Options** menu. This action will create a copy of the .vmoptions file in the IDE config directory and open an editor where you can change them.

If that method is unavailabe the articles lists out manual ways to edit those files. Read it carefully!

# VM Options, Explained!

## Heap on the VM

Intellij is run on the Java Virtual Machine (VM). The **[heap](https://javarevisited.blogspot.com/2011/05/java-heap-space-memory-size-jvm.html)** is the memory (in RAM) that is allocated to the java virtual machine. **So the heap size is total amount of memory that Intellij can use to operate.**

**-Xmx[memoryVal]** - [Specifies the maximum memory allocation of the heap for the VM.](https://stackoverflow.com/a/14763095) This is the **maximum** amount of memory Intellij can use.
* If you are frequently recieving `OutOfMemoryError` errors from Intellij you should increase this value.
* Increasing this value should also speed up processes like inspection time for large files as Intellij will have to do less swapping of memory to stay inside the limit (?)

**-Xms[memoryVal]** - [Specifies the initial memory allocation of the heap for the VM.](https://stackoverflow.com/a/14763095) This will be the amount of memory Intellij starts with.
* Increasing this value should make Intellij start up faster as it will have a larger pool initially and so will not have to wait to allocate more memory as it spins up.

## Garbage Collection

The **garbage collector** (GC) is a program in the VM that removes unused objects/resources and frees up space on the heap.

From the garbage collector's perspective there are two main sections, called **generations**, inside the heap: 

* Young Generation - New objects are created in the young generation.
* Old Generation - Objects that are "live" and have survived a few GC (garbage collections) get moved to the old generation, as well as any objects too big for the young generation.

[Garbage collection is done on both generations, but each has different consequences.](http://crunchify.com/jvm-tuning-heapsize-stacksize-garbage-collection-fundamental/)

* GC on the young generation is "little". Most objects "die" in the young generation, never to be used again. So collection on this generation is considered minor in terms of impact on the VM. 
* GC on the old generation is considered "full GC" and has a much larger impact on the VM as it collects across the entire heap. GC here only happens when there is not enough memory on the heap to allocate new objects.

[Controlling the size of the young generation is an important tuning tool](https://blog.codecentric.de/en/2012/08/useful-jvm-flags-part-5-young-generation-garbage-collection/):
>If the young generation is too small, short-lived objects will quickly be moved into the old generation where they are harder to collect. Conversely, if the young generation is too large, we will have lots of unnecessary copying for long-lived objects that will later be moved to the old generation anyway.

So choosing the right size depends on how you use Intellij. If you are constantly opening new files and jumping around projects you may want a larger young generation so that objects aren't being moved to the old generation quickly. If you are mostly static in your usage (staying in one area of  a project or working on small scopes) a smaller young generation may be better for you as you won't be pushing new objects into the heap as often.

There are two ways to manage the young generation:

**-XX:NewRatio=[ratio]** - [Specifies the young generation size in relation to the size of the old generation.](https://blog.codecentric.de/en/2012/08/useful-jvm-flags-part-5-young-generation-garbage-collection/) 
* The potential advantage of this approach is that the young generation will grow and shrink automatically when the JVM dynamically adjusts the total heap size at run time.
* Use this if the projects you work with vary drastically in their memory usage.
* I recommend a ratio between 2 and 4. This means the young generation will be 1/3 or 1/4 the size of the old generation.

**-Xmn[memoryVal]** - [Specifies the memory allocated to the young generation.](http://blog.sokolenko.me/2014/11/javavm-options-production.html) 
* [This flag is equivalent](https://blogs.oracle.com/jonthecollector/entry/the_second_most_important_gc) to -XX:NewSize and -XX:MaxNewSize, setting the same value for both.
* The only advantage this has compared to NewRatio is it allows more granular control of the young generation size.

### GC Algorithm

(IMO) There are two main GC approaches that can be used.

**Note: If your heap is larger than 4GB you should *always* use G1**

#### CMS 

* **-XX:+UseConcMarkSweepGC** 
* **-XX:+UseParNewGC**

This particular combination [uses mutliple threads to attempt to do GC in the background as to avoid application stopping](http://www.fasterj.com/articles/oraclecollectors1.shtml). If you experience Intellij becoming jerky/unresponsive during heavy usage this may alleviate those problem.

#### G1 (Recommended)

* **-XX:+UseG1GC**
* **-XX:-UseParNewGC**
* **-XX:-UseConcMarkSweepGC**

Introduced in JDK 7 Update 4 [G1 is similar to CMS but is built to support large heaps (4GB and larger).](http://blog.takipi.com/garbage-collectors-serial-vs-parallel-vs-cms-vs-the-g1-and-whats-new-in-java-8/) It breaks up the generations into smaller regions and keeps tracks of regions that produce more garbage. This is supposed to prevent large "stop the world" GC events that can occur in large heaps.

### Other GC Flags

**-XX:ParallelGCThreads=[value]** - [Specifies the number of GC thread to use for parallel GC](https://blog.codecentric.de/en/2013/01/useful-jvm-flags-part-6-throughput-collector/) (ParNewGC)
* If not explicitly set this flag, the JVM will use a default value which is computed based on the number of available (virtual) processors. This is what you normally want to do (not set it).
* Useful if you have more than one JVM running and you want to reduce resource use.

**-XX:SurvivorRatio=[ratio]** - Specifies the size of survivor generations inside the young generation.
* A survivor generation holds all of the objects still "live" after a GC on the young generation but not yet passed to the old generation.
* Normally not a huge factor in performance.
* Most values I see are between 8 and 6.

## Everything Else

**-Xss=[memoryValue]** - [Specifies the memory allocated to a new stack](http://www.onkarjoshi.com/blog/209/using-xss-to-adjust-java-default-thread-stack-size-to-save-memory-and-prevent-stackoverflowerror/)
* Every thread created in the VM gets its own stack space. This is space used to store local variables and references.
* Generally does not need to modified. However a smaller stack size will use less memory.

**-XX:PermSize=[memoryValue]** and **-XX:MaxPermSize=[memoryValue]**
* These refer to the **perm size** which is the memory used by the VM for data **outside of your application, just for the VM**. 
* Normally does not need to be adjusted but if you are seeing `Java.lang.OutOfMemoryError: PermGen space ` events in Intellij often you can increase this value. Recommend **512m** and increase from their until the errors disappear.

**-server** - [In a nutshell it tells the JVM to spend more time on the optimization of the fragments of codes that are most often used (hotspots). It  leads to better performance at the price of a higher overhead at startup.](https://victorpillac.com/2011/09/11/notes-on-the-java-server-flag/)

**-XX:ReservedCodeCacheSize=[memoryValue]** - [Used to store the native code generated for compiled methods.](https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-4-heap-tuning/).
* Jetbrains recommends [**240m**](https://intellij-support.jetbrains.com/hc/en-us/articles/206544869-Configuring-JVM-options-and-platform-properties) for this value.

**-XX:-OmitStackTraceInFastThrow** - This is a flag recommended by Jetbrain. A description of what this flag does can be found [here](http://www.oracle.com/technetwork/java/javase/relnotes-139183.html). 
* Essentially Intellij may check for certain built-in exceptions thrown in the VM -- or they may be thrown by misbehaving plugins -- and producing stack traces for an exception requires a lot of overhead. Using this option prevents stack traces from being generated for these exceptions and may therefore reduce jitteriness caused by resources being used for this purpose when the exceptions are insignificant.
* There is [insignificant](https://groups.google.com/a/jclarity.com/d/msg/friends/4JOO6M29Jr0/IV5682yWh0QJ) overhead associated with using this option so it is recommended to use it unless you want to debug Intellij itself or a misbehaving plugin.

**-XX:+HeapDumpOnOutOfMemoryError** - Will cause a dump of the heap when an `OutOfMemoryError` error occurs (as explained [here](http://www.oracle.com/technetwork/java/javase/clopts-139448.html#gbzrr)). 
* There is no overhead for using this option so it may be used by default however it is not necessary unless you are experiencing `OutOfMemoryError` errors often.

**-XX:MaxJavaStackTraceDepth=[integer]** - [This specifies how many entries a stack trace for a thrown error or exception can have](https://stackoverflow.com/a/19331083) before a `StackOverflowError` is thrown.
* The JVM has a default of 1024 entries before throwing `StackOverflowError`.
* If you are dealing with huge stack traces you can use this option to increase the allowed entriers before overflow.

**-ea** - This option enables assertions.
* If you are debugging Intellij or a plugin for it you need to use this option in order to have any assertions throw `AssertionError`. There is no performance overhead with this option if you are not using assertions (or not coding in Java).

**-Dsun.io.useCanonCaches=[boolean]** - Specifies whether canonical file paths are cached.
* [By default java caches filenames for 30 seconds](https://stackoverflow.com/a/7479642).
* Jetbrians recommend setting the value of this option to [**false**](https://intellij-support.jetbrains.com/hc/en-us/articles/206544869-Configuring-JVM-options-and-platform-properties). I'm guessing this is because IntelliJ frequently deals with file paths/names and frequent changes can cause a performance hit when these properties change and the filenames are still cached.

**-XX:+AggressiveOpts** - [Turns on some peformance optimizations](https://dzone.com/articles/turbo-charging-eclipse#AO) that are expected to be on by default in upcoming releases. 
* These are minor changes and will not affect the rest of your configuration. Not a necessary flag but may see some performance improvements. 
* Keep in mind the changes this flag turns on change for each java release so make sure to re-evaluate when/if Intellij changes the version of Java packaged with Intellij!

**-Djava.net.preferIPv4Stack=[boolean]** - Specifies whether Intellij should prefer using IPv4 connections.
* If you are experiencing connectivity issues when running Intellij under Java 1.7 Jetbrains [recommends](https://intellij-support.jetbrains.com/hc/en-us/articles/207241215-Network-connectivity-issues-when-running-under-Java-1-7) setting this flag to **true**.
* Alternatively if your environment only accepts IPv6 connections use the flag `-Djava.net.preferIPv6Addresses=true`

# Contributing

Please feel free to make a PR to add/modify any options. I'm still learning about this stuff myself
