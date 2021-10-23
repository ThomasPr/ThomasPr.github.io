---
title:  "Java Streams are not always a good fit for readability"
date:   2021-03-25 23:03:03 +0100
categories: blog
excerpt: "Java Streams are powerful but need to get special attention to keep the implementation readable."
header:
  og_image: /assets/images/2021-03-25/streams.png
---


I really like functional programming, it offers powerful expressions with only a few lines of code. When Streams have been introduced in Java 8, they where a huge improvement for someone like me who got used to ruby and its functional power for quite some time.

Even if Java Streams are a good choice for many problems, they might be not the best choice for readability. I want to show you an example where I struggeled for a good solution. Java Streams sound like the perfect solution for that kind of problem.

But let's get started:


## The problem

I cannot give any details about the actual business requirement about that. But I hopefully got a really neat story instead.

I want to get a list of all people and their pets and the possibiliets they can go for a walk. Each person wants to walk with all of their pets in all possible combinations. But it is not allowed two walk two pets of the same kind at the same time.

Example: Thomas has two dogs, two cats and a pig. The result shold be:

| `Thomas` | `Dog 1` | `Cat 1` | `Pig` | 
| `Thomas` | `Dog 2` | `Cat 1` | `Pig` | 
| `Thomas` | `Dog 1` | `Cat 2` | `Pig` | 
| `Thomas` | `Dog 2` | `Cat 2` | `Pig` | 

Keep in mind that even if Thomas would have no cat, he wants wo walk his dogs and his pig.

| `Thomas` | `Dog 1` | | `Pig` | 
| `Thomas` | `Dog 2` | | `Pig` | 


## Start with some coding

The people and animals are fetched from independent external sources and could look like that:

```java
List<String> people = getPeopleInTown("Freiburg");

Map<String, List<Cat>> = getCatsForPeople(people);
Map<String, List<Dog>> = getDogsForPeople(people);
Map<String, List<Pig>> = getPigsForPeople(people);
```

To model a `Walk` the simple Java class will be used:

```java
public class Walk {

	String person;
	Cat cat;
	Dog dog;
	Pig pig;

	public Walk(String person, Cat cat, Dog dog, Pig pig) {
		this.person = person;
		this.cat = cat;
		this.dog = dog;
		this.pig = pig;
	}

	// getters and setters omitted for brevity
}
```


To build the walk, we introduce a `buildWalks` method.

```java
public List<Walk> buildWalks(
    List<String> people,
    Map<String, List<Cat>> cats,
    Map<String, List<Dog>> dogs,
    Map<String, List<Pig>> pigs) {

  List<Walk> walks = new ArrayList<>();

  // do the magic

  return walks;
}
```


## The naive implementation

My first appraoch was to use just a couple of `for`-loops. It looks gorgeous and is easy to read.

```java
public List<Walk> buildWalks(
    List<String> people,
    Map<String, List<Cat>> cats,
    Map<String, List<Dog>> dogs,
    Map<String, List<Pig>> pigs) {

  List<Walk> walks = new ArrayList<>();

  for(String person : people) {
    for (Cat cat : cats.get(person)) {
      for (Dog dog : dogs.get(person)) {
        for (Pig pig : pigs.get(person)) {
          walks.add(new Walk(person, cat, dog, pig));
        }
      }
    }
  }

  return walks;
}
```

Unfortunately, it doesn't work. If Thomas has no cat, he won't be able to do any walk. That's bad for his other pets.

If the list of cats is empty, the `for`-loop will not be executed and therefore the `walks.add()`-method will never be called.

Ok, let's fix it. We need to make sure that every `for`-loop will be executed at least once:


```java
public List<Walk> buildWalks(
    List<String> people,
    Map<String, List<Cat>> cats,
    Map<String, List<Dog>> dogs,
    Map<String, List<Pig>> pigs) {

  List<Walk> walks = new ArrayList<>();

  for(String person : people) {
    for (Cat cat : atLeastOnce(cats.get(person))) {
      for (Dog dog : atLeastOnce(dogs.get(person))) {
        for (Pig pig : atLeastOnce(pigs.get(person))) {
          walks.add(new Walk(person, cat, dog, pig));
        }
      }
    }
  }

  return walks;
}

private <T> List<T> atLeastOnce(List<T> animals) {
  if (animals == null || animals.isEmpty()) {
    return getNulledList();
  }
  return animals;
}

private <T> List<T> getNulledList() {
  List<T> list = new ArrayList<>();
  list.add(null);
  return list;
}
```


## Streams to the rescue?

Let's use Java Streams to implement the `buildWalks()` again.

```java
public List<Walk> buildWalks(
    List<String> names,
    Map<String, List<Cat>> cats,
    Map<String, List<Dog>> dogs,
    Map<String, List<Pig>> pigs) {

  List<Walk> walks = new ArrayList<>();

  names.forEach(name -> {
    forEachAtLeastOnce(cats.get(name), cat -> {
      forEachAtLeastOnce(dogs.get(name), dog -> {
        forEachAtLeastOnce(pigs.get(name), pig -> {
          walks.add(new Walk(name, cat, dog, pig));
        });
      });
    });
  });

  return walks;
}

private T forEachAtLeastOnce(List<T> animals, Consumer<T> consumer) {
  if (animals == null || animals.isEmpty()) {
    consumer.accept(null);
  }
  animals.forEach(consumer);
}
```

If you look at the code, do you see at a glance what's going on? Indee, I need some time to read through every single line to know what the method actually returns.


## Conclusion


The main difference of the implementations is the idea how the permuations will be created.


```java
for(String person : people) {
  for (Cat cat : atLeastOnce(cats.get(person))) {
    for (Dog dog : atLeastOnce(dogs.get(person))) {
      for (Pig pig : atLeastOnce(pigs.get(person))) {
        walks.add(new Walk(person, cat, dog, pig));
      }
    }
  }
}
```

```java
names.forEach(name -> {
  forEachAtLeastOnce(cats.get(name), cat -> {
    forEachAtLeastOnce(dogs.get(name), dog -> {
      forEachAtLeastOnce(pigs.get(name), pig -> {
        walks.add(new Walk(name, cat, dog, pig));
      });
    });
  });
});
```

If you read the code, do you prefer nested for-loops or streams?

IMHO the nested for-loops can be understood more easily than streams. Therefore, I prefer the nested for-loops.

Please keep in mind, that Java Streams is a powerful performance improvement when you have to deal with large data sets. But you need to take special attention on readability if you don't want to get a trade-off for your readers.
