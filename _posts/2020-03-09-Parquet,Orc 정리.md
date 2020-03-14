---
layout: post
title: "Parquet, ORC 정리"
categories:
  - Posts
tags:
  - parquet
  - orc
last_modified_at: 2020-03-04T12:57:42+09:00P
---





## Parquet

컬럼 기준 저장 포맷으로 파일크기, 쿼리 성능 모두 효율성을 가진다.

- 파일크기: 동일한 컬럼을 나란히 모아 저장하기 때문에 인코딩 효율이 높음
- 쿼리성능: 쿼리에 필요하지 않는 컬럼은 처리하지 않기 때문에 효율이 높음



용어

- Block (hdfs block): hdfs 에서 사용하는 block과 같은 용어이며, 변하지 않는 것이 특징이다.
- File: metadata 정보를 포함하며, 실제 data를 포함하지 않는다.
- Row group: data를 논리적인 horizontal partitioning (샤딩)로, 물리적인 구조는 아니다. row group은 dataset 내 각 column을 위한 column chunk들이 포함되어 있다.
- Column chunk: 일부 column의 data chunk로 특정 row group에 있으며 연속적이다.
- Page:  나눌수 없는 가장 작은 unit이며, Column chunk를 여러 page로 나눈다.



File Format

- `헤더` - **Block 1** - **Block 2** - **Block N** - `꼬리말`
  - `헤더`는 파케이 포맷의 파일임을 알려주는 4byte magic number로 "PAR1"만 가짐
  - `꼬리말`은 file metadata 정보를 가지며, 모든 column metadata의 시작 location을 포함한다.
  - **Block** 내 `Column Chunk N개`, `M개 Row Group`, Column Metadata로 이루어져 있다.

![0](https://user-images.githubusercontent.com/22383120/76186044-77fa9400-6214-11ea-97c4-44fb1c787ac7.PNG)

```
4-byte magic number "PAR1"
<Column 1 Chunk 1 + Column Metadata>
<Column 2 Chunk 1 + Column Metadata>
...
<Column N Chunk 1 + Column Metadata> // 행 그룹 1 (N 개의 Column Chunk)
<Column 1 Chunk 2 + Column Metadata>
<Column 2 Chunk 2 + Column Metadata>
...
<Column N Chunk 2 + Column Metadata> // 행 그룸 2 (N 개의 Column Chunk)
...
<Column 1 Chunk M + Column Metadata>
<Column 2 Chunk M + Column Metadata>
...
<Column N Chunk M + Column Metadata> // 행그룹 M (N 개의 Column Chunk)
File Metadata
4-byte length in bytes of file metadata
```

![1](https://user-images.githubusercontent.com/22383120/76186043-7761fd80-6214-11ea-9996-aef642944a09.PNG)

- 테이블 형태로 표현하면 위와 같다.

![2](https://user-images.githubusercontent.com/22383120/76186041-7630d080-6214-11ea-9288-3c848a4545e1.PNG)



- 꼬리말에 모든 Metadata 정보가 들어있으며, offset of first data page (page 시작 offset 정보)를 가지고 있음



![3](https://user-images.githubusercontent.com/22383120/76186042-7761fd80-6214-11ea-96ca-575167377d1a.PNG)

- Parquet에 필요한 속성들을 볼 수 있다.

- 블록 크기는 Scan 효율성과, 메모리 사용률에 trade off 관계이다.
  - 블록 크기가 크면, 더 많은 Row들을 가지며, IO 성능을 높여 `효율적인 Scan 이 가능`하다. 허나 `메모리 사용량은 증가`한다.
  - 블록 크기가 작아지면 반대
- 블록은 HDFS 블록에서 읽을수 있어야 하기 때문에 (?), 블록 크기가 HDFS 블록 크기보다 크면 안된다.



Mimumum/Maximum Filtering

- Parquet 2.0 이후부터, row group 별 min/max statistics 값을 가진다. 이를 통해 필요 없는 row group들을 제거할 수 있다.
- 만약 `SELECT * FROM TABLE WHERE A < 10` 의 Query가 요청되었을 때, 
  - 각 Row group이 가지는 min, max 값이랑 overlap이 되는 row group만 반환하게 된다.



Page Index

- Page가 어느 file에 위치하는지, 어느 row를 포함하는지, mim/max statistics 값 등에 관한 정보를 가진다.
- Page Index는 footer에 저장된다.

- Query 요청이 들어왔을때,
  - footer에 Page Index를 먼저 읽어, 수행할 Page들이 무엇인지 결정한다.
- Scan 요청이 들어왔을때,
  - footer를 읽지 않아도 된다. 어처피 다 읽어야 하기 때문에
- Page Index는 Sorted 데이터에서 제일 빠르게 동작한다. 



## ORC

Parquet와 마찬가지로 컬럼 기준 저장 포맷으로, Hive에 특화된 포맷. Impala, Pig, Spark 등 다른 쿼리 엔진에서 사용하기엔 부적합하다. (범용 스토리지 포맷이 아니다.)

- 초기 Hive는 TextFile, SequenceFile 포맷 성능을 높이기 위해 RCFile이 개발되었음.
- RCFile은 각 컬럼을 하나의 파일 묶음으로 만들었으나, 분산 처리된 데이터를 모으는 비용이 많이 들어 ORC 포맷이 개발되었다.



용어

- stripes: row data의 group을 stripes라 함. stripe의 기본 사이즈는 256MB 이다.
  - data: column 당 multiple stream로 구성되었다.
  - index: skipping row를 위해 필요하며, default skip은 10,000 row이다.
  - footer: stream location 정보를 가진다.

![6](https://user-images.githubusercontent.com/22383120/76189519-44bd0280-621e-11ea-96b2-7fb18be721b1.PNG)

- file footer: 보조 정보 (metadata ?)를 저장한다. 예를들어 `stripe location`이나 각 `column의 count, min, max, sub 정보 (통계 정보)`, `데이터 스키마 정보` 등을 가진다.

![5](https://user-images.githubusercontent.com/22383120/76189522-45ee2f80-621e-11ea-9b01-a486904fc165.PNG)

- postscript: 파일의 끝을 나타내며, compression parameter랑 compressied footer의 크기(?)를 가진다.



File Format

![4](https://user-images.githubusercontent.com/22383120/76189535-50102e00-621e-11ea-8366-d2253d487e80.PNG)

- stripes내 Row data들이 있으며, 각 Column의 Index data와 실제 data들이 저장된다.
  - Index data는 각 column의 min, max 값 (?), 각 column의 row position이 저장된다.
  - `Index data는 stripes 및 row group 선택에 사용되며 Query 응답에는 사용되지 않는다.`

- secondary key를 통해 table을 정렬하여, 실행 시간을 크게 줄일 수 있다.
  - 만약 primery partition은 date라 했을 때, state, zip code, last name 등으로 정렬을 할 수 있고 이를 통해 모든 state를 읽지 않고 원하는 state만 looking 할 수 있다.



ORC 속성

![7](https://user-images.githubusercontent.com/22383120/76189625-851c8080-621e-11ea-872a-7f2b7bd464f8.PNG)





### 참고

https://parquet.apache.org/documentation/latest/

https://blog.cloudera.com/speeding-up-select-queries-with-parquet-page-indexes/

https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ORC

https://118k.tistory.com/408
