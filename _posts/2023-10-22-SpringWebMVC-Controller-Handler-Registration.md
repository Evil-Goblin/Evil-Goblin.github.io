---
title: SpringWebMVC Controller Handler 등록
author: Evil-Goblin
date: 2023-10-22 00:32:00 +0900
categories: [Spring, SpringWebMVC]
tags: [Spring, SpringWebMVC, SpringBoot3, Controller, RequestMapping]
math: true
mermaid: true
---
## Controller Handler 등록
기존 `Spring` 의 경우 `@Controller` 또는 `@RequestMapping` 어노테이션이 등록되어 있어야 컨트롤러로 등록되었다.

하지만 사용 중이던 `SpringBoot3.1.1` 에서 `@RequestMapping` 을 통해 컨트롤러로 등록하려 하였더니 컨트롤러 핸들러가 하나도 등록되지 않는 문제가 발생하였다.

이에 무엇이 문제인지 확인해보니 `SpringBoot3` 이상부터는 `@RequestMapping` 이 컨트롤러 인식 조건문에서 제외되었다는 것을 알게 되었다.
- [커밋](https://github.com/spring-projects/spring-framework/commit/3600644ed1776dce35c4a42d74799a90b90e359e), [이슈](https://github.com/spring-projects/spring-framework/issues/22154)
![커밋내역](https://github.com/Evil-Goblin/springio-guide/assets/74400861/686b3352-d971-493a-be17-0a13dac46eca)

### 결론
- `SpringBoot3` 이상부터는 `@Controller` 어노테이션을 통해서만 핸들러 등록이 가능하다.
