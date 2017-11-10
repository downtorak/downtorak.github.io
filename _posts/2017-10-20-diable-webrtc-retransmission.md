---
layout:     post
title:      WebRTC의 패킷 재전송 기능 제어
date:       2017-10-20 17:00:00 +0900
categories: WebRTC
summary:    WebRTC의 패킷 재전송 기능을 끄는 방법
---

* content
{:toc}

### WebRTC의 재전송 기능
일반적으로 비디오를 실시간 전송할때는 UDP를 사용한다.
TCP를 사용하면 패킷 헤더가 커지는 것은 물론이고 패킷 손실이 발생했을 때
재전송을 시도하기 때문에 Delay가 증가한다.
데이터 손실이 일부 발생하더라도 낮은 Delay를 더 중요시하는
실시간 비디오 전송에서는 TCP가 적합하지 않다는 것이다.

WebRTC에서도 동일한 이유로(*아마도?*)
UDP를 기본적으로 사용한다(*TCP도 사용은 가능*).
그런데 WebRTC 내부를 보면 UDP를 사용하는데도 재전송 기능을 가지고 있다.
비디오 수신 측은 RTCP Feedback으로 손실된 RTP패킷의 Sequence Number를 보내고
송신 측은 손실된 RTP패킷을 재전송한다.

WebRTC를 써보면서 이 기능이 정말 필요한지는 의문이다.
네트워크 상태가 좋을 때에는 패킷 손실은 거의 나타나지 않고,
네트워크 상태가 나쁠 때에는 안그래도 낮은 대역폭에
재전송 패킷까지 대역폭을 차지하기 때문이다.
또, 재전송을 기다리다 Delay가 늘어나기도 한다.
당연히 패킷 한두개쯤 빠지는 어중간한 네트워크 상태라면 유용할 수 있다.

그래서 이 재전송 기능을 꺼보기로 했다.
재전송 기능은 기본으로 사용하게 되어있고, 외부에서 설정하는 API도 없다.
코드 내부에서 끌 수 있다.

### WebRTC의 재전송 끄기
`webrtc/media/engine/webrtcvideoengine2.cc` 파일을 보면 다음의 소스를 찾을 수 있다.
```cpp
void AddDefaultFeedbackParams(VideoCodec* codec) {
  codec->AddFeedbackParam(FeedbackParam(kRtcpFbParamCcm, kRtcpFbCcmParamFir));
  codec->AddFeedbackParam(FeedbackParam(kRtcpFbParamNack, kParamValueEmpty));   
  codec->AddFeedbackParam(FeedbackParam(kRtcpFbParamNack, kRtcpFbNackParamPli));
  codec->AddFeedbackParam(FeedbackParam(kRtcpFbParamRemb, kParamValueEmpty));
  codec->AddFeedbackParam(
      FeedbackParam(kRtcpFbParamTransportCc, kParamValueEmpty));
}
```

비디오 코덱에 Feedback 관련 설정을 추가하는 곳이다.
이 중 재전송과 관련된 `kRtcpFbParamNack`가 포함된 두 줄이다.
두 줄만 지워주면 재전송은 하지 않는다.

두 줄이 지워지면 SDP에 표시가 난다. 보통 WebRTC의 SDP에서 볼 수 있는
아래의 내용이 보이지 않게된다.
```no-highlight
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
```

당연한 얘기지만 보내는 쪽, 받는 쪽 모두 재전송 기능을 제공해야만
해당 기능이 가능하므로 서로 제공하는지 확인하기 위해 SDP에 적히는 것이다.

참고로 `Pli`는 Packet Loss Indication의 약자라고 한다.


