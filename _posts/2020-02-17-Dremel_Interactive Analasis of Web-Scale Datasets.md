```
layout: post
title: "Dremel_Interactive Analasis of Web-Scale Datasets"
categories:
  - Posts
tags:
  - bigquery
  - paper
  - dremel
last_modified_at: 2020-02-17T12:57:42+09:00
```



Nested Columnar Storage

Repetition and Definition Level



Repetition Level

- value의 반복되는 'level'을 알기 위함

Definition Level

- value의 Missing의 정도(?)를 알기 위함

![1](https://user-images.githubusercontent.com/22383120/74655680-196f6680-51d0-11ea-8cba-96e1c14a9604.PNG)

![2](https://user-images.githubusercontent.com/22383120/74655684-1aa09380-51d0-11ea-801b-6716b8934402.PNG)

예제 (그림과 같이 설명)

- Repetition Level
  - new record는 0으로 시작한다. 즉, 0이면 새로운 Record를 의미한다.
  - Name.Language.Code
    - `en-us`는 record 시작을 알린다 (Repetition Level 0)
    - `en`는 1번째 Name은 Name, Language 까지 두 번 repeated 해야 알 수 있다. (Repetition Level 2)
    - `NULL`은 2번째 Name은 Name만 repeated 해도 알 수 있다. (Repetition Level 1)
    - `en-gb`1은 3번째 Name은 Name만 repeated 해도 알 수 있다. (Repetition Level 1)
    - `NULL`은 record 시작을 알린다 (Repetition Level 0)
  - Name.Forward
    - `20`는 record 시작을 알린다 (Repetition Level 0)
    - `40`은 1번째 Links는 Links만 repeated 해도 알 수 있다. (Repetition Level 1)
    - `60`은 1번째 Links는 Links만 repeated 해도 알 수 있다. (Repetition Level 1)
    - `80`는 record 시작을 알린다. (Repetition Level 0)
- Definition Level
  - Name.Language.Code
    - `en-us`는 Name.Language.Code 3개가 정의되어 있다.
      - 허나 Code는 Optional, repeated가 아니므로 세지 않아 **Definition Level 2**로 본다.
      - **optional, repeated는 Missing value가 들어가기 때문에 Definition Level을 세야 한다.**
      - **Required는 무조건 1개의 value가 들어가야 하기 때문에 Definition Level을 세지 않는다.**
    - `en`은 Name.Languega.Code 3개가 정의되어 있다.
    - `NULL`은 Name만 정의되어 있다. (Definition Level 1)
    - `en-gb`는 Name.Language.Code 3개가 정의되어 있다. (Definition Level 3)
    - `NULL`은 Name만 정의되어 있다. (Definition Level 1) 
  - Name.Languege.Country는 Name.Language.Code와 Definition Level이 NULL이 아닌 value는 1이 더 높은데, Country가 optional, repeated 이기 때문에 Definition Level을 3으로 본다.
  - Links.Backward
    - `NULL`은 Links만 정의되어 있다. (Definition Level 1)
    - `10, 30`은 Links,Backward 2개가 정의되어 있다. (Definition Level 2)