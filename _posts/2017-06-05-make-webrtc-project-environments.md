---
layout: post
title:  "WebRTC M59 소스 다운로드 및 GitLab 프로젝트 생성"
date:   2017-06-05 16:00:00 +0900
categories: WebRTC
---

* content
{:toc}


### WebRTC Android M59 소스 다운로드

<br>

#### 소스 디렉토리 생성
```sh
$ mkdir webrtc-checkout
$ cd webrtc-checkout
```

<br>

#### 소스 다운로드
```sh
$ fetch --nohooks webrtc_android
$ cd src
```
> Note:
> prerequisite software는 이미 깔려 있어서 설치 과정 생략.
> 나중에 다시 설치할 일이 있으면 추가하겠음.
> ([prerequisite software](https://webrtc.org/native-code/development/prerequisite-sw/) 설치 참조)

<br>

#### submodules 다운
```sh
$ gclient sync --with_branch_heads
```

<br>

#### M59 release로 코드 Checkout
```sh
$ git checkout -b My_M59 branch-heads/59
```
> Note:
> "-b new_branch_name" 옵션을 통해 Checkout과 동시에 새 branch를 만들 수 있음.
> 만들필요 없으면 빼도 무관.

<br>

#### submodules 동기화
```sh
$ gclient sync
```

<br>

### GitLab에 M59 프로젝트 생성

<br>

#### GitLab 사이트에서 프로젝트 생성
> Note:
> GitLab에 대해서는 나중에 다시 설치할 일이 있으면 추가하겠음.

<br>

#### 프로젝트 Clone
아직 빈프로젝트 상태.
```sh
$ git clone git@minwoo-android-dev:root/WebRTC_M59_Android.git
```

<br>

#### "WebRTC_M59_Android" 디렉토리 생성 확인
생겼으면 OK!

<br>

### M59 프로젝트 소스 등록
기존의 WebRTC Git을 그대로 올려도 되지만
난 깨끗하게 시작하고 싶어서 기존 Git은 다 지울꺼임. 내맘.

<br>

#### WebRTC Android M59 소스에서 기존 Git 관련 파일 제거
```sh
$ find ./webrtc-checkout/ -name ".git*" -exec rm -rf {} \;
```

실행 시, 나오는 에러(No such file or directory)는
디렉토리 지우고 나서 그안에 지우려던 파일들 때문으로 보임.

<br>

#### M59 소스 복사
"webrtc-checkout" 디렉토리에서 "WebRTC_M59_Android" 디렉토리로...
내 경우는 숨겨진 파일이 빠질 수 있으니까 걍 디렉토리를 이동

```sh
$ mv WebRTC_M59_Android/.git webrtc-checkout/src/  --> 결국 .git만 필요해
$ rm -rf WebRTC_M59_Android
$ mv webrtc-checkout/src WebRTC_M59_Android --> src 그대로 옮기면 경로가 길어져서 git 동작도 느리게함
```

<br>

#### 소스 정보 파일 복사
혹시 필요할지 몰라서 하는 거라 안해도 됨

```sh
$ mv webrtc-checkout/.gclient WebRTC_M59_Android
$ mv webrtc-checkout/.gclient_entries WebRTC_M59_Android
```

<br>

#### 출력 폴더는 Commit 대상에서 제외
```sh
$ echo "/out" > WebRTC_M59_Android/.gitignore
```

<br>

#### 최초 Commit
```sh
$ cd WebRTC_M59_Android
$ git add . --> "git add *" 은 히든파일 추가 안됨..조심
$ git commit -m "최초 소스 업로드"
```

<br>

#### repository 용량 줄이기
git gui가 강력하게 해달라해서 하는거

```sh
$ git gui
```

git gui 실행하면 다음과 같은 팝업 발생

> "This repository currently has approximately 272896 loose objects.
> <br> To maintain optimal performance it is strongly recommended that you compress the database.
> <br> Compress the database now?"

클릭 "Yes" 누르면 오래 걸림...
결국 "git gc" 하는거라 Console에서 해도 될듯..

<br>

#### 최초 Push
```sh
$ git push origin master
```

<br>
