# Optional

wrapper 클래스

Optional 타입으로 감싼 객체는
orElseThrow 메서드를 통해
`null이면 커스텀 예외를 던진다`
를 간단하게 표현할 수 있다!

# Enum

- enum은 서로 관련된 상수들을 하나의 타입으로 묶어서, **타입 안전성**을 보장하고 의미 있는 동작까지 추가할 수 있는 특수한 클래스입니다.
- 타입 안정성이란 컴파일 시점에 내가 선언한 상수만 타입으로 인정하도록 하여, 그 외 값은 걸러주는 것을 의미합니다.
- 이넘 상수(ex POST_NOT_FOUND)는 필드와 메서드를 갖습니다. 즉 데이터를 가지고 동작이 가능합니다.
- 각 enum 상수는 컴파일 시점에 변환된 'static final' **객체**이다. jvm이 관리한다.
- enum 상수는 해당 enum 타입의 인스턴스인데, static final로 관리되기 때문에 JVM 내에서 단 하나만 존재하는 싱글톤이에요. 그래서 == 비교가 안전하고 thread-safe합니다.
- 싱글톤이기 때문에 == 비교시 같은 이름의 이넘 상수라면, 무조건 같은 객체이다.
- thread-safe가 깨지는 건 누군가 수정하는 도중에 다른 사람이 읽을 때예요. enum은 한 번 만들어지면 절대 수정이 안 되니까, 몇 개의 스레드가 동시에 접근해도 문제가 없는 거예요.
- final → 참조 변경 불가 (다른 객체로 교체하거나 null로 만들 수 없음)
- 메모리 삭제 → GC 담당인데, static이라 프로그램 종료 전까지 삭제 안 됨
- "enum 상수는 static final로 관리되기 때문에 생성 이후 참조 변경도, 메모리 삭제도 불가능해요. 결국 조회만 가능한 구조라서 여러 스레드가 동시에 접근해도 충돌이 생길 여지가 없고, 그래서 thread-safe합니다."

# mybatis PostResponse 완전 쉽게 이해하기

### 🍕 피자 가게 비유로 생각해보자

식당 주방에는 **재료 원본**이 있어. 양파, 치즈, 밀가루...
근데 손님한테 줄 때는 **완성된 피자**로 포장해서 줘야 하잖아.

코드에서도 똑같아.

| 주방 재료 (원본)                 | 손님에게 주는 것 (포장)                          |
| -------------------------------- | ------------------------------------------------ |
| `Post` (DB에서 꺼낸 날것 데이터) | `PostResponse` (클라이언트에 보낼 깔끔한 데이터) |

### 왜 `Post`를 그냥 주면 안 되나요?

`Post`엔 DB용 정보가 너무 많이 들어있어.
비밀번호, 내부 설정값, 민감한 데이터 등등...

```
Post (DB 원본)
├── id
├── title
├── content
├── createdAt
├── user
│   ├── id
│   ├── email
│   ├── password   ❌ 이런 거 클라이언트에 보내면 안 됨!
│   ├── nickname
│   └── role       ❌ 이런 것도
└── ...
```

그래서 **필요한 것만 골라서** `PostResponse`로 포장해서 줘.

### `record`가 뭐야?

그냥 **데이터 보관함**이야. 클래스랑 비슷한데, 더 단순해.

```java
// 이게 전부야. 생성자, getter 자동으로 다 만들어줌
public record PostResponse(
  Long id,
  String title,
  String content,
  LocalDateTime createdAt,
  Author author      // ← 작성자 정보도 같이 담음
) { }
```

### `Author`는 왜 안에 또 있어? (중첩 record)

응답 JSON이 이런 모양이어야 하거든.

```json
{
  "id": 1,
  "title": "제목",
  "content": "내용",
  "createdAt": "2024-01-01",
  "author": {              ← 이 중괄호 때문에!
    "id": 5,
    "email": "kim@test.com",
    "nickname": "김철수"
  }
}
```

`author`가 단순 문자열이 아니라 **객체 안에 객체** 구조라서,
`Author`라는 작은 보관함을 따로 만들어서 `PostResponse` 안에 넣은 거야.

### `from()`이 하는 일

`from()`은 **변환 담당 직원**이야.

```
Post (날것 재료)  →  [from() 직원]  →  PostResponse (포장 완료)
```

```java
public static PostResponse from(Post post) {
    User user = post.getUser();  // Post에서 User 꺼내기

    return new PostResponse(
        post.getId(),        // Post에서 하나씩 꺼내서
        post.getTitle(),
        post.getContent(),
        post.getCreatedAt(),
        user != null         // 작성자가 있으면 Author로 포장, 없으면 null
            ? new Author(user.getId(), user.getEmail(), user.getNickname())
            : null
    );
}
```

`user != null ? ... : null` 이 부분은  
→ **"작성자 정보가 있으면 Author 만들고, 없으면 그냥 null 줘"** 라는 뜻.

### 서비스 코드 흐름 전체 그림

```
[클라이언트 요청: id=1인 게시글 줘!]
         ↓
findById(1L)
         ↓
postMapper.findById(1L)   → DB에서 Post 꺼내옴
         ↓
없으면? → CustomException 던지기 (404 에러)
있으면? ↓
PostResponse.from(post)   → Post를 PostResponse로 변환
         ↓
[클라이언트에게 깔끔한 JSON 응답]
```

```java
public PostResponse findById(Long id) {
    Post post = postMapper.findById(id)
        .orElseThrow(() -> new CustomException(ErrorCode.POST_NOT_FOUND));
        //               ↑ 없으면 에러, 있으면 post 변수에 저장

    return PostResponse.from(post);
    //     ↑ Post → PostResponse 변환해서 반환
}
```

> `Post`는 **DB용 날것 데이터**, `PostResponse`는 **클라이언트에게 줄 포장 데이터**.  
> `from()`은 둘을 **변환해주는 직원**이고, 서비스는 그 직원을 **호출하는 매니저**야.

# Mybatis 파라미터 바인딩 ${} 문법

`단순 치환`

> 테이블 이름, 필드명, 정렬 키워드 -> 파라미터화 가능

```xml
<select id="findAll" resultMap="postWithUser">
  SELECT p.id as post_id, p.user_id, p.title, p.content, p.created_at as post_created_at,
         u.email, u.nickname, u.created_at as user_created_at
  FROM users u
  INNER JOIN posts p ON u.id = p.user_id
  ORDER BY post_id ${sort}
  LIMIT #{offset}, #{size}
</select>
```

**2. `ORDER BY post_id #{sort}` — 동적 정렬은 `${}` 사용**

`#{}` 는 PreparedStatement의 **값(value)** 바인딩이라 `'ASC'`처럼 따옴표가 붙어버려요.
컬럼명이나 ASC/DESC 같은 **SQL 키워드**는 `${}` 를 써야 해요.

```sql
ORDER BY post_id ${sort}   -- ✅
ORDER BY post_id #{sort}   -- ❌ → ORDER BY post_id 'ASC' 로 해석됨
```

> ⚠️ `${}` 는 SQL Injection 위험이 있으니, 서비스 레이어에서 `"ASC"/"DESC"` 만 허용하도록 검증 필수.


