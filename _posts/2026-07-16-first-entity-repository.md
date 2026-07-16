---
title: "개발 환경 세팅기 6편 — 첫 CRUD, 그리고 의존성 주입이 뭔지 알게 된 날"
date: 2026-07-16 12:00:00 +0900
categories: [개발일지, 세팅]
tags: [spring-boot, jpa, flyway, hibernate, dependency-injection]
---

## 5편 요약

첫 API(`/api/hello`)까지는 문자열이 왔다 갔다 하는 수준이었다. 5편 끝에서 예고한 대로, 이번엔 진짜 DB를 다루는 코드 — Entity와 Repository를 만들 차례였다.

## JPA가 알아서 테이블을 만들어주는 거 아니었어?

Entity를 만들기 전에 먼저 걸린 게 있었다. 지금 프로젝트 설정을 보니 `ddl-auto`가 `none`으로 되어 있었다. "하이버네이트가 Entity 보고 알아서 다 해준다며, 근데 왜 스키마 자동 생성은 꺼놨지?"

답은 **Flyway**였다. Spring Initializr에서 고를 때부터 넣어뒀던 의존성인데, 그게 정확히 이 역할이었다. Hibernate의 자동 스키마 생성(`ddl-auto=update`나 `create`)은 편하지만 "지금 운영 DB에 정확히 뭐가 어떻게 바뀌었는지"를 코드로 추적할 수가 없다. Flyway는 그 반대다:

- SQL 마이그레이션 파일을 `V1__설명.sql`, `V2__설명.sql` 식으로 순서대로 쌓는다
- 앱이 뜰 때마다 아직 안 돌린 파일이 있으면 순서대로 실행한다
- 실행한 이력을 DB 안의 `flyway_schema_history` 테이블에 직접 남긴다
- **되돌리기가 없다.** 잘못 만들었으면 되돌리는 마이그레이션이 아니라 "고치는" 새 마이그레이션을 또 만들어서 앞으로 나가야 한다

Git이 코드 변경 이력을 커밋으로 남기듯, Flyway는 스키마 변경 이력을 파일로 남긴다는 느낌으로 이해하니 확 와닿았다.

## 첫 마이그레이션

`V1__create_app_user.sql`:

```sql
CREATE TABLE app_user (
    user_id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email        text NOT NULL UNIQUE,
    display_name text NOT NULL,
    role         text NOT NULL DEFAULT 'proposer'
                CHECK (role IN ('proposer', 'reviewer', 'admin')),
    created_at   timestamptz NOT NULL DEFAULT now()
);
```

앱 실행 후 `docker exec`로 컨테이너에 직접 들어가서 확인했다:

```
flyway_schema_history  →  version 1, "create app user", success = t
\d app_user             →  컬럼 구조 정확히 일치
```

## Entity — 근데 이거 record로 만들면 안 돼?

`record`가 그렇게 편하다고 감탄해놓고, 당연히 Entity도 record로 만들려고 했다. 안 됐다.

이유를 찾아보니 Hibernate가 객체를 다루는 방식 자체가 record의 불변성과 충돌했다. Hibernate는 DB에서 값을 읽어온 뒤 필드를 하나씩 채워넣고("생성 후 채우기"), 나중에 값이 바뀌면 그 변화를 감지해서("dirty checking") 자동으로 UPDATE 쿼리를 만든다. 둘 다 필드가 나중에 바뀔 수 있어야(mutable) 가능한 동작인데, record는 애초에 "한번 만들면 절대 안 바뀐다"는 게 존재 이유라 정반대다.

그래서 결론: **DTO처럼 한 번 담고 버릴 데이터는 record, Hibernate가 계속 들고 관리해야 하는 건 일반 클래스.**

```java
@Entity
@Table(name = "app_user")
public class AppUser {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "user_id")
    private Long userId;

    @Column(name = "email", nullable = false, unique = true)
    private String email;

    @Column(name = "display_name", nullable = false)
    private String displayName;

    @Column(name = "role", nullable = false)
    private String role;

    protected AppUser() {}

    public AppUser(String email, String displayName, String role) {
        this.email = email;
        this.displayName = displayName;
        this.role = role;
    }

    public Long getUserId() { return userId; }
    public String getEmail() { return email; }
    public String getDisplayName() { return displayName; }
    public String getRole() { return role; }
}
```

`created_at`은 일부러 안 넣었다. DB의 `DEFAULT now()`가 알아서 채워주니, 여기서 또 값을 강제하면 오히려 충돌만 생긴다.

## Repository — 구현체를 한 줄도 안 썼는데 동작한다

```java
public interface AppUserRepository extends JpaRepository<AppUser, Long> {
    Optional<AppUser> findByEmail(String email);
}
```

이게 끝이다. `interface`라 원래 구현이 없어야 정상인데, `JpaRepository`를 상속받는 것만으로 `save()`, `findAll()`, `findById()` 같은 기본 CRUD가 전부 공짜로 생긴다. 심지어 `findByEmail`처럼 내가 이름만 지어낸 메서드도, Spring Data JPA가 메서드 **이름 자체를 해석**해서 `WHERE email = ?` 쿼리를 알아서 만들어준다. 구현 코드는 어디에도 없다 — 실행 시점에 스프링이 동적으로 만들어서 끼워넣는다.

## Controller에서 만난 벽 — 의존성 주입(DI)

```java
@RestController
@RequestMapping("/api/users")
public class AppUserController {
    private final AppUserRepository appUserRepository;

    public AppUserController(AppUserRepository appUserRepository) {
        this.appUserRepository = appUserRepository;
    }

    record CreateUserRequest(String email, String displayName, String role) {}

    @PostMapping
    public AppUser createUser(@RequestBody CreateUserRequest request) {
        AppUser user = new AppUser(request.email(), request.displayName(), request.role());
        return appUserRepository.save(user);
    }

    @GetMapping
    public List<AppUser> listUsers() {
        return appUserRepository.findAll(); 
    }
}
```

생성자에 `AppUserRepository`를 파라미터로 받는 부분에서 한참 막혔다. `new`가 어디에도 없는데 어떻게 동작하는지 감이 안 왔다. 몇 번을 다시 물어보고서야 정리됐다:

- 컨트롤러가 HTTP 요청을 받을 때 그 순간 Repository를 만드는 게 아니다. **앱이 켜지는 순간, 요청이 오기 한참 전에** 스프링이 필요한 객체들을 미리 다 만들어서 조립해둔다.
- 컨트롤러 코드는 "나는 이 타입이 필요하다"고 선언만 할 뿐, 그걸 누가 어떻게 만드는지는 몰라도 된다.
- 이렇게 "쓰는 쪽"과 "만드는 결정"을 분리하는 것 자체가 **의존성 주입(DI)**이다 — 정확히는 그 결과물(객체)이 아니라, 이미 만들어진 걸 건네주는 **행위**를 가리키는 말이다.

신입사원이 입사 서류에 "노트북 필요합니다"라고만 적어두면, 인사팀이 출근 첫날 이미 준비해둔 노트북을 건네주는 것과 같은 그림이다. 신입사원은 노트북을 직접 조립하지 않고, 그 노트북도 "그때그때"가 아니라 "출근 첫날 딱 한 번" 받는다.

## eGovFrame이랑 비교하니 오히려 더 잘 이해됐다

전자정부 할 때 봤던 구조랑 나란히 놓아보니 확 정리됐다.

| eGovFrame (MyBatis)                     | 지금 (Spring Boot + JPA)                              |
| --------------------------------------- | ----------------------------------------------------- |
| Controller → Service → DAO → Mapper.xml | Controller → Repository (SQL 없음)                    |
| `context-*.xml`의 `<bean>`              | 애노테이션 스캔                                       |
| SQL을 XML에 직접 작성                   | Hibernate가 자동 생성                                 |
| `@Controller` + 뷰 이름 리턴            | `@RestController` (= `@Controller` + `@ResponseBody`) |

제일 헷갈렸던 부분: XML의 `<bean>` 설정도 사실 이미 DI였다는 것. "XML(하드코딩) → 애노테이션(DI)"이 아니라, "직접 `new`(DI 아님) → XML이든 애노테이션이든 바깥에서 결정해서 건네줌(둘 다 DI, 표기법만 다름)"이 맞는 구분이었다.

## 실행하니 두 개나 터졌다

**1. DB 연결 거부**

```
Connection to localhost:15432 refused. Check that the hostname and port
are correct and that the postmaster is accepting TCP/IP connections.
```

Docker 컨테이너가 꺼져 있었다. Docker Desktop 켜고 Containers 탭에서 재생 버튼 누르니 해결. 볼륨은 그대로 남아있어서 기존 테이블도 안 날아갔다.

**2. `emial`**

```
column au1_0.emial does not exist
Hint: Perhaps you meant to reference the column "au1_0.email".
```

`@Column(name = "emial")` — 오타였다. 이게 재밌는 지점인데, 이런 오타는 **컴파일 타임엔 절대 안 걸린다.** `@Column(name = "...")`은 그냥 문자열이라, 자바 컴파일러 입장에선 `"emial"`이든 `"email"`이든 완전히 유효한 코드다. 진짜 쿼리가 DB에 날아가는 런타임에야 비로소 걸린다. 앱은 멀쩡히 켜지고, `/api/users`를 호출한 순간에만 터지는 이유였다.

## 검증 — 근데 왜 DB에 직접 안 넣고 이렇게 해?

두 버그를 잡은 뒤 PowerShell로 직접 요청을 보내 확인했다:

```powershell
Invoke-RestMethod -Uri "http://localhost:8080/api/users" -Method Post `
  -ContentType "application/json" `
  -Body '{"email":"test@example.com","displayName":"테스트","role":"proposer"}'
```

`userId`가 채워진 응답이 왔고, 브라우저로 GET을 다시 열어보니 방금 만든 유저가 배열에 들어있었다. Entity → Repository → Controller → DB, 전체 경로가 실제로 연결됐다는 증거였다.

그런데 문득 궁금해졌다 — DB에 직접 `INSERT` 하면 더 빠른데 왜 굳이 API로 넣지? 답은 "지금 확인하려는 게 DB가 아니라 **우리 코드**"라는 거였다. SQL로 직접 넣으면 Entity, Repository, Controller를 전부 건너뛰기 때문에 아무것도 검증이 안 된다 (`emial` 오타도 SQL로 직접 넣었으면 절대 못 잡았을 거다 — DB 컬럼은 멀쩡하니까). 그리고 이게 앞으로도 계속 이런 식이다. Next.js 프론트도 나중에 DB를 직접 안 건드리고 지금 이 API만 거친다. 회사 프로젝트의 `ajax_*.php`가 프론트와 DB 사이를 지키고 있는 것과 똑같은 그림이다.

## 오늘 정리

- Flyway로 스키마를 버전 관리하는 이유
- record와 Entity가 왜 같이 못 쓰이는지 (불변 vs dirty checking)
- Repository 인터페이스 하나로 CRUD가 전부 생기는 원리
- 의존성 주입(DI) — 행위 자체를 가리키는 말이라는 것, IoC 컨테이너가 앱 시작 시점에 미리 조립해둔다는 것
- eGovFrame과 나란히 놓고 보니 XML도 결국 DI였다는 것
- `@Column(name=...)` 오타는 컴파일 타임에 못 잡고 런타임에만 걸린다는 것
- API를 거쳐서 테스트하는 이유 — DB 확인이 아니라 우리 코드 확인

## 다음 편 예고

첫 CRUD 체인은 완성됐다. 다음은 이 브랜치를 커밋하고 PR 워크플로우를 실제로 한번 돌려볼 차례 — 그리고 슬슬 Service 계층이 필요해질 것 같다.
