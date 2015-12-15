---
layout: post
title: Idiomatic Optionals
date: 2015-12-15T17:43:00+01:00
excerpt: "Using java.util.Optional in an idiomatic way"
tags: [java]
comments: true
share: true
---

Java 8 was released over 20 months ago and Java 7 already reached end of life, yet it's not that hard to stumble upon new code that uses classes from Java 8 but looks like it was coded in a Java 7.
The anti-pattern I'm talking about is most common when conditional logic needs to be applied to [java.util.Optional](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html) and it looks like this:

{% highlight java %}
public Optional<String> getClientNumber(String personalId) {
    Optional<Client> client = clientRepository.getByPersonalId(personalId);
    return client.isPresent() ? Optional.of(client.get().getNumber()) : Optional.empty();
}
{% endhighlight %}

while it should look like that:

{% highlight java %}
public Optional<String> getClientNumber(String personalId) {
    return clientRepository.getByPersonalId(personalId)
            .map(Client::getNumber);
}
{% endhighlight %}

As a rule of thumb everytime you use an `if-else` block or ternary operator to consume Optional's value or produce an alternative one take a moment and check if it can be written in an idiomatic way.
Most of the time it can and should be written in such way because it makes code more readable.

The only case I came across that requires ternary operator is when you need to convert an `Optional<T>` to a `Stream<T>`, fortunately that will be [taken care of in Java 9](https://bugs.openjdk.java.net/browse/JDK-8050820).
