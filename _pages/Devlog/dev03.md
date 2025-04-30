---
title: 'IntelliJ 디버깅에 대해'
tags:
  - IntelliJ
  - debugging
date: '2025-04-30'
thumbnail: '/assets/img/devlog/03/dev03_0.png'
---

최근 알고리즘 문제 풀이 언어를 `Java`로 변경하면서, 기존에도 자주 사용하던 `IntelliJ`의 활용 빈도가 더욱 높아졌습니다. 이번 기회에 `IntelliJ`의 디버그 모드를 보다 능숙하게 활용할 수 있도록 정리하고자 합니다.

# 디버그란?

---

> 프로그래밍 과정 중에 발생하는 오류나 비정상적인 연산,
> 즉 버그를 찾고 수정하는 것. 이 과정을 디버깅(Debugging)이라고 하기도 한다.
> 출처: https://namu.wiki/w/%EB%94%94%EB%B2%84%EA%B7%B8

프로그래밍을 하는 누구나 공감하는 격언 중 하나는 “버그가 없는 프로그램은 없다” 혹은 “한 번에 제대로 작동하는 프로그램은 없다”는 말입니다. 이는 단순한 경험담을 넘어서, 프로그래밍이라는 행위 자체가 불확실성과의 싸움임을 나타냅니다.

아무리 뛰어난 실력을 갖춘 개발자라도 버그를 완전히 피할 수는 없습니다. 대신, 오류를 최소화하고 발생하더라도 빠르게 추적하고 수정할 수 있도록 견고한 설계와 효과적인 디버깅 습관을 갖추는 것이 중요합니다.

실제로 프로그램 제작 과정에서 코딩이 2할이라면, 디버깅은 나머지 8할을 차지한다고 해도 과언이 아닙니다. 사람이 작성하는 코드에는 필연적으로 오류가 생기며, 이를 잡아내는 **디버깅**은 프로그래밍 과정의 핵심입니다.

# Intellij 디버그 모드

---

<img src="/assets/img/devlog/03/dev03_1.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

**`Step Over`** (F8)

> - 다음 **줄**로 넘어감
> - method call 안으로 들어가지는 않음

**`Step Into`** (F7)

> - 현재 대기하고 있는 method call 내부로 들어감

**`Step Out`** (Shift + F8)

> - 현재 메서드의 실행을 끝내고 이 메서드를 호출한 상위 메서드로 복귀

**`Resume Program`** (F9)

> - 다음 **break point**로 이동

**`Evaluate Expression`** (Alt + F8)

> - **멈춘 지점**에서 여러 값을 계산해볼 수 있음
> - println을 일일이 하지 않고도 원하는 지점에서 원하는 값을 조회할 수 있음
> - Evaluate 실행된 코드는 **실제로 실행**이 되기 때문에 주의!!

## Conditional Breakpoint

반복문이 많은 상황에서 특정 조건만 디버깅하고 싶은 경우가 종종 있습니다. 예를 들어, 100번 반복하는 for문이 있을 때 90번째 값만 확인하고 싶다면, Step Over(F8)을 90번 눌러야 할까요...?

<img src="/assets/img/devlog/03/dev03_2.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 400px; max-height: 400px;">

이러한 상황에 `Conditional Breakpoint`를 사용하는 것이 매우 효과적입니다. IntelliJ에서는 `breakpoint`를 우클릭한 후 조건식을 설정할 수 있습니다. 실제로는 for문을 돌지만, 설정한 조건식일 때만 디버거가 멈추도록 조건부 **break point**를 설정하는 것입니다.

<img src="/assets/img/devlog/03/dev03_3.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

반복 변수인 subject가 "Spring 평가"일 때만 멈추고 싶다면, subject.equals("Spring 평가")와 같이 작성하면 됩니다.

# Intellij 디버그 모드 실습

---

한 학생의 과목별 점수를 입력받고 평균 점수를 계산하는 간단한 실습용 코드를 작성해봤습니다.

<img src="/assets/img/devlog/03/dev03_4.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

`Main.java`의 `calculator.calculateAverage(student)`에서 **breakpoint** 설정을 했습니다. **Variables**창에서 현재 스코프의 모든 변수를 확인할 수 있습니다. **Step Into**로 `GradeCalculator` 내부로 진입하겠습니다.

<img src="/assets/img/devlog/03/dev03_5.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

**Step Over**로 다음 라인으로 넘어가면서 `score`, `sum`, `count`가 지역 스코프에서 어떻게 변화하는지 확인할 수 있습니다.

글만 읽어서는 완전히 이해하기 어려울 수 있습니다. 디버깅은 아무래도 직접 실습해보는 것이 가장 빠르게 체감할 수 있는 방법입니다. 위에서 소개한 내용을 바탕으로 직접 따라 해보신다면 금방 감을 잡으실 수 있을 것입니다.
