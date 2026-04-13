# Repository + Unit of Work Pattern — Detailed Instruction File

> This file is loaded automatically when the user selects **Repository + Unit of Work Pattern** in `scaffold.md`.
> Follow every section completely. Do not skip layers, rules, or code examples.

---

## 1. Architecture Overview

The Repository + Unit of Work pattern separates data access logic from business logic using a classic layered architecture. Each layer has a clear, single responsibility.

```
┌──────────────────────────────────────────────────────┐
│                      API Layer                       │
│     Controllers, Middleware, DI Registration         │
├──────────────────────────────────────────────────────┤
│                    Service Layer                     │
│     Business logic, orchestration, token service     │
├──────────────────────────────────────────────────────┤
│         Repository + Unit of Work Layer              │
│     Data access, IRepository, IUnitOfWork            │
├──────────────────────────────────────────────────────┤
│                   Database Layer                     │
│     EF Core DbContext, Configurations, Migrations    │
├──────────────────────────────────────────────────────┤
│                   Entities Layer                     │
│     Domain entities, Request/Response models,Enums   │
├──────────────────────────────────────────────────────┤
│                  Extension Layer                     │
│     Extension methods, helper utilities              │
├──────────────────────────────────────────────────────┤
│                  ThirdParty Layer                    │
│     External service wrappers and integrations       │
└──────────────────────────────────────────────────────┘
```

---

## 2. Solution Structure

### Solution Name: `{Project}`

```
{Project}.sln
│
├── src/
│   ├── {Project}.Api/
│   ├── {Project}.Service/
│   ├── {Project}.Repository/
│   ├── {Project}.Database/
│   ├── {Project}.Entities/
│   ├── {Project}.Extension/
│   └── {Project}.ThirdParty/
│
└── tests/  (if test projects selected)
    ├── {Project}.Tests.Unit/
    └── {Project}.Tests.Integration/
```

---

## 3. Project Descriptions

| Project | Type | Responsibility |
|---------|------|----------------|
| `{Project}.Api` | ASP.NET Core Web API | Controllers, middleware, filters, DI wiring, Swagger |
| `{Project}.Service` | Class Library | Business logic, service classes, JWT token service |
| `{Project}.Repository` | Class Library | Generic + specific repositories, Unit of Work |
| `{Project}.Database` | Class Library | EF Core DbContext, entity configurations, migrations |
| `{Project}.Entities` | Class Library | Domain entities, request/response DTOs, validators, enums, common models |
| `{Project}.Extension` | Class Library | Extension methods, helper utilities — no domain dependency |
| `{Project}.ThirdParty` | Class Library | External integrations (email, SMS, cloud storage, etc.) |

---

## 4. Project Reference Rules

```
{Project}.Api
    ├── {Project}.Service
    ├── {Project}.Entities
    ├── {Project}.Extension
    └── {Project}.ThirdParty

{Project}.Service
    ├── {Project}.Repository
    └── {Project}.Entities

{Project}.Repository
    ├── {Project}.Database
    └── {Project}.Entities

{Project}.Database
    └── {Project}.Entities

{Project}.Entities
    └── NO references

{Project}.Extension
    └── NO references

{Project}.ThirdParty
    └── NO references
```

> **Critical rules:**
> - `{Project}.Api` must NOT directly reference `{Project}.Repository` or `{Project}.Database`.
> - `{Project}.Database` must NOT reference `{Project}.Service` or `{Project}.Repository`.
> - `{Project}.Entities` must remain dependency-free.

---

## 5. Naming Conventions

| Concept | Pattern | Example |
|---------|---------|---------|
| Solution/Project | `{Project}.{Layer}` | `Ordering.Api` |
| Service Interface | `I{Entity}Service` | `IUserService` |
| Service Class | `{Entity}Service` | `UserService` |
| Repository Interface | `I{Entity}Repository` | `IUserRepository` |
| Repository Class | `{Entity}Repository` | `UserRepository` |
| Generic Repository | `IGenericRepository<T>` / `GenericRepository<T>` | — |
| Unit of Work Interface | `IUnitOfWork` | — |
| Unit of Work Class | `UnitOfWork` | — |
| Entity | PascalCase noun | `User`, `Order`, `Product` |
| Request Model | `{Action}{Entity}Request` | `CreateUserRequest` |
| Response Model | `{Entity}Response` | `UserResponse` |
| Validator | `{Request}Validator` | `CreateUserRequestValidator` |
| Entity Config | `{Entity}Configuration` | `UserConfiguration` |
| Controller | `{Entity}sController` | `UsersController` |

---

## 6. Folder Structure

### `{Project}.Entities`

```
{Project}.Entities/
├── Domain/
│   ├── BaseEntity.cs
│   └── User.cs
├── RequestModel/
│   ├── Auth/
│   │   ├── LoginRequest.cs
│   │   └── RegisterRequest.cs
│   └── Users/
│       ├── CreateUserRequest.cs
│       └── UpdateUserRequest.cs
├── ResponseModel/
│   ├── Auth/
│   │   └── LoginResponse.cs
│   └── Users/
│       └── UserResponse.cs
├── Validators/
│   ├── Auth/
│   │   ├── LoginRequestValidator.cs
│   │   └── RegisterRequestValidator.cs
│   └── Users/
│       ├── CreateUserRequestValidator.cs
│       └── UpdateUserRequestValidator.cs
├── Enums/
│   └── UserStatus.cs
└── Common/
    └── ApiResponse.cs
```

### `{Project}.Database`

```
{Project}.Database/
├── Context/
│   └── ApplicationDbContext.cs
├── Configurations/
│   └── UserConfiguration.cs
├── Migrations/
└── Extensions/
    └── DatabaseServiceExtensions.cs
```

### `{Project}.Repository`

```
{Project}.Repository/
├── Interfaces/
│   ├── IGenericRepository.cs
│   ├── IUserRepository.cs
│   └── IUnitOfWork.cs
├── Implementations/
│   ├── GenericRepository.cs
│   ├── UserRepository.cs
│   └── UnitOfWork.cs
└── Extensions/
    └── RepositoryServiceExtensions.cs
```

### `{Project}.Service`

```
{Project}.Service/
├── Interfaces/
│   ├── IUserService.cs
│   └── ITokenService.cs
├── Implementations/
│   ├── UserService.cs
│   └── TokenService.cs
└── Extensions/
    └── ServiceExtensions.cs
```

### `{Project}.Api`

```
{Project}.Api/
├── Controllers/
│   ├── BaseApiController.cs
│   ├── AuthController.cs
│   └── UsersController.cs
├── Middleware/
│   └── ExceptionHandlingMiddleware.cs
├── appsettings.json
├── appsettings.Development.json
└── Program.cs
```

---

## 7. Entities Layer — Implementation Details

### BaseEntity

```csharp
// Domain/BaseEntity.cs
namespace {Project}.Entities.Domain;

public abstract class BaseEntity
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; set; }
    public bool IsDeleted { get; set; } = false;
}
```

### User Entity

```csharp
// Domain/User.cs
namespace {Project}.Entities.Domain;

public sealed class User : BaseEntity
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string? RefreshToken { get; set; }
    public DateTime? RefreshTokenExpiry { get; set; }
    public UserStatus Status { get; set; } = UserStatus.Active;
}
```

### Request Model

```csharp
// RequestModel/Users/CreateUserRequest.cs
namespace {Project}.Entities.RequestModel.Users;

public sealed class CreateUserRequest
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
}
```

### Response Model

```csharp
// ResponseModel/Users/UserResponse.cs
namespace {Project}.Entities.ResponseModel.Users;

public sealed class UserResponse
{
    public Guid Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string FullName => $"{FirstName} {LastName}";
    public UserStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

### ApiResponse Wrapper

```csharp
// Common/ApiResponse.cs
namespace {Project}.Entities.Common;

public sealed class ApiResponse<T>
{
    public bool Success { get; init; }
    public string? Message { get; init; }
    public T? Data { get; init; }
    public IEnumerable<string>? Errors { get; init; }

    public static ApiResponse<T> Ok(T data, string? message = null)
        => new() { Success = true, Data = data, Message = message };

    public static ApiResponse<T> Fail(string message, IEnumerable<string>? errors = null)
        => new() { Success = false, Message = message, Errors = errors };
}
```

### Validator

```csharp
// Validators/Users/CreateUserRequestValidator.cs
namespace {Project}.Entities.Validators.Users;

public sealed class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.LastName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Password)
            .NotEmpty().MinimumLength(8)
            .Matches(@"[A-Z]").WithMessage("Must contain uppercase.")
            .Matches(@"[0-9]").WithMessage("Must contain a digit.");
    }
}
```

---

## 8. Database Layer — Implementation Details

### ApplicationDbContext

```csharp
// Context/ApplicationDbContext.cs
namespace {Project}.Database.Context;

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
        foreach (var entry in ChangeTracker.Entries<BaseEntity>())
        {
            if (entry.State == EntityState.Modified)
                entry.Entity.UpdatedAt = DateTime.UtcNow;
        }
        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

### Entity Configuration

```csharp
// Configurations/UserConfiguration.cs
namespace {Project}.Database.Configurations;

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

### Database Service Extensions

```csharp
// Extensions/DatabaseServiceExtensions.cs
namespace {Project}.Database.Extensions;

public static class DatabaseServiceExtensions
{
    public static IServiceCollection AddDatabase(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName)));

        return services;
    }
}
```

---

## 9. Repository Layer — Implementation Details

### Generic Repository Interface

```csharp
// Interfaces/IGenericRepository.cs
namespace {Project}.Repository.Interfaces;

public interface IGenericRepository<T> where T : BaseEntity
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default);
    Task<T> AddAsync(T entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(T entity, CancellationToken cancellationToken = default);  // Soft delete
    Task<bool> ExistsAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default);
    Task<int> CountAsync(Expression<Func<T, bool>>? predicate = null, CancellationToken cancellationToken = default);
}
```

### Generic Repository Implementation

```csharp
// Implementations/GenericRepository.cs
namespace {Project}.Repository.Implementations;

public class GenericRepository<T> : IGenericRepository<T> where T : BaseEntity
{
    protected readonly ApplicationDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public GenericRepository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
        => await _dbSet.FirstOrDefaultAsync(x => x.Id == id, cancellationToken);

    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken cancellationToken = default)
        => await _dbSet.ToListAsync(cancellationToken);

    public async Task<IReadOnlyList<T>> FindAsync(
        Expression<Func<T, bool>> predicate,
        CancellationToken cancellationToken = default)
        => await _dbSet.Where(predicate).ToListAsync(cancellationToken);

    public async Task<T> AddAsync(T entity, CancellationToken cancellationToken = default)
    {
        await _dbSet.AddAsync(entity, cancellationToken);
        return entity;
    }

    public Task UpdateAsync(T entity, CancellationToken cancellationToken = default)
    {
        _dbSet.Update(entity);
        return Task.CompletedTask;
    }

    public Task DeleteAsync(T entity, CancellationToken cancellationToken = default)
    {
        entity.IsDeleted = true;
        entity.UpdatedAt = DateTime.UtcNow;
        _dbSet.Update(entity);
        return Task.CompletedTask;
    }

    public async Task<bool> ExistsAsync(
        Expression<Func<T, bool>> predicate,
        CancellationToken cancellationToken = default)
        => await _dbSet.AnyAsync(predicate, cancellationToken);

    public async Task<int> CountAsync(
        Expression<Func<T, bool>>? predicate = null,
        CancellationToken cancellationToken = default)
        => predicate is null
            ? await _dbSet.CountAsync(cancellationToken)
            : await _dbSet.CountAsync(predicate, cancellationToken);
}
```

### Specific Repository Interface

```csharp
// Interfaces/IUserRepository.cs
namespace {Project}.Repository.Interfaces;

public interface IUserRepository : IGenericRepository<User>
{
    Task<User?> GetByEmailAsync(string email, CancellationToken cancellationToken = default);
    Task<User?> GetByRefreshTokenAsync(string refreshToken, CancellationToken cancellationToken = default);
}
```

### Specific Repository Implementation

```csharp
// Implementations/UserRepository.cs
namespace {Project}.Repository.Implementations;

public sealed class UserRepository : GenericRepository<User>, IUserRepository
{
    public UserRepository(ApplicationDbContext context) : base(context) { }

    public async Task<User?> GetByEmailAsync(string email, CancellationToken cancellationToken = default)
        => await _dbSet.FirstOrDefaultAsync(
            x => x.Email == email.ToLowerInvariant(), cancellationToken);

    public async Task<User?> GetByRefreshTokenAsync(string refreshToken, CancellationToken cancellationToken = default)
        => await _dbSet.FirstOrDefaultAsync(
            x => x.RefreshToken == refreshToken &&
                 x.RefreshTokenExpiry > DateTime.UtcNow, cancellationToken);
}
```

### Unit of Work Interface

```csharp
// Interfaces/IUnitOfWork.cs
namespace {Project}.Repository.Interfaces;

public interface IUnitOfWork : IAsyncDisposable
{
    IUserRepository Users { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
    Task BeginTransactionAsync(CancellationToken cancellationToken = default);
    Task CommitTransactionAsync(CancellationToken cancellationToken = default);
    Task RollbackTransactionAsync(CancellationToken cancellationToken = default);
}
```

### Unit of Work Implementation

```csharp
// Implementations/UnitOfWork.cs
namespace {Project}.Repository.Implementations;

public sealed class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction? _transaction;

    public IUserRepository Users { get; }

    public UnitOfWork(ApplicationDbContext context, IUserRepository userRepository)
    {
        _context = context;
        Users = userRepository;
    }

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        => await _context.SaveChangesAsync(cancellationToken);

    public async Task BeginTransactionAsync(CancellationToken cancellationToken = default)
        => _transaction = await _context.Database.BeginTransactionAsync(cancellationToken);

    public async Task CommitTransactionAsync(CancellationToken cancellationToken = default)
    {
        if (_transaction is not null)
            await _transaction.CommitAsync(cancellationToken);
    }

    public async Task RollbackTransactionAsync(CancellationToken cancellationToken = default)
    {
        if (_transaction is not null)
            await _transaction.RollbackAsync(cancellationToken);
    }

    public async ValueTask DisposeAsync()
    {
        if (_transaction is not null)
            await _transaction.DisposeAsync();
        await _context.DisposeAsync();
    }
}
```

### Repository Service Extensions

```csharp
// Extensions/RepositoryServiceExtensions.cs
namespace {Project}.Repository.Extensions;

public static class RepositoryServiceExtensions
{
    public static IServiceCollection AddRepositories(this IServiceCollection services)
    {
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        return services;
    }
}
```

---

## 10. Service Layer — Implementation Details

### Service Interface

```csharp
// Interfaces/IUserService.cs
namespace {Project}.Service.Interfaces;

public interface IUserService
{
    Task<UserResponse> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<UserResponse>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<UserResponse> CreateAsync(CreateUserRequest request, CancellationToken cancellationToken = default);
    Task<UserResponse> UpdateAsync(Guid id, UpdateUserRequest request, CancellationToken cancellationToken = default);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken = default);
}
```

### Service Implementation

```csharp
// Implementations/UserService.cs
namespace {Project}.Service.Implementations;

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
            ?? throw new KeyNotFoundException($"User with ID '{id}' not found.");

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
        var emailInUse = await _unitOfWork.Users.ExistsAsync(
            u => u.Email == request.Email.ToLowerInvariant(), cancellationToken);

        if (emailInUse)
            throw new InvalidOperationException($"Email '{request.Email}' is already registered.");

        var user = new User
        {
            FirstName = request.FirstName,
            LastName = request.LastName,
            Email = request.Email.ToLowerInvariant(),
            PasswordHash = BCrypt.Net.BCrypt.HashPassword(request.Password)
        };

        await _unitOfWork.Users.AddAsync(user, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        _logger.LogInformation("User {Email} created with ID {Id}", user.Email, user.Id);
        return MapToResponse(user);
    }

    public async Task<UserResponse> UpdateAsync(
        Guid id,
        UpdateUserRequest request,
        CancellationToken cancellationToken = default)
    {
        var user = await _unitOfWork.Users.GetByIdAsync(id, cancellationToken)
            ?? throw new KeyNotFoundException($"User with ID '{id}' not found.");

        user.FirstName = request.FirstName;
        user.LastName = request.LastName;
        user.UpdatedAt = DateTime.UtcNow;

        await _unitOfWork.Users.UpdateAsync(user, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return MapToResponse(user);
    }

    public async Task DeleteAsync(Guid id, CancellationToken cancellationToken = default)
    {
        var user = await _unitOfWork.Users.GetByIdAsync(id, cancellationToken)
            ?? throw new KeyNotFoundException($"User with ID '{id}' not found.");

        await _unitOfWork.Users.DeleteAsync(user, cancellationToken);
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

### TokenService

```csharp
// Implementations/TokenService.cs
namespace {Project}.Service.Implementations;

public sealed class TokenService : ITokenService
{
    private readonly JwtSettings _jwtSettings;

    public TokenService(IOptions<JwtSettings> jwtSettings) => _jwtSettings = jwtSettings.Value;

    public string GenerateAccessToken(User user, IList<string>? roles = null)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64)
        };

        if (roles is not null)
            claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.Secret));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpiryMinutes),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
        => Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
}
```

### Service Extensions

```csharp
// Extensions/ServiceExtensions.cs
namespace {Project}.Service.Extensions;

public static class ServiceExtensions
{
    public static IServiceCollection AddServices(this IServiceCollection services, IConfiguration configuration)
    {
        services.Configure<JwtSettings>(configuration.GetSection(nameof(JwtSettings)));
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<ITokenService, TokenService>();
        services.AddScoped<IAuthService, AuthService>();
        return services;
    }
}
```

---

## 11. API Layer — Implementation Details

### Program.cs

```csharp
using {Project}.Database.Extensions;
using {Project}.Repository.Extensions;
using {Project}.Service.Extensions;
using Serilog;
using SharpGrip.FluentValidation.AutoValidation.Mvc.Extensions;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day));

builder.Services.AddDatabase(builder.Configuration);
builder.Services.AddRepositories();
builder.Services.AddServices(builder.Configuration);

builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(/* configure as in scaffold.md */);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var jwtSettings = builder.Configuration.GetSection("JwtSettings");
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = jwtSettings["Issuer"],
            ValidAudience = jwtSettings["Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtSettings["Secret"]!))
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

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
        [FromBody] CreateUserRequest request,
        CancellationToken cancellationToken)
    {
        var user = await _userService.CreateAsync(request, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id = user.Id },
            ApiResponse<UserResponse>.Ok(user, "User created successfully."));
    }

    [HttpPut("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<UserResponse>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(
        Guid id,
        [FromBody] UpdateUserRequest request,
        CancellationToken cancellationToken)
    {
        var user = await _userService.UpdateAsync(id, request, cancellationToken);
        return Ok(ApiResponse<UserResponse>.Ok(user, "User updated successfully."));
    }

    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(Guid id, CancellationToken cancellationToken)
    {
        await _userService.DeleteAsync(id, cancellationToken);
        return NoContent();
    }
}
```

---

## 12. Exception Handling Middleware

```csharp
// Middleware/ExceptionHandlingMiddleware.cs
namespace {Project}.Api.Middleware;

public sealed class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            var errors = ex.Errors.Select(e => e.ErrorMessage);
            await WriteResponseAsync(context, StatusCodes.Status400BadRequest,
                ApiResponse<object>.Fail("Validation failed.", errors));
        }
        catch (KeyNotFoundException ex)
        {
            await WriteResponseAsync(context, StatusCodes.Status404NotFound,
                ApiResponse<object>.Fail(ex.Message));
        }
        catch (InvalidOperationException ex)
        {
            await WriteResponseAsync(context, StatusCodes.Status409Conflict,
                ApiResponse<object>.Fail(ex.Message));
        }
        catch (UnauthorizedAccessException ex)
        {
            await WriteResponseAsync(context, StatusCodes.Status401Unauthorized,
                ApiResponse<object>.Fail(ex.Message));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            await WriteResponseAsync(context, StatusCodes.Status500InternalServerError,
                ApiResponse<object>.Fail("An unexpected error occurred."));
        }
    }

    private static async Task WriteResponseAsync<T>(HttpContext context, int statusCode, ApiResponse<T> body)
    {
        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(body);
    }
}
```

---

## 13. appsettings.json Template

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database={Project}Db;Trusted_Connection=True;TrustServerCertificate=True"
  },
  "JwtSettings": {
    "Secret": "YOUR-SECRET-KEY-MINIMUM-32-CHARACTERS-LONG",
    "Issuer": "{Project}",
    "Audience": "{Project}Users",
    "ExpiryMinutes": 60,
    "RefreshTokenExpiryDays": 7
  },
  "AllowedHosts": "*"
}
```

---

## 14. NuGet Packages Per Project

### `{Project}.Api`

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Swashbuckle.AspNetCore
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package SharpGrip.FluentValidation.AutoValidation.Mvc
```

### `{Project}.Service`

```bash
dotnet add package BCrypt.Net-Next
dotnet add package Microsoft.Extensions.Options
dotnet add package Microsoft.IdentityModel.Tokens
dotnet add package System.IdentityModel.Tokens.Jwt
dotnet add package Microsoft.Extensions.Logging.Abstractions
```

### `{Project}.Repository`

```bash
dotnet add package Microsoft.EntityFrameworkCore
```

### `{Project}.Database`

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer    # or Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Design
```

### `{Project}.Entities`

```bash
dotnet add package FluentValidation
dotnet add package FluentValidation.AspNetCore
```

---

## 15. CLI Commands to Scaffold the Solution

```bash
dotnet new sln -n {Project}

dotnet new webapi -o src/{Project}.Api --framework net10.0
dotnet new classlib -o src/{Project}.Service --framework net10.0
dotnet new classlib -o src/{Project}.Repository --framework net10.0
dotnet new classlib -o src/{Project}.Database --framework net10.0
dotnet new classlib -o src/{Project}.Entities --framework net10.0
dotnet new classlib -o src/{Project}.Extension --framework net10.0
dotnet new classlib -o src/{Project}.ThirdParty --framework net10.0

dotnet sln add src/{Project}.Api/{Project}.Api.csproj
dotnet sln add src/{Project}.Service/{Project}.Service.csproj
dotnet sln add src/{Project}.Repository/{Project}.Repository.csproj
dotnet sln add src/{Project}.Database/{Project}.Database.csproj
dotnet sln add src/{Project}.Entities/{Project}.Entities.csproj
dotnet sln add src/{Project}.Extension/{Project}.Extension.csproj
dotnet sln add src/{Project}.ThirdParty/{Project}.ThirdParty.csproj

# References
dotnet add src/{Project}.Api reference src/{Project}.Service src/{Project}.Entities src/{Project}.Extension src/{Project}.ThirdParty
dotnet add src/{Project}.Service reference src/{Project}.Repository src/{Project}.Entities
dotnet add src/{Project}.Repository reference src/{Project}.Database src/{Project}.Entities
dotnet add src/{Project}.Database reference src/{Project}.Entities

# Test projects (if selected)
dotnet new xunit -o tests/{Project}.Tests.Unit --framework net10.0
dotnet new xunit -o tests/{Project}.Tests.Integration --framework net10.0
dotnet sln add tests/{Project}.Tests.Unit/{Project}.Tests.Unit.csproj
dotnet sln add tests/{Project}.Tests.Integration/{Project}.Tests.Integration.csproj
dotnet add tests/{Project}.Tests.Unit reference src/{Project}.Service src/{Project}.Repository src/{Project}.Entities
dotnet add tests/{Project}.Tests.Integration reference src/{Project}.Api src/{Project}.Database src/{Project}.Entities
```

---

## 16. Unit Testing Strategy

- Test **Service classes** independently by mocking `IUnitOfWork` and its repositories.
- Test **Validators** independently by calling `.Validate()` directly.
- Never touch real DB or real DbContext in unit tests.

```csharp
public class UserServiceTests
{
    private readonly Mock<IUnitOfWork> _uowMock = new();
    private readonly Mock<IUserRepository> _userRepoMock = new();
    private readonly Mock<ILogger<UserService>> _loggerMock = new();
    private readonly UserService _sut;
    private readonly IFixture _fixture = new Fixture().Customize(new AutoMoqCustomization());

    public UserServiceTests()
    {
        _uowMock.Setup(u => u.Users).Returns(_userRepoMock.Object);
        _sut = new UserService(_uowMock.Object, _loggerMock.Object);
    }

    [Fact]
    public async Task CreateAsync_WithUniqueEmail_ReturnsUserResponse()
    {
        var request = _fixture.Create<CreateUserRequest>();
        _userRepoMock.Setup(r => r.ExistsAsync(It.IsAny<Expression<Func<User, bool>>>(),
            It.IsAny<CancellationToken>())).ReturnsAsync(false);
        _userRepoMock.Setup(r => r.AddAsync(It.IsAny<User>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync((User u, CancellationToken _) => u);
        _uowMock.Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>())).ReturnsAsync(1);

        var result = await _sut.CreateAsync(request);

        result.Should().NotBeNull();
        result.Email.Should().Be(request.Email.ToLowerInvariant());
    }
}
```

---

## 17. Integration Testing Strategy

- Use `WebApplicationFactory<Program>` with EF Core InMemory DB.
- Replace `ApplicationDbContext` registration with InMemory.
- Use `HttpClient` to test full HTTP request/response cycles.
- Seed data via scoped `ApplicationDbContext` in factory setup.

---

## 18. Cloud Provider Services (Conditional — apply only if cloud provider selected)

> Generate this section only when the user selected **Azure** or **AWS** in Question 1.8 of `scaffold.md`.

In the Repository + Unit of Work architecture, all cloud/external integrations live in the **`{Project}.ThirdParty`** project.

### 18.1 — Folder Structure

```
{Project}.ThirdParty/
├── Interfaces/
│   ├── IStorageService.cs          ← if storage selected
│   └── ISecretService.cs           ← if Key Vault / Secrets Manager selected
├── CloudServices/
│   ├── Storage/
│   │   ├── AzureBlobStorageService.cs    ← Azure only
│   │   └── AwsS3StorageService.cs        ← AWS only
│   └── Secrets/
│       ├── AzureKeyVaultSecretService.cs ← Azure only
│       └── AwsSecretsManagerSecretService.cs ← AWS only
└── Extensions/
    └── ThirdPartyServiceExtensions.cs
```

### 18.2 — Interfaces

```csharp
// {Project}.ThirdParty/Interfaces/IStorageService.cs
/// <summary>Abstraction for cloud blob/object storage.</summary>
public interface IStorageService
{
    Task<string> UploadAsync(string containerName, string blobName, Stream content,
        string contentType, CancellationToken cancellationToken = default);
    Task<Stream> DownloadAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
    Task DeleteAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
}

// {Project}.ThirdParty/Interfaces/ISecretService.cs
/// <summary>Abstraction for runtime secret retrieval.</summary>
public interface ISecretService
{
    Task<string?> GetSecretAsync(string secretName, CancellationToken cancellationToken = default);
}
```

### 18.3 — Registration (`ThirdPartyServiceExtensions.cs`)

```csharp
// {Project}.ThirdParty/Extensions/ThirdPartyServiceExtensions.cs
public static class ThirdPartyServiceExtensions
{
    public static IServiceCollection AddThirdPartyServices(
        this IServiceCollection services, IConfiguration configuration)
    {
        // ── Azure Blob Storage ────────────────────────────────────────────
        services.AddSingleton<Azure.Core.TokenCredential>(_ => new DefaultAzureCredential());
        services.AddSingleton(sp =>
        {
            var accountName = configuration["AzureBlobStorage:AccountName"]!;
            return new BlobServiceClient(
                new Uri($"https://{accountName}.blob.core.windows.net"),
                sp.GetRequiredService<Azure.Core.TokenCredential>());
        });
        services.AddScoped<IStorageService, AzureBlobStorageService>();

        // ── Azure Key Vault ───────────────────────────────────────────────
        services.AddSingleton(sp =>
            new SecretClient(new Uri(configuration["Azure:KeyVaultUri"]!),
                sp.GetRequiredService<Azure.Core.TokenCredential>()));
        services.AddScoped<ISecretService, AzureKeyVaultSecretService>();

        // ── AWS S3 (replace Azure storage above if AWS) ───────────────────
        services.AddAWSService<IAmazonS3>();
        services.AddScoped<IStorageService, AwsS3StorageService>();

        // ── AWS Secrets Manager (replace Azure secrets above if AWS) ──────
        services.AddAWSService<IAmazonSecretsManager>();
        services.AddScoped<ISecretService, AwsSecretsManagerSecretService>();

        return services;
    }
}
```

### 18.4 — Usage in Service Layer

Inject `IStorageService` or `ISecretService` into `{Project}.Service` service classes via constructor. The Service layer depends on `{Project}.ThirdParty` interfaces; it never references AWS or Azure SDK packages directly.

```csharp
// {Project}.Service/Implementations/DocumentService.cs
public sealed class DocumentService : IDocumentService
{
    private readonly IStorageService _storageService;

    public DocumentService(IStorageService storageService)
        => _storageService = storageService;

    public async Task<string> UploadAsync(string fileName, Stream content,
        string contentType, CancellationToken cancellationToken)
    {
        var blobName = $"documents/{Guid.NewGuid()}_{fileName}";
        return await _storageService.UploadAsync("app-documents", blobName, content,
            contentType, cancellationToken);
    }
}
```

> **Note:** `{Project}.Api` references `{Project}.ThirdParty` to wire up DI. Call `services.AddThirdPartyServices(configuration)` in `Program.cs`.

---

## 19. Third-Party HTTP Client Integration with Polly (Conditional)

> Generate this section only when the user selected **Yes** in Question 1.9 of `scaffold.md`.

All typed HTTP client work lives in the **`{Project}.ThirdParty`** project.

### 19.1 — Folder Structure

```
{Project}.ThirdParty/
├── Interfaces/
│   └── I{ServiceName}Client.cs          ← one per third-party service
├── HttpClients/
│   └── {ServiceName}Client.cs           ← typed HttpClient implementation
└── Extensions/
    └── ThirdPartyServiceExtensions.cs   ← updated with HTTP client registrations
```

### 19.2 — Interface Pattern

```csharp
// {Project}.ThirdParty/Interfaces/IPaymentServiceClient.cs
/// <summary>Typed HTTP client for the Payment external service.</summary>
public interface IPaymentServiceClient
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request, CancellationToken cancellationToken = default);
}
```

### 19.3 — Implementation Pattern

```csharp
// {Project}.ThirdParty/HttpClients/PaymentServiceClient.cs
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

### 19.4 — Polly Registration (added to `ThirdPartyServiceExtensions.cs`)

```csharp
// Add inside AddThirdPartyServices method
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
```

### 19.5 — Usage in Service Layer

Inject `I{ServiceName}Client` into `{Project}.Service` service classes. The Service layer depends on the `{Project}.ThirdParty` interface — never on the HTTP client implementation directly.

```csharp
// {Project}.Service/Implementations/OrderService.cs
public sealed class OrderService : IOrderService
{
    private readonly IUnitOfWork _uow;
    private readonly IPaymentServiceClient _paymentClient;

    public OrderService(IUnitOfWork uow, IPaymentServiceClient paymentClient)
    {
        _uow = uow;
        _paymentClient = paymentClient;
    }

    public async Task<OrderResponse> PlaceOrderAsync(CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        var order = Order.Create(request.UserId, request.Items);
        await _uow.Orders.AddAsync(order, cancellationToken);

        var paymentResult = await _paymentClient.ProcessPaymentAsync(
            new PaymentRequest { Amount = order.Total, OrderId = order.Id }, cancellationToken);

        order.ConfirmPayment(paymentResult.TransactionId);
        await _uow.SaveChangesAsync(cancellationToken);

        return order.ToResponse();
    }
}
```
