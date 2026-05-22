---
title : C# Clean Architecture + DI 구조 설계하기 (3) - Infrastructure + DI 연결하기
categories : [C#, tips]
tags : [clean-architecture, di, repository, service, dotnet, efcore]
img_path: /assets/img/
pin : true
---

이번 편에서는

**Infrastructure 구현 + DI 연결 + 실제 실행까지 완성한다**

---

## 지금 상태 다시 보기

현재 구조

```csharp
Controller → Service → Repository(interface)
```
▶ 문제

Repository 구현체 없음
DB 연결 없음
DI 등록 안 됨

즉,

▶ 껍데기만 있는 상태

## 1. EF Core DbContext 만들기

먼저 DB 역할을 할 DbContext를 만든다.
```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public DbSet<User> Users => Set<User>();

    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }
}
```
▶ 여기서 중요한 점

Domain의 User를 그대로 사용
DB 설정은 전부 Infrastructure에 위치
### 2. Repository 구현체 만들기

이제 인터페이스를 구현한다.
```csharp
public class UserRepository : IUserRepository
{
    private readonly AppDbContext _context;

    public UserRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<User?> GetByIdAsync(Guid id)
    {
        return await _context.Users.FindAsync(id);
    }

    public async Task AddAsync(User user)
    {
        await _context.Users.AddAsync(user);
        await _context.SaveChangesAsync();
    }
}
```
▶ 핵심

Application의 인터페이스를 구현
EF Core는 여기서만 사용

▶ Application은 여전히 DB 모름

### 3. DI (Dependency Injection) 등록

이제 모든 걸 연결해야 한다.

Program.cs에서 등록한다.
```csharp
var builder = WebApplication.CreateBuilder(args);

// DbContext (InMemory로 테스트)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseInMemoryDatabase("TestDb"));

// Repository
builder.Services.AddScoped<IUserRepository, UserRepository>();

// Service
builder.Services.AddScoped<IUserService, UserService>();

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();
app.Run();
```
▶ 여기서 중요한 포인트

Interface → Implementation 매핑
수명 주기: Scoped 사용
### 4. 실행 흐름 (전체 연결)

이제 드디어 흐름이 완성됐다.

HTTP 요청
→ Controller
→ Service
→ Repository
→ DbContext
→ DB

▶ 그리고 다시 반대로 응답

### 5. 실제 테스트

POST 요청

{
  "name": "gommi"
}

▶ 응답

"생성된 Guid"

GET 요청

/api/users/{id}

▶ 정상적으로 데이터 조회

이 구조의 진짜 의미

여기까지 만들고 나면 느껴진다.

▶ 각 계층이 완전히 분리되어 있음

예를 들어

DB를 InMemory → MSSQL 변경
Repository 로직 변경
외부 API 추가

▶ Application / Domain은 수정 없음

확장 포인트

여기서부터 진짜 실무가 시작된다.

Transaction 처리
Logging / Middleware
Validation (FluentValidation)
CQRS 구조 확장
MediatR 적용

▶ Clean Architecture는 “기초 뼈대”일 뿐이다