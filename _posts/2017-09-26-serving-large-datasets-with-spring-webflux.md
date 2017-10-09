---
title: Serving large datasets with Spring WebFlux
date: 2017-09-26T22:51:32+02:00
excerpt: "Using old solutions dressed in new API"
tags:
  - reactive
---

If you're serving large datasets from your web service you might like one of the upcoming Spring Framework 5.0 features.
But before we get to this feature let's see how a naive implementation of such service might look like:

```java
@GetMapping(path = "items", produces = "application/json")
List<Item> allItems() {
    return itemRepository.findAll();
}
```

The naive part of this implementation is that we try to return the whole dataset at once and this can easily make such service unresponsive.

If this dataset is larger than what we can fit into memory or if there are many clients asking for this large dataset at the same time we'll end up seeing OutOfMemoryError.
And even if we have heap large enough to handle such cases there's a high chance that our client's won't be that lucky and would fail with OutOfMemoryError reading the response.
Moreover there is an often overlooked latency issue hidden there - with such implementation client can start processing that dataset only after it's fully loaded, serialized and delivered to him by the server:

![alt text](../assets/images/posts/serving-large-datasets-with-spring-webflux/single-json-document.gif "returning whole dataset at once")

Such problems are usually mitigated to some extent by introducing paging.
However if client is interested in the whole dataset he now has to issue multiple requests which is not the most convenient solution (not to mention that if no consistency control mechanisms are in place he might get duplicates and/or miss some data).

So can we do better than that?
As microservices were the answer to all the questions in the last few years now reactive is the golden hammer.
Luckily our problem seems to look more like a nail than a screw, so let's stab it with a reactive hammer:

```java
@GetMapping(path = "items", produces = "application/json")
Flux<Item> allItems() {
    return itemRepository.findAll();
}
```

The important part here is that the data source should support backpressure or allow to load data in chunks and at speed that pose no issues for the receiver.
Assuming that our repository supports backpressure and given that Flux is capable of supporting it the problem should be solved since WebFlux (reactive HTTP component, part of Spring Framework 5.0) handles Flux payloads quite well.

Unfortunately this implementation still has all the mentioned problems.
It's because we still need all data in place just to serialize it before we start sending the response.

The fix is rather obvious - we need reactive JSON serializer.
As I told nowadays reactive is an answer to all problems.
Just kidding, we don't need any reactive serializers (I don't even know what that means).
Good old Jackson, Gson, or any other JSON serializer you prefer should be sufficient.
What we need to change is not how we serialize but what we serialize.
Let me show you the implementation before I explain what I mean by saying "change what we serialize".

```java
@GetMapping(path = "items", produces = "application/stream+json")
Flux<Item> allItems() {
    return itemRepository.findAll();
}
```

As you can see we're still returning a Flux of Items however now the response has different mediatype.
Now instead of returning one large serialized JSON document containing all Items we return a stream of individually serialized Items (a stream of JSON documents):

![alt text](../assets/images/posts/serving-large-datasets-with-spring-webflux/stream-of-json-documents.gif "returning stream of JSON documents")

Under the hood whenever an Item is emitted from the repository it gets serialized, the response buffer is flushed (meaning that bytes start flowing to the client) but the connection is kept open until all documents are emitted, serialized and sent.
This doesn't sound like some magical or new solution, e.g. you might remember tricks like Comet that date back several years in the past.

Of course handling such responses requires clients being able to decode them but that's not a rocket science and there already exist implementations that can do that (including Spring Framework 5.0 WebClient).

In the last iteration of this service we got rid of OutOfMemoryError issue on the server side as well as significantly reduced the time needed for the client to start processing the first Item in the dataset returned by our service.
Another issue we had was OutOfMemoryError on the client side - here all depends on the client being able to process incoming Items as fast as the server is sending them or being able to buffer unprocessed part of the sent dataset.
It's not the perfect solution to this problem but having in mind that we're communicating over request-response protocol it might be an acceptable one, especially that we have significantly reduced the probability of OutOfMemoryError on the client side.
