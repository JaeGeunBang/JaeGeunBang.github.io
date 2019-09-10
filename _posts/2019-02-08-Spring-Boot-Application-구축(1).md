---
layout: post
title: "Spring boot Application 구축 (1)"
categories:
  - Posts
tags:
  - springboot
last_modified_at: 2019-02-08T12:57:42+09:00
---

스프링 부트 프로젝트 생성 및 간단히 Rest Controller를 통한 웹 어플리케이션 개발을 해보자



### 개발 환경

VSCode



### STS 설치

VSCode에서 Spring Boot Extension Pack을 Install 한다.

설치 후 Spring Initializr를 통해스프링 부트 어플리케이션을 구축할 수 있다.



### 프로젝트 생성

1. ctrl + shift + p를 누르고 spring initalizr 입력
1. 언어 `Java`
2. group id `com.example`
3. artifact id `demo`
4. 스프링 부트 버전 `2.1.2`
6. 의존성 추가 
   - `	web`  톰캣, 스프링 MVC를 통해 웹 어플리케이션 개발을 위함
   - `jpa` 하이버네이트 등 JPA 를 위함
   - `h2` RDB는 H2를 사용함
   - `lombok` 기본 코드를 자동으로 포함하기 위한 annotation library
6. 폴더 선택



### 프로젝트 구조

![spring_1](https://user-images.githubusercontent.com/22383120/52461676-68dd8880-2bb3-11e9-970b-0e1da0fb2498.PNG)



### 코드 분석

**DemoApplication.java**

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}
```
`@SpringBootApplication`은 `@ComponentScan` `@EnableAutoConfiguration` `@Configuration` 이 포함되어 있다.

> @ComponentScan: @Component가 붙은 클래스 Bean을 찾아 어플리케이션 컨텍스트에 Bean을 등록한다.

> @EnableAutoConfiguration: 어플리케이션 컨텍스트를 만들 때 자동으로 설정하는 기능을 켠다. 만약 tomcat-embed-core.jar가 존재한다면, 자동으로 톰캣 서버가 setting 된다.

> @Configuration: JavaConfig 클래스임을 알리며, @Bean 애노테이션을 메서드에 붙여, 해당 메서드를 어플리케이션 컨텍스트에 등록한다.



### 간단한 실행

DemoApplication.java에 아래 메서드를 추가한다.
```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;

@RestController // @RestController 추가
@SpringBootApplication
public class DemoApplication {

	// @RequestMapping 추가
	@RequestMapping("/")
	String name() {
		return "Hello World";
	}

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

} 
```
위 코드 추가 후 F5(디버깅 실행)를 눌러 실행하면, Debug Console 창에 스프링 부트 어플리케이션이 실행됨을 볼 수 있고, 그 중 `Tomcat started on port(s): 8080 (http) with context path ''` 로그를 볼 수 있다.

처음 의존성 추가 때 선택했던 `web`에 톰캣 서버가 내장되어 있기 때문에 자동으로 8080 포트로 톰켓 서버가 구동됨을 볼 수 있다.

이 후 웹 브라우저에서 localhost:8080을 통해 "Hello World"가 출력됨을 볼 수 있다.



### 코드
[https://github.com/JaeGeunBang/springboot_sts_tutorial/tree/master/spring_boot_application_2](https://github.com/JaeGeunBang/springboot_sts_tutorial/tree/master/spring_boot_application_2)
