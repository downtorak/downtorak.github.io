---
layout: post
title:  "AppRTC constraint 설정"
date:   2017-06-14 14:44:00 +0900
categories: WebRTC
---

* content
{:toc}

### AppRTC 소개
AppRTC([https://appr.tc](https://appr.tc))는 Google에서 제공하는 WebRTC 기반 1:1 화상통화 사이트다.

![apprtc_main]({{ site.url }}/assets/2017-06-14-set-apprtc-parameters.1.apprtc-main.jpg)

room name을 입력하고 JOIN하면 같은 room name을 입력한 사람과 화상통화를 시작한다.

<br>

### AppRTC의 활용
WebRTC를 이용해 개발하는 입장에서 AppRTC가 참 고마운 이유는
WebRTC Android의 example 앱이 동일한 시그널링 서버를 사용한다는 것.

즉, 스마트폰의 WebRTC example 앱과 PC의 웹브라우저로 실행한 AppRTC 사이트 간에 화상통화가 가능하다.

스마트폰 한개만 있어도 PC와 연결하여 테스트가 가능하며
Chrome을 사용하면 webrtc-internel를 통한 분석도 가능하다.
> Note: webrtc-internel에 대해서는 나중에... 시간나면...

여기까지는 사실 서론이었고 얘기하고자 하는 것은 이 AppRTC에서도 constraint를 조정할 수 있다는 것이다.

<br>

### AppRTC constraint 설정
AppRTC 서버도 오픈 소스로 [여기](https://github.com/webrtc/apprtc)에서 확인 가능하다.
> Note: AppRTC 서버 구축하는 것도 나중에... 다시 할일있으면...

소스 중 `apprtc.py`를 보면 다음과 같은 의미심장한 주석을 발견할 수 있다.
```no-highlight
  # Use "audio" and "video" to set the media stream constraints. Defined here:
  # http://goo.gl/V7cZg
  #
  # "true" and "false" are recognized and interpreted as bools, for example:
  #   "?audio=true&video=false" (Start an audio-only call.)
  #   "?audio=false" (Start a video-only call.)
  # If unspecified, the stream constraint defaults to True.
  #
  # To specify media track constraints, pass in a comma-separated list of
  # key/value pairs, separated by a "=". Examples:
  #   "?audio=googEchoCancellation=false,googAutoGainControl=true"
  #   (Disable echo cancellation and enable gain control.)
  #
  #   "?video=minWidth=1280,minHeight=720,googNoiseReduction=true"
  #   (Set the minimum resolution to 1280x720 and enable noise reduction.)
  #
  # Keys starting with "goog" will be added to the "optional" key; all others
  # will be added to the "mandatory" key.
  # To override this default behavior, add a "mandatory" or "optional" prefix
  # to each key, e.g.
  #   "?video=optional:minWidth=1280,optional:minHeight=720,
  #           mandatory:googNoiseReduction=true"
  #   (Try to do 1280x720, but be willing to live with less; enable
  #    noise reduction or die trying.)
  #
  # The audio keys are defined here: talk/app/webrtc/localaudiosource.cc
  # The video keys are defined here: talk/app/webrtc/videosource.cc
```

url 주소로 constraint를 설정할 수 있다는 것.

<br>

### AppRTC constraint 설정 예제
AppRTC의 constraint를 설정하는 간단한 예제들. (room name은 "10000")

<br>

오디오만 사용
```no-highlight
https://appr.tc/r/10000?audio=true&video=false
```

<br>

비디오만 사용
```no-highlight
https://appr.tc/r/10000?audio=false&video=true
```

<br>


비디오 최저 해상도 지정
```no-highlight
https://appr.tc/r/10000?audio=true&video=minminWidth=1280,minHeight=720
```

<br>

여기까지는 WebRTC 표준에 정의된 constraint를 조정하는 것으로 Chrome, Firefox에서 모두 적용된다.
다음 예제들은 `goog` 접두어가 붙은 옵션들로 Chrome에서만 가능하다.

<br>

비디오 노이즈 제거 켜기 (나도 써본적은 없음)
```no-highlight
https://appr.tc/r/10000?audio=true&video=googNoiseReduction=true
```

<br>

오디오 EchoCancellation 끄기 (나도 써본적은 없음)
```no-highlight
https://appr.tc/r/10000?audio=googEchoCancellation&video=true
```

<br>

이렇게 설정할 수 있는 `goog`옵션들은 WebRTC 소스 코드 중
`webrtc/api/mediaconstraintsinterface.cc` 파일에서 확인할 수 있다.

<br>

### AppRTC constraint 설정의 한계
WebRTC 비디오의 Dynamic Adaptation은 좋은 기능이지만 일정한 환경에서 테스트해야할 때는 걸림돌이다.
다시말해 Bitrate, Resolution, Framerate가 계속 바뀌는 상황에서 무엇이 문제를 발생시켰는지 찾기 어렵다는 것이다.

이 AppRTC constraint 설정을 열심히 찾았던 이유는 Dynamic Adaptation 기능을 OFF 하기 위함이었다.
특히 Resolution이 바뀌면 수신 측의 Decoder도 Reset되기 때문에 Resolution이라도 유지 시키고 싶었다.

WebRTC 소스를 분석해 보면 `googCpuOveruseDetection`을 `false`로 설정했을때 Resolution의 변경을 막을 수 있는데
아쉽게도 AppRTC constraint 설정으로 `googCpuOveruseDetection` 값이 변경되지 않는 것으로 보인다.

누가 알면 좀 알려죠...




