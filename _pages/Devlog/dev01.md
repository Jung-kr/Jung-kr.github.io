---
title: '개발자 블로그, Velog에서 GitHub Pages로'
tags:
  - Github Pages
  - Jekyll
  - Giscus
date: '2025-03-25'
thumbnail: '/assets/img/dev01_0.png'
---

최근, 블로그를 **`Velog`**에서 **`GitHub Pages`** 기반 블로그로 이전했습니다. 이번 포스팅에서는 **`GitHub Pages`** 블로그로 옮기게 된 이유와, 그 과정을 통해 새롭게 배운 점들을 정리해 보려 합니다.

# 블로그 이전을 결심한 이유

---

사실 **`Velog`**도 장점이 정말 많습니다. 개인적으로는 링크 처리나 이미지/파일 업로드 같은 **사용 편의성** 면에서는, 다른 어떤 블로그보다 뛰어나다고 생각합니다. 그리고 모든 개발자가 디자인에 신경 쓰는 건 아니고, 오히려 그런 부분에서 스트레스받고 싶지 않은 분들도 많죠. **`Velog`**의 똑같은 디자인 오히려, 좋을 수도...?
그렇게 좋은 블로그인데도, 왜 저는 블로그 이전을 고민하게 됐을까요?

<img src="/assets/img/dev01_1.png" alt="velog 404" style="max-width: 600px; max-height: 400px;">

- **잊을 만하면 찾아오는 404 에러...**
  **`Velog`** 내부 서버의 문제인지는 모르겠지만, 생각보다 자주 **`404 에러`**가 떴습니다. **`Tistory`**도 가끔 그런 문제가 있다는 이야기를 들었고요. 그러다 보니, **`GitHub`** 서버는 다른 블로그들보단 **안정적이지 않을까?** 라는 막연한 생각이 들었습니다.

<img src="/assets/img/dev01_2.png" alt="velog UI" style="max-width: 600px; max-height: 400px;">

- **너무 비슷한 스타일...**
  사실 저도 디자인을 크게 신경 쓰는 타입은 아닙니다. 그런데 **`Velog`** 블로그를 운영하면서, 구글링을 통해 다른 분들의 글을 볼 때마다 디자인이 죄다 비슷해서 어느 글이 어느 분 건지 헷갈릴 때가 많았습니다. 뭔가, **나만의 공간**이라는 느낌...

<img src="/assets/img/dev01_3.png" alt="짱구" style="max-width: 600px; max-height: 400px;">

- **무엇보다... 간지!**
  개발자라면 **본인만의 도메인**으로 운영하는 블로그 하나쯤은 있어야 하지 않을까?"라는 철없는(?) 생각도 했습니다... ㅋㅋ

이런 이유로 다양한 블로그 플랫폼을 비교하게 되었고, 결국 **`GitHub Pages`** + **`Jekyll`** 조합을 선택했습니다. 이후에는 **`GitHub Pages`**가 어떤 서비스인지 간단히 소개하고, **`Hugo`**나 **`Next.js`**처럼 많은 **`정적 페이지 생성기`**들 중에서 **`Jekyll`**을 선택한 이유에 대해 정리해 보려고 합니다.

# 정적 페이지와 GitHub Pages

---

> #### 정적 페이지란

<img src="/assets/img/dev01_4.png" alt="img" style="max-width: 600px; max-height: 400px;">

- 웹 서버에 저장된 파일 (**`HTML`**, **`CSS`**, 이미지 파일, **`JavaScript`** 파일 등)을 클라이언트에게 전송하는 웹 페이지

> #### GitHub Pages란

**`GitHub Pages`**는 저희가 만든 **정적 웹사이트**를 인터넷에 공개할 수 있도록 도와주는 무료 **호스팅 서비스**입니다. 이 기능을 활용하면, **정적 블로그**도 매우 간편하게 배포할 수 있습니다. 혹시 **`github.io`**로 끝나는 블로그를 보신 적 있으신가요? 이런 블로그들은 대부분분 **`GitHub Pages`**를 통해 만들어진 것입니다.

다만, **`GitHub Pages`**는 정적 파일을 보여주는 역할만 하므로 **HTML, CSS, JS** 같은 파일을 매번 직접 작성하고 수정하기엔 번거롭고 비효율적이죠. 그래서 등장한 것이 **정적 사이트 생성기**입니다

> #### 정적 사이트 생성기

정적 사이트 생성기는 많은 페이지를 **효율적으로 만들고 관리할 수 있도록 도와주는 자동화 도구**입니다. **`GitHub Pages`**는 **정적 파일을 보여주는 역할**만 한다면, **`정적 사이트 생성기`**는 그 정적 파일들을 **자동으로 만들어주는 역할**을 합니다.

대표적인 도구로는 **`Hugo`**, **`Next.js`**, **`Jekyll`** 등이 있고, 저는 이 중에서 **`Jekyll`**을 선택했습니다.

# Why Jekyll?

---

제가 **`Jekyll`**을 선택한 이유는, **`GitHub Pages`**에서 기본으로 지원하는 **정적 사이트 생성기**이기 때문입니다. **`GitHub Pages`**에는 **`Jekyll`**이 내장되어 있어서, **`Markdown`** 파일만 올려도 **자동으로 빌드되고 배포**됩니다.

<img src="/assets/img/dev01_5.png" alt="img" style="max-width: 600px; max-height: 400px;">

위의 그림처럼, 원격저장소에 **`git push`**만 해주면 별도의 빌드나 배포 과정 없이 **정적 블로그가 자동으로 생성**됩니다.

# Jekyll 테마 선택 및 댓글 기능 추가

---

**`Jekyll`**의 장점 중 하나는 사용할 수 있는 테마가 정말 많다는 점인데요, 당연히 활용해야겠죠? 그냥 **`"Jekyll Theme"`**로 구글링해 보셔도 되고, 아래 링크에서 맘에 드는 테마를 골라서 적용하셔도 됩니다.

> - [https://jamstackthemes.dev/ssg/jekyll/](https://jamstackthemes.dev/ssg/jekyll/)
> - [http://jekyllthemes.org/](http://jekyllthemes.org/)
> - [https://jekyllthemes.io/](https://jekyllthemes.io/)

원하는 테마의 **`GitHub`** 저장소에 들어가서 **`Fork`**를 뜨고, 본인 스타일에 맞게 커스터마이징하면 됩니다. 저는 [이 블로그](https://velog.io/@shg4821/%EA%B9%83%ED%97%88%EB%B8%8C-%EB%B8%94%EB%A1%9C%EA%B7%B8-%EB%A7%8C%EB%93%A4%EA%B8%B0-1)를 참고했습니다

<img src="/assets/img/dev01_6.png" alt="img" style="max-width: 600px; max-height: 400px;">

댓글 기능은 **`Giscus`** 라는 외부 서비스를 이용했습니다. **정적 사이트 생성기**로 만든 블로그는 기본적으로 **서버가 없기** 때문에, 댓글 기능을 직접 만들 수는 없습니다. 그래서 보통은 **`Disqus`**, **`utterances`**, **`Giscus`** 등 외부 서비스를 활용합니다. 저는, 제가 사용하는 테마 제작자가 추천한 **`Giscus`**를 그대로 적용했습니다. 댓글도 많이 달아주세요!

# 앞으로의 블로그 운영 계획

---

그럼, 기존에 운영하던 **`Velog`**는 어떻게 되느냐... 사실, 버리기엔 너무 **편한 블로그**입니다. 😂
그래서 앞으로는 공부한 내용을 기록할 땐 **`Velog`**, 그리고 공유하고 싶은 글이나 정리된 글은 **`GitHub Pages`** 블로그에 작성할 계획입니다.

블로그 이전을 한 지 얼마 되지 않아서 아직 **`GitHub Pages`** 블로그의 장단점을 확실히 느끼진 못했지만, 앞으로 포스팅을 해나가면서 그 과정에서 느낀 점들도 함께 정리해 보겠습니다.

<img src="/assets/img/dev01_7.png" alt="img" style="max-width: 600px; max-height: 400px;">
