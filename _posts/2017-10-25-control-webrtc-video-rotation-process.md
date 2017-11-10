---
layout:     post
title:      WebRTC의 Video Rotation 방식 제어
date:       2017-10-25 11:00:00 +0900
categories: WebRTC
summary:    WebRTC의 Video Rotation 처리 방식을 제어하는 방법
---

* content
{:toc}

### Video Rotation
PC간의 비디오 전송에는 Rotation이 큰 의미가 없다.
일반적인 PC환경에서는 카메라가 고정되어 있기 때문이다.

하지만 모바일환경에서는 카메라가 얼마든지 다른 방향으로 회전할 수 있기 때문에
Rotation이 중요할 수 밖에 없다. 이러한 Rotation에 대한 처리가 이루어지지 않으면
옆으로 누워지거나 180도 돌아간 비디오 화면을 보게 될 것이다.

<br>

### WebRTC의 Video Rotation 처리
WebRTC의 SDP를 보면 아래와 같은 내용을 찾을 수 있다.
```no-highlight
a=extmap:4 urn:3gpp:video-orientation
```

3GPP에서 정의한 Video Orientation 관련 RTP Extension을 지원한다는 의미이다.
(*3GPP TS 26.114 참고, [여기](https://www.ietf.org/mail-archive/web/rtcweb/current/msg08977.html)도 참고*)

즉, 송신측이 수신측에 Rotation 정보를 주고 수신측은 Rotation 정보에 따라
수신 영상을 회전 시키는 방식이다.

![with-rtp-extension-zero-degree]({{ site.url }}/assets/2017-10-25-control-webrtc-video-rotation-process/01.with-rtp-extension-zero-degree.png)
![with-rtp-extension-ninety-degree]({{ site.url }}/assets/2017-10-25-control-webrtc-video-rotation-process/02.with-rtp-extension-ninety-degree.png)

위의 그림은 RTP Extension을 통해 Rotation 정보를 전달했을 때
Rotation 처리과정을 도식화 한 것이다.
중요하게 볼 것은 회전에 따라 해상도가 변경되지 않았고,
수신측에서 화면에 표시할 때 회전을 적용했다는 것이다.

![without-rtp-extension-zero-degree]({{ site.url }}/assets/2017-10-25-control-webrtc-video-rotation-process/03.without-rtp-extension-zero-degree.png)
![without-rtp-extension-ninety-degree]({{ site.url }}/assets/2017-10-25-control-webrtc-video-rotation-process/04.without-rtp-extension-ninety-degree.png)

위의 그림은 Rotation 정보를 전달하지 않은
Rotation 처리과정을 도식화 한 것이다.
송신측에서 미리 회전하여 전송함으로써 수신측은 받은대로 화면에 표시한다.
어쩌면 이것도 괜찮은 방법으로 보일 수 있지만 그렇지 않다.

회전할 때마다 해상도가 변경되기 때문에
송신측의 Encoder와 수신측의 Decoder 모두에 해상도 변경에 따른 처리가
추가로 필요하게된다. 이 처리는 심하면 Encoder/Decoder 재시작일 수도 있다.

WebRTC에서는 기본으로 Rotation 정보를 전달하는 방식을 사용하고 있고,
Rotation 정보를 전달하지 않는 방식에 대해서도 구현되어있다.
Video Orientation 관련 RTP Extension을 지원하지 않는 WebRTC Client와의
영상 송수신을 위함이다.

<br>

### WebRTC의 Video Rotation 처리 방식 변경
앞서 설명한 바와 같이
Rotation 정보를 전달하는 방식을 쓸 수 있다면 사용하는 것이 좋다.
그래도 한 번 강제로 Rotation 정보를 전달하지 않는 방식을 써보기로 한다.

`webrtc/media/engine/webrtcvideoengine2.cc` 파일을 보면 다음의 소스를 찾을 수 있다.
```cpp
RtpCapabilities WebRtcVideoEngine2::GetCapabilities() const {
  RtpCapabilities capabilities;
  capabilities.header_extensions.push_back(
      webrtc::RtpExtension(webrtc::RtpExtension::kTimestampOffsetUri,
                           webrtc::RtpExtension::kTimestampOffsetDefaultId));
  capabilities.header_extensions.push_back(
      webrtc::RtpExtension(webrtc::RtpExtension::kAbsSendTimeUri,
                           webrtc::RtpExtension::kAbsSendTimeDefaultId));
  capabilities.header_extensions.push_back(
      webrtc::RtpExtension(webrtc::RtpExtension::kVideoRotationUri,
                           webrtc::RtpExtension::kVideoRotationDefaultId));
  capabilities.header_extensions.push_back(webrtc::RtpExtension(
      webrtc::RtpExtension::kTransportSequenceNumberUri,
      webrtc::RtpExtension::kTransportSequenceNumberDefaultId));
  capabilities.header_extensions.push_back(
      webrtc::RtpExtension(webrtc::RtpExtension::kPlayoutDelayUri,
                           webrtc::RtpExtension::kPlayoutDelayDefaultId));
  return capabilities;
}
```

WebRTC에서 지원하는 RTP Extension 목록을 획득하는 곳이다.
이 중 Rotation와 관련된 것은 다음과 같다.
```cpp
  capabilities.header_extensions.push_back(
      webrtc::RtpExtension(webrtc::RtpExtension::kVideoRotationUri,
                           webrtc::RtpExtension::kVideoRotationDefaultId));
```
해당 라인을 지워주면 RTP Extension 목록에서 Rotation관련된 것은 빠지게 되고,
Rotation 정보를 전달하지 않는 방식만 사용하게 된다.


