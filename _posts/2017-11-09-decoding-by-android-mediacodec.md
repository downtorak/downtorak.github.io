---
layout:     post
title:      Android MediaCodec을 이용한 비디오 Decoding
date:       2017-11-09 11:00:00 +0900
categories: "Android Video"
summary:    Android MediaCodec을 이용한 비디오 Decoding 방법
---

* content
{:toc}

### MediaCodec Decoder 생성
MediaCodec Decoder를 생성하기 위한 API는 두 가지가 있다.

첫 번째는 코덱이름을 사용하여 생성하는 API이다.
(코덱이름을 확인하는 방법은 이전의 `Android MediaCodec 정보 확인` 포스트를 참조)
```java
static MediaCodec createByCodecName(String name)
```

예를 들면 아래와 같이 사용할 수 있다.
```java
MediaCodec mediaCodec = MediaCodec.createByCodecName("OMX.google.h264.decoder");
```

주의할 사항은 `createByCodecName()` API는 Encoder를 생성하는데도 사용할 수
있기 때문에 `MediaCodecInfo`의 `isEncoder()` API를 통해 Encoder가 아님을 확인하고
사용할 필요가 있다.

두 번째는 MIME Type을 사용하여 생성하는 API이다.
```java
static MediaCodec createDecoderByType(String type)
```

예를 들면 아래와 같이 사용할 수 있다.
```java
MediaCodec mediaCodec = MediaCodec.createDecoderByType("video/avc");
```

<br>

### MediaCodec Decoder 설정
MediaCodec Decoder를 설정하기 위한 API는 아래와 같다.
```java
void configure(MediaFormat format, Surface surface, MediaCrypto crypto, int flags)
```
- `format`은 입력의 포멧을 설정하는 인자이다.
- `surface`는 RAW 이미지 출력방식에 따라 달라질 수 있어서
Decoding 방식에 대한 설명 때 다시 다루겠다.
- `crypto`는 특정 코덱의 암호화에 사용된다는데 써본적은 없다. 안쓰면 `null`이다.
- `flags`는 플레그 설정인데 아직 Encoder를 위한 종류 밖에 없다.
그러므로 Decoder 사용 시, `flags`는 `0`이다.
  - `CONFIGURE_FLAG_ENCODER`: Encoder 사용을 의미한다.

Decoder를 사용하기 위한 `configure()` 호출 예를 들면 아래와 같다.
```java
mediaCodec.configure(format, null, null, 0);
```

MediaCodec Decoder를 설정하기 위한 MediaFormat은 Encoder와 달리,
특별한 경우가 아니라면 속성을 추가할 필요는 없다.
비디오 MediaFormat을 생성하는 API는 다음과 같다.
```java
static final MediaFormat createVideoFormat(String mime, int width, int height)
```

AVC 코덱, FullHD의 설정은 다음과 같이 사용할 수 있겠다.
```java
MediaFormat format = MediaFormat.createVideoFormat("video/avc", 1920, 1080);
```

<br>

### MediaCodec Decoder 시작
MediaCodec Decoder를 시작하기 위한 API는 아래와 같다.
```java
final void start()
```

아래와 같이 호출하면 MediaCodec Decoder가 시작된다.
```java
mediaCodec.start();
```

<br>

### MediaCodec Decoder 입력
MediaCodec Decoder에서 데이터를 입력하는 API는 아래와 같다.
```java
final int dequeueInputBuffer(long timeoutUs)
ByteBuffer getInputBuffer(int index)
final void queueInputBuffer(int index, int offset, int size, long presentationTimeUs, int flags)
```

`dequeueInputBuffer()`는 데이터를 입력할 버퍼의 인덱스를 반환한다.
MediaCodec에는 데이터 입력을 위한 일정 크기의 버퍼를 만들고,
우리는 해당 버퍼에 데이터를 복사하면 되는 것이다. 모든 버퍼가 사용 중이라면,
`-1`을 반환한다.
- `timeoutUs`: 사용 가능한 버퍼가 생길 때까지 대기하고자 한다면,
`timeoutUs`에 대기 시간을 설정하면 된다. 0이면 대기하지 않고, 음수면 계속 대기한다.

`getInputBuffer()`는 `dequeueInputBuffer()`에서 사용 가능하다고 알린 인덱스의
실제 버퍼를 반환하는 API이다.
- `index`: 사용할 버퍼의 인덱스를 입력한다.

`queueInputBuffer()`는 버퍼에 데이터 복사 후 사용하는 API로,
해당 버퍼를 MediaCodec에 입력한다.
- `index`: MediaCodec에 입력할 버퍼의 인덱스를 입력한다.
- `offset`: 입력 버퍼의 데이터 시작 지점을 입력한다. 보통 0이겠지.
- `size`: 데이터의 크기를 입력한다.
- `presentationTimeUs`: timestamp를 입력한다.
- `flags`: 쓸일 없어 Pass.

이상의 API를 사용하는 예를 들어보면 다음과 같다.
```java
public boolean decodeBuffer(byte[] input, long presentationTimeUs) {
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

<br>

### MediaCodec Decoder 출력
MediaCodec Decoder에서 출력 방법에는 두 가지 방법이 존재한다.
첫 번째는 RAW 이미지를 `ByteBuffer`로 출력하는 것이고, 두 번째는 Surface에
RAW 이미지를 Draw하여 출력하는 것이다.
여기에서는 첫 번째를 Byte Decoding 방식,
두 번째를 Texture Decoding 방식이라 하겠다.

#### Byte Decoding 방식
Byte Decoding 방식을 위해서는 MediaCodec 설정 시, `configure()`에서
`surface` 인자를 `null`로 설정하여야 한다.

Byte Decoding 방식을 위해 사용되는 MediaCodec API는 다음과 같다.
```java
final int dequeueOutputBuffer(MediaCodec.BufferInfo info, long timeoutUs)
ByteBuffer getOutputBuffer(int index)
final void releaseOutputBuffer(int index, boolean render)
```

`dequeueOutputBuffer()`는 출력 RAW 이미지 데이터를 담은 버퍼의 인덱스를 반환한다.
입력과 마찬가지로 MediaCodec에는 데이터를 출력하기 위한 일정 크기의 버퍼가
존재한다. 출력된 RAW 이미지 데이터가 없다면 `-1`을 반환한다.
- `info`: 출력된 RAW 이미지 데이터의 부가 정보를 전달한다.
  - `offset`: 버퍼의 데이터 시작 지점이다.
  - `size`: 데이터의 크기 이다.
  - `presentationTimeUs`: 데이터의 timestamp이다.
  - `flags`:
    - `BUFFER_FLAG_KEY_FRAME`: 데이터가 Key Frame인 경우이다.
    - `BUFFER_FLAG_END_OF_STREAM`: 데이터가 마지막인 경우이다.
- `timeoutUs`: 데이터가 출력될 때까지 대기하고자 한다면,
`timeoutUs`에 대기 시간을 설정하면 된다. 0이면 대기하지 않고, 음수면 계속 대기한다.

`getOutputBuffer()`는 `dequeueOutputBuffer()`에서 얻은 인덱스를 통해 실제 버퍼를
반환하는 API이다.
- `index`: 사용할 버퍼의 인덱스를 입력한다.

`releaseOutputBuffer()`는 사용한 버퍼를 반환하는 API이다.
- `index`: 반환할 버퍼의 인덱스를 입력한다.
- `render`: Byte Decoding 방식을 사용 시에는 `false`이다.

이상의 API를 사용하는 예를 들어보면 다음과 같다.
```java
class RawImageData {
  byte[] data;
  long presentationTimeUs;
}

public boolean getRawImageData(RawImageData rawImageData) {
  MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
  int outputBufferIndex = mediaCodec.dequeueOutputBuffer(bufferInfo, DEQUEUE_TIMEOUT);
  if (outputBufferIndex < 0) {
    return false;
  }
  
  ByteBuffer outputBuffer = mediaCodec.getOutputBuffer(outputBufferIndex);
  rawImageData.data = new byte[bufferInfo.size];
  outputBuffer.get(rawImageData.data);
  rawImageData.presentationTimeUs = bufferInfo.presentationTimeUs;
  mediaCodec.releaseOutputBuffer(outputBufferIndex, false);
  return true;
}
```

위의 코드는 예를 들기위한 것이므로 RAW 이미지 데이터를 복사하는 짓은
웬만하면 하지말아야겠다.

Byte Decoding 방식은 거의 쓸일이 없으므로 이런게 있구나 하고 넘어가자.

<br>

#### Texture Decoding 방식
Texture Decoding 방식을 위해서는 MediaCodec 설정 시, `configure()`에서
`surface` 인자에 Decoding 된 RAW 이미지를 출력할 Surface를 입력해야 한다.

Texture Decoding 방식을 위해 사용되는 MediaCodec API는 다음과 같다.
```java
final int dequeueOutputBuffer(MediaCodec.BufferInfo info, long timeoutUs)
final void releaseOutputBuffer(int index, boolean render)
```

`dequeueOutputBuffer()`는 Byte Decoding 방식과 동일하게 Decoding 결과를 반환한다.

`releaseOutputBuffer()`는 사용한 버퍼를 반환하는 역할과 Surface에 RAW 이미지를
출력하는 역할을 동시에 수행한다.
- `index`: RAW 이미지를 출력하고, 출력 후 반환할 버퍼의 인덱스를 입력한다.
- `render`: RAW 이미지를 출력을 하려면 `true`, 반환만 하려면 `false`이다.

즉, `releaseOutputBuffer()` 호출 시에 `render` 인자를 `true`로 설정하면,
`configure()`에서 입력한 Surface에 RAW 이미지를 출력하게 된다.

이상의 API를 사용하는 예를 들어보면 다음과 같다.
```java
public boolean drawRawImage() {
  MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
  int outputBufferIndex = mediaCodec.dequeueOutputBuffer(bufferInfo, DEQUEUE_TIMEOUT);
  if (outputBufferIndex < 0) {
    return false;
  }
  /*
  // 여기서 bufferInfo.presentationTimeUs에 의거한 대기가 필요함.
  */
  mediaCodec.releaseOutputBuffer(outputBufferIndex, true);
  return true;
}
```

Texture Decoding 방식을 사용하면 화면에 표시되는 `SurfaceView` 등에 바로
Rendering 할 수 있기 때문에 주로 사용되는 방식이다.

<br>

### MediaCodec Decoder 정지
MediaCodec Decoder를 정지하기 위한 API는 아래와 같다.
```java
final void stop()
```

아래와 같이 호출하면 MediaCodec Decoder가 정지된다.
```java
mediaCodec.stop();
```

<br>

### MediaCodec Decoder 해제
MediaCodec Decoder를 해제하기 위한 API는 아래와 같다.
```java
final void release()
```

아래와 같이 호출하면 MediaCodec Decoder의 자원이 해제된다.
```java
mediaCodec.release();
```