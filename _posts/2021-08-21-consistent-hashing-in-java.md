---
layout: post
title:  "Consistent Hashing in Java"
tags: ['java', 'Algorithms']
---

## Consistent Hashing in Java

In computer science, consistent hashing is a hashing technique used to re-balance the minimum entries in the current set.
For example, it is especially used in distributed systems when you want to implement a distributed cache in memory.

But, first, let's see what problems it solved.

### ➡️ Naive Hashing

Suppose that you want to implement a distributed cache across `n` servers, something like that:

<img src="/static/blog/consistent-hashing/01.svg" width="400" style="margin: 0"/>

In this case, when you want to check that a given key is stored in the cache, you have two solutions:

- Check each server, and stop as soon as one of them returns a positive answer.
- Use a hash function to map a key to a server and check one and only one server!

Note that the `hash` function can be implemented using any algorithm (MD5, SHA-1, FNV1, etc.), it does not matter: as long as it can be converted to an Integer, it's easy to choose a server using a very simple modulo.

For example, suppose the key `"Hello World"`, and suppose we have those four servers:

<img src="/static/blog/consistent-hashing/02.svg" width="400" style="margin: 0"/>

The main advantage of this technique is that it is straightforward to implement. For example, here is a very basic implementation in Java:

```java
public interface HashFunction {
  int compute(String value);
}
```

```java
public final class Node {
  private final String id;

  public Node(String id) {
    this.id = id;
  }

  public String getId() {
    return id;
  }
}
```

```java
public final class Cluster {
  private final List<Node> nodes;
  private final HashFunction hashFunction;

  public Cluster(List<Node> nodes, HashFunction hashFunction) {
    this.nodes = new ArrayList<>(nodes);
    this.hashFunction = hashFunction;
  }

  public Node findNode(String value) {
    if (nodes.isEmpty()) {
      throw new IllegalStateException("Cannot find node in an empty cluster");
    }

    int hash = Math.abs(hashFunction.compute(value));
    return nodes.get(hash % nodes.size());
  }
}
```

And that's it!

BUT: the main problem with this solution is when you need to add/remove node: when this happens, you have to re-map (or invalidate) almost all the keys!

The solution to this problem is: **consistent hashing**.

### ➡️ Consistent Hashing

Consistent Hashing is a technique that sees the cluster as a "ring" in which each node is positioned.
For example:

<img src="/static/blog/consistent-hashing/03.svg" width="400" style="margin: 0"/>

Here, we have four nodes: `srv1`, `srv2`, `srv3`, and `srv4`.

Now, suppose that we want to map a key to a given node: we use the same hashing function and put the resulting key on the ring: the next server (clockwise) on the ring will be the winner!

Let's see an example:

<img src="/static/blog/consistent-hashing/04.svg" width="400" style="margin: 0"/>

- We still have our four nodes: `srv1`, `srv2`, `srv3`, and `srv4`
- We can now map `k1`, `k2`, and `k3` 🚀

The main advantage of this technique is that adding or removing a node in the cluster only needs to re-map a few keys:

<img src="/static/blog/consistent-hashing/05.svg" width="400" style="margin: 0"/>

In the example below, we suppose that `srv4` goes down: all we have to do is to re-map keys located between `srv3` and `srv4` to `srv1` 🎉

Let's see how to implement this in Java:

```java
public interface HashFunction {
  int compute(String value);
}
```

```java
public final class Node {
  private final String id;

  public Node(String id) {
    this.id = id;
  }

  public String getId() {
    return id;
  }
}
```

```java
public final class Cluster {
  private final SortedMap<Integer, Node> nodes;
  private final HashFunction hashFunction;

  public Cluster(List<Node> nodes, HashFunction hashFunction) {
    this.nodes = new TreeMap<>();
    this.hashFunction = hashFunction;

    for (Node node : nodes) {
      this.nodes.put(computeHash(node), node);
    }
  }

  public Node findNode(String value) {
    if (nodes.isEmpty()) {
      throw new IllegalStateException("Cannot find node in an empty cluster");
    }

    int hash = computeHash(value);
    if (nodes.containsKey(hash)) {
      return nodes.get(hash);
    }

    // Get next node on the ring.
    SortedMap<Integer, RingNode> tailMap = nodes.tailMap(hash);
    if (tailMap.isEmpty()) {
      // If we don't have a "next" node, get the first one.
      tailMap = nodes;
    }

    return tailMap.get(tailMap.firstKey());
  }

  private int computeHash(String value) {
    return Math.abs(hashFunction.compute(value));
  }
}
```

Let's see how it works:

👉 Our ring is implemented using a `TreeMap`, this allows us to:
  - Add node in the ring in `O(log n)`
  - Remove node in the ring in `O(log n)`
  - Find node in the ring in `O(log n)`

👉 An alternative would be to implement the ring using a sorted list, in this case:
  - Adding can be implemented in `O(n)`
  - Removing a node can be implemented in `O(n)`
  - Finding a node can be implemented in `O(log n)` (using a simple binary lookup).

Even if I did not implement node addition/removal in this example, I chose to use a `TreeMap` because it seems more efficient :)

### ➡️ Going Further

Consistent Hashing is a great technique to have in your "besace" if you work on a distributed system.

The implementation above is straightforward. If you want to go further, I suggest you to:

- Implement node addition/removal
- Implement virtual nodes*

_Virtual nodes is a technique where we can add, for any node in the ring, additional "virtual" nodes (node that does not exist, but "points" to the actual node), the idea being to adjust & balance key mapping across the ring_

Or, if you want to take a look, checkout my [GitHub project](https://github.com/mjeanroy/consistent-hashing-4j) where I implemented all of this and let me know what you think :)
