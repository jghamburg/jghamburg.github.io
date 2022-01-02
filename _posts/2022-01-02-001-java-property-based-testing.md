# Property Based Testing - An Evaluation  

In the past I did a lot of unit and integration testing in Java. Most of the time I found it easy to reason about the
imperative process of services within the test structure. What I was missing was the possibility 
to reason about rules or properties that my software should follow to conform to the business requirements.  
Something in the line of questions like:  

* Does the order of processing of an order within my business process make any difference (in mathematics: commutativity)
* or is it allowed to process different subsets within a process and later on collect the results (in mathematics: associativity)

During reading a lot about functional programming lately I stumbled over the term "Property Based Testing" aka PBT.

The basic idea:

1. Specify rules or properties that the software under test should follow.
2. And verify this behavior with generated test data like in the 100 or 1.000 instead of at most ten examples like I used to do.

As I first asked in my company about experiences on PBT I found that it is still not widely known in the Java community.  
And on the other hand I heard things like:

* "I already use specification by example. So why jet another framefork to learn and use?"
* "What is the benefit of this? - I already have a very high test coverage."

So I got curious about this rather old idea. The first steps of PBT were taken at the beginning of 
this millennium and were presented by a paper of Coen Claessen and John Hughes using Haskell. In the
following paragraphs I will motivate my line of thoughts - why I think it is worth it to invest into
the ideas of PBT.

## My Framework of Choice to Explore PBT

I selected the framework [jqwik][jqwik The Framework] maintained by Johannes Link.  
In contrast to the older and more widely used framework [JUnit-quickcheck][JUnit QuickCheck] which
is based on JUnit 4 jqwik is integrated into the JUnit5 platform concept and is implemented as test 
engine. In my opinion this is a more future prove way to proceed.

## Examples

In a microservice-driven world there is a lot of communication going on between the different
systems and components. And perhaps you use events to trigger actions. And these events are
represented in format JSON as they were sent back and forth between components.

So my goal is to setup an event wrapper that can be used to represent different business events but
has a common wrapper for all of them.

Here is my first take on the master event class. I will use Lombok @Value to make the code more
concise.

```java
@Value
@With
public class EventWrapper<T> {
  String eventId;
  String eventType;
  // 2020-09-22T11:47:22.634+0000
  OffsetDateTime eventTime;
  T data;
}
```

and a simple object providing deeper insight within my business domain.

```java
@Value
public class BusinessData {
  String firstValue;
  String secondValue;
}
```

With this in place I want to verify that `EventWraper<BusinessData>` will be correctly encoded to
and decoded from its JSON representation.  
The first could look like this:

```java
class EventWrapperTest {
  private EventWrapper<BusinessData> eventWrapper;

  @BeforeEach
  void setUp() {
    mapper = MyObjectMapperFactory.get();
    // "yyyy-MM-dd'T'HH:mm:ss[.SSS[SSS]]XXX"
    eventTime = OffsetDateTime.parse("2021-11-21T10:15:01.123Z");
    eventWrapper = new EventWrapper<BusinessData>(EVENT_ID, EVENT_TYPE, eventTime, null);
    businessEvent = new BusinessData("firstValue", "secondValue");
  }

  @Test
  void verifyObjectToJsonAndBackWithoutData() throws JsonProcessingException {
    // given: an event
    // when transforming to json
    String value = mapper.writeValueAsString(eventWrapper);
    // then a valid json string is provided
    assertThat(value).isEqualTo("""
        {"event-id":"12345-67890","event-type":"context.TYPE","event-time":"2021-11-21T10:15:01.123Z","data":null}""");
    // and back again
    final EventWrapper baseObject = mapper.readValue(value, EventWrapper.class);
    // then: start and end should be equal
    assertThat(eventWrapper).isEqualTo(baseObject);
  }
}
```

=> all green.  
This was pretty straight forward. And the extra mile with PBT does not seam to give more insights.  
Let's proceed and see.    
Now lets start to look into PBT with a basic setup.

```java
class EventWrapperPbTest {

  void anyEventDataToJsonAndBackIsEqual(
      @ForAll("validEventWrapper") EventWrapper wrapper
  ) throws JsonProcessingException {
    // when: transforming to JSON and back to object
    final EventWrapper<BusinessData> eventWrapperAgain = 
            objectMapper.readValue(objectMapper.writeValueAsString(wrapper)
                    , EventWrapper.class);
    // then: start and end values are equal
    assertThat(eventWrapperAgain).isEqualTo(wrapper);
  }

  @Provide
  Arbitrary<EventWrapper<BusinessData>> validEventWrapper() {
    Arbitrary<String> eventId = Arbitraries.strings();
    Arbitrary<String> eventType = Arbitraries.strings()
        .alpha();
    final Arbitrary<OffsetDateTime> eventTime = DateTimes
        .offsetDateTimes()
        .between(LocalDateTime.of(2021, 1, 01, 01, 01, 01, 001),
            LocalDateTime.of(2022, 01, 01, 01, 01, 01, 01));
    return Combinators.combine(eventId, eventType, eventTime, validBusinessData())
        .as(EventWrapper::new);
  }

  @Provide
  static Arbitrary<BusinessData> validBusinessData() {
    Arbitrary<String> firstValue = Arbitraries.strings()
        .withCharRange('a', 'z');
    Arbitrary<String> secondValue = Arbitraries.strings()
        .withCharRange('0', '9');
    return Combinators.combine(firstValue, secondValue).as(BusinessData::new);
  }
}
```

What do I get for slightly more code?  
Basically first of all a red test.

looks reasonable to me. 

```log
timestamp = 2021-12-16T19:39:55.322492, EventWrapperPbTest:events transformed to json format and back will be equal. = 
  org.opentest4j.AssertionFailedError:
    expected: EventWrapper(eventId=, eventType=, eventTime=2021-01-01T01:01:01.000000001Z, data=BusinessData(firstValue=, secondValue=))
     but was: EventWrapper(eventId=, eventType=, eventTime=2021-01-01T01:01:01.000000001Z, data={first-value=, second-value=})

                              |-------------------jqwik-------------------
tries = 1                     | # of calls to property
checks = 1                    | # of not rejected calls
generation = RANDOMIZED       | parameters are randomly generated
after-failure = PREVIOUS_SEED | use the previous seed
when-fixed-seed = ALLOW       | fixing the random seed is allowed
edge-cases#mode = MIXIN       | edge cases are mixed in
edge-cases#total = 972        | # of all combined edge cases
edge-cases#tried = 0          | # of edge cases tried in current run
seed = 8855018484525384750    | random seed to reproduce generated values

Shrunk Sample (7 steps)
-----------------------
  wrapper:
    EventWrapper(eventId=, eventType=, eventTime=2021-01-01T01:01:01.000000001Z, data=BusinessData(firstValue=, secondValue=))

Original Sample
---------------
  wrapper:
    EventWrapper(eventId=꼑櫽⑚塠䙵䗝祗, eventType=⼒퐴壗艕ఌ䙝邭ﺷ캒呣悑㬔긆ൗఉ뒾❰庫䤚ᑭ䘗ꚻ揞, eventTime=2021-01-30T16:42:48.323253210+07:45, data=BusinessData(firstValue=, secondValue= ))
```

A few observations on the output:

* There seems to be a problem with the provided type BusinessEvent.
* It found counter example on the first attempt - nice
* The counter example looks quite nicely reduced with interesting values I never planed to use in my production code
* What is this about Shrunken example (7 steps) and Original example?

### The Shrinking business  

Like in a lot of other disciplines PBT shines when finding counter examples that reject the
assertion of a property. PBT introduces the concept of shrinking trying to find the simplest example
to reproduce the error. And this can ease up debugging and fixing the error a lot.    

After finding a counter example to falsify the @Property jqwik implements a technic called shrinking
is used to find the "smallest" possible counter example. - There are a lot of details behind this approach and  
the meaning of "smallest" is always in context of the provided testdata domain.  

The one thing to remember the shrunken counter examples are much nicer to handle within example based
test that will be used to fix the identified problems.

## What have I learned so far?  

Why on earth should I tolerate empty, blank or null Strings for Ids and eventTypes?  
How do I handle input values to my system in times of log4shell and other exploits?  

Here's the improved version:  

```java
@Value
@With
public class EventWrapper2<T> {
  // ...
  @JsonCreator
  public EventWrapper2(@JsonProperty(EVENT_ID_NAME) String eventId,
      @JsonProperty(EVENT_TYPE_NAME) String eventType,
      @JsonProperty(EVENT_TIME_NAME) OffsetDateTime eventTime,
      @JsonProperty(DATA_NAME) T data) {
    notBlank(eventId, EVENT_ID_NAME);
    this.eventId = eventId;
    notBlank(eventType, EVENT_TYPE_NAME);
    this.eventType = eventType;
    notNull(eventTime, EVENT_TIME_NAME);
    this.eventTime = eventTime;
    notNull(data, DATA_NAME);
    this.data = data;
  }
  public String toJson() throws JsonProcessingException {
    return ObjectMapperFactory.get().writeValueAsString(this);
  }

  public static EventWrapper2 fromJson(String jsonString, String dataType) {
    final String checkedJsonString = notEmpty(notNull(jsonString, "jsonString"), "jsonString");
    inclusiveBetween(2,2048, checkedJsonString.length());
    final String checkedDataType = notEmpty(notNull(dataType, "dataType"), "dataType");
    TypeReference ref;
    if ("businessdata".equalsIgnoreCase(checkedDataType.toLowerCase())) {
      ref = new TypeReference<EventWrapper2<BusinessData>>() {
      };
    } else {
      ref = new TypeReference<EventWrapper2>() {
      };
    }
    EventWrapper2 eventObject;
    try {
      eventObject = (EventWrapper2) ObjectMapperFactory.get().readValue(jsonString, ref);
    } catch (JsonProcessingException ex) {
      throw new RuntimeException("Error parsing json object");
    }
    return eventObject;
  }
```

With these changes in place there is just one thing left. Comparing the OffsetDateTime correctly.  
The error:  

```log
  org.opentest4j.AssertionFailedError:
    expected: EventWrapper2(eventId=    , eventType=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA, eventTime=2021-01-01T01:01:01.000000001+00:15, data=BusinessData(firstValue=AA, secondValue=00))
     but was: EventWrapper2(eventId=    , eventType=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA, eventTime=2021-01-01T00:46:01.000000001Z, data=BusinessData(firstValue=AA, secondValue=00))
```

So basically just refine our understanding of equality. => two OffsetDateTime are equal if they contain 
the same Instance in time.  
For the implementation please look into the code provided on [github jghamburg/pbt-demo][github jghamburg/pbt-demo].

Now we have verified the behavior or property with 1000 generated testdata. And on every new run the 
next randomly generated testdata will be used - until a counter example is found.  
(seed = 4143122016923828875    | random seed to reproduce generated values) This seed will change on 
every new run.

```log
timestamp = 2021-12-16T20:46:12.921615, EventWrapperWithRestrictionsPbTest:events transformed to json format and back will be equal. = 
                              |-------------------jqwik-------------------
tries = 1000                  | # of calls to property
checks = 1000                 | # of not rejected calls
generation = RANDOMIZED       | parameters are randomly generated
after-failure = PREVIOUS_SEED | use the previous seed
when-fixed-seed = ALLOW       | fixing the random seed is allowed
edge-cases#mode = MIXIN       | edge cases are mixed in
edge-cases#total = 192        | # of all combined edge cases
edge-cases#tried = 182        | # of edge cases tried in current run
seed = 4143122016923828875    | random seed to reproduce generated values
```

## Patterns that are designated for PBT

There are some patterns that come in handy if you are not sure if your problem is a nail.  
So I curated three of these patterns just as an appetizer.

### Inverse Functions

There are a lot of functions that have a reverse. E.g. encode/decode JSON. So basically for all
inputs

```text
ForAll x:encode(decode(x))==x  
```

### Fuzzying  

Code should always be written that it will behave nicely. It should not spit into your face only
because unexpected data examples are feed into your services under test. As a few examples you could
check for:

* No exceptions occur, at least no unexpected ones - wrong data is identified on entering the
  service udner test.
* No 5xx return codes in http requests, maybe you even require 2xx all the time
* All return values are valid

### Test Oracle

On the other hand there are a lot of functions and algorithms where you want to optimize our e.g.
old one to improve speed. So you have to verify that

```text
ForAll x:old(x) == new(x)
```

This kind of verification is called test oracle.  
There are a few sources where the alternatives can come from:

* Simple and slow versus complicated but fast
* Self-made versus commercial
* Old (pre-refactoring) versus new (post-refactoring)

## Lessons Learned

### But why yet another testing framework?

Sometimes it is just fun to get a deeper understanding of frameworks you already use. Like JUnit5 plattform
and its engine extensions.

### What did I learn by diving into PBT?

* Think outside the box. Find ways to question your system under test to get a deeper insight of
the functioning of the system.  
* On the other hand exploring new areas might lead to new ways of thinking and progressing in the
realm of problem-solving. You can extend your repertoire/portfolio of tools to use for solving
different kind of problems/challenges.

	If the only tool you have at hand is a hammer every problem is a nail. 
		Unknown - at least to me

Things that came up during the experiments:

* am I restricting my input data correctly?
* will my system crash under some allowed but bizzar input that never would come to my mind. 
  - Ever tried to use chinese input characters?

This approach is definitly not for the faint of heart.  
You have to think hard to find behavior/properties that represent your system on the one hand.  
And it is highly challenging to describe data generators to prove specific properties.   

In my experience PBT will be a good solution if you find structures and algorithms where your gut feeling
starts to moan and the existing tests do not provide sufficient security and understanding of the system.

Myself I was quite astonished how many ideas and insights I generated only by providing some unexpected input data.  
and to be clear I would not have started out to analyse and structure some input data into categories
and provide examples for these.  

At the end

* a lot of fun
* a cool way to generate test data
* and a new tool for my belt

## Resources

For all the examples I use jqwik provided by Johannes Link, who is well known in the java testing
community.  
The sample project with the code and my article you can find on 
[github jghamburg/pbt-demo][github jghamburg/pbt-demo].

__Articles about PBT and usage__:

* [The original paper by Claessen-Hughes 2000](https://www.semanticscholar.org/paper/QuickCheck%3A-a-lightweight-tool-for-random-testing-Claessen-Hughes/75d28729e96691eb85ae2b34e791473a24062ce5)
* [How to Specify it! In Java! | How to Specify It! In Java!](https://johanneslink.net/how-to-specify-it/)
* [Property-based Testing in Java: Introduction - My Not So Private Tech Life](https://blog.johanneslink.net/2018/03/24/property-based-testing-in-java-introduction)
* [Know for Sure with Property-Based Testing](https://blogs.oracle.com/javamagazine/post/know-for-sure-with-property-based-testing)
* [Intro to PBT with F#](https://fsharpforfunandprofit.com/posts/property-based-testing/)
* [Step by Step to Property based Testing - Dave Nicolette 2018](https://www.leadingagile.com/2018/04/step-by-step-toward-property-based-testing/)
    - a good introduction
* [Patterns to Identify Properties by Johannes Link 2018](https://blog.johanneslink.net/2018/07/16/patterns-to-find-properties)
* [Property-based Testing Patterns by Sanj 2016](https://blog.ssanj.net/posts/2016-06-26-property-based-testing-patterns.html)

__The Framework jqwik and Exensions__:  

* [jqwik The Framework][jqwik The Framework]
* [jqwik Users Guide][jqwik Users Guide]
* [jqwik-spring: extension to support testing of Spring-Boot][jqwik-spring: extension to support testing of Spring-Boot]
  -> spring boot support: (example):
* [jqwik-spring-boot-gradle example by Johannes Link][jqwik-spring-boot-gradle example]
  -> how to PBT a rest controller 
* [PBT Workshop Handout by Johannes Link](https://johanneslink.net/downloads/pbt-workshop-english.pdf)

__Interesting short series about testing__:

* [Multiplying the Quality of your Tests - with Parameterized Tests - Sergio Sastre Florez - 2021](https://sergiosastre.hashnode.dev/multiplying-the-quality-of-your-unit-tests-part-1)
  Pros and Cons (kotlin code examples)

[Testing General Rules through Property-Based Tests - A. Rothuis - 2020](https://arothuis.nl/posts/property-based-testing-rock-paper-scissors/)
-> nice appetizer - more academic approach

__Examples based on json date conversion__:  

These articles actually triggered my ideas for this project.

* [How PBT helps me to be a better developer - Andriy Redko - 2020](https://aredko.blogspot.com/2020/02/)
* [Differences Between ZonedDateTime and OffsetDateTime | Baeldung](https://www.baeldung.com/java-zoneddatetime-offsetdatetime)
  Recommendation: Use OffsetDateTime for database use.
* [Why “Always use UTC” is bad advice](https://engineering.q42.nl/why-always-use-utc-is-bad-advice/)
  Basically use ZonedDateTime if you want to calculate events in the future.
* [Jackson , java.time , ISO 8601 , serialize without milliseconds | Newbedev](https://newbedev.com/jackson-java-time-iso-8601-serialize-without-milliseconds)
* [Jackson JSON - Using @JsonSerialize and @JsonDeserialize with Converter for custom conversion](https://www.logicbig.com/tutorials/misc/jackson/json-serialize-deserialize-converter.html)
* [JSON deserialize generic types using Gson and Jackson | JSBlogs](https://blogs.jsbisht.com/blogs/2016/09/07/json-deserialize-generic-types-using-gson-and-jackson)


[github jghamburg/pbt-demo]: https://github.com/jghamburg/pbt-demo
[jqwik The Framework]: https://jqwik.net
[jqwik Users Guide]: https://jqwik.net/docs/current/user-guide.html
[jqwik-spring: extension to support testing of Spring-Boot]: https://github.com/jlink/jqwik-spring
[jqwik-spring-boot-gradle example]: https://github.com/jlink/jqwik-samples/tree/main/jqwik-spring-boot-gradle
[JUnit QuickCheck]: https://github.com/pholser/junit-quickcheck

