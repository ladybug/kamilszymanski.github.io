---
title: Refactoring stringly-typed systems
date: 2018-01-21T16:45:17+01:00
excerpt: "Picking low-hanging fruits on the way to improve domain model"
tags:
  - Java
---

Last year I joined a project that was taken over from another software house that failed to satisfy client demands.
As you can probably tell there were many things that could and should be improved in that "inherited" project and its codebase.
Sadly (but not surprisingly) the domain model was one of such orphaned, long-forgotten areas that screamed for help the most.

We knew we needed to put our hands dirty but how do you improve the domain model in an unfamiliar project where everything is so mixed up, tangled and overgrown with accidental complexity?
You set boundaries (divide and conquer!), apply small improvements in one area, then move to the other while getting to know the landscape and discovering bigger issues that hide behind those scary, obvious things that hurt your eyes from the first sight.
You would be surprised how much you can achieve by making small improvements and picking low-hanging fruits, yet at the same time you would be a fool thinking that they could solve major issues that have grown up there due to the lack of (or not enough) modeling efforts taken right from the dawn of the project.
Nevertheless without those small improvements it would be way harder to tackle most of the major domain model issues.

For me bringing more expressiveness and type-safety into code by introducing simple value objects was always one of the lowest-hanging fruits.
It's a trick that always works, especially when dealing with codebases stinking with primitive obsession code smell and the mentioned system was a stringly-typed one.
It was full of code looking like this:
```java
public void verifyAccountOwnership(String accountId, String customerId) {...}
```
while I bet everyone would prefer it to look more like that:
```java
public void verifyAccountOwnership(AccountId accountId, CustomerId customerId) {...}
```

It's not a rocket science!
I'd say it's a no-brainer and it always surprises me how easy it is to find implementations operating on e.g. vague, contextless BigDecimals instead of Amounts, Quantities or Percentages.

Code that uses domain specific value objects instead of contextless primitives is:
* much more expressive (you don't need to map strings into a customer identifiers in your head nor worry about any of those strings being an empty string)
* easier to grasp (invariants are protected in one place instead of being scattered all around the codebase in ubiquitous if statements)
* less buggy (did I put all of those strings in the right order?)
* easier to develop (explicit definitions are more obvious and invariants are protected right where you would expect it)
* faster to develop (IDE offers much more help and the compiler provides fast feedback cycles)

and those are just a few of the things you get almost for free (you just have to use common sense ^^).

Refactoring towards value objects sounds like a piece of cake (naming things is not taken into account here), you simply extract class here, migrate type there, nothing spectacular.
It usually is that simple, especially when the code you have to deal with lives inside a single code repository and runs in a single process.
This time however it wasn't so trivial.
Not that it was much more complicated, it just required a tiny bit more thinking (and it makes for a nice piece of work to be described ^^).

It was a distributed system that had service boundaries set at wrong places and shared too much code (including model) between services .
The boundaries were set so bad that many crucial operations in the system required numerous interactions (mostly synchronous) with multiple services.
There's a challenge (not so big) in applying mentioned refactoring in a described context in a way that doesn't end up as an exercise of creating unnecessary layers and introducing accidental complexity at service boundaries.
Before jumping to refactoring I had to set some rules, or rather one crucial rule: no changes should be visible from the outside of the service, including backing services.
To put it simple all published contracts stay the same and there are no changes required on backing services side (e.g. no  database schema changes).
Easily said and frankly speaking easily done with a bit of dull work.

Let's take `String accountId` for a ride and demonstrate necessary steps.
We want to turn such code:
```java
public class Account {

    private String accountId;

    // rest omitted for brevity
}
```
into this:
```java
public class Account {

    private AccountId accountId;

    // rest omitted for brevity
}
```
This can be achieved by introducing AccountId value object:
```java
@ToString
@EqualsAndHashCode
public class AccountId {

    private final String accountId;

    private AccountId(String accountId) {
        if (accountId == null || accountId.isEmpty()) {
            throw new IllegalArgumentException("accountId cannot be null nor empty");
        }
        // can account ID be 20 characters long?
        // are special characters allowed?
        // can I put a new line feed in the account ID?
        this.accountId = accountId;
    }

    public static AccountId of(String accountId) {
        return new AccountId(accountId);
    }

    public String asString() {
        return accountId;
    }
}
```
AccountId is just a value object, it has no identity, it doesn't change over time, hence it is immutable.
It performs all validations in a single place and fails fast on incorrect inputs by failing to instantiate AccountId instead of failing later on on an if statement buried down several layers down the call stack.
If it needs to protect any invariants you know where to put them and where to look for them.

So far so good, but what if AccountId needs to be persisted in a database?
Well, you just implement an attribute converter:
```java
public class AccountIdConverter implements AttributeConverter<AccountId, String> {

    @Override
    public String convertToDatabaseColumn(AccountId accountId) {
        return accountId.asString();
    }

    @Override
    public AccountId convertToEntityAttribute(String accountId) {
        return AccountId.of(accountId);
    }
}
```
Then you enable the converter by either `@Converter(autoApply = true)` set directly on the converter implementation or `@Convert(converter = AccountIdConverter.class)` set on the entity field.

Of course not everything spins around databases and luckily amongst many not so good design decision applied in the mentioned project there were also many good ones.
One of such good decisions was to standardize the data format used for out of process communication.
In the mentioned case it was JSON, hence I needed to make JSON payload immune to the performed refactoring.
The easiest way (if you use Jackson) is to sprinkle the implementation with a couple of Jackson annotations:
```java
public class AccountId {

    @JsonCreator
    public static AccountId of(@JsonProperty("accountId") String accountId) {
        return new AccountId(accountId);
    }

    @JsonValue
    public String asString() {
        return accountId;
    }

    // rest omitted for brevity
}
```

I started with the easiest solution.
It wasn't ideal but it was good enough and at that time we had more important issues to deal with.
Having both JSON serialization and database types conversion taken care of after less than 3 hours I have moved first 2 services from stringly-typed identifiers to the value object based ones for the identifiers most commonly used within the system.
It took so long due to 2 reasons.

The first one was obvious: along the way I had to check if null values were not possible (and if they would then state that explicitly).
Without this the whole refactoring would be just a code polishing exercise.

The second one was something I almost missed - do you remember the requirement that the change should not be visible from the outside?
After turning account ID into a value object swagger definitions changed as well, now account ID was no longer a string but an object.
This was also easy to fix, it just required specifying swagger model substitution.
In case of [swagger-maven-plugin](https://github.com/kongchen/swagger-maven-plugin) all you need to do is [feed it with the file containing model substitution mappings](https://github.com/kongchen/swagger-maven-plugin#model-substitution):
```
com.example.AccountId: java.lang.String
```

Was the result of performed refactoring a significant improvement?
Rather not, but you improve a lot by making lots of small improvements.
Nevertheless this wasn't a tiny improvement, it brought a lot of clarity into the code and made further improvements easier.
Was it worth the effort - I would definitely say: yes, it was.
A good indicator of this is that other teams adopted that approach.

Fast-forward a few sprints, having solved some of the more important issues and having started turning inherited, heavily tangled mess into a bit nicer solution based on hexagonal architecture, the time has come to deal with the drawbacks of the taken easiest approach to support JSON serialization.
What we needed to do was decouple AccountId domain object from things not related to the domain.
Namely we had to move out of the domain the part defining how to serialize this value object and remove domain coupling to Jackson.
In order to achieve that we created Jackson Module that handled AccountId serialization:
```java
class AccountIdSerializer extends StdSerializer<AccountId> {

    AccountIdSerializer() {
        super(AccountId.class);
    }

    @Override
    public void serialize(AccountId accountId, JsonGenerator generator, SerializerProvider provider) throws IOException {
        generator.writeString(accountId.asString());
    }
}

class AccountIdDeserializer extends StdDeserializer<AccountId> {

    AccountIdDeserializer() {
        super(AccountId.class);
    }

    @Override
    public AccountId deserialize(JsonParser json, DeserializationContext cxt) throws IOException {
        String accountId = json.readValueAs(String.class);
        return AccountId.of(accountId);
    }
}

class AccountIdSerializationModule extends Module {

    @Override
    public void setupModule(SetupContext setupContext) {
        setupContext.addSerializers(createSerializers());
        setupContext.addDeserializers(createDeserializers());
    }

    private Serializers createSerializers() {
        SimpleSerializers serializers = new SimpleSerializers();
        serializers.addSerializer(new AccountIdSerializer());
        return serializers;
    }

    private Deserializers createDeserializers() {
        SimpleDeserializers deserializers = new SimpleDeserializers();
        deserializers.addDeserializer(AccountId.class, new AccountIdDeserializer());
        return deserializers;
    }

    // rest omitted for brevity
}
```

If you’re using Spring Boot configuring such module requires simply registering it in the application context:
```java
@Configuration
class JacksonConfig {

    @Bean
    Module accountIdSerializationModule() {
        return new AccountIdSerializationModule();
    }
}
```

Implementing custom serializers was also something we needed because along all the improvements we have identified more value objects and some of them were a bit more complex - but that's something for another article.
