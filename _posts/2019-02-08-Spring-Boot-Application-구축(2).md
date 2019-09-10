---
layout: post
title: "Spring boot Application 구축 (2)"
categories:
  - Posts
tags:
  - springboot
last_modified_at: 2019-02-08T12:57:42+09:00
---

[Spring boot Application 구축(1)](https://jaegeunbang.github.io/web.springboot/Spring-Boot-Application-%EA%B5%AC%EC%B6%95(1))

앞서 구축한 프로젝트를 바탕으로 H2 DB를 연동해보자.

## 프로젝트 구조
아래 폴더와 java 파일을 추가한다.

![spring_2](https://user-images.githubusercontent.com/22383120/52461702-86aaed80-2bb3-11e9-9e7b-8720b4fa57ba.PNG)

> config: 어플리케이션의 config를 정의한다. (여기선 DB Console 경로를 설정한다.)

> domain: 하이버네이트를 통해 db에 table을 생성한다. (DDL)

> repository: 하이버네이트를 통해 db에 데이터를 조작한다. (DML)

> restController: 외부 client에 Rest API 요청을 받는다.

> service: 서비스에 필요한 작업을 처리한다.

이번 글에선 config, domain 까지 설정을 해보자

## H2 DB 셋팅
### config
DBConfig.java는 아래 코드와 같다.

```java
package com.example.demo.config;

import org.h2.server.web.*;
import org.springframework.boot.web.servlet.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DBConfig {
	@Bean
	public ServletRegistrationBean<?> h2servletRegistration() {
        ServletRegistrationBean<?> registration = new ServletRegistrationBean<>(new WebServlet());
		registration.addUrlMappings("/console/*");
		return registration;
	}
}
```
> @Configuration, @Bean을 통해 h2servletRegistration() 메서드를 어플리케이션 컨텍스트가 관리한다. 해당 메서드를 등록 후 /console/로 접근하면 DB Console에 접근할 수 있다.

### resources/application.yml
application.properties를 application.yml로 바꾼 후 아래와 같이 설정한다.
```java
spring:
    datasource:
        driverClassName: org.h2.Driver
        url: jdbc:h2:~/test;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE;AUTO_RECONNECT=TRUE
        username: sa
        password:
    jpa:
        hibernate:
            ddl-auto: update
```
DB 접속에 필요한 정보이며, H2 DB는 기본 In-memory DB이기 때문에 영속성이 없는데, 위 설정을 통해 영속성을 가질 수 있다.

### domain
Customer.java는 아래 코드와 같다.
```java
package com.example.demo.domain;

import javax.persistence.*;
import lombok.*;

@Entity
@Table (name="customer")
@Data
@NoArgsConstructor @AllArgsConstructor
public class Customer{
    @Id
    @GeneratedValue
    private Integer id;

    @Column(nullable = false)
    private String firatName;

    @Column(nullable = false)
    private String lastName; 
}
```
> @Entity: 엔티티임을 알리는 애노테이션이다.
> @Table: 엔티티에 대응하는 테이블 이름을 지정한다. (table: customer)

>@Data: 추가한 의존성인 lombok이 제공하는 에노테이션으로, 각 필드의 getter / setter와 toString(), equals(), hashCode() 메서드가 자동으로 생성된다.
>@NoArgsContructor, AllArgsConstructor: 생성자를 자동으로 생성해주는 에노테이션이다.

> @Id: 해당 필드가 기본키 임을 알린다.
> @GeneratedValue: 기본 키 번호를 자동으로 매긴다.
> @Column: 객체의 필드와 테이블 칼럼의 이름이나 제약 조건을 설정한다.

위 Customer.java를 구현하면 스프링 부트 어플리케이션 실행 시, 해당 객체를 자동으로 테이블로 생성해준다.

스프링 부트 어플리케이션을 실행 후 localhost:8080/console 로 접속하면 DB Console을 확인할 수 있다.

> **Note**: lombok을 추가하고 getter, setter를 호출 할 수 없는 경우가 있는데, 이는 VSCode에서 Lombok Annotations Support for VS Code 를 추가해주어야 한다.

## H2 DB 콘솔 접속
로그인 화면

![spring_3](https://user-images.githubusercontent.com/22383120/52461711-91658280-2bb3-11e9-973d-c3f5ee1ea86a.PNG)

H2 DB Console 화면

![spring_4](https://user-images.githubusercontent.com/22383120/52461719-9aeeea80-2bb3-11e9-9146-8248126bae77.PNG)

생성된 customer 테이블을 볼 수 있다. 
현재 테이블엔 아무런 값도 없기 때문에 insert를 통해 데이터를 입력한다.
그 후 스프링 부트 어플리케이션을 재시작 후에 테이블에 값이 있는지 확인한다.


### 코드
[https://github.com/JaeGeunBang/springboot_sts_tutorial/tree/master/spring_boot_application_2](https://github.com/JaeGeunBang/springboot_sts_tutorial/tree/master/spring_boot_application_2)
