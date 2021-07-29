# Using Lambda Expressions in Java  

One lamb will not make a flock and multiple lambda in Java will not make Java a functional programming language.  

In martial arts it's not about opposing forces but complementary engergy - the concept of yin / yang. So what if we take functional concepts to complement ideas from object orientend programming? Lets join forces to find a more concise solution.

Lets start with an example taken from Vankat Subramaniem: Functional Programming in Java.

## Separation of Concerns - method level  

We are in the finance market and need to manage assets.  
So we start with a rather simple definition.  

```java
public​ ​class​ Asset {
  ​public​ ​enum​ AssetType { BOND, STOCK };
  ​private​ ​final​ AssetType type;
  ​private​ ​final​ ​int​ value;

  ​public​ Asset(​final​ AssetType assetType, ​final​ ​int​ assetValue) {​ 
    type = assetType;
    value = assetValue;
  }
  ​public​ AssetType getType() { ​return​ type; }	
  ​public​ ​int​ getValue() { ​return​ value; }
}
```

ith this standard Java beans class we represent bonds and stock objects. 
The first requirement will be to sum up the total value of our assets. 

We implement this in a AssetUtils.class like this.

```java
public​ ​static​ ​int​ totalAssetValues(​final​ ​List​<Asset> assets) {​ 
  ​return​ assets.stream()
    .mapToInt(Asset::getValue)
    .sum();
}
```

We don't do imperative loops here, we use streams, map functions and the Collectors helper class from Java8 stream api.  
It is concise !  
- take the list of assets, extract the values from the assets and sum everything up.

For checking the correctness you can use a list of assets like this one

```java
final List>Asset> assets = List.of(
  Asset.AssetType.BOND, 1000),
  Asset.AssetType.BOND, 2000),
  Asset.AssetType.STOCK, 3000),
  Asset.AssetType.STOCK, 4000)
);
```

What's next? The boss will ask us to sum up only the bonds.  
- No problem. We copy the totalAssetValues and adapt.

```java
public​ ​static​ ​int​ totalValues(​final​ ​List​<Asset> assets) {​ 
  ​return​ assets.stream()
    .mapToInt(asset ->
              asset.getType() == AssetType.BOND ? asset.getValue() : 0
    )
    .sum();
}
```

Simple isn't it? - But remember this is are only two methods. In real life projects there are more to come and maintain.  
So next requirement: we need the value of our stocks as well. 
We already know how to do this.

```java
public​ ​static​ ​int​ totalValues(​final​ ​List​<Asset> assets) {​ 
  ​return​ assets.stream()
    .mapToInt(asset ->
              asset.getType() == AssetType.STOCK ? asset.getValue() : 0
    )
    .sum();
}
```

At this point there are three methods almost looking the same but with subtle differences. What are the key concerns?  
It's about iterating over a list, filter content, sum up values.  
We found three concerns in this functions where only the filtering differs.  

This looks like  
  Replacing Conditional Logic with Strategy  
  Conditialal logic in a method controls which of serveral variants of a calculation are executed.
    [RefacToPat], p. 129f

For the refactoring we use a functional interface as parameter.  

```java
public static int totalAssetValues(final List<Asset> assets
  , final Predicate<Asset> assetSelector) {
  return assets.stream()
    .filter(assetSelector)
    .mapTpInt(Asset::getValue)
    .sum();
}
```

On a scientific level we introduced a higher order function and isolated the concern of selection into a functional interface.  
In practice we can use a lambda expression as functional interface.  

```java
  System.out.println("Total of all bonds: " + AssetUtils.totalAssetValues(assets, asset -> 
    asset.getType == AssetType.BOND ? true: false));
// Total of bonds: 3000
  System.out.println("Total of all stock: " + AssetUtils.totalAssetValues(assets, asset -> 
    asset.getType == AssetType.STOCK ? true: false));
// Total of stocks: 7000
  System.out.println("Total of all assets: " + AssetUtils.totalAssetValues(assets, asset -> 
    true));
// Total of all: 10000
```

And we reduced the complexity of our code from three almost identical to one method which takes the filtering concern as parameter. For simplicity and not to introduce our own intercase or inner class we used Java8 functional interface Predicate.  

With higher order functions we have used one of key concepts of functional programming paradigm.  
Using functions that take functions as parameters.  

Just a glimps of what we can achieve with the functional interfaces introduced in Java8.  

If we wnat to use filter criteria like  

```java  
Predicate<Integer> isGreaterThan3 = number -> number > 3;
// and next time we need something like
Predicate<Integer> isGreaterThan5 = number -> number > 5;
```

now we can introduce a function that takes a pivotal element and returns a predicate to reduce the douplication above. - Sounds like killing sparows with cannons at first sight.  

```java
Function<Integer, Predicate<Integer>> isGreaterThan(Integer pivot) = pivot -> 
  number -> number > pivot;
```

A call to this function looks like this

```java
System.out.println("Total all assets with value greater than 2500: " +
  AssetUtils.totalAssetValues(assets, isGreaterThan.apply(2500)));
// Total all assets with value greater than 2500: 7000
```
There are excellent talks on youtube about this topic.  
And I highly recomment the talks by Vankat Subramaniam.  

There are more ideas to embrace! You will see.

# Resources

* [FPJ]: Functional Programmming in Java by Venkat Subramaniam, PragProg Pub 2014
* [Refac]: Refactoring (second edition) by Martin Fowler, Addison Wesley New York 2019
* [RefacToPat]: Refactoring to Pattern by Joshua Kerievsky, Addison Wesley New York 2005
* [RefacWork]: Refactoring Workbook by William C. Wake, Addison Wesley New York 2004
* [DuckDuck]: search for "youtube venkat subramaniam functional interfaces"