---
title: "개발 환경 세팅기 5편 — 손으로 만든 API, 그리고 질문들"
date: 2026-07-15 16:00:00 +0900
categories: [개발일지, 세팅]
tags: [spring-boot, jackson, json, record, mybatis]
---

## 포트 문제 해결 후, 드디어

4편에서 포트 충돌을 잡고 나니 서버는 뜬다. 이제 진짜 API를 만들 차례다.

## 첫 컨트롤러

`controller` 패키지 만들고, `HelloController.java`:

```java
package com.kosbi.backend.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/api/hello")
    public String hello() {
        return "Hello, project!";
    }
}
```

`http://localhost:8080/api/hello` 열었더니 `Hello, kproject!`가 그대로 떴다. 별거 아닌 것 같은데 묘하게 뿌듯했다 — 오늘 하루 종일 환경 세팅만 하다가 처음으로 "내가 짠 코드"가 화면에 나온 순간.

## 근데 이게 정확히 왜 여기서 실행되는 거지?

`BackendApplication.java`의 ▶ 버튼을 눌러서 실행했는데, 왜 하필 이 클래스인지 궁금해졌다. 답은 Java 자체의 규칙이었다 — `public static void main`이 있는 클래스가 프로그램의 시작점이라는 건 Spring이랑 상관없는 Java 기본 규칙이고, `@SpringBootApplication` 애노테이션이 "여기서부터 하위 패키지 전부 스캔해라"를 의미해서 우리가 만든 `HelloController`가 어디 등록도 안 했는데 자동으로 잡힌 거였다.

그리고 이게 로컬 전용 방식이라는 것도 다시 확인했다. 실제 배포는 `./gradlew build`로 JAR 하나로 뭉친 다음, 그걸 Docker 이미지로 포장해서 서버에서 `docker compose up`으로 띄우는 흐름이 될 거다. 근데 재밌는 건 — 로컬이든 서버든 **실행되는 건 결국 똑같은 `BackendApplication.main()`**이라는 것. 누가 눌러주냐만 다르다 (IntelliJ vs Docker).

## 문자열 대신 JSON으로

실제 API는 문자열이 아니라 구조화된 데이터를 돌려줘야 한다. Java 21의 `record` 문법을 처음 써봤다:

```java
@RestController
public class HelloController {

    record HelloResponse(String message) {}

    @GetMapping("/api/hello")
    public HelloResponse hello() {
        return new HelloResponse("Hello, project!");
    }
}
```

브라우저에 `{"message":"Hello, project!"}`가 떴다. `record` 한 줄이 예전 Java였으면 생성자·getter·equals까지 다 손으로 써야 했던 걸 대신해준다는 게 신기했다.

## 근데 이 JSON 변환은 누가 하는 거지?

여기서부터 진짜 궁금해져서 한참 파고들었다. 정리하면 세 개가 나눠서 하는 일이었다:

1. **Spring Framework**(전자정부 때 쓰던 그 Spring MVC와 같은 뿌리)의 Web 모듈이 `@RestController`, `@ResponseBody` 같은 애노테이션과 "변환해서 내보내라"는 규칙만 정의한다.
2. 실제 변환 작업은 **Jackson**이라는, Spring 팀이 만든 게 아닌 완전히 별개의 서드파티 라이브러리가 한다.
3. **Spring Boot**가 이 둘을 자동으로 이어붙여준다 — Initializr에서 고른 "Spring Web"이 사실은 Spring Framework Web 모듈 + 내장 Tomcat + Jackson을 한 세트로 묶어놓은 거라, Jackson을 고른 적도 없는데 자동으로 딸려온 거였다.

그리고 이게 "무조건" 되는 것도 아니었다. 처음에 문자열을 리턴했을 때는 따옴표 없이 순수 텍스트로 나왔는데, `record`로 바꾸니 JSON으로 나왔다 — 문자열은 Jackson을 거치지도 않고 별도 변환기가 그냥 텍스트로 바로 내보내는 거였다. "객체를 리턴하면 JSON, 단순 문자열이면 그냥 텍스트"가 정확한 규칙이었다.

## 그러다 옛날 생각이 났다 — MyBatis

JSON 얘기를 하다 보니 "이거 XML이랑 뭐가 다르지" 싶어졌다. 찾아보니 JSON은 XML을 개선해서 만든 후속작이 아니라, 아예 다른 뿌리(자바스크립트 객체 문법)에서 나온 별개의 형식이었다. 다만 API 데이터 포맷의 표준 자리를 XML한테서 넘겨받은 건 맞았다 — 더 가볍고, 브라우저(JS)가 파싱 없이 그냥 바로 쓸 수 있어서.

그러다 문득 전자정부 시절 SQL을 XML 파일에 썼던 게 떠올랐다. MyBatis였다. `<select id="...">` 안에 SQL을 직접 쓰고, 자바 인터페이스 메서드랑 `id`로 연결되던 그 방식. 지금 우리가 넣은 Spring Data JPA는 정확히 반대 철학이다 — SQL을 내가 쓰는 게 아니라 Hibernate가 자바 객체(Entity)를 보고 알아서 SQL을 만들어준다. 다음에 첫 Entity를 만들 때 이 대비가 꽤 도움이 될 것 같다.

## 오늘 정리

- Java의 진입점 규칙, 로컬 실행과 서버 배포의 차이
- `record`로 보일러플레이트 없애기
- Spring Framework / Spring Boot / Jackson, 세 개가 나뉘어 있다는 것
- JSON이 "무조건"은 아니라는 것 (리턴 타입에 따라 갈림)
- JSON과 XML의 진짜 관계, 그리고 MyBatis와의 재회

## 다음 편 예고

이제 진짜 DB를 다루는 코드 — 첫 Entity와 Repository를 만들 차례다. MyBatis 경험이 있으니 비교하면서 이해하기로 했다.
