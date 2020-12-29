---
title:  "Build a Cartesian Product with Java Streams"
date:   2020-12-29 21:40:00 +0100
categories: blog
excerpt: "Buildung a Cartesian Product or Permutation is challenging. I used Java Streams to implement a readable algorithm."
header:
  og_image: /assets/images/2020-12-29/cartesian-product.png
---

Recently, I ran over an old post by Baeldung: [Permutations of an Array in Java
](https://www.baeldung.com/java-array-permutations). He presented very well solutions, as usual. I just want to provide another point of view: Readability. IMHO, readability is one of the most important aspects for code. Some time ago I had to solve a similiar issue and used Java Streams to create a Cartesian Product out of Lists.


```java
import static java.util.Collections.unmodifiableList;
import static java.util.stream.Collectors.toList;

import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.function.BinaryOperator;
import java.util.stream.Stream;

public final class CartesianProductUtil {

  private CartesianProductUtil() { }

  /**
   * Compute the cartesian product for n lists.
   * The algorithm employs that A x B x C = (A x B) x C
   *
   * @param listsToJoin [a, b], [x, y], [1, 2]
   * @return [a, x, 1], [a, x, 2], [a, y, 1], [a, y, 2], [b, x, 1], [b, x, 2], [b, y, 1], [b, y, 2]
   */
  public static <T> List<List<T>> cartesianProduct(List<List<T>> listsToJoin) {
    if (listsToJoin.isEmpty()) {
      return new ArrayList<>();
    }

    listsToJoin = new ArrayList<>(listsToJoin);
    List<T> firstListToJoin = listsToJoin.remove(0);
    Stream<List<T>> startProduct = joinLists(new ArrayList<T>(), firstListToJoin);

    BinaryOperator<Stream<List<T>>> noOp = (a, b) -> null;

    return listsToJoin.stream() //
        .filter(Objects::nonNull) //
        .filter(list -> !list.isEmpty()) //
        .reduce(startProduct, CartesianProductUtil::joinToCartesianProduct, noOp) //
        .collect(toList());
  }

  /**
   * @param products [a, b], [x, y]
   * @param toJoin   [1, 2]
   * @return [a, b, 1], [a, b, 2], [x, y, 1], [x, y, 2]
   */
  private static <T> Stream<List<T>> joinToCartesianProduct(Stream<List<T>> products, List<T> toJoin) {
    return products.flatMap(product -> joinLists(product, toJoin));
  }

  /**
   * @param list   [a, b]
   * @param toJoin [1, 2]
   * @return [a, b, 1], [a, b, 2]
   */
  private static <T> Stream<List<T>> joinLists(List<T> list, List<T> toJoin) {
    return toJoin.stream().map(element -> appendElementToList(list, element));
  }

  /**
   * @param list    [a, b]
   * @param element 1
   * @return [a, b, 1]
   */
  private static <T> List<T> appendElementToList(List<T> list, T element) {
    int capacity = list.size() + 1;
    ArrayList<T> newList = new ArrayList<>(capacity);
    newList.addAll(list);
    newList.add(element);
    return unmodifiableList(newList);
  }
}
```

The code is also available on this [GitHub Gist](https://gist.github.com/ThomasPr/8e038d5ebca97261940bf1dd13d3417d).
