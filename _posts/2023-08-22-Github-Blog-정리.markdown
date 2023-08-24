---
layout: single
title: "Github Blog 정리"
date: 2023-08-22 14:57
last_modified_at: 2023-08-24 13:48
search : true
categories: Github
gallery:
  - url: /images/Github-Blog-정리/1.jpg
    image_path: /images/Github-Blog-정리
    alt: "placeholder image 1"
    title: "Image 1 title caption"
---

### Github Blog 만드는 법

### New repository 만들기

```yaml
New repository 생성
꼭 사용자이름.github.io로 할 것
Add a README file 체크
```
<!-- ```yaml
gallery:
  - url: /images/Github-Blog-정리/1.jpg
    image_path: /images/Github-Blog-정리/1.jpg
    # alt: "placeholder image 1"
    title: "Image 1 title caption"
``` -->
<!-- ```liquid
{% raw %}{% include gallery caption="This is a sample gallery with **Markdown support**." %}{% endraw %}
``` -->
{% include gallery caption="This is a sample gallery with **Markdown support**." %}

### Clone

```yaml
Git을 설치하고
cmd에서 repository 주소 복사 후,
원하는 경로에 clone 하기
```

### 블로그 생성
cmd에서 clone한 경로에서 할 것

```yaml
echo "Hello World" > index.html
git add --all
git commit -m "Initial commit" (자신이 원하는 메세지)
git push -u origin main
```
기본 블로그 생성 후 "사용자이름.github.io" 접속하여 블로그가 잘 생성되었는지 확인하기!

### 블로그 테마 적용
clone한 파일에서 index.html, README.md 삭제하고 진행하기?
Ruby 설치하고 진행할것!
Ruby 설치시에 개발툴 깔기(3번)

```yaml
gem install Jekyll (gem update하라고 하면 하기)
gem install Jekyll bundler
```

폴더 안 내용 모두 제거 후 원하는 테마 zip 파일 다운로드
압축 해제 후 Clone한 파일에 덮어쓰기

_config.yml 파일 수정하기! 
테마와 파일 수정 모두 이 곳 참고
(https://honbabzone.com/jekyll/start-gitHubBlog/)

### 로컬서버에서 잘 적용되었는지 확인하기
127.0.0.1:4000에 접속하여 확인하기

```yaml
jekyll new ./
bundle exec jekyll serve
```

### 블로그에 최종 적용
로컬서버에서 잘 적용되었을 경우 git에 push
commit 메세지는 본인이 결정하는 것

```yaml
git add .
git commit -m "Hello ENF"
git push
```

사용자이름.github.io에 접속하여 잘 적용되었는지 확인하기!

### 번외

### 블로그 업데이트
원래의 블로그 파일은 백업 후,
127.0.0.1:4000에서 잘 적용되는지 확인하고
업데이트 하는 것 추천!

```yaml
bundle install
bundle exec jekyll serve
```

```yaml
git add .
git commit -m "커밋 메세지"
git push
```