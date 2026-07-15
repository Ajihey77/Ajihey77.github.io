---
title: "개발 환경 세팅기 4편 — 비밀번호는 맞는데 왜 안 될까"
date: 2026-07-15 11:00:00 +0900
categories: [개발일지, 트러블슈팅]
tags: [postgresql, docker, networking, spring-boot]
---

## 3편 요약

Spring Boot를 처음 실행했더니 `SQL State: 28P01`(비밀번호 인증 실패)로 죽었다. `docker-compose.yml`이랑 `application.properties`의 계정/비번을 몇 번을 대조해봐도 완전히 똑같은데.

## 용의선상에서 하나씩 지우기

**1. 파일에 안 보이는 문자가 있나?** — `cat -A`로 줄 끝 확인, 바이트 수까지 세봤다. 트레일링 공백도, CRLF도 없이 깨끗했다.

**2. 환경변수가 덮어쓰고 있나?** — `SPRING_DATASOURCE_*` 관련 환경변수 검색, 아무것도 없음.

**3. Gradle이 옛날 빌드를 쓰고 있나?** — `build/resources/main/application.properties`(실제 실행에 쓰이는 복사본)까지 열어봤는데 소스랑 완전히 동일. 최신 상태 맞음.

**4. IntelliJ만의 문제인가?** — 아예 터미널에서 `./gradlew bootRun`으로 직접 돌려봤다. 똑같이 실패. IntelliJ 탓은 아니라는 뜻.

여기까지 확인했는데도 안 되니 슬슬 이상해졌다. 결정적으로, **컨테이너 안에 직접 들어가서(`docker exec`) 같은 비밀번호로 접속하면 멀쩡히 성공**했다. 그러니까 비밀번호 자체는 확실히 맞다. 그런데 Spring Boot(내 컴퓨터, 즉 컨테이너 "밖")에서 접속하면 거부당한다.

## 진짜 원인 — 포트를 다른 애가 이미 쓰고 있었다

컨테이너 "안"에서는 되고 "밖"에서는 안 된다는 게 힌트였다. Windows에서 실제로 5432 포트를 누가 물고 있는지 확인해보니:

```
Get-Service | Where-Object { $_.Name -match "postgres" }

Name              DisplayName                          Status
----              -----------                           ------
postgresql-x64-12 PostgreSQL Server 12                  Running
postgresql-x64-16 PostgreSQL Server 16                  Running
```

로컬에 PostgreSQL이 **두 버전이나 이미 서비스로 설치되어 실행 중**이었다. 이 존재를 완전히 잊고 있었다. `docker compose ps`는 분명 "5432 포트 열었다"고 보고했지만, Windows에서 `localhost:5432`로 접속하면 실제로는 먼저 자리 잡고 있던 이 네이티브 PostgreSQL로 연결되고 있었던 거다. 거기엔 당연히 계정이 없으니 인증이 거부된 것 — Docker가 거짓말을 한 게 아니라, 같은 포트를 두고 실제로는 다른 프로그램이 응답하고 있었던 상황.

## 포트를 바꿨는데도 또 실패

"그럼 포트만 바꾸면 되겠네" 하고 `5433`으로 바꿨다. `docker-compose.yml`이랑 `application.properties`둘 다 수정하고 재실행 — **또 똑같은 에러.**

이번엔 5433도 확인해보니:

```
Get-NetTCPConnection -LocalPort 5433 -State Listen
LocalAddress  PID  ProcessName
------------  ---  -----------
0.0.0.0       8132 postgres
```

또 postgres 프로세스였다. 로컬에 PostgreSQL 두 버전(12, 16)을 같이 설치하면, 설치 프로그램이 서로 충돌 안 나게 자동으로 **하나는 5432, 하나는 5433에 배정**해버린다는 걸 이때 알았다. 하필 그 두 포트를 이미 둘 다 선점하고 있던 것.

## 이번엔 확인부터 하고 바꿨다

같은 실수를 세 번 하지 않으려고, 포트를 정하기 전에 먼저 비어있는지부터 확인했다.

```
Get-NetTCPConnection -LocalPort 15432 -State Listen   # 아무것도 안 뜸 = 비어있음
```

`15432`로 확정하고 `docker-compose.yml`, `application.properties` 반영 → 컨테이너 재생성 → 실행:

```
HikariPool-1 - Added connection ...
Schema "public" is up to date. No migration necessary.
Started BackendApplication in 3.43 seconds
```

브라우저로 `http://localhost:8080/actuator/health` 열어보니:

```json
{"status":"UP"}
```

드디어.

## 배운 것 — 포트 번호는 그냥 숫자다

이번 일로 제일 크게 느낀 건, 포트 번호가 어디 미리 정해진 목록에서 골라야 하는 게 아니라 **그냥 0~65535 사이 숫자 중 아무거나(단, 이미 안 쓰이는 것만)** 써도 된다는 거였다. `5432`가 PostgreSQL 기본값인 것도 그냥 관습이지 법칙이 아니다. 앞으로는 새 포트를 정할 때 감으로 찍지 말고 `Get-NetTCPConnection`으로 먼저 확인하는 습관을 들이기로 했다.

그리고 `docker-compose.yml`의 `"15432:5432"`에서 오른쪽(컨테이너 내부 포트)은 안 건드렸다는 것도 다시 짚을 만하다 — 컨테이너 안은 독립된 세계라 거기서는 그냥 5432를 그대로 써도 아무와도 안 겹친다. 우리가 바꾼 건 바깥에서 그 방으로 들어가는 입구 번호뿐이다.

## 다음 편 예고

환경 문제는 해결됐으니, 이제 진짜 손으로 첫 API 컨트롤러를 써볼 차례다.
