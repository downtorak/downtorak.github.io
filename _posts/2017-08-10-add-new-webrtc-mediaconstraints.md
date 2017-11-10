---
layout:     post
title:      WebRTC에 Custom Constraint 추가하기
date:       2017-08-10 17:00:00 +0900
categories: WebRTC
summary:    WebRTC에 Custom Constraint를 추가하는 방법
---

* content
{:toc}

### Constraint 입력 분석
WebRTC Android 소스에 기본 제공되는 Sample App `AppRTCMobile`에서 Constraints 입력부(webrtc/examples/androidapp/src/org/appspot/apprtc/PeerConnectionClient.java)중 일부를 보면 다음과 같다.

```java
    // Create audio constraints.
    audioConstraints = new MediaConstraints();
    // added for audio performance measurements
    if (peerConnectionParameters.noAudioProcessing) {
      Log.d(TAG, "Disabling audio processing");
      audioConstraints.mandatory.add(
          new MediaConstraints.KeyValuePair(AUDIO_ECHO_CANCELLATION_CONSTRAINT, "false"));
      audioConstraints.mandatory.add(
          new MediaConstraints.KeyValuePair(AUDIO_AUTO_GAIN_CONTROL_CONSTRAINT, "false"));
      audioConstraints.mandatory.add(
          new MediaConstraints.KeyValuePair(AUDIO_HIGH_PASS_FILTER_CONSTRAINT, "false"));
      audioConstraints.mandatory.add(
          new MediaConstraints.KeyValuePair(AUDIO_NOISE_SUPPRESSION_CONSTRAINT, "false"));
    }
    if (peerConnectionParameters.enableLevelControl) {
      Log.d(TAG, "Enabling level control.");
      audioConstraints.mandatory.add(
          new MediaConstraints.KeyValuePair(AUDIO_LEVEL_CONTROL_CONSTRAINT, "true"));
    }
```

Constraint는 String 타입의 key와 String 타입의 value로 구성된 `KeyValuePair`객체로 입력된다. 이렇게 입력된 Constraint는 C++ Level의 MediaConstraintsInterface 객체로 변환되어 저장된다.
`webrtc/api/mediaconstraintsinterface.h` 파일을 보면 다음의 소스를 찾을 수 있다.
```cpp
class MediaConstraintsInterface {
 public:
  struct Constraint {
    Constraint() {}
    Constraint(const std::string& key, const std::string value)
        : key(key), value(value) {
    }
    std::string key;
    std::string value;
  };

  class Constraints : public std::vector<Constraint> {
   public:
    bool FindFirst(const std::string& key, std::string* value) const;
  };

  virtual const Constraints& GetMandatory() const = 0;
  virtual const Constraints& GetOptional() const = 0;
```

입력받은 모든 Constraint는 지원하든 않든 일단 key와 value로 vector에 저장된다.
저장된 Constraint는 필요로하는 곳에서 파싱되서 사용된다.

따라서 Constraint를 파싱하고 사용하는 부분을 추가함으로써 새로 정의한 Constraint를 사용할 수 있다.

여기서는 WebRTC의 Echo Cancellation Module의 세부설정을 변경하는 Constraint를 추가해보겠다.
WebRTC Android에는 Echo Cancellation Module의 AECM 모드를 사용한다(참고: WebRtcVoiceEngine::ApplyOptions()).
이 AECM 모드의 세부설정 중에 RoutingMode라는 것이 있는데 기본값은 `kSpeakerphone`이다.
이 값을 Constraint를 통해 `kLoudSpeakerphone`으로 변경해보자.

<br>

### Custom Constraint 파싱부 추가
Constraint의 key 변수는 `webrtc/api/mediaconstraintsinterface.h`에 정의되어 있다.
```cpp
  static const char kHighpassFilter[];  // googHighpassFilter
  static const char kTypingNoiseDetection[];  // googTypingNoiseDetection
  static const char kAudioMirroring[];  // googAudioMirroring
  static const char
      kAudioNetworkAdaptorConfig[];  // goodAudioNetworkAdaptorConfig

  // (jominwoo): My Custom constraint keys for Audio Constraints.
  static const char kAECMLoudSpeakerphone[];  // customAECMLoudSpeakerphone

  // Google-specific constraint keys for a local video source
  static const char kNoiseReduction[];  // googNoiseReduction
```

중간에 보이는 것처럼 `kAECMLoudSpeakerphone`이라는 key 변수를 추가했다.

다음으로 `webrtc/api/mediaconstraintsinterface.cc`에 key 값을 추가한다.
```cpp

const char MediaConstraintsInterface::kHighpassFilter[] =
    "googHighpassFilter";
const char MediaConstraintsInterface::kTypingNoiseDetection[] =
    "googTypingNoiseDetection";
const char MediaConstraintsInterface::kAudioMirroring[] = "googAudioMirroring";
const char MediaConstraintsInterface::kAudioNetworkAdaptorConfig[] =
    "googAudioNetworkAdaptorConfig";

// (jominwoo): My Custom constraint keys for Audio Constraints.
const char MediaConstraintsInterface::kAECMLoudSpeakerphone[] =
    "customAECMLoudSpeakerphone";

// Google-specific constraint keys for a local video source (getUserMedia).
const char MediaConstraintsInterface::kNoiseReduction[] = "googNoiseReduction";
```

key 값으로 `customAECMLoudSpeakerphone`을 추가했다.

정의한 Constraint는 Audio 관련 설정이기 때문에 AudioOptions 객체에 추가하면 된다.
AudioOptions 객체는 `webrtc/media/base/mediachannel.h`에 정의되어 있다.
```cpp
struct AudioOptions {
  void SetAll(const AudioOptions& change) {
    ...
    SetFrom(&aecm_generate_comfort_noise, change.aecm_generate_comfort_noise);
    SetFrom(&aecm_loud_speakerphone, change.aecm_loud_speakerphone);
    SetFrom(&adjust_agc_delta, change.adjust_agc_delta);
    ...
  }
  bool operator==(const AudioOptions& o) const {
    return echo_cancellation == o.echo_cancellation &&
           ...
           aecm_generate_comfort_noise == o.aecm_generate_comfort_noise &&
           aecm_loud_speakerphone == o.aecm_loud_speakerphone &&
           experimental_agc == o.experimental_agc &&
           ...
  }
  bool operator!=(const AudioOptions& o) const { return !(*this == o); }
  std::string ToString() const {
    std::ostringstream ost;
    ost << "AudioOptions {";
    ...
    ost << ToStringIfSet("comfort_noise", aecm_generate_comfort_noise);
    ost << ToStringIfSet("aecm_loud_speakerphone", aecm_loud_speakerphone);
    ost << ToStringIfSet("agc_delta", adjust_agc_delta);
    ...
  }
  ...
  rtc::Optional<bool> aecm_generate_comfort_noise;
  rtc::Optional<bool> aecm_loud_speakerphone; // (jominwoo): My Custom Option
  rtc::Optional<int> adjust_agc_delta;
  ...
};
```

AudioOptions에 `aecm_loud_speakerphone`를 추가하고 멤버함수에 처리부도 추가하였다.

이제 정의한 Constraint가 AudioOptions에 적용될 수 있도록 파싱부를 추가하자.
MediaConstraintsInterface의 Constraint를 AudioOptions로 파싱하는 부분은
`webrtc/api/mediaconstraintsinterface.cc`에 있다.

```cpp
void CopyConstraintsIntoAudioOptions(
    const MediaConstraintsInterface* constraints,
    cricket::AudioOptions* options) {
  if (!constraints) {
    return;
  }

  ConstraintToOptional<bool>(constraints,
                             MediaConstraintsInterface::kGoogEchoCancellation,
                             &options->echo_cancellation);
  // (jominwoo): My Custom constraint processing.
  ConstraintToOptional<bool>(
      constraints, MediaConstraintsInterface::kAECMLoudSpeakerphone,
      &options->aecm_loud_speakerphone);
  ConstraintToOptional<bool>(
      constraints, MediaConstraintsInterface::kExtendedFilterEchoCancellation,
      &options->extended_filter_aec);
  ConstraintToOptional<bool>(constraints,
                             MediaConstraintsInterface::kDAEchoCancellation,
                             &options->delay_agnostic_aec);
  ConstraintToOptional<bool>(constraints,
                             MediaConstraintsInterface::kAutoGainControl,
                             &options->auto_gain_control);
  ...
}
```

중간에 추가한 파싱부를 통해 `kAECMLoudSpeakerphone` key로 입력된 bool형 value는
AudioOptions에 `aecm_loud_speakerphone`에 저장된다.

<br>

### Custom Constraint 사용부 추가
파싱부는 어떤 종류의 Constraint를 추가하든 유사하겠지만,
사용부는 Constraint 종류에 따라 서로 다를 것이다.
그래도 참고를 위해 사용부 예시를 남겨둔다.

위에서 추가한 `aecm_loud_speakerphone`을 적용하기 위해 AECM 모드를 설정하는 곳을 찾아보면
`webrtc/media/engine/webrtcvoiceengine.cc`의 ApplyOptions()에서 찾을 수 있다.
```cpp
bool WebRtcVoiceEngine::ApplyOptions(const AudioOptions& options_in) {
  RTC_DCHECK(worker_thread_checker_.CalledOnValidThread());
  LOG(LS_INFO) << "WebRtcVoiceEngine::ApplyOptions: " << options_in.ToString();
  AudioOptions options = options_in;  // The options are modified below.
  
  ...
  
  if (options.echo_cancellation) {
    // Check if platform supports built-in EC. Currently only supported on
    // Android and in combination with Java based audio layer.
    // TODO(henrika): investigate possibility to support built-in EC also
    // in combination with Open SL ES audio.
    const bool built_in_aec = adm()->BuiltInAECIsAvailable();
    if (built_in_aec) {
      // Built-in EC exists on this device and use_delay_agnostic_aec is not
      // overriding it. Enable/Disable it according to the echo_cancellation
      // audio option.
      const bool enable_built_in_aec =
          *options.echo_cancellation && !use_delay_agnostic_aec;
      if (adm()->EnableBuiltInAEC(enable_built_in_aec) == 0 &&
          enable_built_in_aec) {
        // Disable internal software EC if built-in EC is enabled,
        // i.e., replace the software EC with the built-in EC.
        options.echo_cancellation = rtc::Optional<bool>(false);
        LOG(LS_INFO) << "Disabling EC since built-in EC will be used instead";
      }
    }
    webrtc::apm_helpers::SetEcStatus(
        apm(), *options.echo_cancellation, ec_mode);
#if !defined(ANDROID)
    webrtc::apm_helpers::SetEcMetricsStatus(apm(), *options.echo_cancellation);
#endif
    if (ec_mode == webrtc::kEcAecm) {
      bool cn = options.aecm_generate_comfort_noise.value_or(false);
      webrtc::apm_helpers::SetAecmMode(apm(), cn);
    }
  }
  
  ...
```

보면 AudioOptions에 따라 WebRTC의 Audio Processing Module을 설정하고 있다.
그리고 SetAecmMode()를 통해 AECM 모드를 설정하는 것을 확인할 수 있다.

이제 `webrtc/media/engine/apm_helpers.cc`에서 SetAecmMode()를 확인해 보자.
```cpp
void SetAecmMode(AudioProcessing* apm, bool enable) {
  RTC_DCHECK(apm);
  EchoControlMobile* ecm = apm->echo_control_mobile();
  RTC_DCHECK_EQ(EchoControlMobile::kSpeakerphone, ecm->routing_mode());
  if (ecm->enable_comfort_noise(enable) != 0) {
    LOG(LS_ERROR) << "Failed to enable/disable CNG: " << enable;
    return;
  }
  LOG(LS_INFO) << "CNG set to " << enable;
}
```

위에서 보면 `aecm_generate_comfort_noise`옵션이 `EchoControlMobile`객체의
enable_comfort_noise()를 통해 설정되는 것을 확인할 수 있다.

이와 유사하게 `EchoControlMobile`객체의 set_routing_mode()을 통해 RoutingMode 변경이 가능하다.
RoutingMode를 옵션에 따라 `kLoudSpeakerphone`으로 변경이 가능하도록 SetAecmMode()를 수정해보자.
```cpp
void SetAecmMode(AudioProcessing* apm, bool enable_cng,
                 bool enable_aecm_loud_speakerphone) {
  RTC_DCHECK(apm);
  EchoControlMobile* ecm = apm->echo_control_mobile();
  // (jominwoo): Ignore below line for changing routing mode.
  // RTC_DCHECK_EQ(EchoControlMobile::kSpeakerphone, ecm->routing_mode());
  if (ecm->enable_comfort_noise(enable_cng) != 0) {
    LOG(LS_ERROR) << "Failed to enable/disable CNG: " << enable_cng;
    return;
  }
  LOG(LS_INFO) << "CNG set to " << enable_cng;
  EchoControlMobile::RoutingMode mode = (enable_aecm_loud_speakerphone)
                                        ? EchoControlMobile::kLoudSpeakerphone
                                        : EchoControlMobile::kSpeakerphone;
  if (ecm->set_routing_mode(mode) != 0) {
    LOG(LS_ERROR) << "Failed to set AECM Routing mode: " << mode;
    return;
  }
  LOG(LS_INFO) << "AECM Routing mode set to " << mode;
}
```

함수 인자를 추가하고 값에 따라 RoutingMode를 `kSpeakerphone` 또는 `kLoudSpeakerphone`로 설정하도록 하였다.

변경한 SetAecmMode()를 Constraint에 따라 달리 호출 할 수 있도록
`webrtc/media/engine/webrtcvoiceengine.cc`의 ApplyOptions()를 수정해보자.
```cpp
    if (ec_mode == webrtc::kEcAecm) {
      bool cn = options.aecm_generate_comfort_noise.value_or(false);
      bool ls = options.aecm_loud_speakerphone.value_or(false);
      webrtc::apm_helpers::SetAecmMode(apm(), cn, ls);
    }
```

AudioOptions의 `aecm_loud_speakerphone`의 값을 SetAecmMode()의 인자로 사용했다.

이렇게 Constraint 사용부 추가를 완료하였다. 이제 아래와 같이 Constraint를 설정하면
RoutingMode가 `kLoudSpeakerphone`으로 적용되게 된다.
```java
audioConstraints.mandatory.add(
          new MediaConstraints.KeyValuePair("customAECMLoudSpeakerphone", "true"));
```

