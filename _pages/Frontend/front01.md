---
title: '재사용성을 고려한 UI 컴포넌트 만들기'
tags:
  - Vue.js
  - Tailwind CSS
date: '2025-04-23'
thumbnail: '/assets/img/frontend/01/front01_0.png'
---

얼마 전, `Vue`를 이용해 가계부 스켈레톤 프로젝트를 진행했습니다. 컴포넌트 기반 개발을 지향했지만, 재사용성이 높은 코드에 대한 이해가 부족해 같은 코드를 반복해서 작성하는 일이 많았습니다.
이번 기회에 재사용성 높은 컴포넌트에 대해 공부해보고, 그 내용을 정리하여 다음 프로젝트에 효과적으로 적용해보고자 합니다.

# 🗹 재사용 가능한 컴포넌트가 왜 중요한가?

---

서비스가 커질수록 모든 UI 컴포넌트를 일일이 직접 작성하는 것은 비효율적일 뿐 아니라, 유지보수성도 크게 떨어집니다.

<img src="/assets/img/frontend/01/front01_3.png" alt="velog 404"
style="display: block; margin: 0 auto; max-width: 300px; max-height: 200px;">

가장 흔히 사용하는 버튼만 봐도, `primary`, `secondary`, `danger` 등 버튼의 목적에 따라 다양한 색상과 스타일을 적용하게 됩니다. 게다가 최근에는 다크 모드 지원까지 고려해야 하니 필요한 버튼 스타일은 더욱 많아질 수밖에 없습니다. 이처럼 여러 종류의 버튼을 **일관성 있게 관리**하려면 어떻게 해야 할까요?

```html
<button />
<PrimaryButton />
<SecondaryButton />
<DangerButton />
<MediumButton />
```

이렇게 각각의 버튼을 별도의 컴포넌트로 만들어두면 관리하기 어렵고, 코드 중복도 늘어나게 됩니다. 이를 피하기 위해, **재사용 가능한 컴포넌트**를 만들어 다양한 버튼을 **하나의 컴포넌트로** 일관성 있게 관리할 필요가 있습니다.

# 🗹 Tailwind CSS란?

---

<img src="/assets/img/frontend/01/front01_2.png" alt="velog 404"
     style="display: block; margin: 40px auto; max-width: 300px; max-height: 200px;">

`Tailwind`에 대해 자세한 설명은 이 포스트의 주제가 아니기 때문에 간단하게 소개만 하고 넘어가겠습니다. `Tailwind`는 수많은 유틸리티 클래스로 이뤄진 `CSS` 프레임워크입니다. 유틸리티 클래스들을 조합하면 별도의 CSS 코드 작성없이 단순히 HTML 요소의 `class`속성에 원하는 클래스를 지정하는 것만으로 손쉽게 스타일링이 가능합니다.

```html
<button
  class="rounded bg-blue-500 px-4 py-2 font-bold text-white hover:bg-blue-600"
>
  Primary
</button>

<button
  class="rounded bg-red-500 px-4 py-2 font-bold text-white hover:bg-red-600"
>
  Danger
</button>
```

<img src="/assets/img/frontend/01/front01_1.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 150px; max-height: 100px;">

추가적인 `CSS` 코드 없이, HTML 태그에 클래스만 지정해도 스타일을 바로 적용할 수 있습니다.이렇게 동작하는 이유는, `Tailwind CSS`가 빌드 시점에 사용된 유틸리티 클래스만 모아 필요한 CSS 만 자동으로 생성해주기 때문입니다. 아래는 `Tailwind CSS` 빌드 과정입니다.

- 코드에서 실제로 사용된 클래스 이름(bg-blue-500, px-4 등)을 모두 추출
- 해당 클래스에 맞는 CSS 규칙만 최종 CSS 파일에 포함
- 완성된 CSS 파일이 빌드 과정에서 프로젝트에 자동으로 추가됨

# 🗹 재사용 가능한 버튼 컴포넌트 구현

---

`Vue` + `Tailwind` 조합으로 **재사용 가능**한 버튼 컴포넌트를 만들어보겠습니다. `React`에서는 일반적으로 **cva + clsx** 또는 **cva + classnames** 조합을 사용하지만, `Vue`에서는 반응형 시스템과 템플릿 문법이 강력하기 때문에 보통 **computed + :class** 조합으로 구현합니다.

```javascript
<script setup>
//...
const props = defineProps({
  variant: {
    type: String,
    default: 'primary',
    validator: (val) => ['primary', 'secondary', 'danger'].includes(val),
  },
  size: {
    type: String,
    default: 'md',
    validator: (val) => ['sm', 'md', 'lg'].includes(val),
  },
  type: {
    type: String,
    default: 'button',
  },
  disabled: {
    type: Boolean,
    default: false,
  },
  block: {
    type: Boolean,
    default: false,
  },
  rounded: {
    type: Boolean,
    default: true,
  },
});
//...
</script>
```

먼저 `<script setup>` 내부에서 부모로부터 전달받을 `props`를 정의합니다.
각 `prop`의 역할은 다음과 같습니다:

- `variant`: 버튼 스타일 유형 (**primary**, **secondary**, **danger** 중 선택)
- `size`: 버튼 크기기 (**sm**, **md**, **lg**)
- `type`: 버튼의 HTML 타입 속성 (**button**, **submit**, **reset**)
- `disabled`: 버튼을 비활성화할지 여부
- `block`: 버튼을 **w-full**로 전체 너비로 만들지 여부
- `rounded`: 버튼에 **rounded-lg** 클래스를 적용할지 여부

```javascript
<script setup>
import { computed } from 'vue';
//...
const baseClass = 'font-semibold focus:outline-none transition duration-200';

const variantClasses = {
  primary: 'bg-blue-600 text-white hover:bg-blue-700',
  secondary: 'bg-gray-200 text-gray-900 hover:bg-gray-300',
  danger: 'bg-red-600 text-white hover:bg-red-700',
};

const sizeClasses = {
  sm: 'text-sm px-3 py-1.5',
  md: 'text-base px-4 py-2',
  lg: 'text-lg px-6 py-3',
};

const runtimeClass = computed(() => [
  baseClass,
  variantClasses[props.variant],
  sizeClasses[props.size],
  props.block && 'w-full',
  props.rounded ? 'rounded-lg' : 'rounded-none',
  props.disabled && 'opacity-50 cursor-not-allowed',
]);
</script>
```

위에서 받은 props를 바탕으로 `<button>` 태그에 적용할 클래스를 정의해줍니다.

`baseClass`: 모든 버튼에 공통 적용되는 기본 클래스

| 클래스             | 의미                                             |
| ------------------ | ------------------------------------------------ |
| font-semibold      | 폰트 두께를 semi-bold로 설정 (600)               |
| focus:outline-none | 버튼에 포커스됐을 때 브라우저 기본 아웃라인 제거 |
| transition         | 기본 트랜지션 효과 활성화                        |
| duration-200       | 트랜지션 지속 시간을 200ms로 설정                |

`variantClasses`: props의 variant 속성 값에 따라 버튼 색상 결정

| 클래스            | 의미                                  |
| ----------------- | ------------------------------------- |
| bg-blue-600       | 배경색: Tailwind의 파란색 계열 600번  |
| text-white        | 색상: 흰색                            |
| hover:bg-blue-700 | hover 시 배경색을 700번 파랑으로 변경 |
| bg-gray-200       | 배경색: 밝은 회색 (200번)             |
| text-gray-900     | 텍스트 색상: 거의 검정색 (900번)      |
| hover:bg-gray-300 | hover 시 회색(300번)으로 변경         |
| bg-red-600        | 배경색: 진한 빨강 (600번)             |
| text-white        | 텍스트: 흰색                          |
| hover:bg-red-700  | hover 시 더 진한 빨강 (700번)         |

`sizeClasses`: props의 size 속성 값에 따라 글자 크기와 패딩 설정

| 클래스                        | 의미                           |
| ----------------------------- | ------------------------------ |
| text-sm / text-base / text-lg | 텍스트 크기: 작은/기본/큰 글씨 |
| px-3 / px-4 / px-6            | 좌우(Padding X) 여백 설정      |
| py-1.5 / py-2 / py-3          | 위아래(Padding Y) 여백 설정    |

`runtimeClass`는 props의 값에 따라 동적으로 클래스 목록을 계산하여,
`<button :class="runtimeClass">` 형태로 적용됩니다.
<br>

```html
<template>
  <button :type="type" :class="runtimeClass" :disabled="disabled"></button>
</template>
```

`<button>`: 부모로부터 전달받은 prop에 따라 유연하게 스타일링, 타입, 비활성화를 설정해줄 수 있도록 구성

> 💡 disabled이 이미 runtimeClass 안에서 처리되고 있는데 HTML 속성인 :disabled="disabled"도 또 명시하는 이유는?
>
> - Tailwind 클래스는 "스타일"만 적용하고, 실제 버튼 기능은 막지 못함
> - :disabled 속성은 HTML 레벨에서 실제 버튼 기능을 비활성화하는 역할
>   `disabled`: 실제 버튼의 클릭·포커스 등 기능 차단
>   `runtimeClass`: 시각적으로 흐리게 보이고, 커서도 변경되는 효과

## 사용법과 실제 렌더링 결과

위에서 작성한 컴포넌트는 아래처럼 사용할 수 있습니다.

```html
<CustomButton variant="secondary" size="lg">큰 회색 버튼</CustomButton>
<CustomButton variant="danger" size="md" :disabled="true"
  >사용 불가 버튼</CustomButton
>
```

<img src="/assets/img/frontend/01/front01_3.png" alt="velog 404"
style="display: block; margin: 0 auto; max-width: 300px; max-height: 200px;">

클래스를 병합하고 `Vue`가 최종적으로 렌더링 해주는 HTML은 아래가 됩니다.

```html
<button
  type="button"
  class="font-semibold focus:outline-none transition duration-200 bg-gray-200 text-gray-900 hover:bg-gray-300 text-lg px-6 py-3 rounded-lg"
>
  큰 회색 버튼
</button>

<button
  type="button"
  class="font-semibold focus:outline-none transition duration-200 bg-red-600 text-white hover:bg-red-700 text-base px-4 py-2 rounded-lg opacity-50 cursor-not-allowed"
  disabled
>
  사용 불가 버튼
</button>
```
