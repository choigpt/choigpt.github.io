---
layout: post
title: "Google Java 스타일 가이드와 IntelliJ 적용법"
date: 2025-04-29 10:30:00 -0700
---

# Google Java 스타일 가이드와 IntelliJ 적용법

구글 자바 스타일을 처음 알게 된 건 우테코 프리코스 때였다. 코드 리뷰에서 구글 자바 스타일을 따라달라는 피드백을 받았고, 알고 보니 업계에서 꽤 유명한 코딩 스타일 가이드였다.

## 그냥 따르면 편해진다

코딩 스타일에 대해 고민하는 시간이 얼마나 많은가. 탭이냐 스페이스냐, 중괄호는 어디에 두느냐, 들여쓰기는 몇 칸이냐... 마치 조선시대 예송 논쟁처럼 끝이 없다. 정해진 규칙을 그냥 따르면 그 고민의 시간을 실제 문제 해결에 쓸 수 있다.

가끔은 정해진 규칙을 따르는 게 반감이 들기도 한다. 내 스타일이 아니니까. 하지만 어차피 작동 면에서는 문제가 없으니 개인 취향보다 일관성이 중요하다. 구글 스타일 가이드의 핵심은:

- 소스 파일은 UTF-8로 인코딩, 들여쓰기는 공백 2칸
- 한 줄은 최대 100자까지만
- 클래스명은 `UpperCamelCase`, 메서드와 변수는 `lowerCamelCase`, 상수는 `CONSTANT_CASE`
- 중괄호는 K&R 스타일(같은 줄에 여는 중괄호)
- 단일 문장이라도 중괄호 생략 안 함

개인 프로젝트든 뭐든 이런 규칙 하나 정해두면 적어도 스타일에 대한 고민은 없어진다.

## 인텔리제이에 적용하기

개인적으로 기록용으로 남기는 인텔리제이 적용법이다. 플러그인 하나로 끝나니 참 편하다.

### 플러그인 설치

1. 인텔리제이에서 `File > Settings`
2. `Plugins` 메뉴로 가서 `Marketplace` 탭 선택
3. "Google Java Format" 검색해서 설치
4. IDE 재시작

### 설정하기

1. 다시 `Settings`로 들어가서 `Other Settings > Google Java Format` 선택
2. `Enable Google Java Format` 체크박스에 체크
3. `Google Style` 선택

### 플러그인 에러 발생시

플러그인에서 오류가 발생하면 아래 JVM 옵션을 추가하라고 할 수 있다. 난 그냥 시키는대로 했다:

```
--add-exports=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED
```

Help > Edit Custom VM Options에 복붙하면 된다. 왜 필요한지는 모르겠고 그냥 하라니까 했다.

### 코드에 적용하기

- 현재 파일에 적용: `Ctrl+Alt+L`
- 프로젝트 전체 적용: `Ctrl+Alt+Shift+L`

## 자동 포맷팅

매번 단축키 누르는 것도 귀찮다면 저장할 때마다 자동으로 포맷팅되게 하면 된다.

1. `Settings > Tools > Actions on Save`
2. `Reformat code` 체크

이제 파일 저장할 때마다 자동으로 구글 스타일이 적용된다. 이거 하나로 스타일 고민이 사라진다.

## 나에게 남기는 메모

구글 자바 스타일을 처음 접했을 때는 왜 굳이 이렇게까지 하나 싶었다. 특히 2칸 들여쓰기는 좀 좁아 보여서 별로였다. 하지만 쓰다 보니 익숙해지고, 무엇보다 이게 맞나 하는 고민이 사라진 게 제일 좋다.

처음에는 내 스타일을 고집하고 싶었지만, 여러 프로젝트에서 다양한 스타일을 오가다 보니 그냥 하나로 통일하는 게 정신건강에 이롭다는 걸 깨달았다. 이제는 개인 프로젝트도 전부 구글 스타일로 한다.

하다보니 진짜 편하다. 요즘엔 어지간한 건 다 플러그인 같은 게 잘 되어있어서, 아 이거 누가 안 해놨나 싶으면 이미 누가 다 해놨다. 개발자들이 귀찮은 걸 제일 싫어하니까.

스타일 가이드를 따르는 건 코드를 어떻게 작성할지에 대한 고민을 줄여준다. 그 시간에 더 중요한 문제를 해결하는 게 낫다. 무조건 구글 스타일이 최고라서가 아니라, 그냥 정해진 규칙을 따르는 게 편해서다.

적어도 이 플러그인 덕분에 코드 스타일에 대한 고민은 끝났다. 이제 다른 고민을 할 시간이다.
