### application.yml vs application.yaml
둘의 기능적 차이 없음
관습적을로 사람들이 아래와 같이 구분해서 씀
- 인프라 설정 -> yaml 풀네임 씀
- 개발 코드 -> yml 씀

### yml 야믈의 문법

- 콜론과 들여쓰기로 계층을 구분함
  - (1) **탭을 쓰지 말고** 무조건 스페이스 두번 해야함
  - (2) 콜론 다음에 스페이스 한번

### 환경 변수 관리

```
application-prod.yml
application-dev.yml
application.yml -> 개발 야믈, 운영 야믈 어떤 프로파일을 적용할지 결정 + 공통 내용
```

## SQL과 자바 매핑
자바 LONG = BIGINT
String = VARCHAR
LocalDateTime은 `DATETIME` 이랑 매핑돼!

| Java | MySQL |
|---|---|
| `LocalDateTime` | `DATETIME` |
| `LocalDate` | `DATE` |
| `LocalTime` | `TIME` |
| `ZonedDateTime` | `DATETIME` + 타임존 별도 관리 |

`LocalDateTime`은 **타임존 정보가 없어**.

| | DATETIME | TIMESTAMP |
|---|---|---|
| 범위 | 1000년 ~ 9999년 | 1970년 ~ 2038년 |
| 타임존 | 영향 없음 | DB 타임존 따라 변환됨 |
| 용도 | 생년월일, 예약일 등 | 생성일시, 수정일시 등 |

`LocalDateTime`은 타임존 정보가 없어서 보통 `DATETIME`이랑 매핑하고, 
타임존이 필요하면 `ZonedDateTime` + `TIMESTAMP` 조합을 써.

> 한 줄 요약: `LocalDateTime`은 타임존 정보가 없어서 브라우저가 멋대로 해석

## DB용 DTO와 요청/응답용 DTO

| 레이어 | 구성 요소 | 불변 객체 |
|---|---|---|
| 클라이언트 | React | - |
| Presentation Layer | Controller, 요청/응답 DTO | ✅ 가능 (Record) |
| Business Layer | Service | - |
| Persistence Layer | Repository, Entity, DB용 DTO | ❌ 불가 (JPA 리플렉션) |
| DB | MySQL | - |

`한 줄 요약: DB 관련은 퍼시스턴스 레이어 — 영속성(데이터를 영구 저장)을 담당한다는 의미에서 붙은 이름이야.`

# db dto 는 왜 불변이면 안돼?

```java
JPA가 객체 만드는 방식
DB에서 데이터 조회
      ↓
기본 생성자로 빈 객체 생성 (리플렉션)
      ↓
각 필드에 setter로 값 주입
근데 불변 객체는:

final 필드라 setter로 값 못 넣음 ❌
기본 생성자 없을 수도 있음 ❌
```

`한 줄 요약: JPA가 DB 데이터를 객체에 담을 때 기본 생성자 + 값 주입 방식을 쓰는데, 불변 객체는 이 과정이 불가능해서야.`

### mybatis가 좋은 이유
- 쿼리 볼 수 있음
- 성능에 대한 예측 가능함
- select에서만 사용해도 됨 - 동적 쿼리 원활하게 짤 수 있음

## 필요한 것
```java
(1) dto/ -> db용 dto 객체
(2) mapper/ ->  @Mapper // 나는 xml mapper를 호출할 때 사용하는 인터페이스
(3) resources/mapper/ -> xml 매퍼 
(4) resources/ -> schema.sql과 data.sql (ddl, dml sql문)
```

- select는 result map이 필요하다 (dto와 sql 사이 필드 매핑을 위해)
  -  property에는 dto 필드명, coulum에는 컬럼명
- insert, delete, update의 결과값은 영향을 받은 행의 개수로 int이다.
- select를 제외한 나머지 dml은 result map 적지 않는다. 적으면 오류


```xml
<!-- "이 resultMap은 Post 객체를 만들어" -->
<resultMap id="postWithUser" type="Post">

    <!-- Post의 필드들 -->
    <result property="title" column="title"/>

    <!-- "그 중 user 필드는 User 객체야" -->
    <association property="user" javaType="User">
        <result property="email" column="email"/>
    </association>

</resultMap>
```
## 레이어별 객체 역할
        
```
MyBatis : Service → Mapper → DB
JPA     : Service → Repository → DB
```
매퍼와 레포지토리 모두 db 접근 레이어

| 객체 | 역할 | 오가는 레이어 |
|---|---|---|
| `Post`, `User` | 도메인/엔티티 | Service ↔ Mapper ↔ DB |
| `CreatePostRequest` | 요청 DTO | 프론트 → Controller → Service |
| `PostResponse` | 응답 DTO | Service → Controller → 프론트 |

### 엔티티 vs DTO

- **엔티티**: DB 테이블 표현 + 비즈니스 상태를 담는 객체
- **DTO**: 데이터 전달 목적의 단순 객체

엄밀히 말하면 엔티티는 DTO가 아니다.
단, MyBatis는 JPA처럼 영속성 관리를 하지 않아서 경계가 흐린 편이다.

- **Entity**: DB 테이블과 1:1 매핑되는 객체. JPA는 `@Entity`, MyBatis는 어노테이션 없이 사용
- **Domain**: 비즈니스 로직을 담는 핵심 객체. 실무에서는 Entity와 혼용해서 부르는 경우가 많음

**DTO** (`PostRequest`, `PostResponse`)
- 클라이언트에게 필요한 데이터만 담음
- DB 구조가 바뀌어도 API 응답 형태 유지 가능
- `@Valid` 같은 입력 검증을 여기서 처리

**Domain** (`Post`)
- DB 테이블 구조를 반영
- 비즈니스 로직을 담을 수 있음
- 클라이언트에게 노출하면 안 되는 필드 포함 가능 (ex. `password`)


### 변환

```java
// JPA
public void createPost(PostRequest dto) {
    Post post = Post.from(dto);        // Service가 변환 (동일)
    postRepository.save(post);         // Repository가 DB 접근
}

// MyBatis
public void createPost(PostRequest dto) {
    Post post = Post.from(dto);        // Service가 변환 (동일)
    postMapper.insert(post);           // Mapper가 DB 접근
}
```

## useGeneratedKeys의 필요성

``` java
@Service
@RequiredArgsConstructor
public class PostService {
  private final PostMapper postMapper;
  public void createPost(CreatePostRequest createPostRequest){
    
    // post의 id, 날짜 없음
    Post post = Post.builder()
      .userId(createPostRequest.userId())
      .title(createPostRequest.title())
      .content(createPostRequest.content())
      .build();
    
    // 참조무결성 제약조건 위배를 대비한 코드
    // xml 매퍼에서 useGeneratedKeys == pk 생성 -> keyProperty="id"에 집어 넣음
    // 위를 통해 id가 생김
    postMapper.save(post);    

    // 포스트 리턴시 createAt 제외한 모든 값 반환됨
    // createAt까지 채우고 싶다면 post의 id를 통해 select 한 뒤 채우기
    
  }

}
```

`부모의 pk값을 넣어야 자식이 fk 값으로 가져와서 사용할 수 있다.`

 