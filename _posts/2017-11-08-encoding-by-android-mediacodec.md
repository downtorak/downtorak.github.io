---
layout:     post
title:      Android MediaCodec을 이용한 비디오 Encoding
date:       2017-11-08 11:00:00 +0900
categories: "Android Video"
summary:    Android MediaCodec을 이용한 비디오 Encoding 방법
---

* content
{:toc}

### MediaCodec Encoder 생성
MediaCodec Encoder를 생성하기 위한 API는 두 가지가 있다.

첫 번째는 코덱이름을 사용하여 생성하는 API이다.
(코덱이름을 확인하는 방법은 이전의 `Android MediaCodec 정보 확인` 포스트를 참조)
```java
static MediaCodec createByCodecName(String name)
```

예를 들면 아래와 같이 사용할 수 있다.
```java
MediaCodec mediaCodec = MediaCodec.createByCodecName("OMX.google.h264.encoder");
```

주의할 사항은 `createByCodecName()` API는 Decoder를 생성하는데도 사용할 수
있기 때문에 `MediaCodecInfo`의 `isEncoder()` API를 통해 Encoder 인지 확인하고
사용할 필요가 있다.

두 번째는 MIME Type을 사용하여 생성하는 API이다.
```java
static MediaCodec createEncoderByType(String type)
```

예를 들면 아래와 같이 사용할 수 있다.
```java
MediaCodec mediaCodec = MediaCodec.createEncoderByType("video/avc");
```

<br>

### MediaCodec Encoder 설정
MediaCodec Encoder를 설정하기 위한 API는 아래와 같다.
```java
void configure(MediaFormat format, Surface surface, MediaCrypto crypto, int flags)
```
- `format`은 출력의 포멧을 설정하는 인자이다.
- `surface`는 Decoder 설정에 사용되는 인자로 Encoder 설정 시에는 `null`이다.
- `crypto`는 특정 코덱의 암호화에 사용된다는데 써본적은 없다. 안쓰면 `null`이다.
- `flags`는 플레그 설정인데 아직 한 종류 밖에 없다.
  - `CONFIGURE_FLAG_ENCODER`: Encoder 사용을 의미한다.

Encoder를 사용하기 위한 `configure()` 호출 예를 들면 아래와 같다.
```java
mediaCodec.configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODER);
```

결국 MediaCodec Encoder를 설정하는데 핵심은 `MediaFormat`이다.
그래서 `MediaFormat`에 대해 아래에 좀더 기술한다.

#### MediaFormat 생성
이 문서의 타겟은 비디오이므로 비디오에 대해서만 다루겠다.
비디오 MediaFormat을 생성하는 API는 다음과 같다.
```java
static final MediaFormat createVideoFormat(String mime, int width, int height)
```

AVC 코덱, FullHD의 설정은 다음과 같이 사용할 수 있겠다.
```java
MediaFormat format = MediaFormat.createVideoFormat("video/avc", 1920, 1080);
```

#### MediaFormat 속성 추가
MediaFormat의 기본 설정은 생성 시에 입력되고, 그 외의 속성은 Key, Value 형태로
추가하게 된다. 비디오 Encoder와 관련된 속성은 다음과 같다.
- `KEY_BIT_RATE`: 출력 데이터의 비트레이트를 설정한다.
- `KEY_COLOR_FORMAT`: 입력 RAW 이미지의 Color Format을 설정한다.
- `KEY_FRAME_RATE`: Frame Rate를 설정한다.
- `KEY_I_FRAME_INTERVAL`: I-Frame(Key Frame)의 주기를 설정한다. (초 단위)

이 외에도 몇 가지 더 있지만, 이 정도만 알아도 충분할 듯 하다.

FullHD 30fps, AVC 코덱, 5Mbps, 1초 주기 I-Frame은 다음과 같이 사용할 수 있겠다.
```java
MediaFormat format = MediaFormat.createVideoFormat("video/avc", 1920, 1080);
format.setInteger(MediaFormat.KEY_BIT_RATE, 5 * 1024 * 1024);
format.setInteger(MediaFormat.KEY_FRAME_RATE, targetFps);
format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, keyFrameIntervalSec);
```

`KEY_COLOR_FORMAT`은 RAW 이미지 입력방식에 따라 달라질 수 있어서
Encoding 방식에 대한 설명 때 다시 다루겠다.

<br>

### MediaCodec Encoder 시작
MediaCodec Encoder를 시작하기 위한 API는 아래와 같다.
```java
final void start()
```

아래와 같이 호출하면 MediaCodec Encoder가 시작된다.
```java
mediaCodec.start();
```

<br>

### MediaCodec Encoder 입력
MediaCodec Encoder에 RAW 이미지를 입력하는 방법에는 두 가지 방법이 존재한다.
첫 번째는 RAW 이미지를 `ByteBuffer`로 입력한 것이고, 두 번째는 Surface에
RAW 이미지를 Draw하여 입력하는 것이다.
여기에서는 첫 번째를 Byte Encoding 방식,
두 번째를 Texture Encoding 방식이라 하겠다.

#### Byte Encoding 방식
Byte Encoding 방식을 위해 사용되는 MediaCodec API는 다음과 같다.
```java
final int dequeueInputBuffer(long timeoutUs)
ByteBuffer getInputBuffer(int index)
final void queueInputBuffer(int index, int offset, int size, long presentationTimeUs, int flags)
```

`dequeueInputBuffer()`는 RAW 이미지 데이터를 입력할 버퍼의 인덱스를 반환한다.
MediaCodec에는 RAW 이미지 입력을 위한 일정 크기의 버퍼를 만들고,
우리는 해당 버퍼에 데이터를 복사하면 되는 것이다. 모든 버퍼가 사용 중이라면,
`-1`을 반환한다.
- `timeoutUs`: 사용 가능한 버퍼가 생길 때까지 대기하고자 한다면,
`timeoutUs`에 대기 시간을 설정하면 된다. 0이면 대기하지 않고, 음수면 계속 대기한다.

`getInputBuffer()`는 `dequeueInputBuffer()`에서 사용 가능하다고 알린 인덱스의
실제 버퍼를 반환하는 API이다.
- `index`: 사용할 버퍼의 인덱스를 입력한다.

`queueInputBuffer()`는 버퍼에 RAW 이미지 데이터 복사 후 사용하는 API로,
해당 버퍼를 MediaCodec에 입력한다.
- `index`: MediaCodec에 입력할 버퍼의 인덱스를 입력한다.
- `offset`: 입력 버퍼의 데이터 시작 지점을 입력한다. 보통 0이겠지.
- `size`: 데이터의 크기를 입력한다.
- `presentationTimeUs`: timestamp를 입력한다.
- `flags`: 쓸일 없어 Pass.

이상의 API를 사용하는 예를 들어보면 다음과 같다.
```java
public boolean encodeBuffer(byte[] input, long presentationTimeUs) {
  int inputBufferIndex = mediaCodec.dequeueInputBuffer(DEQUEUE_TIMEOUT);
  if (inputBufferIndex < 0) {
    return false;
  }
  
  ByteBuffer inputBuffer = mediaCodec.getInputBuffer(inputBufferIndex);
  inputBuffer.put(input);
  mediaCodec.queueInputBuffer(inputBufferIndex, 0, input.length, presentationTimeUs, 0);
  return true;
}
```

Byte Encoding 방식은 yuv파일 등과 같이
RAW 이미지 데이터를 바이트 형태로 입력받을 경우 사용할 수 있으며,
Texture Encoding 방식 보다는 성능이 떨어지는 것으로 알려져있다.

Byte Encoding 방식을 사용할 때는 MediaFormat에 `KEY_COLOR_FORMAT`으로
입력 RAW 이미지에 해당하는 Color Format을 설정해 주어야 한다.

#### Texture Encoding 방식
Texture Encoding 방식을 위해 사용되는 MediaCodec API는 다음과 같다.
```java
final Surface createInputSurface()
```

`createInputSurface()`는 RAW 이미지를 Draw할 Surface를 반환한다. 해당 Surface에
Draw가 되면 MediaCodec은 Draw된 Texture를 입력으로 Encoding을 수행한다.

Texture Encoding 방식은 카메라에서 Texture로 받은 RAW 이미지를 입력하거나,
Transcoding을 위해 MediaCodec Decoder의 출력을 입력하거나,
OpenGL 등으로 처리한 Texture를 입력하는 경우에 사용하기에 용이한 방식이다.

Texture Encoding 방식을 사용할 때는 MediaFormat에 `KEY_COLOR_FORMAT`으로
`MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface`를 설정해 주어야 한다.

<br>

### MediaCodec Encoder 출력
MediaCodec Encoder에서 출력을 받기 위한 API는 아래와 같다.
```java
final int dequeueOutputBuffer(MediaCodec.BufferInfo info, long timeoutUs)
ByteBuffer getOutputBuffer(int index)
final void releaseOutputBuffer(int index, boolean render)
```

`dequeueOutputBuffer()`는 출력 데이터를 담은 버퍼의 인덱스를 반환한다.
입력과 마찬가지로 MediaCodec에는 데이터를 출력하기 위한 일정 크기의 버퍼가
존재한다. 출력된 데이터가 없다면 `-1`을 반환한다.
- `info`: 출력된 데이터의 부가 정보를 전달한다.
  - `offset`: 버퍼의 데이터 시작 지점이다.
  - `size`: 데이터의 크기 이다.
  - `presentationTimeUs`: 데이터의 timestamp이다.
  - `flags`:
    - `BUFFER_FLAG_CODEC_CONFIG`: 코덱의 설정 데이터 (e.g. H.264의 SPS/PPS)인 경우이다.
    - `BUFFER_FLAG_KEY_FRAME`: 데이터가 Key Frame인 경우이다.
    - `BUFFER_FLAG_END_OF_STREAM`: 데이터가 마지막인 경우이다.
- `timeoutUs`: 데이터가 출력될 때까지 대기하고자 한다면,
`timeoutUs`에 대기 시간을 설정하면 된다. 0이면 대기하지 않고, 음수면 계속 대기한다.

`getOutputBuffer()`는 `dequeueOutputBuffer()`에서 얻은 인덱스를 통해 실제 버퍼를
반환하는 API이다.
- `index`: 사용할 버퍼의 인덱스를 입력한다.

`releaseOutputBuffer()`는 사용한 버퍼를 반환하는 API이다.
- `index`: 반환할 버퍼의 인덱스를 입력한다.
- `render`: Decoder를 위한 인자로 Encoder 사용 시에는 `false`이다.

이상의 API를 사용하는 예를 들어보면 다음과 같다.
```java
class EncodedData {
  byte[] data;
  long presentationTimeUs;
}

public boolean getEncodedData(EncodedData encodedData) {
  MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
  int outputBufferIndex = mediaCodec.dequeueOutputBuffer(bufferInfo, DEQUEUE_TIMEOUT);
  if (outputBufferIndex < 0) {
    return false;
  }
  
  ByteBuffer outputBuffer = mediaCodec.getOutputBuffer(outputBufferIndex);
  encodedData.data = new byte[bufferInfo.size];
  outputBuffer.get(encodedData.data);
  encodedData.presentationTimeUs = bufferInfo.presentationTimeUs;
  mediaCodec.releaseOutputBuffer(outputBufferIndex, false);
  return true;
}
```

<br>

### MediaCodec Encoder 정지
MediaCodec Encoder를 정지하기 위한 API는 아래와 같다.
```java
final void stop()
```

아래와 같이 호출하면 MediaCodec Encoder가 정지된다.
```java
mediaCodec.stop();
```

<br>

### MediaCodec Encoder 해제
MediaCodec Encoder를 해제하기 위한 API는 아래와 같다.
```java
final void release()
```

아래와 같이 호출하면 MediaCodec Encoder의 자원이 해제된다.
```java
mediaCodec.release();
```