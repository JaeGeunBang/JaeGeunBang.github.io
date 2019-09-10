---
layout: post
title: "Spring boot Application 구축 (3)"
categories:
  - Posts
tags:
  - springboot
last_modified_at: 2019-02-08T12:57:42+09:00
---

[Spring boot Application 구축(1)](https://jaegeunbang.github.io/posts/2019/02/08/Spring-Boot-Application-구축(1).html)

[Spring boot Application 구축(2)](https://jaegeunbang.github.io/posts/2019/02/08/Spring-Boot-Application-구축(1).html)

앞서 구축한 프로젝트를 바탕으로 Rest API를 받아보자.



### 프로젝트 구조

![spring_2](https://user-images.githubusercontent.com/22383120/52461702-86aaed80-2bb3-11e9-9e7b-8720b4fa57ba.PNG)

> ~~config: 어플리케이션의 config를 정의한다. (여기선 DB Console 경로를 설정한다.)~~

> ~~domain: 하이버네이트를 통해 db에 table을 생성한다. (DDL)~~

> repository: 하이버네이트를 통해 db에 데이터를 조작한다. (DML)

> restController: 외부 client에 Rest API 요청을 받는다.

> service: 서비스에 필요한 작업을 처리한다.




### repository
```java

package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import com.example.demo.domain.*;
import java.util.*;

@Repository
public interface CustomerRepository extends JpaRepository<Customer, Integer> {
    @Query(value = "SELECT * FROM customer x ORDER BY x.first_name", nativeQuery = true)
    List<Customer> getAllCustomer() ;
}
```
> @Repository: repository 클래스임을 알린다. Repository는 JpaRepository 클래스를 상속 받아 사용하며, 기본 CRUD 조작용 메서드를 사용할 수 있다.
findAll(), saveAll(), deleteAll() 등

> @Query: SQL문을 통해 테이블에서 데이터를 조작할 수 있다. 옵션으로 nativeQuery =  true로 함으로써 순수한 SQL문을 사용할 수 있다.
>
> > 디폴트는 JPQL이다. (nativeQuery = true를 지우면 된다.)



### JPQL, SQL 차이는?
JPQL은 객체를 기준으로 데이터를 조작하고, SQL은 테이블을 기준으로 데이터를 조작한다.

 **만약 JPQL을 쓴다면, SQL문에 정의된 테이블 이름, 필드 이름은 객체에 선언된 클래스명, 필드명을 써야한다.**

> @Query(value =  "SELECT x FROM **Customer** x ORDER BY **x.firstName**")
> Customer와 firstName은 객체의 이름과 필드명이다. 



JPQL을 사용하면 사용자가 DB에 직접 조작하는 것 없이, 객체 작업만으로도 DB를 조작할 수 있기 때문에 SQL문을 사용하는 것보다 JPQL을 쓰는 것이 좋다.

(이를 가능하게 해주는 기술이 `Hibernate`이다.)



해당 예제에서는 간단한 예제라 순수 SQL문을 써봤다.



### service

service.java는 아래와 같다.

```java

package com.example.demo.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.example.demo.repository.*;
import com.example.demo.domain.*;

import java.util.*;

@Service
public class CustomerService{
    @Autowired
    CustomerRepository customerRepository ;
    public List<Customer> getAllCustomer(){
        return customerRepository.getAllCustomer();
    }
}
```
> @Service: 해당 클래스가 service 클래스임을 알린다.
> @Autowired: 의존성 주입(DI)을 위해 사용한다. 
> 즉, CustomerService와 CustomerRepository의 의존 관계가 생성되며, 이는 스프링의 어플리케이션 컨텍스트가 객체의 생명 주기와 의존 관계를 관리한다.



### restController

CustomerRestController.java는 아래와 같다.

```java
package com.example.demo.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import com.example.demo.service.*;
import com.example.demo.domain.*;
import java.util.*;

@RestController
@RequestMapping("api")
public class CustomerRestController {
    @Autowired
    CustomerService customerService;

    @RequestMapping(value="/customer", method=RequestMethod.GET)
    public List<Customer> getAllCustomer() {
        return customerService.getAllCustomer() ;
    }
}
```
> @RestController: client로 부터 Rest API 요청을 받기 위한 클래스임을 알린다 (=서블릿 클래스)

> @RequestMapping: 
> 1. 해당 애노테이션이 클래스에 붙어있다면 클래스 내 있는 메서드는 모두 정의된 Path로 접근할 수 있다. (localhost:8080/api)
> 2. 해당 에노테이션이 메서드에 붙어있다면, 클래스에 붙은 Path와 더해진 Path로 접근할 수 있다. (localhost:8080/api/customer)



## Rest API 확인

Rest API 요청을 확인하기 위해 Postman을 사용했다. (그냥 브라우저에 입력해도 된다.)

URL: localhost:8080/api/customer

![spring_5](https://user-images.githubusercontent.com/22383120/52464375-de4e5680-2bbd-11e9-805c-e6a39eb7939b.PNG)



### 코드

[https://github.com/JaeGeunBang/springboot_sts_tutorial/tree/master/spring_boot_application_2](https://github.com/JaeGeunBang/springboot_sts_tutorial/tree/master/spring_boot_application_2)
