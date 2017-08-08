---
layout: post
title:  "WebRTC Android 빌드"
date:   2017-06-05 17:34:00 +0900
categories: WebRTC
---

* content
{:toc}


### WebRTC Android 빌드

<br>

#### 프로젝트 생성
```sh
$ gn gen out/debug --args='target_os="android" target_cpu="arm" is_debug=true'
$ gn gen out/release --args='target_os="android" target_cpu="arm" is_debug=false'
```

<br>

#### Compile
전체 Compile
```sh
$ ninja -C out/debug
$ ninja -C out/release
```

libjingle_peerconnection_so 만 Compile
```sh
$ ninja -C out/debug libjingle_peerconnection_so
$ ninja -C out/release libjingle_peerconnection_so
```

> Note: 부분 Compile 가능한 목록은 out/debug/build.ninja 파일에서 확인

<br>

### AppRTCMobile 앱을 Android Studio로 빌드
AppRTCMobile은 WebRTC에서 제공하는 안드로이드 Example 앱.
AppRTCMobile의 gradle 프로젝트 생성은 최근에 추가되었음.

<br>

#### AppRTCMobile Compile
```sh
$ ninja -C out/debug AppRTCMobile
```

<br>

#### Android Studio용 gradle 프로젝트 생성
```sh
$ ./build/android/gradle/generate_gradle.py --output-directory $PWD/out/debug --target "//webrtc/examples:AppRTCMobile" --use-gradle-process-resources --split-projects
```

> Note: `--use-gradle-process-resources --split-projects' 이 두 옵션의 효과가 정확히 무엇인지는 모르겠음.
> 어쨌든 두 옵션을 빼면 모든 소스가 하나의 모듈에 몰려서 생성됨. 근데 빌드가 안됨...!?

<br>

#### Linux Android Studio에서 열기
File-Open에서 "out/debug/gradle" 선택하고 OK!

<br>

#### Windows Android Studio에서 열기
> Note: 내 경우엔 WebRTC는 Linux 서버에서 빌드하고
> Android Studio는 Windows에서 사용하기 때문에 이 과정이 필요했음.
> 이렇게까지 해야하나 싶으면 안해도됨.

준비사항
1. Linux에서 Samba로 WebRTC 소스 폴더를 공유
2. Windows에서 "컴퓨터-네트워크 드라이브 연결"로 WebRTC 소스 폴더를 적당한 드라이브로 지정
> Note: Android Studio에서 공유폴더를 직접 지정할 수 없으나 네트워크 드라이브로는 접근 가능.

<br>

여기서 그냥 "out/debug/gradle"을 열게되면 빌드 시 에러발생.
이유는 Project Name에 따라 '>'가 들어간 파일을 만들려고해서...
Linux에서는 '>'가 들어간 파일을 만들 수 있으나 Windows에서는 불가.

그래서 Project Name에 '>'가 들어가지 않도록 "./build/android/gradle/generate_gradle.py" 파일을 수정.
ProjectName() 함수에서 '>'를 '.'으로 변경.

Line 177:

수정 전
```python
  def ProjectName(self):
    """Returns the Gradle project name."""
    return self.GradleSubdir().replace(os.path.sep, '>')
```

수정 후
```python
  def ProjectName(self):
    """Returns the Gradle project name."""
    return self.GradleSubdir().replace(os.path.sep, '.')
```

<br>

#### Android Studio 2.3에서 열기
Android Studio 2.3에서 빌드 시, "dummy.package" 관련 에러발생.
package 이름에 "package"가 들어가면 안되는가봄.

해당 버그는 M29 release 이후에 fix되었음. ([참조](https://codereview.chromium.org/2827923002))

<br>

#### Project Open 실패
하다보니 어쩌다 Project를 Open하면 Gradle 위치를 못찾겠다면서 위치를 설정하는 창이 팝업.
어거지로 `Finish`를 해보면 `Minimum supported gradle version is 3.3...` 이런 에러가 발생.
이때 보면 일부 모듈을 로드하지 못했다는 알림이 뜨는데,
Android Studio가 최신이 아니면 일부 모듈을 로드하지 못하는 현상이 있는 것으로 보임.
알림 클릭해서 업데이트를 수행하고 나서 Project를 Open하면 잘 열림.
