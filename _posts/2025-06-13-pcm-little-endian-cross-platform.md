---
title: "PCM 오디오가 무음으로 재생됐다 — Little-Endian과 Big-Endian"
date: 2025-06-13 22:00:00 +0900
categories: [Dev, Backend]
tags: [java, javascript, react-native, pcm, audio, endianness, little-endian, base64, android]
description: "Android 네이티브 모듈에서 PCM 오디오를 Base64로 JS에 넘겼는데 무음. Java ByteBuffer의 기본 Big-Endian과 JS의 Little-Endian이 달라서였다. Endianness란 무엇이고 왜 중요한지 정리한다."
---

## 1. 상황: Android → JS로 PCM 오디오 전달

React Native 앱에서 음성 녹음 기능을 만들고 있었다. 구조는 이렇다.

1. Android 네이티브 모듈(Java)에서 `AudioRecord`로 마이크 입력을 받는다.
2. VAD(Voice Activity Detection)를 거쳐 음성 구간만 `float[]`로 추출한다.
3. `float[]`를 `ByteBuffer`에 담고, Base64로 인코딩해서 JS로 넘긴다.
4. JS에서 Base64를 디코딩하고, `DataView`로 float 배열을 복원해서 재생한다.

```
AudioRecord → VAD → float[] → ByteBuffer → Base64 → JS → DataView → 재생
```

JS에서 디코딩까지는 정상이었다. 바이트 배열의 길이도 맞고, Base64 문자열도 비어있지 않았다. 그런데 **재생하면 무음**이었다.

데이터는 분명히 있는데, 소리가 안 났다.

---

## 2. Endianness란

멀티바이트 데이터를 메모리에 저장할 때, 바이트를 **어떤 순서로** 배치하느냐의 문제다.

- **Little-Endian**: 낮은 바이트(LSB)를 앞(낮은 주소)에 저장한다.
- **Big-Endian**: 높은 바이트(MSB)를 앞(낮은 주소)에 저장한다.

예를 들어 `0x12345678`을 4바이트로 저장하면:

```
Little-Endian: 78 56 34 12
Big-Endian:    12 34 56 78
```

같은 값인데 바이트 순서가 정반대다.

### 누가 뭘 쓰나

| 환경 | Endianness |
|------|-----------|
| x86 / ARM (대부분의 디바이스) | Little-Endian |
| 네트워크 바이트 순서 (TCP/IP) | Big-Endian |
| Java `ByteBuffer` 기본값 | **Big-Endian** |
| JS `DataView` / `TypedArray` | 플랫폼 의존 (대부분 **Little-Endian**) |

Java는 네트워크 바이트 순서를 따르고, JS는 실행 환경(거의 대부분 x86/ARM)을 따른다. 둘이 기본값이 다르다.

---

## 3. 원인: ByteBuffer vs DataView

Java 쪽 코드다.

```java
ByteBuffer byteBuffer = ByteBuffer.allocate(floatBuffer.length * 4);
// order 미지정 → 기본값 BIG_ENDIAN
for (float value : floatBuffer) {
    byteBuffer.putFloat(value);
}
String base64 = Base64.encodeToString(byteBuffer.array(), Base64.NO_WRAP);
```

`ByteBuffer.allocate()`의 기본 바이트 순서는 `BIG_ENDIAN`이다. `order()`를 호출하지 않으면 float 값이 Big-Endian으로 쓰인다.

JS 쪽에서는 이렇게 읽는다.

```javascript
const bytes = Uint8Array.from(atob(base64), c => c.charCodeAt(0));
const dataView = new DataView(bytes.buffer);

for (let i = 0; i < bytes.length; i += 4) {
    const sample = dataView.getFloat32(i, true); // true = Little-Endian
}
```

`getFloat32`의 두 번째 인자 `true`는 Little-Endian으로 읽겠다는 뜻이다.

Java가 Big-Endian으로 쓰고, JS가 Little-Endian으로 읽으면 float 값이 완전히 다른 값이 된다.

### 무음이 되는 이유

float `0.5`의 IEEE 754 표현은 `3F000000`이다.

```
Big-Endian 저장:    3F 00 00 00
Little-Endian 읽기: 00 00 00 3F → 0x0000003F → 약 8.8e-44
```

`0.5`가 `0.000000000000000000000000000000000000000000088`이 된다. 오디오 샘플 범위(-1.0 ~ 1.0)에서 이 값은 사실상 **0**이다. 모든 샘플이 이런 식으로 극미한 값이 되니까, 재생하면 무음이 나온다.

데이터가 없는 게 아니라, 바이트 순서가 뒤집혀서 값이 전부 0에 수렴한 것이다.

### 같은 코드에서 WAV는 왜 됐나

같은 네이티브 모듈에 WAV 파일을 만드는 함수도 있었는데, 그쪽은 정상이었다. 이유는 간단했다. WAV 헤더와 PCM 데이터를 `writeShort`로 직접 바이트를 조합하면서 LSB부터 썼기 때문에 이미 Little-Endian이었다.

```java
// writeShort는 수동으로 LSB부터 쓴다
private void writeShort(DataOutputStream out, short value) throws IOException {
    out.write(value & 0xFF);        // 낮은 바이트 먼저
    out.write((value >> 8) & 0xFF); // 높은 바이트 나중에
}
```

`ByteBuffer.putFloat()`을 거치는 경로만 Big-Endian 기본값에 걸린 것이다.

---

## 4. 수정: 한 줄

```java
ByteBuffer byteBuffer = ByteBuffer.allocate(floatBuffer.length * 4);
byteBuffer.order(ByteOrder.LITTLE_ENDIAN); // 이 한 줄 추가
for (float value : floatBuffer) {
    byteBuffer.putFloat(value);
}
String base64 = Base64.encodeToString(byteBuffer.array(), Base64.NO_WRAP);
```

`ByteBuffer`를 할당한 직후에 `order(ByteOrder.LITTLE_ENDIAN)`을 호출하면, `putFloat()`이 Little-Endian으로 바이트를 쓴다. JS에서 `getFloat32(i, true)`로 읽으면 같은 바이트 순서이므로 원래 float 값이 복원된다.

한 줄로 무음이 해결됐다.

---

## 5. 돌아보며

1. **Java `ByteBuffer`의 기본값은 Big-Endian이다.** 바이너리를 다룰 때 `order()`를 항상 명시하자.
2. **JS `TypedArray`/`DataView`는 플랫폼 Endianness를 따른다.** 대부분 Little-Endian이다.
3. **바이너리 데이터를 언어 간 주고받을 때, 양쪽의 바이트 순서를 맞춰야 한다.** 안 맞으면 값이 깨진다.
4. **증상이 "무음"이면 데이터가 없는 게 아니라 바이트 순서가 틀린 걸 의심하자.** 데이터 길이가 정상인데 값이 전부 0에 가깝다면 Endianness 문제다.
5. **Base64는 바이트를 문자열로 바꿀 뿐, 바이트 순서와는 무관하다.** Base64 인코딩/디코딩은 바이트 배열을 그대로 보존한다. 순서가 틀린 바이트를 Base64로 감싸봤자 틀린 순서 그대로 전달된다.
