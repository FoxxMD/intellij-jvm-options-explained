# Intellij Jvm Options Explained

Writing this as a guide for myself and other Intellij users who want to tune JVM options but have no idea what any of them do.

I will attempt to explain each, in context, and also document sources on where i found explainations (if any).

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
-Xmx[memoryValue, at least 1/4 total physical memory. Recommend 1/2 to 3/4.]
-Xms[memoryValue, at least 1/2 Xmx. Can be = to Xmx]
-XX:+UseConcMarkSweepGC
-XX:+UseParNewGC
-XX:ReservedCodeCacheSize=[between 128m and 240m, depending on how much physical memory you have available]
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-XX:MaxJavaStackTraceDepth=-1
```

# Heap on the VM

Intellij is run on the Java Virtual Machine (VM). The **[heap](https://javarevisited.blogspot.com/2011/05/java-heap-space-memory-size-jvm.html)** is the memory (in RAM) that is allocated to the java virtual machine. **So the heap size is total amount of memory that Intellij can use to operate.**

**-Xmx[memoryVal]** - [Specifies the maximum memory allocation of the heap for the VM.](https://stackoverflow.com/a/14763095) This is the **maximum** amount of memory Intellij can use.
* If you are frequently recieving `OutOfMemoryError` errors from Intellij you should increase this value.
* Increasing this value should also speed up processes like inspection time for large files as Intellij will have to do less swapping of memory to stay inside the limit (?)

**-Xms[memoryVal]** - [Specifies the initial memory allocation of the heap for the VM.](https://stackoverflow.com/a/14763095) This will be the amount of memory Intellij starts with.
* Increasing this value should make Intellij start up faster as it will have a larger pool initially and so will not have to wait to allocate more memory as it spins up.

# Garbage Collection

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

## Other GC Flags

I recommend using these two flags together:

* **-XX:+UseConcMarkSweepGC** 
* **-XX:+UseParNewGC**

These control what algorithm is used for garbage collection. This particular combination [uses mutliple threads to attempt to do GC in the background as to avoid application stopping](http://www.fasterj.com/articles/oraclecollectors1.shtml). If you experience Intellij becoming jerky/unresponsive during heavy usage this may alleviate those problem. There is no real reason not to use them.

**-XX:ParallelGCThreads=[value]** - [Specifies the number of GC thread to use for parallel GC](https://blog.codecentric.de/en/2013/01/useful-jvm-flags-part-6-throughput-collector/) (ParNewGC)
* If not explicitly set this flag, the JVM will use a default value which is computed based on the number of available (virtual) processors. This is what you normally want to do (not set it).
* Useful if you have more than one JVM running and you want to reduce resource use.

**-XX:SurvivorRatio=[ratio]** - Specifies the size of survivor generations inside the young generation.
* A survivor generation holds all of the objects still "live" after a GC on the young generation but not yet passed to the old generation.
* Normally not a huge factor in performance.
* Most values I see are between 8 and 6.

# Everything Else

**-Xss=[memoryValue]** - [Specifies the memory allocated to a new stack](http://www.onkarjoshi.com/blog/209/using-xss-to-adjust-java-default-thread-stack-size-to-save-memory-and-prevent-stackoverflowerror/)
* Every thread created in the VM gets its own stack space. This is space used to store local variables and references.
* Generally does not need to modified. However a smaller stack size will use less memory.

**-XX:PermSize=[memoryValue]** and **-XX:MaxPermSize=[memoryValue]**
* These refer to the **perm size** which is the memory used by the VM for data **outside of your application, just for the VM**. 
* Normally does not need to be adjusted but if you are seeing `Java.lang.OutOfMemoryError: PermGen space ` events in Intellij often you can increase this value. Recommend **512m** and increase from their until the errors disappear.

**-server** - [In a nutshell it tells the JVM to spend more time on the optimization of the fragments of codes that are most often used (hotspots). It  leads to better performance at the price of a higher overhead at startup.](https://victorpillac.com/2011/09/11/notes-on-the-java-server-flag/)

**-XX:ReservedCodeCacheSize=[memoryValue]** - [Used to store the native code generated for compiled methods.](https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-4-heap-tuning/).
* Jetbrains recommends [**240m**](https://intellij-support.jetbrains.com/hc/en-us/articles/206544869-Configuring-JVM-options-and-platform-properties) for this value.

# Contributing

Please feel free to make a PR to add/modify any options. I'm still learning about this stuff myself
