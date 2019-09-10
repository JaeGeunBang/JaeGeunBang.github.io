---
layout: post
title: "Spark-Junit Test"
categories:
  - Posts
tags:
  - scala
  - spark
  - junit
last_modified_at: 2019-03-14T12:57:42+09:00
---

스파크로 개발 후 주요 기능에 대해 테스트를 진행해보자.

먼저 pom.xml에 아래 dependency를 추가한다.

```xml
<dependency>  
 <groupId>junit</groupId>  
 <artifactId>junit</artifactId>  
 <version>4.7</version>  
 <scope>test</scope>  
</dependency>  
 <groupId>org.scalatest</groupId>  
 <artifactId>scalatest_2.11</artifactId>  
 <version>3.0.5</version>  
 <scope>test</scope>  
</dependency>
```



아래 wordCount 메서드를 테스트해보자.

```scala
package com.example.test

import org.apache.spark.SparkConf
import org.apache.spark.SparkContext
import org.apache.spark.rdd.RDD.rddToPairRDDFunctions

object WordCount {  
  // 아래 메서드를 테스트해보자.
  def wordCount(fileRDD: RDD) = {  
    fileRDD.flatMap(line => line.split(" "))  
      .map(word => (word, 1)).reduceByKey(_ + _)  
  }  
  
  def main(args: Array[String]): Unit =  
  {  
    val inputPath = args(0)  
    val outputPath = args(1)  
      
    val conf = new SparkConf().setMaster("master").setAppName("wordCount")  
    val sc = new SparkContext(conf)  
    sc.setLogLevel("ERROR")  
    val fileRDD = sc.textFile(inputPath)  
      
    val resultRDD = wordCount(fileRDD)  
    resultRDD.saveAsTextFile(outputPath)  
  }  
}
```



test 폴더에 test를 위한 클래스를 생성 후 ScalaTest가 제공하는 FunSuite를 통해 쉽게 테스트를 할 수 있다.

```scala
import org.scalatest._  
import org.junit.Assert._ 

class WordCount extends FunSuite with SharedSparkContext {  
  test("testWordCount") {  // 테스트 함수 이름
    try {  
      val s = "Hi hi hi bye bye bye word count"  
	  val seq = s.split(" ")  
      val rdd = sc2.parallelize(seq)  
      val result = WordCount.wordCount(rdd)  // wordCount 수행
      assertEquals(8, result.count())  // 결과 확인
        
    } catch {  
      case e: Exception =>  
        e.printStackTrace()  
    }  
  }  
}
```


여기서 SharedSparkContext는 아래와 같이 정의한다. 

SharedSparkContext는 테스트에서 SparkContext 호출이 필요한 메서드들이 공통으로 사용할 수 있다.

```scala
import org.apache.spark.SparkContext  
import org.scalatest._  
  
trait SharedSparkContext extends BeforeAndAfterAll { self: Suite =>  
  @transient private var _sc: SparkContext = _  
  
  def sc: SparkContext = _sc  
  
  override def beforeAll() {  
    _sc = SparkContext("local[4]", "test")  
    super.beforeAll()  
  }  
  
  override def afterAll(): Unit = {  
    _sc.stop()  
    _sc = null  
 super.afterAll()  
  }  
}
```
실행 후 테스트 결과를 확인한다. <br>



### private method 테스트 방법

만약 위 wordCount 기능이 private으로 선언되어있다면, 외부 Test Class에서 호출할 수 없다.

이를 위해 리플렉션을 통해 해당 메서드를 접근할 수 있다.

```scala
import org.scalatest._  
import org.junit.Assert._ 

class WordCount extends FunSuite with SharedSparkContext {  
  test("testWordCount") {  
    try {  
      val s = "Hi hi hi bye bye bye word count"  
  val seq = s.split(" ")  
      val rdd = sc.parallelize(seq)\  
  
      val method = WordCount.getClass().getDeclaredMethod("wordCount", classOf[RDD])  
      method.setAccessible(true)  // accessible을 true로 설정함.
      val result = method.invoke(WordCount, rdd).asInstanceOf[DataFrame]  
      
      assertEquals(8, result.count())  
  
    } catch {  
      case e: Exception =>  
        e.printStackTrace()  
    }  
  }  
}
```



### 참고

[http://www.scalatest.org/getting_started_with_fun_suite](http://www.scalatest.org/getting_started_with_fun_suite)

[https://www.slideshare.net/SparkSummit/spark-summit-eu-talk-by-ted-malaska](https://www.slideshare.net/SparkSummit/spark-summit-eu-talk-by-ted-malaska)