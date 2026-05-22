---
title : Android TTSManager 구현기 (Queue + Watchdog + 복구 로직까지)
categories : [Kotlin, tips]
tags : [tts, texttospeech, android, kotlin, queue, watchdog]
img_path: /assets/img/
pin : true
---

안드로이드에서 **TextToSpeech(TTS)** 를 사용하다 보면 생각보다 다양한 문제가 생긴다.  
잘 나오다가 갑자기 멈춘다든지, 콜백이 안 오는 경우도 있고, 심하면 TTS 엔진이 죽어버리는 상황도 생긴다️

그래서 이번에는 **안정성을 최우선으로 둔 TTSManager**를 하나 만들어봤다.  
단순히 speak만 하는 수준이 아니라, 실제 서비스에서도 버틸 수 있게 방어 로직을 꽤 많이 넣은 구조다.

---

## 전체 구조 한눈에 보기

핵심은 크게 3가지다.

- Queue 기반 순차 재생
- Watchdog으로 멈춤 감지
- 자동 복구 (재생 실패 / timeout 대응)

---

## 1. Queue 기반 재생

TTS는 한 번에 하나의 음성만 재생되기 때문에
여러 요청을 안정적으로 처리하려면 큐 구조가 필요하다.

```kotlin
private val queue: LinkedList<String> = LinkedList()

fun speak(text: String?) {
    val safeText = text?.trim().orEmpty()
    if (safeText.isEmpty()) return

    queue.add(safeText)
    playNext()
}
```

텍스트를 큐에 쌓고,
현재 재생 중이 아닐 때 하나씩 꺼내서 처리하는 방식이다.

▶ isSpeakingNow 플래그로 중복 재생을 막는 게 중요하다.

## 2. UtteranceListener로 상태 추적

TTS의 실제 상태는 Listener를 통해서 관리한다.

tts.setOnUtteranceProgressListener(object : UtteranceProgressListener() {

onStart() → 재생 시작
onDone() → 정상 종료
onError() → 오류 발생
override fun onDone(utteranceId: String?) {
    handleUtteranceFinished(utteranceId)
}

정상 종료 시 다음 큐를 이어서 실행한다.

▶ 이 구조 덕분에 자연스럽게 "연속 읽기"가 된다.

## 3. Watchdog으로 멈춤 감지

가끔 TTS는 콜백 없이 멈추는 경우가 있다.
이걸 잡기 위해 watchdog을 사용한다.
```kotlin
private val watchdogRunnable = Runnable {
    if (isSpeakingNow) {
        recoverTts("timeout")
    }
}
mainHandler.postDelayed(watchdogRunnable, 30_000L)
```

일정 시간(30초) 동안 응답이 없으면
▶ TTS가 멈췄다고 판단하고 복구를 시도한다.

## 4. TTS 자동 복구 로직
```kotlin
private fun recoverTts(reason: String)
```

복구 흐름은 다음과 같다.

현재 재생 중 텍스트 저장
상태 초기화
TTS stop / shutdown
새로운 인스턴스 생성
실패한 문장을 큐 맨 앞에 다시 넣기
if (!retryText.isNullOrBlank()) {
    queue.addFirst(retryText)
}

▶ 사용자 입장에서는 끊김 없이 이어지는 것처럼 들린다.

## 5. 과도한 복구 방지

무한 재시작을 막기 위한 제한도 추가했다.
```kotlin
private const val RECOVER_WINDOW_MS = 120_000L
private const val MAX_RECOVER_IN_WINDOW = 3
if (!canRecoverNow()) {
    stopInternal(resetCurrent = false)
    return
}
```
▶ 일정 횟수 이상 실패하면 더 이상 복구하지 않는다.
(무한 루프 방지 + 배터리 보호)

## 6. stop / clear / shutdown 차이
```kotlin
fun stop()
fun clear()
fun shutdown()
stop()      //→ 현재 재생만 중지
clear()     //→ 큐 포함 전체 초기화
shutdown()  //→ TTS 완전 종료
fun shutdown() 
{
    tts.stop()
    tts.shutdown()
}
```

▶ Activity나 Service 종료 시에는 꼭 호출해주는 게 좋다.

▶ 정리

이 TTSManager의 핵심은 안정성이다.

Queue로 순차 처리
Listener 기반 상태 관리
Watchdog으로 멈춤 감지
자동 복구 + 재시도
과도한 복구 제한
=>  마무리

TTS는 간단해 보이지만 실제로는 예외 상황이 꽤 많다.

onDone이 안 오는 경우
speak 실패
엔진 자체가 죽는 경우

이런 문제들을 한 번씩 겪다 보면
결국 안정성을 위한 코드가 계속 추가된다...