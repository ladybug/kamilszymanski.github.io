---
title: Controlling parallelism level of Java parallel streams
date: 2017-10-16T08:21:23+02:00
excerpt: "Busting the most common misconception about Java parallel streams"
tags:
  - java
---

With recent Java 9 release we got many new goodies to play with and improve our solutions once we grasp those new features.
The release of Java 9 is also a good time to revise whether we have grasped Java 8 features.

In this post I'd like to bust the most common misconception about Java parallel streams.
It's often said that you cannot control parallel streams' parallelism level in a programmatic way, that parallel streams always run on shared ForkJoinPool.commonPool() and there's nothing you can do about it.
This is the case if you make your stream parallel by just adding parallel() call to the call chain.
That might be sufficient in some cases, e.g. if you perform only lightweight operations on that stream, however if you need to gain more control over your stream's parallel execution you need to do a bit more than just calling parallel().

Instead of diving in into theory and technicalities let's jump straight to the self-documenting example.

Having a parallel stream being processed on shared ForkJoinPool.commonPool():

```java
Set<FormattedMessage> formatMessages(Set<RawMessage> messages) {
    return messages.stream()
            .parallel()
            .map(MessageFormatter::format)
            .collect(toSet());
}
```

let's move parallel processing to a pool that we can control and don't have to share:

```java
private static final int PARALLELISM_LEVEL = 8;

Set<FormattedMessage> formatMessages(Set<RawMessage> messages) {
    ForkJoinPool forkJoinPool = new ForkJoinPool(PARALLELISM_LEVEL);
    try {
        return forkJoinPool.submit(() -> formatMessagesInParallel(messages))
                .get();
    } catch (InterruptedException | ExecutionException e) {
        // handle exceptions
    } finally {
        forkJoinPool.shutdown();
    }
}

private Set<FormattedMessage> formatMessagesInParallel(Set<RawMessage> messages) {
    return messages.stream()
            .parallel()
            .map(MessageFormatter::format)
            .collect(toSet());
}
```

In this example we're interested only in the parallelism level of the ForkJoinPool though we can also control ThreadFactory and UncaughtExceptionHandler if needed.

Under the hood the ForkJoinPool scheduler will take care of everything, including incorporating work-stealing algorithm to improve parallel processing efficiency.
Having said that it's worth to mention that manual processing using ThreadPoolExecutor might be more efficient in some cases, e.g. if the workload is evenly distributed over worker threads.
