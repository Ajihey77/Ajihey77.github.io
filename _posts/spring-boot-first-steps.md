---
title: "개발 환경 세팅기 3편 — Spring Boot 첫걸음, 그리고 뭔가 잘못됐다"
date: 2026-07-15 22:23:00 +0900
categories: [개발일지, 세팅]
tags: [spring-boot, gradle, intellij, jpa, flyway]
---

## 드디어 코드다

지금까지는 Docker, GitHub, 블로그 세팅 같은 주변 정리였고, 이번에야 진짜 프로젝트 코드를 만들기 시작했다. 백엔드 뼈대부터.

## Spring Initializr — 프로젝트 뼈대 자동 생성기

IntelliJ 내장 마법사 대신 **start.spring.io**를 웹으로 열어서 생성했다. IntelliJ가 최근에 Community/Ultimate 통합판으로 바뀌면서 메뉴가 달라졌을 수 있어서, UI가 안정적인 웹 쪽을 택했다.

설정:

| 항목 | 값 |
|---|---|
| Project | Gradle - Groovy |
| Language | Java 21 |
| Group / Artifact | com. / backend |
| Packaging | Jar |

의존성은 딱 5개 + Actuator만 골랐다: **Spring Web, Spring Data JPA, Validation, PostgreSQL Driver, Flyway, Spring Boot Actuator**. Security는 일부러 뺐다 — 처음부터 넣으면 모든 요청이 401 떠서 디버깅만 힘들어진다고 해서.

## WAR가 아니라 Jar인 이유

전자정부프레임워크 할 때는 항상 WAR였는데, 이번엔 Jar를 골랐다. 차이를 찾아보니:

- **WAR**: 이미 켜져 있는 Tomcat 서버 "안에" 배포하는 방식. 서버(Tomcat)와 앱이 분리되어 있고, 운영팀이 관리하는 공용 서버에 여러 부서 WAR가 같이 얹히는 구조 — 공공기관에서 흔한 모델.
- **Jar**: 서버가 통째로 내장된 실행 파일. `java -jar` 한 줄로 뜬다.

우리 배포 계획이 EC2 + Docker Compose인 게 결정적이었다. Docker는 "컨테이너 하나 = 독립 앱 하나"가 기본 철학인데, 이게 Jar 방식과 정확히 맞아떨어진다. WAR로 갔으면 컨테이너 안에 Tomcat까지 따로 설치해야 하는 번거로움이 하나 더 생겼을 거다.

## properties vs yml — 어제 데인 사람의 선택

`application.properties`(기본)로 갈지 `application.yml`로 바꿀지 잠깐 고민했는데, 둘이 담는 정보는 완전히 같고 문법만 다르다는 걸 알고 나서 바로 결정했다. **properties로 간다.** 어제 `docker-compose.yml` 들여쓰기 때문에 한참 씨름했던 사람이 오늘 또 yml을 늘리는 건 리스크 관리가 안 되는 선택이라고 판단했다.

## Gradle 동기화 — 첫 관문

압축 푼 `backend` 폴더를 IntelliJ로 열었더니 우측 하단에서 Gradle 동기화가 자동으로 돌았다. 2~3분 정도 걸렸지만 별문제 없이 통과.

## DB 연결 설정 후 실행 — 어라?

`application.properties`에 이 세 줄 추가:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/
spring.datasource.username=
spring.datasource.password=
```

Docker로 띄워둔 DB 정보 그대로다. `BackendApplication.java`의 ▶ 버튼을 누르니 Spring Boot 배너까지는 잘 뜨는데, 곧바로 에러가 터졌다:

```
Caused by: org.postgresql.util.PSQLException: (한글 메시지, 인코딩 깨짐)
SQL State  : 28P01
```

`28P01`을 찾아보니 PostgreSQL의 표준 에러코드로 **"비밀번호 인증 실패"**란다. 근데 이 비밀번호, `docker-compose.yml`에 내가 직접 넣은 것과 토씨 하나 안 틀리고 같은데?

껐다 켜봐도, 파일을 다시 확인해도 똑같았다. Actuator가 자동으로 DB 상태까지 검사해준다길래 기대하고 있었는데, 그 전에 애플리케이션 자체가 뜨지도 못하는 상황.

## 다음 편에 계속

여기서 막혔다. 파일은 분명 맞는데 왜 거부당하는지, 다음 편에서 추적기를 이어간다.
