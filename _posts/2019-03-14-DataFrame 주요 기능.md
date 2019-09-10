---
layout: post
title: "DataFrame 주요 기능"
categories:
  - Posts
tags:
  - scala
  - spark
  - dataframe
last_modified_at: 2019-03-14T12:57:42+09:00
---


### 1. 날짜 포맷 변경

source 데이터의 time은 아래와 같으며, 이를 날짜 형식으로 바꾸어 보자.

Before:
>time=1551836848116

After:
> refined_date=2019-02-20 04:02:13

초기 데이터 포맷은 Unix Time이며 이는 `from_unixtime`을 통해 변경할 수 있다.

```scala
import org.apache.spark.sql.functions._

dataframe.select (
from_unixtime(
	get_json_object(col("url"), "$.time").cast("bigint"), 
	"YYYY-MM-dd HH:mm:ss").as("refined_date").cast("timestamp"),
col("data")
)
```
> get_json_object: `col("url")`은 DataFrame에 있는 column 중 하나이며, json 정보가 저장되어 있다. `"$.time"`은 json에서 time Key를 가진 Value를 반환한다.
> 
> "YYYY-MM-dd hh:mm:ss": 원하는 날짜 포맷을 기재한다. 포맷은 아래 문서 참고.
[https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html)
>
> as("refined_date"): 변경할 column 이름
>
> cast("timestamp"): 변경할 데이터 타입, timestamp로 변경하면 시간,분,초 까지 모두 나타낸다. 만약 date로 변경하면 시간 정보를 제외한 년,월,일 정보만 표시한다.

### 2. 사용자 정의 함수

깔끔한 코드 구현을 위해 `사용자 정의 함수(UDF)` 사용법을 알아보자.

```scala
import org.apache.spark.sql.functions._

// UDF 선언
// UDF는 val 타입으로 선언한다.
val isCheckUDF= udf(
	(data: Boolean) => {  
	  if(data == true) "success"
	  else "fail" 
})  
  
dataframe.select(
  isCheckUDF(col("data")).as("isChecked"))
```
> isCheckUDF 이름을 가진 UDF를 정의한다.
> > udf ( (<파라미터>: <타입>) => { <내용> } )
>
> 선언된 내용을 dataframe의 'data' 필드에 적용한다. `isCheckUDF(col("data")`

### 3. 집계 함수 (Group By, Agg 등)

간단하게 GroupBy, Agg를 통해 집계를 해보았다. Agg를 쓰려면 Group By를 먼저 진행해야 한다. 하고자 하는 집계는 아래와 같다.

**input Table:** <br>

|Text | Sub_Text |
|:--------|:--------|
|테스트1 | 서브테스트1|
|테스트1 | 서브테스트2|
|테스트1 | 서브테스트2|
|테스트2 | 서브테스트1|
|테스트2 | 서브테스트2|
|테스트2 | 서브테스트3|
|테스트1 | 서브테스트2|
  
**output Table:**  <br>

|Text | Sub_Text_list|
|:--------|:--------|
|테스트1 | 서브테스트1-2, 서브테스트2-2|
|테스트2 | 서브테스트1-1, 서브테스트2-1, 서브테스트3-1|

각 테스트를 그룹으로 묶고 서브 테스트 별로 Count를 세려고 한다.  <br>

```scala
Dataframe.groupBy("Text", "Sub_Text")  
  .count()  
  .orderBy(col("Text"), desc("count"))  
  .withColumn("Sub_TextAndCount", concat(concat(col("Sub_Text"), lit("-")), col("count")))  
// 먼저 Text, Sub_Text를 그룹으로 묶고 Count를 구한다. 이후 Count 별로 내림차순을 진행 후 SubText와 Count를 하나의 열로 만들어 준다.
/*
테스트1 | 서브테스트1-2
테스트1 | 서브테스트2-2
테스트2 | 서브테스트1-1
테스트2 | 서브테스트2-1
테스트2 | 서브테스트3-1
*/
  .groupBy("query")  
  .agg(concat_ws(",", collect_list("scAndCount")) as "scAndCount_list")  
  .orderBy(col("query"))
// 이후 query를 다시 그룹으로 묶고, 각 그룹들의 문자들을 concat_ws를 통해 묶어준다.
```


