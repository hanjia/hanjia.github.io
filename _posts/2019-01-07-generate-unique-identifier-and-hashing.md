---
layout: post
title: Generate unique id in a distributed system
date: 2019-01-07
tag:
  - scala
  - java
  - algorithm
  - programming
---



Define Spark Context and Spark SQL Context.

```scala
val sparkConext = new SparkContext(conf)
val sqlConext = new SQLContext(sparkConext)
```



Understand .hashcode() and .toString() function

```java
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```




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


How hashCode() is implemented in the class java.lang.String

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

org.apache.spark.sql.Row

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

Using MurmurHash3 lib provided by Scala







Timed UUID seems to be safe but sometimes it's still possible to see collision in a distributed system. We have seen such a case many times in our system. We are using [Datastax driver's implementation](https://docs.datastax.com/en/drivers/java/2.0/com/datastax/driver/core/utils/UUIDs.html#timeBased--) for generating timed UUID. Because if we run on a large number of partitions.


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
