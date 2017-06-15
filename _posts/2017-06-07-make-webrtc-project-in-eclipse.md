---
layout: post
title:  "WebRTC 소스 Eclipse에서 보기"
date:   2017-06-07 12:51:00 +0900
categories: WebRTC
---

* content
{:toc}


WebRTC 소스 분석을 편하게 하기위해 Eclipse를 사용한다.
즉, WebRTC 소스로 Project를 구성해서 Eclipse의 Indexer 기능을 활용하기 위함이다.
([참고](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_eclipse_dev.md))

<br>

### Eclipse 설치 및 설정
Eclipse는 C++용 "Eclipse CDT"를 다운로드.
현재 내가 받은것은 Neon3 x64.
적당한데 압출풀고 다음의 설정 변경을 수행.

<br>

#### Heap 크기 조정
메모리 부족 현상을 막기위한 수정

eclipse/eclipse.ini
Line 19:

수정 전
```no-highlight
-Xms256m
-Xmx1024m
```

수정 후
```no-highlight
-Xms256m
-Xmx3072m
```

<br>

#### Hyperlink detection 끄기
Eclipse 실행 후 workspace는 적당히 잡고 시작.
Hyperlink detection을 끄기위해 다음을 수행.

```no-highlight
Windows -> Preferences -> General -> Editors -> Text Editors -> Hyperlinking
-> "Enable on demend hyperlink style navigation" 끄기
```

<br>

#### Build automatically 끄기
Build automatically를 끄기위해 다음을 수행.

```no-highlight
Windows -> Preferences -> General -> Workspace
-> "Build automatically" 끄기
```

<br>

#### Indexer 설정 변경
```no-highlight
Windows -> Preferences -> C/C++ -> Indexer
-> "Automatically update the index" 끄기
```

```no-highlight
Windows -> Preferences -> C/C++ -> Indexer
-> Build configuration for the indexer
-> "Use active build configuration" 선택
```

```no-highlight
Windows -> Preferences -> C/C++ -> Indexer
-> Cache limits -> Index database cache -> Limit relative to the maximum heap size
-> 20%로 변경
```

<br>

### Eclipse에 WebRTC Project 생성

<br>

#### Project 생성
Project 생성을 위해 다음을 수행.

```no-highlight
File -> New -> Makefile Project with Existing Code
-> 이름, 경로, Toolchain 설정하고 "Finish".
```

내 설정은 다음과 같음.

| Project Name                   | WebRTC_M59_Android       |
| Existing Code Location         | (...)\WebRTC_M59_Android |
| Languages                      | [v] C, [v] C++           |
| Toolchain for Indexer Settings | Linux GCC                |

<br>

Project 생성 후 Background로 도는게 생기는데, 괜히 힘빼지 않게 멈춰주자. 아직 할게 많다.

<br>

#### C++11 활성화
WebRTC는 C++11을 사용하므로 설정이 필요. 이를 위해 다음을 수행.

```no-highlight
Project Explorer에서 Project 이름 우클릭 -> Properties
-> C/C++ Build -> Settings -> Tool Settings
-> GCC C++ Compiler -> Miscellaneous
-> Other flags에 "-std=C++11" 추가
```

```no-highlight
Project Explorer에서 Project 이름 우클릭 -> Properties
-> C/C++ General -> Paths and Symbols -> Symbols -> GNU C++ -> Add...
-> Name: __cplusplus, Value: 201103L
```

<br>

#### 필요한 소스만 보이도록 설정
Indexing할 필요없는 파일, 폴더를 Filtering하기 위해 다음을 수행.

```no-highlight
Project Explorer에서 Project 이름 우클릭 -> Properties
-> Resource -> Resource Filters -> Add Filter...
```

다음의 Filter들을 추가.

필요한 파일 타입(c, cc, cpp, h)만 포함.

| Filter type    | Include only                               |
| Applies to     | Files                                      |
|                | [v] All children (recursive)               |
| Filter Details | Name                                       |
|                | matches                                    |
|                | `.*\.(c|cc|cpp|h)`                         |
|                | [v] regular expression                     |

<br>

unittest 파일들은 제외.

| Filter type    | Exclude all                                |
| Applies to     | Files                                      |
|                | [v] All children (recursive)               |
| Filter Details | Name                                       |
|                | matches                                    |
|                | `.*_unittest.cc`                           |
|                | [v] regular expression                     |

<br>

불필요한 폴더(.git, third_party/android_tools, third_party/WebKit) 제외.

| Filter type    | Exclude all                                |
| Applies to     | Folders                                    |
|                | [v] All children (recursive)               |
| Filter Details | Name                                       |
|                | Project Relative Path                      |
|                | `\.git|build|buildtools|testing|third_party/(android_tools|WebKit|webrtc_overrides)` |
|                | [v] regular expression                     |

<br>

#### STL Header 파일들 Include
Windows에는 STL Header 파일들이 없어서 이것들에 대한 Include가 필요.
나는 Cygwin이나 MinGW를 고려하다가 걍 Android Studio에 NDK에 STL Header 파일들을 Include하기로 결정.

우선 Android Studio에서 NDK 설치.
```no-highlight
File -> Settings -> Appearance & Behavior -> System Settings -> Android SDK
-> SDK Tools -> NDK 체크 -> Apply 클릭
```

Eclipse Project에 STL Header 파일들 Include.
```no-highlight
Project Explorer에서 Project 이름 우클릭 -> Properties
-> C/C++ General -> Path and Symbols -> Include -> GNU C++ 클릭
-> Add... 클릭
```

상기의 방법으로 아래의 경로들을 추가.
```no-highlight
(Android SDK Path)\ndk-bundle\sources\cxx-stl\gabi++\include
(Android SDK Path)\ndk-bundle\sources\cxx-stl\gnu-libstdc++\4.9\include
(Android SDK Path)\ndk-bundle\sources\cxx-stl\gnu-libstdc++\4.9\libs\armeabi-v7a\include
(Android SDK Path)\ndk-bundle\sources\cxx-stl\llvm-libc++\include
(Android SDK Path)\ndk-bundle\sources\cxx-stl\system\include
```

Android SDK의 경로는 Default 경로라면 다음과 같다.
 
`C:\Users\(사용자이름)\AppData\Local\Android\sdk\`

설정하면 Indexer Rebuild 할꺼냐고 물어보는데 어차피 아래에서 할거니 해도 그만 안해도 그만.

<br>

#### Project Refresh 및 Indexing
```no-highlight
Project Explorer에서 Project 이름 우클릭 -> Refresh 클릭
```

```no-highlight
Project Explorer에서 Project 이름 우클릭 -> Index -> Rebuild 클릭
```

<br>

### 에필로그
이렇게 해도 빨간줄이 많이 보이지만 나는 이정도로 만족.

<br>









