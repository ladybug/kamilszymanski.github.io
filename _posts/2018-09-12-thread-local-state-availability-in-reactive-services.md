---
title: Thread-local state availability in reactive services
date: 2018-09-12T08:33:14+02:00
excerpt: "All non-trivial abstractions, to some degree, are leaky"
tags:
  - reactive
---

Any architecture decision involves a trade-off.
It's no different if you decide to go reactive, e.g. on one side using [Reactive Streams](http://www.reactive-streams.org/) implementations gives better resources utilization almost out of the box but on the other hand makes debugging harder.
Introducing reactive libraries also has huge impact on your domain, your domain will no longer speak only in terms of `Payment`, `Order` or `Customer`, the reactive lingo will crack in introducing `Flux<Payment>`, `Flux<Order>`, `Mono<Customer>` (or `Observable<Payment>`, `Flowable<Order>`, `Single<Customer>` or whatever Reactive Streams publishers your library of choice provides).
Such trade-offs quickly become evident but as you can probably guess not all of them will be so obvious - [The Law of Leaky Abstractions](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/) guarantees that.

Reactive libraries make it trivial to change threading context.
You can easily subscribe on one scheduler, then execute part of the operator chain on the other and finally hop onto a completely different one.
Such jumping from one thread to another works as long as no thread-local state is involved, you know - the one you don't usually deal with on a day to day basis although it powers crucial parts of your services (e.g. security, transactions, multitenancy).
Changing threading context when a well hidden part of your tech stack depends on thread-local state leads to tricky to nail down bugs.

Let me demonstrate the problem on a simple example:

```java
private static final Logger LOG = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());
private static final String SESSION_ID = "session-id";

@GetMapping("documents/{id}")
Mono<String> getDocument(@PathVariable("id") String documentId) {
    MDC.put(SESSION_ID, UUID.randomUUID().toString());
    LOG.info("Requested document[id={}]", documentId);
    return Mono.just("Lorem ipsum")
            .map(doc -> {
                LOG.debug("Sanitizing document[id={}]", documentId);
                return doc.trim();
            });
}
```

With `MDC.put(SESSION_ID, UUID.randomUUID().toString())` we're putting `session-id` into [Mapped Diagnostic Context](https://logback.qos.ch/manual/mdc.html) of underlying logging library so that we could log it later on.

Let's configure logging pattern in a way that would automatically log `session-id` for us:

```
logging.pattern.console=[%-28thread] [%-36mdc{session-id}] - %-5level - %msg%n
```

When we hit the exposed service with a request (`curl localhost:8080/documents/42`) we will see `session-id` appearing in the log entries:

```
[reactor-http-server-epoll-10] [00c4b05f-a6ee-4a7d-9f92-d9d53dbbb9d0] - INFO  - Requested document[id=42]
[reactor-http-server-epoll-10] [00c4b05f-a6ee-4a7d-9f92-d9d53dbbb9d0] - DEBUG - Sanitizing document[id=42]
```

The situation changes if we switch the execution context (e.g. by subscribing on a different scheduler) after `session-id` is put into MDC:

```java
@GetMapping("documents/{id}")
Mono<String> getDocument(@PathVariable("id") String documentId) {
    MDC.put(SESSION_ID, UUID.randomUUID().toString());
    LOG.info("Requested document[id={}]", documentId);
    return Mono.just("Lorem ipsum")
            .map(doc -> {
                LOG.debug("Sanitizing document[id={}]", documentId);
                return doc.trim();
            })
            .subscribeOn(Schedulers.elastic()); // don't use schedulers with unbounded thread pool in production
}
```

After execution context changes we will notice `session-id` missing from log entries logged by operators scheduled by that scheduler:

```
[reactor-http-server-epoll-10] [c2ceae03-593e-4fb3-bbfa-bc4970322e44] - INFO  - Requested document[id=42]
[elastic-2                   ] [                                    ] - DEBUG - Sanitizing document[id=42]
```

As you can probably guess there's some `ThreadLocal` hidden deep [inside the logging library](https://github.com/qos-ch/logback/blob/master/logback-classic/src/main/java/ch/qos/logback/classic/util/LogbackMDCAdapter.java) we're using.

Some Reactive Streams implementations provide mechanisms that allow to make contextual data available to operators (e.g. [Project Reactor](https://projectreactor.io/) provides [subscriber context](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#subscriberContext-reactor.util.context.Context-)):

```java
@GetMapping("documents/{id}")
Mono<String> getDocument4(@PathVariable("id") String documentId) {
    String sessionId = UUID.randomUUID().toString();
    MDC.put(SESSION_ID, sessionId);
    LOG.info("Requested document[id={}]", documentId);
    return Mono.just("Lorem ipsum")
            .zipWith(Mono.subscriberContext())
            .map(docAndCtxTuple -> {
                try(MDC.MDCCloseable mdc = MDC.putCloseable(SESSION_ID, docAndCtxTuple.getT2().get(SESSION_ID))) {
                    LOG.debug("Sanitizing document[id={}]", documentId);
                    return docAndCtxTuple.getT1().trim();
                }})
            .subscriberContext(Context.of(SESSION_ID, sessionId))
            .subscribeOn(Schedulers.elastic()); // don't use schedulers with unbounded thread pool in production
}
```

Of course making data available is just part of the story.
Once we make `session-id` available (`subscriberContext(Context.of(SESSION_ID, sessionId))`) we not only have to retrieve it but also attach it back to the threading context as well as remember to clean up after ourselves since schedulers are free to reuse threads.

Presented implementation brings back `session-id`:

```
[reactor-http-server-epoll-10] [24351524-f105-4746-8e06-b165036d02e6] - INFO  - Requested document[id=42]
[elastic-2                   ] [24351524-f105-4746-8e06-b165036d02e6] - DEBUG - Sanitizing document[id=42]
```

Nonetheless the code that makes it work is too complex and too invasive to welcome it with open arms in most codebases, especially if it ends up scattered across the codebase.

I’d love to finish this blog post by providing a simple solution to that problem but I haven’t yet stumbled upon such (a.k.a. for now we need to live with such, more complex and invasive, solutions while also trying to move this complexity from business-focused software parts down to it’s infrastructural parts and if possible directly to the libraries themselves).
