---
layout: post
title: Generate unique identifiers in a distributed system
date: 2019-01-07
tag:
  - scala
  - java
  - algorithm
  - programming
---


Recently I worked on a project that we need to create persistent unique identifiers for records in our data processing system. By solving the problem, I had a chance to explore and understand some related topics that are usually easy to be ignored.



Let's first take a look at how .hashcode() and .toString() function are defined in java.lang.Object, which is the root for all Java classes. 

```java
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```
It depends on the hashCode() function.

```java
/**
 *
 * .......
 *
 * As much as is reasonably practical, the hashCode method defined by
 * class {@code Object} does return distinct integers for distinct
 * objects. (This is typically implemented by converting the internal
 * address of the object into an integer, but this implementation
 * technique is not required by the
 * Java&trade; programming language.)
 *
 * ......
 *
 */
public native int hashCode();

```

Another common usage for hashCode() function is to compare strings. The .hashCode() implementation in the class java.lang.String is using a universal hash function with 31 as the base.

```java
public int hashCode() {
  int h = hash;
  if (h == 0 && value.length > 0) {
    char val[] = value;

    for (int i = 0; i < value.length; i++) {
      h = 31 * h + val[i];
    }
    hash = h;
  }
  return h;
}
```


-----------------

Since we are using Spark SQL for the data pipeline so it will useful to check how the hashcode function is implemented in the org.apache.spark.sql.Row object.

```scala
override def hashCode: Int = {
  // Using Scala's Seq hash code implementation.
  var n = 0
  var h = MurmurHash3.seqSeed
  val len = length
  while (n < len) {
    h = MurmurHash3.mix(h, apply(n).##)
    n += 1
  }
  MurmurHash3.finalizeHash(h, n)
}
```

Note that this method is using native MurmurHash3 lib provided by Scala. The good thing is it considers the value of the object.

-----------------


Now back to the original problem. Basically we need to find a way to create unique identifiers to meet the following requirements:

1) Minimize the likelihood of collisions

2) Even the same record should have different ids over different runs 


Timed UUID seems to be a good fit. 

However, it doesn't guarantee the uniqueness in a distributed system. We had seen some duplicate ids in results. We were generating version 1 Timed UUID (see [Datastax driver's timed UUID implementation](https://docs.datastax.com/en/drivers/java/2.0/com/datastax/driver/core/utils/UUIDs.html#timeBased--) for details), which takes date-time and MAC address into account. 
Because we have a large number of data partitions in Spark, so it's still possible to encounter a situation where two or multiple executors (JVMs) are running in parallel and on the same physical machine.


To avoid this kind of collisions, we decided to combine the timed UUID with a hash value generated from the content (a set of strings in our case). This works well because it's very unlikely to have a collision for the hash value, while those two records are being processed by the same physical machine at the exact same time.



The final code is here.

```scala
import java.util.UUID
import java.nio.charset.Charset
import com.google.common.hash.Hashing
import com.datastax.driver.core.utils.UUIDs

object IDGenerator {

  def getUUID: UUID = UUIDs.timeBased

  def getString: String = getUUID.toString

  def getEmptyUUID = new UUID(0l, 0l)

  def getScrambledUUID(originalId: String): String = {
    val hashingFunction = Hashing.murmur3_128()
    val hasher = hashingFunction.newHasher()
    val value = originalId + "_" + getUUID
    hasher.putString(value, Charset.forName("utf-8"))
    hasher.hash().toString()
  }
}

```
------------------

What if we need the id to be persistent over runs? Same input always leads to the same output. Just as the input variable originalId in the example above. 

One important thing to keed in mind is that if your data is a collection, you have to sort the collection first to make sure the elements are in the same order every time before send it to the hasher. Otherwise, the results could still be different even the elements in the collection are the same. 

```scala
def murmur3Hash128BitStringForIdSet(idSet: Seq[Row]): String = {
    val hashingFunction = Hashing.murmur3_128()
    val hasher = hashingFunction.newHasher()
    idSet.sortWith(_.hashCode() < _.hashCode()) // Sort the ids
      .map(entityId => hasher.putString(, Charset.forName("UTF-8")))
    hasher.hash().toString()
  }

```