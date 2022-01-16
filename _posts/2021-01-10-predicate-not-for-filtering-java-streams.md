---
title:  "Predicate.not for filtering Java Streams"
date:   2021-01-10 15:07:00 +0100
excerpt: "Predicate.not is handy to filter streams with method references for the opposite of a provided method"
header:
  og_image: /assets/images/2021-01-10/filter.png
---


Let's start with an example:

```java
userList.stream()
  .filter(user -> user != null)
  .filter(user -> !user.isActivated())
  .count();
```

The idea of this code snippet is to count all deactivated users. At first we have to filter all nulls and secondly remove all activated users. The remaining list entries can just be counted.

I want to focus on the two `filter` methods. IMHO using method references should be preferred. Let me show that by an example which filters for the opposite. Please keep in mind that this will throw a `NullPointerException` since we call `isActivated` only on nulls.

```java
userList.stream()
  .filter(Objects::isNull)
  .filter(User::isActivated)
  .count();
```

Wouldn't you agree that the method reference `User::isActivated` is much easier to read than `user -> !user.isActivated()`? But we do not want to filter for activated users, but for de-activated users. We could achieve that be implementing a `isDeactivated` method in the User class, but I want to show you a different way.

It's the very same idea for remove null objects from the list. It's really nice to just write `.filter(Objects::isNull)` to find only nulls, but I always wrote `.filter(u -> u != null)` in the past to get only non-nulls.

Until today, I never took the time to look for an easier way. But there are two very nice (and obvious) solutions:

## Objects::nonNull

In the [Objects class](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Objects.html) there exists not only the `isNull` method, but also `nonNull` to achieve exactly the behaviour I need.

```java
userList.stream()
  .filter(Objects::nonNull)
  .count()
```

## Predicate::not

The [Predicate interface](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/function/Predicate.html) has a handy `not` method to negate any Predicate. It's concise and easy to understand when using a static import for the `Predicate.not` method

```java
userList.stream()
  .filter(not(Objects::isNull))
  .filter(not(User::isActivated))
  .count();
```
`Predicate.not` is more powerful, it allows to negate any filter option, e.g. to remove empty lists out of a list of lists:


By combining the two approaches, we are again able to use method references to count all deactivated users:

```java
userList.stream()
  .filter(Objects::nonNull)
  .filter(not(User::isActivated)
  .count();
```

Filtering Java Streams can be very concise by using method references. To achieve the opposite of a provided method, Predicate.not is a handy solution.

Comments are welcome on [Twitter](https://twitter.com/TheThomasPr/status/1348273079500333058).
