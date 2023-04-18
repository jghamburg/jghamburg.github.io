# Equality still Matters - Improve Understanding and Maintainability

In my experience equality still matters not only in human world but also within the realm of software developemnt. And it is the basic concepts that take us far before we have to switch to more complex methods to find a solution.   

As I was doing some code reviews lately (2023) I saw a lot of unit test code like this:  

* Request an object like a product with 5+ attributes from a business service 
* compare expected against actual output.  

```java
// Given
ProductId productId =
   new ProductId("af508252-65e7-4e68-ba26-fed86d5e6b96");
// When
  Product product = productDataService.getProduct(partnerVariationId);

// Then
  assertThat(product.getProductId())
   .isEqualTo("af508252-65e7-4e68-ba26-fed86d5e6b96");
  assertThat(product.getArticleNumber()).isEqualTo(16441402);
  assertThat(product.getWarehouseId()).isEqualTo(1);
  assertThat(product.getWarehouseLocation()).isEqualTo("MYWAREHOUSE_LOCATION");
  assertThat(product.getPackages().get(0).getPackageNumber()).isEqualTo(1);
``` 

So far so good - just a standard scenario. Now let's look at the code.  
So what is wrong with this code? - Actually nothing - it is working.    

More prcisely asked: Does this conway the idea and the validation I want to process in a concise way? Is it easy to understand?  

The questions that came to my mind:  

* Do we have a common understanding of equality concerning the product?
* Are there different notions of equality that imposes the finegrained writeup?
* What are different notions of equality that I know of?

So let's dig deeper here.  
Based on DDD (Domain Driven Design) there are two basic notions of identity and associated equality.  

* Two DDD entities are equal if and only if the defined identifier for that domain object are equal.  
* Two DDD value objects are equal if and only if all (nonvolatile) attributes of the instances are equal.  

The major difference between both types of object is that DDD entities have a unique id that is valid for the whole lifecycle of it. And besides it can contain a lot of different attributes that change over time of the lifespan.  
On the one hand Product is a DDD entity. On the other hand I want to verify the exact attribute match during 





A DDD value object does not have the notion of a lifespan. It is created once adn most likely will be associated to an entity object, e.g. as attribute. This could be a Price that contains an ammount and currency attribute.  

Beside the business driven ideas from DDD there is also the notion of equality that is provided by the programming language used. This provides a posibility to implement my idea of business equality.   
For my further observations the language of choise is Java.  
If you do not change standard implementation of equality it means that two objects are equals if the pointer to the in memory instance is equal. So:  

objA == objB &wedgeq; objA.equals(objB)

Only rarely this kind of equality is the one you seek. It holds true for Java String implementation because every representation of a String is handled as static object (same content -> same object).  

In contrast it does not work for our DDD entity example of Product. Even if every attribute is set exactly the same values two different instances will never be equal.  

With that introduction out of the way let's sketch out some ideas how to apply our knowledge.  

The rule of thump that was true in 1996 as I started exploring Java is still valid:  

    Always override equals() and hashCode()

Because there are different interpretations now it is the time to decide what we want to achieve.  
If we are looking for DDD value objects an easy way to provide implementations for equals/hashCode is to use a framework like Lombok. A perhaps more modern approach would be to use Java feature Record (Java 17 lts and up) which directly implements attribute equality and also provides the feature of immutability.  

If you wnat to implement DDD entitiy equality by id you have to override the equals() and hashCode() methods accordingly.  

After all the introduction let's apply this to our initial sample:  

```java
add an improved sample that is more concise and incorporates all the above mentioned ideas.
```

What about the cases were I want to do assertions on a subset of attributes?  
E.g. categorize like on lifecycle state of business object. - Are all SalesOrders already paied after processing?  

For this kind of question we can introduce filter methods. To start always add to the class where this knowledge extension takes place. Only if it got unaccessible/uncomprehensible separate this extension to an external class. - Start small and easy. Only split up for clarity and fast understanding of intentions.  

Before I close this article just one glimps on another but related topic.  
If the validation steps you want to do during assertion grow out of hand and it cannot be solved by introducing a specific notion of equality to the object think about writing your own Asserter class. The framework AssertJ provides easy extension points for this.  

Why did I jump into this topic in the first place?  
It is not always a shiny new idea or approach to overcome your day to day challanges in software development. A lot of times the basic concepts that seem too simple and easy to grasp that bring you a long way.  

The investment into these elementary concepts will provide you with ideas to find more concise and easy to maintain solutions within your code base.  
