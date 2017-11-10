---
layout:     post
title:      WebRTC의 패킷 FEC 기능 제어
date:       2017-10-23 15:00:00 +0900
categories: WebRTC
summary:    WebRTC의 패킷 FEC 기능을 끄는 방법
---

* content
{:toc}

### WebRTC의 FEC 기능
FEC는 Forward Error Correction의 약자로
송신측이 전송할 데이터에 에러 검출 및 정정에 필요한 추가 데이터를 보내고,
수신측이 추가 데이터를 통해 에러 검출 및 정정을 수행한다.
WebRTC에서는 비디오/오디오 데이터에 대해 FEC를 기본으로 적용하고 있다.

WebRTC에 적용된 FEC는
`RED/ULPFEC`([RFC 5109](https://tools.ietf.org/html/rfc5109))이다.
WebRTC에서 비디오/오디오는 RTP패킷으로 전송하기 때문에
RTP 규격에 따른 FEC를 적용하고 있다.

`RED`는 Redundant Coding의 약자로 RTP에서 추가 데이터를 전송하는 규약이고,
`ULP`는 Uneven Level Protection의 약자로 FEC 알고리즘 중 하나이다.
(*깊게는 나도 몰라서..*)

FEC는 불안정한 네트워크 상황에서 유용하지만 네트워크가 안정적일 경우에도
추가 데이터에 따른 대역폭 사용과 에러 검출을 위한 연산량 증가를 요한다.

그래서 이 FEC 기능도 한번 꺼보기로 한다.
FEC 기능도 재전송 기능처럼 기본으로 사용하게 되어있고,
외부에서 설정하는 API도 없다. 코드 내부에서만 끌 수 있다.

### WebRTC의 FEC 끄기
`webrtc/media/engine/internalencoderfactory.cc` 파일을 보면 다음의 소스를 찾을 수 있다.
```cpp
InternalEncoderFactory::InternalEncoderFactory() {
  supported_codecs_.push_back(cricket::VideoCodec(kVp8CodecName));
  if (webrtc::VP9Encoder::IsSupported())
    supported_codecs_.push_back(cricket::VideoCodec(kVp9CodecName));
  if (webrtc::H264Encoder::IsSupported()) {
    cricket::VideoCodec codec(kH264CodecName);
    // TODO(magjed): Move setting these parameters into webrtc::H264Encoder
    // instead.
    codec.SetParam(kH264FmtpProfileLevelId,
                   kH264ProfileLevelConstrainedBaseline);
    codec.SetParam(kH264FmtpLevelAsymmetryAllowed, "1");
    supported_codecs_.push_back(std::move(codec));
  }

  supported_codecs_.push_back(cricket::VideoCodec(kRedCodecName));
  supported_codecs_.push_back(cricket::VideoCodec(kUlpfecCodecName));

  if (IsFlexfecAdvertisedFieldTrialEnabled()) {
    cricket::VideoCodec flexfec_codec(kFlexfecCodecName);
    // This value is currently arbitrarily set to 10 seconds. (The unit
    // is microseconds.) This parameter MUST be present in the SDP, but
    // we never use the actual value anywhere in our code however.
    // TODO(brandtr): Consider honouring this value in the sender and receiver.
    flexfec_codec.SetParam(kFlexfecFmtpRepairWindow, "10000000");
    flexfec_codec.AddFeedbackParam(
        FeedbackParam(kRtcpFbParamTransportCc, kParamValueEmpty));
    flexfec_codec.AddFeedbackParam(
        FeedbackParam(kRtcpFbParamRemb, kParamValueEmpty));
    supported_codecs_.push_back(flexfec_codec);
  }
}
```

WebRTC에서 제공하는 내부 Encoder를 추가하는 곳이다.
이 중 `RED/ULPFEC`와 관련된 것은 중간의 두 줄이다.
```cpp
  supported_codecs_.push_back(cricket::VideoCodec(kRedCodecName));
  supported_codecs_.push_back(cricket::VideoCodec(kUlpfecCodecName));
```
이 두 줄을 지워주면 내부 Encoder 목록에서 `RED`와 `ULPFEC`는 빠지게 된다.

두 줄이 지워지면 마찬가지로 SDP로 확인할 수 있다.
보통 WebRTC의 SDP에서 볼 수 있는 아래의 내용이 보이지 않게된다.
```no-highlight
a=rtpmap:102 red/90000
a=rtpmap:127 ulpfec/90000
```



