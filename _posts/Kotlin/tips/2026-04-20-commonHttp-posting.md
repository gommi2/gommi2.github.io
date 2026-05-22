---
title : Android 공통 HttpClient 설계하기 (CommonHttp + Repository 패턴)
categories : [Kotlin, tips]
tags : [retrofit, okhttp, gson, repository, network]
img_path: /assets/img/
pin : true
---

앱을 만들다 보면 API 통신 코드는 계속 반복된다.  
요청 만들고 → Retrofit 호출하고 → JSON 파싱하고 → 에러 처리하고…

처음엔 그냥 기능마다 따로 만들다가  
어느 순간 보면 **중복 코드 + 유지보수 지옥**이 시작된다

그래서 이번에는  
▶ **공통 HTTP 클라이언트 + Repository 구조**로 정리해봤다.

---

## 전체 구조

구조는 단순하게 2단계로 나눈다.

- `CommonHttpClient` → 실제 통신 담당 (공통 처리)
- `Repository` → API 단위로 래핑 (비즈니스 로직)

---

## 1. Repository 역할

```kotlin
class PaymentRepository {

    suspend fun wechatRequestPayment(
        url: String,
        fCode: Int,
        charge: Int,
        barcode: String
    ): Result<GenericResponse<String>> {
        return CommonHttpClient.wechatRequestPayment(
            fullUrl = url + "req",
            fCode = fCode,
            charge = charge,
            barcode = barcode
        )
    }
}
```
Repository는 단순하다.

▶ "이 API는 어떤 파라미터를 쓰는지"만 정의

URL 조합
파라미터 전달
어떤 API인지 의미 부여

즉,
▶ 비즈니스 로직과 네트워크 로직을 분리하는 역할

## 2. CommonHttpClient 핵심 구조
object CommonHttpClient

모든 통신은 여기서 처리한다.

★ OkHttp 설정

```kotlin
private val okHttpClient: OkHttpClient by lazy {
    OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .readTimeout(15, TimeUnit.SECONDS)
        .writeTimeout(15, TimeUnit.SECONDS)
        .addInterceptor(loggingInterceptor)
        .build()
}
```

타임아웃 설정
로그 인터셉터 추가

▶ 디버깅할 때 BODY 로그는 거의 필수다.

★ Retrofit 생성
```kotlin
private val retrofit: Retrofit by lazy {
    Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(okHttpClient)
        .build()
}
```

여기서 중요한 포인트!

▶ baseUrl은 의미 없음 (127.0.0.1 더미)
▶ 실제 URL은 fullUrl로 동적 전달

## 3. 핵심: 공통 POST 함수
```kotlin
private suspend inline fun <reified TReq : Any, reified TRes> postJson(
    fullUrl: String,
    request: TReq
): Result<GenericResponse<TRes>>
```

이 함수 하나로 모든 API를 처리한다.

= 요청 처리

```kotlin
val requestJson = gson.toJson(request)
val requestBody = requestJson.toRequestBody(mediaType)
```

▶ 어떤 Request든 JSON으로 변환

= 응답 처리
```kotlin
val type = object : TypeToken<GenericResponse<TRes>>() {}.type
val parsed: GenericResponse<TRes> = gson.fromJson(responseText, type)
```

▶ 제네릭으로 Response 파싱

이 구조 덕분에

String 응답
Object 응답
List 응답

▶ 전부 동일한 방식으로 처리 가능

= 에러 처리
```kotlin
if (!response.isSuccessful) {
    Result.failure(
        Exception("HTTP ${response.code()} ...")
    )
}
```

▶ HTTP 에러 / 파싱 에러 모두 Result.failure로 통일

## 4. 실제 API 구현 예시
```kotlin
suspend fun wechatRequestPayment(
    fullUrl: String,
    fCode: Int,
    charge: Int,
    barcode: String
): Result<GenericResponse<String>> {
val request = WechatPaymentRequest(
    f_code = fCode,
    charge = charge,
    barcode = barcode
)
return postJson<WechatPaymentRequest, String>(
    fullUrl = fullUrl,
    request = request
)
```

▶ 핵심 포인트

Request 객체만 만들고
공통 함수 호출

끝이다.

## 5. 사용하는 쪽 코드
```kotlin
val genericResponse = paymentRepository
    .wechatRequestPayment(...)
    .getOrElse { e ->
        GenericResponse(
            success = false,
            error = e.message ?: "통신 오류",
            result = "",
            txNo = ""
        )
    }
```

▶ Result를 사용하는 이유

성공 / 실패를 강제 분기
예외를 안전하게 처리
= 실패 처리
```kotlin
if (!genericResponse.success) {
    if(genericResponse.result.isEmpty())
        throw UserException.TheOtherException("새로운 바코드로 다시 시도해주세요.")
    else
        throw UserException.TheOtherException(genericResponse.result)
}
```

▶ 서버 응답 기준으로 UI 처리

result 비어있으면 공통 메시지
있으면 서버 메시지 그대로 사용
▶ 이 구조의 장점

정리해보면 꽤 깔끔하다.

공통 통신 로직 1곳 집중
API 추가 시 코드 최소화
제네릭으로 응답 재사용
Result로 안전한 에러 처리

▶ 특히 팀 프로젝트에서 유지보수가 훨씬 편해진다.

★ 마무리

처음에는 Retrofit 인터페이스만 써도 충분해 보이지만
프로젝트가 커질수록 결국 이런 구조가 필요해진다.

공통 처리 없으면 코드 중복 폭발
에러 처리 제각각
유지보수 난이도 상승

그래서 개인적으로는

"CommonHttp + Repository 패턴"은 거의 필수 구조라고 본다.