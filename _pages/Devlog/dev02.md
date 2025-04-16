---
title: '테스트 주도 개발... 직접 해보자!'
tags:
  - Test-Driven Development
  - Spring
  - AssertJ
date: '2025-04-16'
thumbnail: '/assets/img/devlog/02/dev02_0.png'
---

최근 켄트 벡의 **『테스트 주도 개발』** 책으로 `TDD` 스터디를 시작하게 됐습니다. 사실 그동안 테스트 코드의 중요성은 잘 알고 있었지만, 항상 프로젝트에서 나중에 하자며 미루기만 했고, 결국엔 손도 못 대고 넘어가는 경우가 대부분이었습니다. 그래서 언젠가는 꼭 제대로 공부해 봐야겠다고 생각했는데, 마침 좋은 기회가 생겨 망설임 없이 스터디에 참여하게 됐습니다.

아직 스터디 초반이라 책의 앞부분을 읽는 중인데, 지금은 솔직히 조금 뜬구름을 잡는 느낌도 듭니다. 중요하고 좋은 건 알겠는데 도대체 테스트 코드는 어떻게 작성하는 건데..!!!

<img src="/assets/img/devlog/02/dev02_1.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 600px; max-height: 400px;">

개발자는 코드를 직접 작성할 때 가장 빠르게 습득한다고 하죠? 그래서 직접 코드를 작성하면서 테스트 주도 개발의 흐름을 이해하고자 합니다.

# 테스트 주도 개발이란?

---

<img src="/assets/img/devlog/02/dev02_2.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 600px; max-height: 400px;">

> **Red 단계**

- 실패하는 테스트 먼저 작성
- 아직 기능을 구현하지 않았기 때문에 테스트는 당연히 실패
- 이 실패가 기능이 필요한 근거가 됨

> **Green 단계**

- 테스트를 통과할 수 있는 최소한의 코드 작성
- 구현이 허술해도, 일단 테스트만 통과하면 OK

> **Refactor 단계**

- Green 단계에서 급하게 짠 코드를 리팩토링
- 중복 제거, 의미 있는 이름 부여, 구조 개선 등
- 이때도 모든 테스트가 여전히 통과해야 리팩토링 성공

# 테스트 주도 개발 흐름

---

## ⚪️ 문제 설명 및 초기 세팅

테스트 주도 개발(TDD)의 흐름을 이해하기 위해 간단한 회원 가입 기능을 구현해보겠습니다. `id`, `name`, `password`를 통해 회원 가입을 하고 중복되는 `name`으로 가입하려고 하면 에러가 발생합니다.
TDD의 원칙은 **테스트 코드를 먼저 작성하는 것**이지만, 먼저 클래스와 메서드의 코드 구조부터 잡아두면 컴파일 오류를 피할 수 있습니다. 물론 이 단계에서는 비즈니스 로직은 작성하지 않습니다.

```java
@Entity
public class Member {
    @Id
    private String id;
    private String name;
    private String password;
}
```

```java
@Repository
public interface MemberRepository extends JpaRepository<User, String> {
}
```

```java
@Service
public class MemberService {
    public Member join(Member member) {
        return null;
    }
}
```

## 1) 회원가입 기능

### 🔴 Red 단계

`Member`, `MemberRepository`, `MemberService` 중 비즈니스 로직을 가지고 있는 유일한 클래스는 `UserService`이므로 해당 클래스에 대한 테스트를 작성합니다. 물론, 아직 기능을 구현하지 않았기 때문에 테스트는 당연히 실패하고 이 실패가 기능이 필요한 근거가 됩니다. 단축키 `Ctrl + Shift + T`로 테스트 코드를 작성할 수 있습니다.

테스트 라이브러리로는 `AssertJ`를 사용하겠습니다.

> **`AssertJ`** vs **`Junit`**
>
> https://velog.io/@bonjugi/assertj-vs-junit

```java
@SpringBootTest
class MemberServiceTest {
    @Autowired
    private MemberService MemberService;
    @Autowired
    private MemberRepository MemberRepository;

    @Test
    void 회원가입() {
        //given
        Member unjoinedMember = new Member("12", "woo0", "1234");
        //when
        memberService.join(unjoinedMember);
        //then
        Member joinedMember = memberRepository.findById(unjoinedMember.getId()).get();
        assertThat(joinedMember).isEqualTo(unjoinedMember);
    }
}
```

<img src="/assets/img/devlog/02/dev02_3.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 600px; max-height: 400px;">

테스트는 당연히 실패하고 컴파일 에러도 있습니다. 이 실패는 근거가 됩니다.

```java
@Entity
@Getter
public class Member {
    @Id
    private String id;
    private String name;
    private String password;

    public Member(String id, String name, String password) {
        this.id = id;
        this.name = name;
        this.password = password;
    }
    public Member() {
    }
}
```

먼저, 테스트 코드가 컴파일되도록 **최소한의 수정만 반영**합니다. 이제 테스트는 실행되지만 실패하는 상태가 되었으므로 **Green 단계**로 넘어갑니다.

## 🟢 Green 단계

테스트를 실행하면 예상대로 `NoSuchElementException`이 발생합니다. 이제 테스트를 통과하기 위해 `MemberService`의 `join`메서드를 수정하겠습니다.

```java
public Member join(Member member) {
    return memberRepository.save(member);
}
```

<img src="/assets/img/devlog/02/dev02_4.png" alt="velog 404" style="max-width: 600px; max-height: 400px;">

## 🟡 Refactor 단계

**Refactor 단계**는 테스트를 통과하기 위해 작성했던 코드를 리팩토링하는 단계입니다. 현재는 `@Autowired`를 통해 의존성을 주입하고 있었지만, 이를 생성자 주입 방식으로 수정하겠습니다.

> 생성자 주입 vs Autowired
> https://madplay.github.io/post/why-constructor-injection-is-better-than-field-injection

```java
@Service
@RequiredArgsConstructor
public class MemberService {
    private final MemberRepository memberRepository;

    public Member join(Member member) {
        return memberRepository.save(member);
    }
}
```

또한, 엔티티 클래스에서는 `@Data` 대신 필요한 애노테이션만 선택적으로 사용하도록 수정하겠습니다.

```java
@Getter
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode
@Entity
public class Member {
    @Id
    private String id;
    private String name;
    private String password;
}
```

하나의 TDD 사이클을 마쳤습니다. 이제 다음 사이클에서는 **중복 가입 시 예외가 발생**하도록 구현해보겠습니다.

## 2) 중복 가입 시 예외

### 🔴 Red 단계

테스트 코드 먼저 작성을 해야겠죠?

```java
@Test
void 중복_회원가입_예외() {
    //given
    Member member1 = new Member("1", "woo0", "123412");
    Member member2 = new Member("2", "woo0", "123434");
    //when
    memberService.join(member1);
    //then
    assertThatThrownBy(() -> memberService.join(member2))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("이미 존재하는 회원입니다.");
}
```

당연히 테스트는 실패합니다.

## 🟢 Green 단계

테스트를 통과하기 위해 `MeberService`의 `join` 매서드를 수정해야 합니다.

```java
@Service
@RequiredArgsConstructor
public class MemberService {
    private final MemberRepository memberRepository;

    public Member join(Member member) {
        validateDuplicateMember(member);
        return memberRepository.save(member);
    }

    private void validateDuplicateMember(Member member) {
        memberRepository.findByName(member.getName())
                .ifPresent(m -> {
                    throw new IllegalStateException("이미 존재하는 회원입니다.");
                });
    }
}
```

<img src="/assets/img/devlog/02/dev02_4.png" alt="velog 404" style="max-width: 600px; max-height: 400px;">

## 🟡 Refactor 단계

`Refactor` 단계에서는 리팩터링할 코드가 없다면 억지로 수정할 필요는 없습니다.따라서 중복 가입 예외 처리 사이클은 여기서 마무리하겠습니다.

두 개의 사이클을 통해 **TDD**의 흐름을 실제로 경험해보며, 시간이 다소 소요되긴 했지만 **올바른 설계**를 위한 좋은 방법이라는 확신이 들었습니다. 앞으로는 스터디를 진행하며 더 깊이 있게 학습해볼 계획입니다.
