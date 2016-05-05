---
layout: post
title:  "How to call Scala UDFs from PySpark"
---

Suppose you decided to speed up your PySpark job. One possible way to this is to make a Scala UDF. If you implement it in Scala instead of Python, the Spark workers will execute the computation themselves rather than ask Python code to do it, and won't need to serialize/deserialize the data to the Python component. Double win!

So, we can write a simple Scala object with a single function in it. Then we wrap the function in `udf`. Make sure to use Scala 2.11 when compiling (as of Spark 2.0.1/2.1.0). 

{% highlight scala %}
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
    true
  }

  def isPrimeUDF: UserDefinedFunction = udf { x: Int => isPrime(x) }
}
{% endhighlight %}

Build a JAR and provide it in the classpath for the Python driver. In this example, I'm running `pyspark` from command line:

	pyspark --driver-class-path out/artifacts/simple_scala_udf_jar/simple-scala-udf.jar


Now we can use the Scala function like this. The prerequisite is that the data should already be in the `DataFrame` format. Be careful with the syntax, the `py4j` exceptions are rarely helpful.

{% highlight python %}
from pyspark.sql import Row
from pyspark.sql.column import Column, _to_java_column, _to_seq

a = range(10)
df = sc.parallelize(a).map(lambda x: Row(number=x)).toDF()

scala_udf_is_prime = sc._jvm.com.example.spark.udfs.Primes.isPrimeUDF()
is_prime_column = lambda col: Column(scala_udf_is_prime.apply(_to_seq(sc, [col], _to_java_column)))
df.withColumn('is_prime', is_prime_column(df['number'])).show()
{% endhighlight %}

This will print:

{% highlight python %}
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
{% endhighlight %}

Tested with Spark 2.0.1.
