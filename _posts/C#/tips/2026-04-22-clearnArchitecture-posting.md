---
title : C# Clean Architecture + DI 구조 설계하기 (1) - 프로젝트 구조 잡기
categories : [C#, tips]
tags : [clean-architecture, di, repository, service, dotnet]
img_path: /assets/img/
pin : true
---

프로젝트를 처음 만들 때는 보통 이렇게 시작한다.  

Controller에서 바로 DB 호출하고  
Service 하나 만들어서 이것저것 다 넣고  
Repository 없이 DbContext를 직접 사용하고…

처음엔 빠르고 편하다.  

근데 기능이 조금만 늘어나면  
▶ **코드가 서로 엉키기 시작한다**

- 수정 하나 하면 여러 파일 같이 터짐  
- 테스트 거의 불가능  
- 유지보수 난이도 급상승  

그래서 결국 필요해지는 게

▶ **Clean Architecture 구조다**

---

## Clean Architecture가 뭐냐?

한 줄로 정리하면 이거다.

▶ **의존성 방향을 "안쪽으로만" 흐르게 만드는 구조**

조금 풀어서 보면

- 핵심 비즈니스 로직은 중심에 둔다  
- 바깥쪽(외부 기술)은 언제든 교체 가능하게 만든다  

---

## 전체 구조

Clean Architecture는 보통 4단계로 나눈다.

- **Domain** → 핵심 비즈니스 (Entity)
- **Application** → 유스케이스 (Service, Interface)
- **Infrastructure** → DB, 외부 API
- **Presentation** → Controller, API

---

## 핵심: 의존성 방향

여기서 제일 중요한 포인트

```csharp
Presentation → Application → Domain
Infrastructure → Application → Domain
```

▶ 무조건 안쪽만 바라본다

- Domain → Infrastructure 의존 금지
- Application → DbContext 직접 사용 금지

이거 하나만 지켜도 구조가 거의 무너지지 않는다.

솔루션 구조 만들기

Visual Studio 기준으로 프로젝트를 이렇게 나눈다.

```csharp
MyProject.sln

- MyProject.Domain
- MyProject.Application
- MyProject.Infrastructure
- MyProject.Api
```

▶ 처음엔 좀 과해 보이지만
프로젝트 커질수록 이 구조가 살려준다

### 1. Domain (핵심 계층)

가장 안쪽, 가장 중요한 계층이다.
```csharp
public class User
{
    public Guid Id { get; set; }
    public string Name { get; set; }
}
```
▶ 규칙

순수한 객체만 존재
DB 코드 없음
외부 라이브러리 의존 없음

즉,

▶ "절대 안 깨져야 하는 영역"

### 2. Application (유스케이스 계층)

# 유스케이스 계층?
<div style="text-align: center;">
  <img src="clean01.jpg" alt="유스케이스" width="700">
  <p><em>한국스마트그리드협회 참조</em></p>
</div>
```
유스케이스 계층은 시스템이 수행해야 할 ‘행위(시나리오)’를 중심으로, 계층형 아키텍처에서 유스케이스를 분리·구조화하는 관점.
스마트그리드 표준화 프레임워크 3.0에서는 계층적 구조에 따라 비즈니스 유스케이스(BUC)와 시스템 유스케이스(SUC)로 구분해 설명합니다
참조 : 한국스마트그리드협회
```

여기서 실제 비즈니스 흐름이 정의된다.

public interface IUserRepository
{
    Task<User?> GetByIdAsync(Guid id);
}

▶ 핵심 포인트

Repository는 인터페이스만 존재
구현은 절대 여기 있으면 안 됨

왜냐하면

▶ 구현은 바뀌지만, 규칙은 안 바뀌기 때문

### 3. Infrastructure (외부 구현 계층)

여기서 실제 DB, API 연결을 담당한다.
```csharp
public class UserRepository : IUserRepository
{
    public Task<User?> GetByIdAsync(Guid id)
    {
        // DB 조회 로직
    }
}
```
▶ 역할

EF Core
외부 API 호출
파일 / 캐시

▶ 전부 여기다 몰아넣는다

### 4. Presentation (API 계층)

사용자와 만나는 부분

```csharp
[ApiController]
[Route("api/users")]
public class UserController : ControllerBase
{
    private readonly IUserService _userService;

    public UserController(IUserService userService)
    {
        _userService = userService;
    }
}
```

▶ 핵심

로직 없음
그냥 호출만 한다
왜 이렇게 나누냐?

한 문장으로 끝난다.

▶ "바뀌는 것과 안 바뀌는 것을 분리하기 위해서"

예를 들어

DB를 MSSQL → MySQL 변경
API 구조 변경
외부 서비스 교체

▶ Domain / Application은 그대로 유지 가능

이 구조의 장점

정리해보면 꽤 크다.

비즈니스 로직 보호
테스트 용이
유지보수 쉬움
팀 협업에 강함

특히

▶ "프로젝트가 커질수록 차이가 난다"

여기까지 상태

지금 상태는

▶ 구조만 잡혀있는 상태다

아직

DB 연결 없음
Service 없음
실제 동작 안 함
다음 편 예고

다음 편에서는

Service (UseCase) 구현
Repository 실제 연결 흐름
DTO / Command 구조

▶ 진짜 로직을 넣어본다