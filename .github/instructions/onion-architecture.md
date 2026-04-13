# Onion Architecture — Detailed Instruction File

> This file is loaded automatically when the user selects **Onion Architecture** in `scaffold.md`.
> Follow every section completely. Do not skip layers, rules, or code examples.

---

## 1. Architecture Overview

Onion Architecture organises code into concentric rings. Dependencies always point **inward** toward the core. The innermost ring defines the domain; outer rings implement technical concerns.

```
        ┌────────────────────────────────────────────┐
        │              API / Presentation             │
        │   Controllers, Middleware, DI Wiring        │
        │  ┌──────────────────────────────────────┐  │
        │  │           Infrastructure             │  │
        │  │  EF Core, Repositories, Identity,    │  │
        │  │  External APIs, Email, Storage        │  │
        │  │  ┌────────────────────────────────┐  │  │
        │  │  │         Application            │  │  │
        │  │  │  Services, DTOs, Validators,   │  │  │
        │  │  │  Interfaces, Mappings          │  │  │
        │  │  │  ┌──────────────────────────┐  │  │  │
        │  │  │  │          Core            │  │  │  │
        │  │  │  │  Domain Entities,        │  │  │  │
        │  │  │  │  Value Objects, Enums,   │  │  │  │
        │  │  │  │  Domain Exceptions,      │  │  │  │
        │  │  │  │  Repository Contracts    │  │  │  │
        │  │  │  └──────────────────────────┘  │  │  │
        │  │  └────────────────────────────────┘  │  │
        │  └──────────────────────────────────────┘  │
        └────────────────────────────────────────────┘
            All dependencies point inward ↑
```

**Core Principle:** The `Core` project must never reference `Application`, `Infrastructure`, or `Api`. Interfaces are defined in `Core`; implementations live in `Infrastructure`.

---

## 2. Solution Structure

### Solution Name: `{Project}`

```
{Project}.sln
│
├── src/
│   ├── {Project}.Api/
│   ├── {Project}.Infrastructure/
│   ├── {Project}.Application/
│   └── {Project}.Core/
│
└── tests/  (if test projects selected)
    ├── {Project}.Tests.Unit/
    └── {Project}.Tests.Integration/
```

---

## 3. Project Descriptions

| Project | Type | Ring | Responsibility |
|---------|------|------|----------------|
| `{Project}.Core` | Class Library | Innermost | Domain entities, value objects, domain enums, domain exceptions, repository contracts/interfaces |
| `{Project}.Application` | Class Library | Middle | Application services, DTOs, request/response models, validators, interface definitions for infrastructure services |
| `{Project}.Infrastructure` | Class Library | Outer | EF Core DbContext, repository implementations, Identity, external services, token service |
| `{Project}.Api` | ASP.NET Core Web API | Outermost | HTTP entry point, controllers, middleware, DI wiring, Swagger |

---

## 4. Project Reference Rules

```
{Project}.Api
    ├── references {Project}.Application
    └── references {Project}.Infrastructure   ← only for DI wiring in Program.cs

{Project}.Infrastructure
    ├── references {Project}.Application
    └── references {Project}.Core

{Project}.Application
    └── references {Project}.Core

{Project}.Core
    └── NO references whatsoever
```

> **Critical Rules:**
> - `{Project}.Core` is **pure** — no EF Core, no HTTP, no FluentValidation.
> - `{Project}.Api` must only reference `{Project}.Infrastructure` for DI setup.
> - All repository contracts live in `{Project}.Core.Interfaces`.
> - All application-level contracts (e.g., `ITokenService`, `IEmailService`) live in `{Project}.Application.Interfaces`.

---

## 5. Naming Conventions

| Concept | Pattern | Example |
|---------|---------|---------|
| Solution/Project | `{Project}.{Ring}` | `Ordering.Core` |
| Domain Entity | PascalCase noun | `User`, `Order` |
| Value Object | PascalCase noun | `Money`, `Address` |
| Domain Exception | `{Entity}{Issue}Exception` | `UserNotFoundException` |
| Repository Interface (Core) | `I{Entity}Repository` | `IUserRepository` |
| Generic Repository Interface | `IRepository<T>` | — |
| Application Service Interface | `I{Feature}Service` | `IUserService` |
| Application Service Impl | `{Feature}Service` | `UserService` |
| Request DTO | `{Action}{Entity}Request` | `CreateUserRequest` |
| Response DTO | `{Entity}Response` | `UserResponse` |
| Validator | `{Request}Validator` | `CreateUserRequestValidator` |
| Entity Config | `{Entity}Configuration` | `UserConfiguration` |
| Infrastructure Service | `{Feature}Service` (impl) | `TokenService`, `EmailService` |

---

## 6. Folder Structure

### `{Project}.Core`

```
{Project}.Core/
├── Entities/
│   ├── BaseEntity.cs
│   └── User.cs
├── Enums/
│   └── UserStatus.cs
├── Exceptions/
│   ├── DomainException.cs
│   ├── UserNotFoundException.cs
│   └── EmailAlreadyExistsException.cs
├── Interfaces/
│   ├── Repositories/
│   │   ├── IRepository.cs
│   │   └── IUserRepository.cs
│   └── Services/
│       └── IUnitOfWork.cs
└── ValueObjects/
    └── Email.cs
```

### `{Project}.Application`

```
{Project}.Application/
├── DTOs/
│   ├── UserDto.cs
│   └── LoginDto.cs
├── Interfaces/
│   ├── IAuthService.cs
│   ├── IUserService.cs
│   └── ITokenService.cs
├── Requests/
│   ├── Auth/
│   │   ├── LoginRequest.cs
│   │   └── RegisterRequest.cs
│   └── Users/
│       ├── CreateUserRequest.cs
│       └── UpdateUserRequest.cs
├── Responses/
│   ├── Auth/
│   │   └── LoginResponse.cs
│   └── Users/
│       └── UserResponse.cs
├── Services/
│   └── UserService.cs
├── Validators/
│   ├── Auth/
│   │   ├── LoginRequestValidator.cs
│   │   └── RegisterRequestValidator.cs
│   └── Users/
│       ├── CreateUserRequestValidator.cs
│       └── UpdateUserRequestValidator.cs
├── Common/
│   └── ApiResponse.cs
└── DependencyInjection.cs
```

### `{Project}.Infrastructure`

```
{Project}.Infrastructure/
├── Persistence/
│   ├── ApplicationDbContext.cs
│   ├── Configurations/
│   │   └── UserConfiguration.cs
│   └── Migrations/
├── Repositories/
│   ├── Repository.cs
│   └── UserRepository.cs
├── Identity/
│   ├── ApplicationUser.cs
│   └── TokenService.cs
├── Services/
│   └── EmailService.cs
└── DependencyInjection.cs
```

### `{Project}.Api`

```
{Project}.Api/
├── Controllers/
│   ├── AuthController.cs
│   └── UsersController.cs
├── Middleware/
│   └── ExceptionHandlingMiddleware.cs
├── appsettings.json
├── appsettings.Development.json
└── Program.cs
```

---

## 7. Core Layer — Implementation Details

### BaseEntity

```csharp
// Core/Entities/BaseEntity.cs
namespace {Project}.Core.Entities;

public abstract class BaseEntity
{
    public Guid Id { get; protected set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; protected set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; protected set; }
    public bool IsDeleted { get; protected set; } = false;

    protected void MarkUpdated() => UpdatedAt = DateTime.UtcNow;
}
```

### User Domain Entity

```csharp
// Core/Entities/User.cs
namespace {Project}.Core.Entities;

public sealed class User : BaseEntity
{
    public string FirstName { get; private set; } = string.Empty;
    public string LastName { get; private set; } = string.Empty;
    public string Email { get; private set; } = string.Empty;
    public string PasswordHash { get; private set; } = string.Empty;
    public UserStatus Status { get; private set; } = UserStatus.Active;
    public string? RefreshToken { get; private set; }
    public DateTime? RefreshTokenExpiry { get; private set; }

    private User() { }  // EF Core

    public static User Create(string firstName, string lastName, string email, string passwordHash)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(firstName);
        ArgumentException.ThrowIfNullOrWhiteSpace(lastName);
        ArgumentException.ThrowIfNullOrWhiteSpace(email);

        return new User
        {
            FirstName = firstName,
            LastName = lastName,
            Email = email.ToLowerInvariant(),
            PasswordHash = passwordHash
        };
    }

    public void UpdateProfile(string firstName, string lastName)
    {
        FirstName = firstName;
        LastName = lastName;
        MarkUpdated();
    }

    public void SetRefreshToken(string token, DateTime expiry)
    {
        RefreshToken = token;
        RefreshTokenExpiry = expiry;
        MarkUpdated();
    }

    public void Deactivate()
    {
        Status = UserStatus.Inactive;
        IsDeleted = true;   // soft delete via backing field
        MarkUpdated();
    }
}
```

### Value Object

```csharp
// Core/ValueObjects/Email.cs
namespace {Project}.Core.ValueObjects;

public sealed class Email
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Email Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainException("Email cannot be empty.");

        if (!value.Contains('@'))
            throw new DomainException($"'{value}' is not a valid email address.");

        return new Email(value.ToLowerInvariant());
    }

    public override string ToString() => Value;
}
```

### Domain Exceptions

```csharp
// Core/Exceptions/DomainException.cs
namespace {Project}.Core.Exceptions;

public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
}
```

```csharp
// Core/Exceptions/UserNotFoundException.cs
namespace {Project}.Core.Exceptions;

public sealed class UserNotFoundException : DomainException
{
    public UserNotFoundException(Guid id)
        : base($"User with ID '{id}' was not found.") { }

    public UserNotFoundException(string email)
        : base($"User with email '{email}' was not found.") { }
}
```

### Repository Interface (Core)

```csharp
// Core/Interfaces/Repositories/IRepository.cs
namespace {Project}.Core.Interfaces.Repositories;

public interface IRepository<T> where T : BaseEntity
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default);
    Task AddAsync(T entity, CancellationToken cancellationToken = default);
    void Update(T entity);
    void Remove(T entity);
    Task<bool> ExistsAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default);
}
```

```csharp
// Core/Interfaces/Repositories/IUserRepository.cs
namespace {Project}.Core.Interfaces.Repositories;

public interface IUserRepository : IRepository<User>
{
    Task<User?> GetByEmailAsync(string email, CancellationToken cancellationToken = default);
}
```

```csharp
// Core/Interfaces/Services/IUnitOfWork.cs
namespace {Project}.Core.Interfaces.Services;

public interface IUnitOfWork
{
    IUserRepository Users { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

---

## 8. Application Layer — Implementation Details

### Application Service Interface

```csharp
// Application/Interfaces/IUserService.cs
namespace {Project}.Application.Interfaces;

public interface IUserService
{
    Task<UserResponse> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<UserResponse>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<UserResponse> CreateAsync(CreateUserRequest request, CancellationToken cancellationToken = default);
    Task<UserResponse> UpdateAsync(Guid id, UpdateUserRequest request, CancellationToken cancellationToken = default);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken = default);
}
```

### Application Service Implementation

```csharp
// Application/Services/UserService.cs
namespace {Project}.Application.Services;

public sealed class UserService : IUserService
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly ILogger<UserService> _logger;

    public UserService(IUnitOfWork unitOfWork, ILogger<UserService> logger)
    {
        _unitOfWork = unitOfWork;
        _logger = logger;
    }

    public async Task<UserResponse> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        var user = await _unitOfWork.Users.GetByIdAsync(id, cancellationToken)
            ?? throw new UserNotFoundException(id);

        return MapToResponse(user);
    }

    public async Task<IReadOnlyList<UserResponse>> GetAllAsync(CancellationToken cancellationToken = default)
    {
        var users = await _unitOfWork.Users.GetAllAsync(cancellationToken);
        return users.Select(MapToResponse).ToList();
    }

    public async Task<UserResponse> CreateAsync(
        CreateUserRequest request,
        CancellationToken cancellationToken = default)
    {
        var emailExists = await _unitOfWork.Users.ExistsAsync(
            u => u.Email == request.Email.ToLowerInvariant(), cancellationToken);

        if (emailExists)
            throw new EmailAlreadyExistsException(request.Email);

        var user = User.Create(
            request.FirstName,
            request.LastName,
            request.Email,
            BCrypt.Net.BCrypt.HashPassword(request.Password));

        await _unitOfWork.Users.AddAsync(user, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        _logger.LogInformation("User created: {Email}", user.Email);
        return MapToResponse(user);
    }

    public async Task<UserResponse> UpdateAsync(
        Guid id,
        UpdateUserRequest request,
        CancellationToken cancellationToken = default)
    {
        var user = await _unitOfWork.Users.GetByIdAsync(id, cancellationToken)
            ?? throw new UserNotFoundException(id);

        user.UpdateProfile(request.FirstName, request.LastName);
        _unitOfWork.Users.Update(user);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return MapToResponse(user);
    }

    public async Task DeleteAsync(Guid id, CancellationToken cancellationToken = default)
    {
        var user = await _unitOfWork.Users.GetByIdAsync(id, cancellationToken)
            ?? throw new UserNotFoundException(id);

        user.Deactivate();
        _unitOfWork.Users.Update(user);
        await _unitOfWork.SaveChangesAsync(cancellationToken);
    }

    private static UserResponse MapToResponse(User user) => new()
    {
        Id = user.Id,
        FirstName = user.FirstName,
        LastName = user.LastName,
        Email = user.Email,
        Status = user.Status,
        CreatedAt = user.CreatedAt
    };
}
```

### Application DependencyInjection

```csharp
// Application/DependencyInjection.cs
namespace {Project}.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IAuthService, AuthService>();
        services.AddFluentValidationAutoValidation();
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
        return services;
    }
}
```

---

## 9. Infrastructure Layer — Implementation Details

### ApplicationDbContext

```csharp
// Infrastructure/Persistence/ApplicationDbContext.cs
namespace {Project}.Infrastructure.Persistence;

public sealed class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }

    public DbSet<User> Users => Set<User>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
        base.OnModelCreating(modelBuilder);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Auto-stamp UpdatedAt via domain method is preferred,
        // but intercept here as a safety net:
        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

### Entity Configuration (IEntityTypeConfiguration)

```csharp
// Infrastructure/Persistence/Configurations/UserConfiguration.cs
namespace {Project}.Infrastructure.Persistence.Configurations;

public sealed class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.ToTable("Users");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Id).ValueGeneratedNever();
        builder.Property(x => x.FirstName).IsRequired().HasMaxLength(100);
        builder.Property(x => x.LastName).IsRequired().HasMaxLength(100);
        builder.Property(x => x.Email).IsRequired().HasMaxLength(256);
        builder.Property(x => x.PasswordHash).IsRequired();
        builder.Property(x => x.Status).HasConversion<string>().HasMaxLength(50);
        builder.HasIndex(x => x.Email).IsUnique();
        builder.HasQueryFilter(x => !x.IsDeleted);
    }
}
```

### Repository Implementation

```csharp
// Infrastructure/Repositories/Repository.cs
namespace {Project}.Infrastructure.Repositories;

public class Repository<T> : IRepository<T> where T : BaseEntity
{
    protected readonly ApplicationDbContext _context;
    private readonly DbSet<T> _dbSet;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
        => await _dbSet.FindAsync([id], cancellationToken);

    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken cancellationToken = default)
        => await _dbSet.ToListAsync(cancellationToken);

    public async Task<IReadOnlyList<T>> FindAsync(
        Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default)
        => await _dbSet.Where(predicate).ToListAsync(cancellationToken);

    public async Task AddAsync(T entity, CancellationToken cancellationToken = default)
        => await _dbSet.AddAsync(entity, cancellationToken);

    public void Update(T entity) => _dbSet.Update(entity);

    public void Remove(T entity)
    {
        // Soft delete — entity's Deactivate() sets IsDeleted
        _dbSet.Update(entity);
    }

    public async Task<bool> ExistsAsync(
        Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default)
        => await _dbSet.AnyAsync(predicate, cancellationToken);
}
```

```csharp
// Infrastructure/Repositories/UserRepository.cs
namespace {Project}.Infrastructure.Repositories;

public sealed class UserRepository : Repository<User>, IUserRepository
{
    public UserRepository(ApplicationDbContext context) : base(context) { }

    public async Task<User?> GetByEmailAsync(string email, CancellationToken cancellationToken = default)
        => await _context.Users
            .FirstOrDefaultAsync(u => u.Email == email.ToLowerInvariant(), cancellationToken);
}
```

### UnitOfWork Implementation

```csharp
// Infrastructure/Repositories/UnitOfWork.cs
namespace {Project}.Infrastructure.Repositories;

public sealed class UnitOfWork : IUnitOfWork, IAsyncDisposable
{
    private readonly ApplicationDbContext _context;

    public IUserRepository Users { get; }

    public UnitOfWork(ApplicationDbContext context, IUserRepository users)
    {
        _context = context;
        Users = users;
    }

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        => await _context.SaveChangesAsync(cancellationToken);

    public async ValueTask DisposeAsync() => await _context.DisposeAsync();
}
```

### Infrastructure DependencyInjection

```csharp
// Infrastructure/DependencyInjection.cs
namespace {Project}.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName)));

        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<ITokenService, TokenService>();

        services.Configure<JwtSettings>(configuration.GetSection(nameof(JwtSettings)));

        return services;
    }
}
```

---

## 10. API Layer — Implementation Details

### Program.cs

```csharp
using {Project}.Application;
using {Project}.Infrastructure;
using Serilog;
using SharpGrip.FluentValidation.AutoValidation.Mvc.Extensions;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day));

builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(/* see scaffold.md */);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var jwt = builder.Configuration.GetSection("JwtSettings");
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = jwt["Issuer"],
            ValidAudience = jwt["Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwt["Secret"]!))
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();
if (app.Environment.IsDevelopment()) { app.UseSwagger(); app.UseSwaggerUI(); }
app.UseSerilogRequestLogging();
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseMiddleware<ExceptionHandlingMiddleware>();
app.MapControllers();
app.Run();

public partial class Program { }
```

### UsersController

```csharp
// Controllers/UsersController.cs
namespace {Project}.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public sealed class UsersController : ControllerBase
{
    private readonly IUserService _userService;

    public UsersController(IUserService userService) => _userService = userService;

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<UserResponse>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        var user = await _userService.GetByIdAsync(id, cancellationToken);
        return Ok(ApiResponse<UserResponse>.Ok(user));
    }

    [HttpGet]
    [ProducesResponseType(typeof(ApiResponse<IReadOnlyList<UserResponse>>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll(CancellationToken cancellationToken)
    {
        var users = await _userService.GetAllAsync(cancellationToken);
        return Ok(ApiResponse<IReadOnlyList<UserResponse>>.Ok(users));
    }

    [HttpPost]
    [AllowAnonymous]
    [ProducesResponseType(typeof(ApiResponse<UserResponse>), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        [FromBody] CreateUserRequest request, CancellationToken cancellationToken)
    {
        var user = await _userService.CreateAsync(request, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id = user.Id },
            ApiResponse<UserResponse>.Ok(user, "User created."));
    }

    [HttpPut("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<UserResponse>), StatusCodes.Status200OK)]
    public async Task<IActionResult> Update(Guid id, [FromBody] UpdateUserRequest request,
        CancellationToken cancellationToken)
    {
        var user = await _userService.UpdateAsync(id, request, cancellationToken);
        return Ok(ApiResponse<UserResponse>.Ok(user));
    }

    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public async Task<IActionResult> Delete(Guid id, CancellationToken cancellationToken)
    {
        await _userService.DeleteAsync(id, cancellationToken);
        return NoContent();
    }
}
```

---

## 11. Exception Handling Middleware

Refer to the pattern in `repository-uow-architecture.md` Section 12. Replace domain-specific exception types with those from `{Project}.Core.Exceptions`:

- `DomainException` → 400
- `UserNotFoundException` → 404
- `EmailAlreadyExistsException` → 409
- `UnauthorizedAccessException` → 401
- All others → 500

---

## 12. NuGet Packages Per Project

### `{Project}.Api`

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Swashbuckle.AspNetCore
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package SharpGrip.FluentValidation.AutoValidation.Mvc
```

### `{Project}.Application`

```bash
dotnet add package FluentValidation
dotnet add package FluentValidation.AspNetCore
dotnet add package SharpGrip.FluentValidation.AutoValidation.Mvc
dotnet add package Microsoft.Extensions.Logging.Abstractions
dotnet add package BCrypt.Net-Next
```

### `{Project}.Infrastructure`

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer    # or Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.IdentityModel.Tokens
dotnet add package System.IdentityModel.Tokens.Jwt
```

### `{Project}.Core`

```bash
# No NuGet packages — Core must remain free of external dependencies
```

---

## 13. CLI Commands to Scaffold the Solution

```bash
dotnet new sln -n {Project}

dotnet new webapi -o src/{Project}.Api --framework net10.0
dotnet new classlib -o src/{Project}.Application --framework net10.0
dotnet new classlib -o src/{Project}.Infrastructure --framework net10.0
dotnet new classlib -o src/{Project}.Core --framework net10.0

dotnet sln add src/{Project}.Api/{Project}.Api.csproj
dotnet sln add src/{Project}.Application/{Project}.Application.csproj
dotnet sln add src/{Project}.Infrastructure/{Project}.Infrastructure.csproj
dotnet sln add src/{Project}.Core/{Project}.Core.csproj

dotnet add src/{Project}.Api reference src/{Project}.Application src/{Project}.Infrastructure
dotnet add src/{Project}.Application reference src/{Project}.Core
dotnet add src/{Project}.Infrastructure reference src/{Project}.Application src/{Project}.Core

# Test projects (if selected)
dotnet new xunit -o tests/{Project}.Tests.Unit --framework net10.0
dotnet new xunit -o tests/{Project}.Tests.Integration --framework net10.0
dotnet sln add tests/{Project}.Tests.Unit/{Project}.Tests.Unit.csproj
dotnet sln add tests/{Project}.Tests.Integration/{Project}.Tests.Integration.csproj
dotnet add tests/{Project}.Tests.Unit reference src/{Project}.Application src/{Project}.Core
dotnet add tests/{Project}.Tests.Integration reference src/{Project}.Api src/{Project}.Infrastructure
```

---

## 14. Unit Testing Rules for Onion Architecture

- Test **Application Services** in isolation — mock `IUnitOfWork` and all repositories.
- Test **Domain Entities** directly — no mocking (pure functions).
- Test **Value Objects** directly — assert creation logic and validation.
- Test **Validators** independently — instantiate and call `.Validate()`.

```csharp
public class UserServiceTests
{
    private readonly Mock<IUnitOfWork> _uowMock = new();
    private readonly Mock<IUserRepository> _userRepoMock = new();
    private readonly Mock<ILogger<UserService>> _loggerMock = new();
    private readonly UserService _sut;

    public UserServiceTests()
    {
        _uowMock.Setup(u => u.Users).Returns(_userRepoMock.Object);
        _sut = new UserService(_uowMock.Object, _loggerMock.Object);
    }

    [Fact]
    public async Task CreateAsync_WithDuplicateEmail_ThrowsEmailAlreadyExistsException()
    {
        _userRepoMock.Setup(r => r.ExistsAsync(It.IsAny<Expression<Func<User, bool>>>(),
            It.IsAny<CancellationToken>())).ReturnsAsync(true);

        var act = () => _sut.CreateAsync(new CreateUserRequest
        {
            FirstName = "John", LastName = "Doe",
            Email = "existing@example.com", Password = "Password1!"
        });

        await act.Should().ThrowAsync<EmailAlreadyExistsException>();
    }
}
```

---

## 15. Integration Testing Rules for Onion Architecture

- Replace `ApplicationDbContext` registration with EF Core InMemory in `TestWebApplicationFactory`.
- Seed test data in `ConfigureWebHost`.
- Test complete HTTP input → Application Service → Repository → InMemory DB cycle.
- Keep test databases isolated per test class using unique DB names.

---

## 16. Cloud Provider Services (Conditional — apply only if cloud provider selected)

> Generate this section only when the user selected **Azure** or **AWS** in Question 1.8 of `scaffold.md`.

### 16.1 — Folder Structure

Add cloud service implementations to `{Project}.Infrastructure`:

```
{Project}.Infrastructure/
├── CloudServices/
│   ├── Storage/
│   │   ├── AzureBlobStorageService.cs    ← Azure only
│   │   └── AwsS3StorageService.cs        ← AWS only
│   └── Secrets/
│       ├── AzureKeyVaultSecretService.cs ← Azure only
│       └── AwsSecretsManagerSecretService.cs ← AWS only
```

Add cloud service interfaces to `{Project}.Application`:

```
{Project}.Application/
├── Interfaces/
│   ├── IStorageService.cs    ← if storage selected
│   └── ISecretService.cs     ← if Key Vault / Secrets Manager selected
```

### 16.2 — Interface Contract

```csharp
// {Project}.Application/Interfaces/IStorageService.cs
/// <summary>Abstraction for cloud blob/object storage operations.</summary>
public interface IStorageService
{
    Task<string> UploadAsync(string containerName, string blobName, Stream content,
        string contentType, CancellationToken cancellationToken = default);
    Task<Stream> DownloadAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
    Task DeleteAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
}

// {Project}.Application/Interfaces/ISecretService.cs
/// <summary>Abstraction for runtime secret retrieval.</summary>
public interface ISecretService
{
    Task<string?> GetSecretAsync(string secretName, CancellationToken cancellationToken = default);
}
```

### 16.3 — Registration (Infrastructure `DependencyInjection.cs`)

```csharp
// Inside AddInfrastructure extension method in {Project}.Infrastructure/DependencyInjection.cs

// ── Azure Blob Storage (if selected) ──────────────────────────────────────
services.AddSingleton<Azure.Core.TokenCredential>(_ => new DefaultAzureCredential());
services.AddSingleton(sp =>
{
    var accountName = configuration["AzureBlobStorage:AccountName"]!;
    return new BlobServiceClient(
        new Uri($"https://{accountName}.blob.core.windows.net"),
        sp.GetRequiredService<Azure.Core.TokenCredential>());
});
services.AddScoped<IStorageService, AzureBlobStorageService>();

// ── Azure Key Vault (if selected) ─────────────────────────────────────────
services.AddSingleton(sp =>
    new SecretClient(new Uri(configuration["Azure:KeyVaultUri"]!),
        sp.GetRequiredService<Azure.Core.TokenCredential>()));
services.AddScoped<ISecretService, AzureKeyVaultSecretService>();

// ── AWS S3 (if selected — replace Azure storage registration above) ────────
services.AddAWSService<IAmazonS3>();
services.AddScoped<IStorageService, AwsS3StorageService>();

// ── AWS Secrets Manager (if selected — replace Azure secrets above) ────────
services.AddAWSService<IAmazonSecretsManager>();
services.AddScoped<ISecretService, AwsSecretsManagerSecretService>();
```

### 16.4 — Usage in Application Services

Inject `IStorageService` or `ISecretService` into Application Service classes (e.g., `UserService`). The service class depends on the interface only — never on the Azure/AWS SDK directly.

```csharp
public sealed class DocumentService : IDocumentService
{
    private readonly IStorageService _storageService;

    public DocumentService(IStorageService storageService)
        => _storageService = storageService;

    public async Task<string> UploadDocumentAsync(string fileName, Stream content,
        string contentType, CancellationToken cancellationToken)
    {
        var blobName = $"documents/{Guid.NewGuid()}_{fileName}";
        return await _storageService.UploadAsync("app-documents", blobName, content,
            contentType, cancellationToken);
    }
}
```

---

## 17. Third-Party HTTP Client Integration with Polly (Conditional)

> Generate this section only when the user selected **Yes** in Question 1.9 of `scaffold.md`.

### 17.1 — Folder Structure

```
{Project}.Application/
├── Interfaces/
│   └── HttpClients/
│       └── I{ServiceName}Client.cs    ← interface per third-party service

{Project}.Infrastructure/
├── HttpClients/
│   ├── {ServiceName}Client.cs         ← typed HttpClient implementation
│   └── HttpClientServiceExtensions.cs ← DI registration with Polly
```

### 17.2 — Interface Pattern

```csharp
// {Project}.Application/Interfaces/HttpClients/IPaymentServiceClient.cs
/// <summary>Typed HTTP client contract for the Payment external service.</summary>
public interface IPaymentServiceClient
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request, CancellationToken cancellationToken = default);
}
```

### 17.3 — Implementation Pattern

```csharp
// {Project}.Infrastructure/HttpClients/PaymentServiceClient.cs
public sealed class PaymentServiceClient : IPaymentServiceClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<PaymentServiceClient> _logger;

    public PaymentServiceClient(HttpClient httpClient, ILogger<PaymentServiceClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request,
        CancellationToken cancellationToken = default)
    {
        var response = await _httpClient.PostAsJsonAsync("payments", request, cancellationToken);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PaymentResult>(cancellationToken: cancellationToken)
               ?? throw new InvalidOperationException("Null response from payment service.");
    }
}
```

### 17.4 — Polly Registration

```csharp
// {Project}.Infrastructure/HttpClients/HttpClientServiceExtensions.cs
public static class HttpClientServiceExtensions
{
    public static IServiceCollection AddHttpClients(
        this IServiceCollection services, IConfiguration configuration)
    {
        services.AddResiliencePipeline("default-http-pipeline", b =>
        {
            b.AddRetry(new RetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromSeconds(2),
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true,
                ShouldHandle = new PredicateBuilder()
                    .Handle<HttpRequestException>()
                    .HandleResult<HttpResponseMessage>(r =>
                        r.StatusCode is HttpStatusCode.InternalServerError
                            or HttpStatusCode.BadGateway
                            or HttpStatusCode.ServiceUnavailable
                            or HttpStatusCode.GatewayTimeout)
            });

            b.AddCircuitBreaker(new CircuitBreakerStrategyOptions
            {
                FailureRatio = 0.5,
                SamplingDuration = TimeSpan.FromSeconds(30),
                MinimumThroughput = 5,
                BreakDuration = TimeSpan.FromSeconds(15),
                ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>()
            });

            b.AddTimeout(TimeSpan.FromSeconds(10));
        });

        services
            .AddHttpClient<IPaymentServiceClient, PaymentServiceClient>(client =>
            {
                client.BaseAddress = new Uri(configuration["HttpClients:PaymentService:BaseUrl"]!);
                client.Timeout = TimeSpan.FromSeconds(
                    configuration.GetValue<int>("HttpClients:PaymentService:TimeoutSeconds", 30));
            })
            .AddResilienceHandler("default-http-pipeline", (b, ctx) =>
            {
                b.AddPipeline(ctx.ServiceProvider
                    .GetRequiredService<ResiliencePipelineProvider<string>>()
                    .GetPipeline("default-http-pipeline"));
            });

        // Repeat AddHttpClient block for each additional service

        return services;
    }
}
```

Call `services.AddHttpClients(configuration)` inside `{Project}.Infrastructure/DependencyInjection.cs`.
