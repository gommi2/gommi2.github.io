---
title : C# Clean Architecture + DI 구조 설계하기 (2) - Service / Repository 구현하기
categories : [C#, tips]
tags : [clean-architecture, di, repository, service, dotnet]
img_path: /assets/img/
pin : true
---

1편에서는 전체 구조를 잡고
- Domain / Application / Infrastructure / Presentation  
- 의존성 방향  
- 프로젝트 분리  

▶ 근데 아직 아무것도 안 돌아간다  

이번 편에서는  
**실제 비즈니스 로직을 넣어본다**

---

## 전체 흐름 다시 보기

이번 편에서 만들 구조는 이거다.

```csharp
Controller → Service → Repository → DB
```

▶ 핵심은

Controller는 요청만 전달
Service는 비즈니스 처리
Repository는 데이터 접근
### 1. Domain (Entity 그대로 사용)

1편에서 만든 Entity 그대로 사용한다.

```csharp
public class User
{
    public Guid Id { get; set; }
    public string Name { get; set; }
}
```

▶ Domain은 건드리지 않는다 (중요)

### 2. Application - Repository 인터페이스

먼저 Repository 인터페이스부터 정의한다.
```csharp
public interface IUserRepository
{
    Task<User?> GetByIdAsync(Guid id);
    Task AddAsync(User user);
}
```
▶ 여기서는 "무엇을 할 수 있는지"만 정의한다
▶ "어떻게 하는지"는 절대 안 씀

### 3. Application - Service (UseCase)

이제 핵심 로직을 만든다.
```csharp
public interface IUserService
{
    Task<User?> GetUserAsync(Guid id);
    Task<Guid> CreateUserAsync(string name);
}
```
구현체는 이렇게 간다.
```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _userRepository;

    public UserService(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }

    public async Task<User?> GetUserAsync(Guid id)
    {
        return await _userRepository.GetByIdAsync(id);
    }

    public async Task<Guid> CreateUserAsync(string name)
    {
        var user = new User
        {
            Id = Guid.NewGuid(),
            Name = name
        };

        await _userRepository.AddAsync(user);

        return user.Id;
    }
}
```
▶ 핵심 포인트

Repository를 인터페이스로 주입
DB가 뭔지 전혀 모름
오직 비즈니스 로직만 담당
### 4. DTO 분리 (중요 👍)

실무에서는 Entity를 그대로 쓰지 않는 게 좋다.
```csharp
public class CreateUserRequest
{
    public string Name { get; set; } = string.Empty;
}
```
▶ 이유

API 모델과 Domain 분리
변경 영향 최소화
유지보수 편함
### 5. Controller 연결

이제 Presentation 계층에서 연결한다.
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

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(Guid id)
    {
        var user = await _userService.GetUserAsync(id);

        if (user == null)
            return NotFound();

        return Ok(user);
    }

    [HttpPost]
    public async Task<IActionResult> Create(CreateUserRequest request)
    {
        var id = await _userService.CreateUserAsync(request.Name);

        return Ok(id);
    }
}
```
▶ Controller는 진짜 얇다

로직 없음
검증/호출만 담당
여기까지 구조 상태

이제 흐름은 완성됐다.

Controller → Service → Repository(interface)

▶ 근데 아직 문제 하나 남았다

- Repository "구현체"가 없음
- DI 연결도 안 됨
- DB도 없음

즉

▶ 아직 실행은 안 된다

이번 편 핵심 정리
Service는 비즈니스 로직 담당
Repository는 인터페이스만 정의
Controller는 최대한 얇게
DTO로 외부 모델 분리

다음 포스팅에서는

EF Core 적용
Repository 실제 구현
DI 등록 (Program.cs)
실제 실행까지 연결

▶ 드디어 "돌아가는 구조" 완성한다

Clean Architecture는

▶ 구조만 보면 어렵고
▶ 한번 흐름 잡으면 오히려 더 단순하다
