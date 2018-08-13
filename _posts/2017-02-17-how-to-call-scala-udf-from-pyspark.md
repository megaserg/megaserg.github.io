---
layout: post
title:  "How to call Scala UDFs from PySpark"
tags: scala, spark, pyspark
---

# Motivation

Suppose we decided to speed up our PySpark job. One possible way to this is to write a Scala UDF.
If we implement it in Scala instead of Python, the Spark workers will execute the computation themselves rather than ask Python code to do it, and won't need to serialize/deserialize the data to the Python component. Double win!

So, we can write a simple Scala object with a single function in it. Then we wrap the function in `udf`. I sincerely apologize for the imperative style.

```scala
package com.example.spark.udfs

import org.apache.spark.sql.expressions.UserDefinedFunction
import org.apache.spark.sql.functions.udf

object Primes {
  def isPrime(x: Int): Boolean = {
    if (x < 2) return false
    var i = 2
    while (i * i <= x) {
      if (x % i == 0) {
        return false
      }
      i += 1
    }
    return true
  }

  def isPrimeUDF: UserDefinedFunction = udf { x: Int => isPrime(x) }
}
```

Save it as `src/com/example/spark/udfs/Primes.scala` or under any other convenient name.


# Compiling Scala

We don't need heavyweight IDEs or `sbt` to compile such a simple function. Let's do it from the command line! We would need to use Scala 2.11 when compiling (as of Spark 2.0.1/2.1.0). [Download Scala 2.11](http://www.scala-lang.org/download/2.11.8.html) and unpack it somewhere (I unpacked it as `~/Downloads/Distribs/scala-2.11.8`).

Setup the environment variables:
```bash
SCALA_HOME=$HOME/Downloads/Distribs/scala-2.11.8
PATH=$PATH:$SCALA_HOME/bin
```

Make sure we also have `$SPARK_HOME` set:
```bash
$ echo $SPARK_HOME
/Users/sserebryakov/spark-2.0.1-bin-hadoop2.7
```

Check that we can invoke `scalac`:
```bash
$ scalac -version
Scala compiler version 2.11.8 -- Copyright 2002-2016, LAMP/EPFL
```

Go to our project folder and build a JAR. Note that the wildcard is just a star, not `*.jar`:
```bash
$ scalac -classpath "$SPARK_HOME/jars/*" src/com/example/spark/udfs/Primes.scala -d primes_udf.jar
```

# Running PySpark

Now we can provide the JAR in the classpath for `pyspark`:
```bash
$ pyspark --jars primes_udf.jar
```

Now we can use the Scala function like this. The prerequisite is that the data should already be in the `DataFrame` format. Be careful with the syntax, the `py4j` exceptions are rarely helpful.

```python
from pyspark.sql import Row
from pyspark.sql.column import Column, _to_java_column, _to_seq

a = range(10)
df = sc.parallelize(a).map(lambda x: Row(number=x)).toDF()

scala_udf_is_prime = sc._jvm.com.example.spark.udfs.Primes.isPrimeUDF()
is_prime_column = lambda col: Column(scala_udf_is_prime.apply(_to_seq(sc, [col], _to_java_column)))
df.withColumn('is_prime', is_prime_column(df['number'])).show()
```

This will print:

```python
+------+--------+
|number|is_prime|
+------+--------+
|     0|   false|
|     1|   false|
|     2|    true|
|     3|    true|
|     4|   false|
|     5|    true|
|     6|   false|
|     7|    true|
|     8|   false|
|     9|   false|
+------+--------+
```

Tested with Spark 2.0.1, but should work for 2.1.0 (the latest as of this writing) as well.


# Currying

If we want to initialize the Scala class with some arguments once, and call the function repeatedly, we can use a `case class`. Example:

```scala
package com.example.spark.udfs

import org.apache.spark.sql.expressions.UserDefinedFunction
import org.apache.spark.sql.functions.udf

case class DistanceComputer(originX: Double, originY: Double) {
  def distanceUDF: UserDefinedFunction = udf { (x: Double, y: Double) =>
    math.sqrt(math.pow(x - originX, 2) + math.pow(y - originY, 2))
  }
}
```

Registering with PySpark would look like:
```python
scala_udf_distance = sc._jvm.com.example.spark.udfs.DistanceComputer(0.0, 0.0).distanceUDF()
```

Make sure to conform exactly to the argument types when calling the constructor. A constructor with `Double` argument should be called with `0.0`, not `0`.
