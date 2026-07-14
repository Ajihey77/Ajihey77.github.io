---
title: "개발 환경 세팅기 2편 — 이 블로그 자체를 만든 이야기"
date: 2026-07-14
categories: [개발일지, 세팅]
tags: [github-pages, jekyll, chirpy, claude-code, 자동화]
---

## 코드보다 블로그가 먼저

1편 마지막에 "다음 편은 Spring Boot 스켈레톤"이라고 예고했는데, 사실 그 전에 삼천포로 한 번 빠졌다 — **지금 이 글을 쓰고 있는 블로그 자체를 만드는 이야기**다. 개발 일지를 남기고 싶다는 생각이 든 김에, 프로젝트 코드를 건드리기 전에 기록할 공간부터 만들기로 했다.

## GitHub Pages는 사실 Netlify다

블로그를 만든다길래 당연히 Netlify나 Vercel 계정을 새로 팔 줄 알았는데, 필요 없었다. 저장소 이름을 **`<내계정>.github.io`** 로 정확히 지으면, GitHub가 그 이름 자체를 "이건 내 Pages 사이트"라는 신호로 인식한다. 그리고 GitHub Actions(빌드) + GitHub Pages(호스팅)가 이미 GitHub 안에 다 내장되어 있어서, 외부 서비스를 하나도 안 거치고 push만으로 배포가 끝났다.

테마는 한국 개발 블로그에서 많이 쓰는 **Chirpy**로 골랐다. `chirpy-starter` 레포를 받아서 내 저장소에 새로 얹는 방식이라, Ruby나 Jekyll을 로컬에 설치할 필요도 없었다 — 마크다운 파일 쓰고 push하면 GitHub 서버가 알아서 빌드해준다.

## 삽질 1: 빌드는 됐는데 테마가 깨질 뻔했다

레포 만들 때 "Add a README" 체크를 했더니, GitHub가 자동으로 Pages를 켜면서 `legacy` 빌드 방식(GitHub 자체 내장 Jekyll 빌더)으로 설정해버렸다. 근데 Chirpy는 커스텀 빌드 스텝(젬 설치, 서브모듈 체크아웃 등)이 필요해서, `legacy` 방식으론 제대로 안 빌드된다.

```
gh api repos/<계정>/<계정>.github.io/pages -X PUT -f build_type=workflow
```

이 API 호출 한 줄로 "GitHub 기본 빌더" 대신 "내가 넣은 GitHub Actions 워크플로"를 쓰도록 바꿨다. 저장소 Settings → Pages → Source에서 클릭 한 번으로도 되는 걸 CLI로 한 것뿐인데, 확인 안 했으면 사이트가 이상하게 뜬 채로 몰랐을 뻔했다.

## 삽질 2: assets/lib 서브모듈

`.gitmodules` 파일이 있길래 봤더니 `assets/lib`이 서브모듈(폰트·JS 라이브러리 묶음)로 연결돼 있었다. 얕은 클론(`--depth 1`)으로 받아온 터라 이 폴더가 빈 채로 남아있었고, `git submodule add`로 다시 제대로 연결해줘야 했다. 빈 폴더가 이미 있으면 `submodule add`가 거부하길래 폴더 지우고 재시도 — 사소하지만 기억해둘 만한 패턴.

## 삽질 3: 커스텀 명령어가 갑자기 안 먹혔다

블로그 초안을 쓸 때마다 매번 설명하기 귀찮아서 `/blog-draft`라는 나만의 명령어를 만들기로 했다. `~/.claude/commands/blog-draft.md`에 마크다운으로 지시사항을 적어뒀는데, 막상 쳐보니 "없다"는 반응이 왔다.

알고 보니 Claude Code의 **커스텀 명령어 체계가 최근 Skills로 통합**됐고, `commands/` 경로는 세션을 재시작해야만 인식되는 반면 `~/.claude/skills/<이름>/SKILL.md` 경로는 세션 중에도 실시간으로 감지된다고 한다. 파일을 옮기고 나니 재시작 없이 바로 됐다.

```yaml
---
description: "..."
disable-model-invocation: true   # 내가 직접 호출할 때만 실행, AI가 알아서 트리거하지 않도록
---
```

`disable-model-invocation: true`를 넣은 게 포인트였다 — 이 스킬은 코드를 건드리는 게 아니라 콘텐츠를 만드는 거라, 내가 원할 때만 실행돼야지 AI가 알아서 "지금 블로그 쓸 때다"라고 판단하면 곤란하다.

## 정리

블로그 하나 여는 데 진짜 필요했던 건 계정 가입이 아니라 **저장소 이름 규칙 하나, 빌드 방식 설정 하나, 서브모듈 하나**였다.

## 다음 편 예고

이제 진짜 다음이다 — **Spring Boot 스켈레톤 만들기.** IntelliJ를 처음 열어보는 이야기.
