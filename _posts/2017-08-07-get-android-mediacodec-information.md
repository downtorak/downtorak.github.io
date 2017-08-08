---
layout: post
title:  "Android MediaCodec 정보 확인"
date:   2017-06-14 15:00:00 +0900
categories: Android_Video
---

* content
{:toc}

### MediaCodec 목록 가져오기
`MediaCodecList`([참조](https://developer.android.com/reference/android/media/MediaCodecList.html))는 Android 단말에서 사용 가능한 `MediaCodec`의 정보를 제공한다. 이 `MediaCodec` 정보는 `MediaCodecInfo`로 제공된다.

아래는 `MediaCodecList` 사용 예제.
```java
public void printMediaCodecInfo() {
  MediaCodecList list = new MediaCodecList(MediaCodecList.REGULAR_CODECS);
  MediaCodecInfo[] infos = list.getCodecInfos();
  for (MediaCodecInfo info : infos) {
    Log.i(TAG, "MediaCodecInfo" +
        " name=" + info.getName() +
        " isEncoder=" + info.isEncoder());
  }
}
```

`MediaCodecList` 생성자의 인자는 `REGULAR_CODECS` 외에 `ALL_CODECS`가 있는데 일반적인 방법으로 사용할 수 있는 `MediaCodec` 정보만 알아보기 위해서는 `REGULAR_CODECS`을 사용하면 된다.

`getCodecInfos()`는 API level 21부터 지원된다. 낮은 버전에서는 아래와 같이 `getCodecCount()`와 `getCodecInfoAt()`을 사용하며, `REGULAR_CODECS`을 설정했을 때와 동일한 결과를 얻을 수 있다.
```java
public void printMediaCodecInfo() {
  for (int i = 0; i < MediaCodecList.getCodecCount(); ++i) {
    MediaCodecInfo info = MediaCodecList.getCodecInfoAt(i);
    Log.i(TAG, "MediaCodecInfo" +
        " name=" + info.getName() +
        " isEncoder=" + info.isEncoder());
  }
}
```

<br>

아래는 상기 예제의 수행 결과.
```no-highlight
MediaCodecInfo name=OMX.Marvell.audio_decoder.mp3 isEncoder=false
MediaCodecInfo name=OMX.Marvell.audio_decoder.mp2 isEncoder=false
MediaCodecInfo name=OMX.Marvell.audio_decoder.g711 isEncoder=false
MediaCodecInfo name=OMX.Marvell.audio_decoder.ac3 isEncoder=false
MediaCodecInfo name=OMX.Marvell.video_decoder.avc isEncoder=false
MediaCodecInfo name=OMX.Marvell.video_decoder.hevc isEncoder=false
MediaCodecInfo name=OMX.Marvell.video_decoder.mpeg4 isEncoder=false
MediaCodecInfo name=OMX.Marvell.video_decoder.vpx isEncoder=false
MediaCodecInfo name=OMX.Marvell.video_decoder.vp9 isEncoder=false
MediaCodecInfo name=OMX.Marvell.video_decoder.h263 isEncoder=false
MediaCodecInfo name=OMX.Marvell.video_decoder.mpeg2 isEncoder=false
MediaCodecInfo name=OMX.Marvell.video_encoder.avc isEncoder=true
MediaCodecInfo name=OMX.Marvell.video_encoder.vp8 isEncoder=true
MediaCodecInfo name=OMX.google.mp3.decoder isEncoder=false
MediaCodecInfo name=OMX.google.amrnb.decoder isEncoder=false
MediaCodecInfo name=OMX.google.amrwb.decoder isEncoder=false
MediaCodecInfo name=OMX.google.aac.decoder isEncoder=false
MediaCodecInfo name=OMX.google.g711.alaw.decoder isEncoder=false
MediaCodecInfo name=OMX.google.g711.mlaw.decoder isEncoder=false
MediaCodecInfo name=OMX.google.vorbis.decoder isEncoder=false
MediaCodecInfo name=OMX.google.opus.decoder isEncoder=false
MediaCodecInfo name=OMX.google.raw.decoder isEncoder=false
MediaCodecInfo name=OMX.google.aac.encoder isEncoder=true
MediaCodecInfo name=OMX.google.amrnb.encoder isEncoder=true
MediaCodecInfo name=OMX.google.amrwb.encoder isEncoder=true
MediaCodecInfo name=OMX.google.flac.encoder isEncoder=true
MediaCodecInfo name=OMX.google.gsm.decoder isEncoder=false
MediaCodecInfo name=OMX.google.mpeg4.decoder isEncoder=false
MediaCodecInfo name=OMX.google.h263.decoder isEncoder=false
MediaCodecInfo name=OMX.google.h264.decoder isEncoder=false
MediaCodecInfo name=OMX.google.hevc.decoder isEncoder=false
MediaCodecInfo name=OMX.google.vp8.decoder isEncoder=false
MediaCodecInfo name=OMX.google.vp9.decoder isEncoder=false
MediaCodecInfo name=OMX.google.h263.encoder isEncoder=true
MediaCodecInfo name=OMX.google.h264.encoder isEncoder=true
MediaCodecInfo name=OMX.google.mpeg4.encoder isEncoder=true
MediaCodecInfo name=OMX.google.vp8.encoder isEncoder=true
```

`OMX`는 `OpenMax`의 약자이고 뒤에는 보통 제조사나 칩셋이름이 붙는다. 위 수행 결과는 Android 셋탑박스에서 출력된 것이라 조금 특이한데, 보통 단말에서는 `Marvell` 대신 `qcom`(퀄컴)이나 삼성폰이라면 `Exynos`가 출력될 것이다.

아래쪽의 `OMX.google`은 SW Encoder/Decoder를 `OpenMax`규격에 맞게 제공하는 것이다. 그래서 당연히 HW Encoder/Decoder 보다 성능이 떨어진다.

<br>

### MediaCodec의 지원 Codec 확인하기
`MediaCodecInfo`의 `getSupportedTypes()`를 통해 `MediaCodec`이 지원하는 Codec의 MIME Type 리스트를 가져올 수 있다.

아래는 `getSupportedTypes()` 사용 예제.
```java
public void printMediaCodecInfo() {
  MediaCodecList list = new MediaCodecList(MediaCodecList.REGULAR_CODECS);
  MediaCodecInfo[] infos = list.getCodecInfos();
  for (MediaCodecInfo info : infos) {
    for (String mimeType : info.getSupportedTypes()) {
      Log.i(TAG, "MediaCodecInfo" +
          " name=" + info.getName() +
          " mimeType=" + mimeType);
    }
  }
}
```

<br>

아래는 상기 예제의 수행 결과의 일부.
```no-highlight
...
MediaCodecInfo name=OMX.Marvell.audio_decoder.ac3 mimeType=audio/ac3
MediaCodecInfo name=OMX.Marvell.audio_decoder.ac3 mimeType=audio/eac3
MediaCodecInfo name=OMX.Marvell.audio_decoder.ac3 mimeType=audio/x-eac3
MediaCodecInfo name=OMX.Marvell.video_decoder.avc mimeType=video/avc
MediaCodecInfo name=OMX.Marvell.video_decoder.hevc mimeType=video/hevc
...
```

보통 `MediaCodec`을 사용할 때는 `createDecoderByType()`를 사용해서 MIME Type만으로 `MediaCodec`을 생성한다. 하지만  `createByCodecName()`로 특정 `MediaCodec`을 선택해서 사용하고자 한다면 `getSupportedTypes()`로 MIME Type을 지원하는지 확인하고 사용해야 한다.

<br>

### MediaCodec의 Codec Capabilities 확인하기
`MediaCodecInfo`의 `getCapabilitiesForType()`를 통해 `MediaCodec`이 지원하는 Codec의 Capabilities를 확인할 수 있다.

아래는 AVC Encoder에 대한 `getCapabilitiesForType()` 사용 예제.
```java
public void printMediaCodecInfo() {
  MediaCodecList list = new MediaCodecList(MediaCodecList.REGULAR_CODECS);
  MediaCodecInfo[] infos = list.getCodecInfos();

  String avcMimeType = "video/avc";
  for (MediaCodecInfo info : infos) {
    boolean supportAVC = false;
    for (String mimeType : info.getSupportedTypes()) {
      if (mimeType.equals(avcMimeType)) {
        supportAVC = true;
        break;
      }
    }

    if (info.isEncoder() && supportAVC) {
      MediaCodecInfo.CodecCapabilities codecCapa = info.getCapabilitiesForType(avcMimeType);
      MediaCodecInfo.VideoCapabilities videoCapa = codecCapa.getVideoCapabilities();
      Range<Integer> widths = videoCapa.getSupportedWidths();
      Range<Integer> heights = videoCapa.getSupportedHeights();
      Range<Integer> frameRates = videoCapa.getSupportedFrameRates();
      Range<Integer> bitrateRange = videoCapa.getBitrateRange();
      Log.i(TAG, "MediaCodecInfo" +
          " name=" + info.getName() +
          " width=[" + widths.getLower() + "," + widths.getUpper() + "]" +
          " height=[" + heights.getLower() + "," + heights.getUpper() + "]" +
          " fps=[" + frameRates.getLower() + "," + frameRates.getUpper() + "]" +
          " bps=[" + bitrateRange.getLower() + "," + bitrateRange.getUpper() + "]");

    }
  }
}
```

<br>

아래는 상기 예제의 수행 결과.
```no-highlight
MediaCodecInfo name=OMX.Marvell.video_encoder.avc width=[144,1920] height=[96,1072] fps=[0,960] bps=[10000,60000000]
MediaCodecInfo name=OMX.google.h264.encoder width=[16,1920] height=[16,1088] fps=[0,960] bps=[1,12000000]
```

`getCapabilitiesForType()`을 사용하면 `MediaCodec`의 스펙을 알 수 있다. 하지만 왠만한 HW Encoder/Decoder는 범용적으로 사용하는 스펙을 대부분 지원하기 때문에 특별한 경우가 아니면 `getCapabilitiesForType()`은 잘 쓰이지는 않는 듯하다.







