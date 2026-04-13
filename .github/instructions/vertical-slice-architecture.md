# Vertical Slice Architecture — Detailed Instruction File

> This file is loaded automatically when the user selects **Vertical Slice Architecture** in `scaffold.md`.
> Follow every section completely. Do not skip features, rules, or code examples.

---

## 1. Architecture Overview

Vertical Slice Architecture organises code by **feature/use case** rather than by technical layer. Each "slice" is a self-contained vertical cut through the codebase: it owns its own request model, validator, handler, response, and data access logic.

```
┌─────────────────────────────────────────────────────────────┐
│                         Features                            │
│                                                             │
│  ┌─────────────────────┐   ┌─────────────────────────────┐ │
│  │    Users/           │   │    Auth/                    │ │
│  │  ├ CreateUser/      │   │  ├ Login/                   │ │
│  │  │  ├ Command.cs    │   │  │  ├ Command.cs            │ │
│  │  │  ├ Handler.cs    │   │  │  ├ Handler.cs            │ │
│  │  │  ├ Validator.cs  │   │  │  └ Validator.cs          │ │
│  │  │  └ Response.cs   │   │  └ RefreshToken/            │ │
│  │  ├ GetUser/         │   │     ├ Command.cs            │ │
│  │  │  ├ Query.cs      │   │     └ Handler.cs            │ │
│  │  │  ├ Handler.cs    │   └─────────────────────────────┘ │
│  │  │  └ Response.cs   │                                   │
│  │  └ DeleteUser/      │                                   │
│  └─────────────────────┘                                   │
│                                                             │
│    Shared: Domain Entities, DbContext, Common Models        │
└─────────────────────────────────────────────────────────────┘
```

**Core Principle:** Each slice is independent. Code within a slice can change without affecting other slices. Slices communicate only through shared domain entities or the MediatR bus — never by directly depending on other slices.

---

## 2. Solution Structure

### Solution Name: `{Project}`

```
{Project}.sln
│
├── src/
│   ├── {Project}.Api/
│   └── {Project}.Infrastructure/
│
└── tests/  (if test projects selected)
    ├── {Project}.Tests.Unit/
    └── {Project}.Tests.Integration/
```

> Unlike other architectures, Vertical Slice consolidates most application code into `{Project}.Api` organised by feature, with `{Project}.Infrastructure` providing only EF Core, Identity, and external integrations.

---

## 3. Project Descriptions

| Project | Type | Responsibility |
|---------|------|----------------|
| `{Project}.Api` | ASP.NET Core Web API | Feature slices (Commands, Queries, Handlers, Validators), controllers or minimal API endpoints, domain entities, common models, DI wiring, middleware |
| `{Project}.Infrastructure` | Class Library | EF Core DbContext, configurations, migrations, external service implementations |

---

## 4. Project Reference Rules

```
{Project}.Api
    └── references {Project}.Infrastructure

{Project}.Infrastructure
    └── NO references to {Project}.Api
```

> The `{Project}.Api` project contains all feature slices. Domain entities and the `ApiResponse<T>` wrapper are defined in the `Common/` folder within `{Project}.Api`.

---

## 5. Naming Conventions

| Concept | Pattern | Example |
|---------|---------|---------|
| Feature folder | `Features/{Domain}/{Action}/` | `Features/Users/CreateUser/` |
| Command | `{Action}{Entity}Command` | `CreateUserCommand` |
| Query | `{Action}{Entity}Query` | `GetUserByIdQuery` |
| Handler | `{Action}{Entity}Handler` | `CreateUserHandler` |
| Validator | `{Action}{Entity}Validator` | `CreateUserValidator` |
| Response | `{Action}{Entity}Response` | `CreateUserResponse` |
| Domain Entity | PascalCase noun | `User`, `Order` |
| Entity Config | `{Entity}Configuration` | `UserConfiguration` |
| Domain Exception | `{Entity}{Issue}Exception` | `UserNotFoundException` |
| Endpoint class | `{Action}{Entity}Endpoint` (optional Minimal API) | `CreateUserEndpoint` |

---

## 6. Folder Structure

### `{Project}.Api`

```
{Project}.Api/
│
├── Features/
│   ├── Users/
│   │   ├── CreateUser/
│   │   │   ├── CreateUserCommand.cs          ← IRequest<CreateUserResponse>
│   │   │   ├── CreateUserHandler.cs          ← IRequestHandler<...>
│   │   │   ├── CreateUserValidator.cs        ← AbstractValidator<...>
│   │   │   └── CreateUserResponse.cs
│   │   ├── GetUserById/
│   │   │   ├── GetUserByIdQuery.cs           ← IRequest<GetUserByIdResponse>
│   │   │   ├── GetUserByIdHandler.cs
│   │   │   └── GetUserByIdResponse.cs
│   │   ├── GetAllUsers/
│   │   │   ├── GetAllUsersQuery.cs
│   │   │   ├── GetAllUsersHandler.cs
│   │   │   └── GetAllUsersResponse.cs
│   │   ├── UpdateUser/
│   │   │   ├── UpdateUserCommand.cs
│   │   │   ├── UpdateUserHandler.cs
│   │   │   ├── UpdateUserValidator.cs
│   │   │   └── UpdateUserResponse.cs
│   │   └── DeleteUser/
│   │       ├── DeleteUserCommand.cs
│   │       └── DeleteUserHandler.cs
│   └── Auth/
│       ├── Login/
│       │   ├── LoginCommand.cs
│       │   ├── LoginHandler.cs
│       │   ├── LoginValidator.cs
│       │   └── LoginResponse.cs
│       ├── Register/
│       │   ├── RegisterCommand.cs
│       │   ├── RegisterHandler.cs
│       │   ├── RegisterValidator.cs
│       │   └── RegisterResponse.cs
│       └── RefreshToken/
│           ├── RefreshTokenCommand.cs
│           ├── RefreshTokenHandler.cs
│           └── RefreshTokenResponse.cs
│
├── Common/
│   ├── Behaviours/
│   │   ├── ValidationBehaviour.cs
│   │   └── LoggingBehaviour.cs
│   ├── Domain/
│   │   ├── BaseEntity.cs
│   │   └── User.cs
│   ├── Enums/
│   │   └── UserStatus.cs
│   ├── Exceptions/
│   │   ├── DomainException.cs
│   │   ├── UserNotFoundException.cs
│   │   └── EmailAlreadyExistsException.cs
│   └── Models/
│       └── ApiResponse.cs
│
├── Controllers/
│   ├── UsersController.cs
│   └── AuthController.cs
│
├── Middleware/
│   └── ExceptionHandlingMiddleware.cs
│
├── appsettings.json
├── appsettings.Development.json
└── Program.cs
```

### `{Project}.Infrastructure`

```
{Project}.Infrastructure/
├── Persistence/
│   ├── ApplicationDbContext.cs
│   ├── Configurations/
│   │   └── UserConfiguration.cs
│   └── Migrations/
├── Identity/
│   ├── ApplicationUser.cs
│   └── TokenService.cs
└── DependencyInjection.cs
```

---

## 7. Common Layer — Domain & Shared Models

### BaseEntity

```csharp
// Common/Domain/BaseEntity.cs
namespace {Project}.Api.Common.Domain;

public abstract class BaseEntity
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; set; }
    public bool IsDeleted { get; set; } = false;
}
```

### User Domain Entity

```csharp
// Common/Domain/User.cs
namespace {Project}.Api.Common.Domain;

public sealed class User : BaseEntity
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public UserStatus Status { get; set; } = UserStatus.Active;
    public string? RefreshToken { get; set; }
    public DateTime? RefreshTokenExpiry { get; set; }
}
```

### ApiResponse Wrapper

```csharp
// Common/Models/ApiResponse.cs
namespace {Project}.Api.Common.Models;

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

---

## 8. Feature Slice — Full Example (CreateUser)

### Command

```csharp
// Features/Users/CreateUser/CreateUserCommand.cs
namespace {Project}.Api.Features.Users.CreateUser;

public sealed record CreateUserCommand(
    string FirstName,
    string LastName,
    string Email,
    string Password) : IRequest<CreateUserResponse>;
```

### Response

```csharp
// Features/Users/CreateUser/CreateUserResponse.cs
namespace {Project}.Api.Features.Users.CreateUser;

public sealed record CreateUserResponse(
    Guid Id,
    string FirstName,
    string LastName,
    string Email,
    DateTime CreatedAt);
```

### Validator

```csharp
// Features/Users/CreateUser/CreateUserValidator.cs
namespace {Project}.Api.Features.Users.CreateUser;

public sealed class CreateUserValidator : AbstractValidator<CreateUserCommand>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.FirstName)
            .NotEmpty().WithMessage("First name is required.")
            .MaximumLength(100).WithMessage("First name must not exceed 100 characters.");

        RuleFor(x => x.LastName)
            .NotEmpty().WithMessage("Last name is required.")
            .MaximumLength(100).WithMessage("Last name must not exceed 100 characters.");

        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required.")
            .EmailAddress().WithMessage("A valid email address is required.");

        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Password is required.")
            .MinimumLength(8).WithMessage("Password must be at least 8 characters.")
            .Matches(@"[A-Z]").WithMessage("Password must contain at least one uppercase letter.")
            .Matches(@"[0-9]").WithMessage("Password must contain at least one digit.");
    }
}
```

### Handler

```csharp
// Features/Users/CreateUser/CreateUserHandler.cs
namespace {Project}.Api.Features.Users.CreateUser;

public sealed class CreateUserHandler : IRequestHandler<CreateUserCommand, CreateUserResponse>
{
    private readonly ApplicationDbContext _dbContext;
    private readonly ILogger<CreateUserHandler> _logger;

    public CreateUserHandler(ApplicationDbContext dbContext, ILogger<CreateUserHandler> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task<CreateUserResponse> Handle(
        CreateUserCommand request,
        CancellationToken cancellationToken)
    {
        var emailExists = await _dbContext.Users
            .AnyAsync(u => u.Email == request.Email.ToLowerInvariant(), cancellationToken);

        if (emailExists)
            throw new EmailAlreadyExistsException(request.Email);

        var user = new User
        {
            FirstName = request.FirstName,
            LastName = request.LastName,
            Email = request.Email.ToLowerInvariant(),
            PasswordHash = BCrypt.Net.BCrypt.HashPassword(request.Password)
        };

        _dbContext.Users.Add(user);
        await _dbContext.SaveChangesAsync(cancellationToken);

        _logger.LogInformation("User {Email} created with ID {Id}", user.Email, user.Id);

        return new CreateUserResponse(user.Id, user.FirstName, user.LastName, user.Email, user.CreatedAt);
    }
}
```

---

## 9. Feature Slice — Full Example (GetUserById)

### Query

```csharp
// Features/Users/GetUserById/GetUserByIdQuery.cs
namespace {Project}.Api.Features.Users.GetUserById;

public sealed record GetUserByIdQuery(Guid Id) : IRequest<GetUserByIdResponse>;
```

### Response

```csharp
// Features/Users/GetUserById/GetUserByIdResponse.cs
namespace {Project}.Api.Features.Users.GetUserById;

public sealed record GetUserByIdResponse(
    Guid Id,
    string FirstName,
    string LastName,
    string Email,
    UserStatus Status,
    DateTime CreatedAt);
```

### Handler

```csharp
// Features/Users/GetUserById/GetUserByIdHandler.cs
namespace {Project}.Api.Features.Users.GetUserById;

public sealed class GetUserByIdHandler : IRequestHandler<GetUserByIdQuery, GetUserByIdResponse>
{
    private readonly ApplicationDbContext _dbContext;

    public GetUserByIdHandler(ApplicationDbContext dbContext) => _dbContext = dbContext;

    public async Task<GetUserByIdResponse> Handle(
        GetUserByIdQuery request,
        CancellationToken cancellationToken)
    {
        var user = await _dbContext.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == request.Id, cancellationToken)
            ?? throw new UserNotFoundException(request.Id);

        return new GetUserByIdResponse(
            user.Id, user.FirstName, user.LastName,
            user.Email, user.Status, user.CreatedAt);
    }
}
```

---

## 10. Feature Slice — Auth (Login)

### Command

```csharp
// Features/Auth/Login/LoginCommand.cs
namespace {Project}.Api.Features.Auth.Login;

public sealed record LoginCommand(string Email, string Password) : IRequest<LoginResponse>;
```

### Response

```csharp
// Features/Auth/Login/LoginResponse.cs
namespace {Project}.Api.Features.Auth.Login;

public sealed record LoginResponse(
    string AccessToken,
    string RefreshToken,
    DateTime ExpiresAt,
    string Email,
    string FullName);
```

### Validator

```csharp
// Features/Auth/Login/LoginValidator.cs
namespace {Project}.Api.Features.Auth.Login;

public sealed class LoginValidator : AbstractValidator<LoginCommand>
{
    public LoginValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Password).NotEmpty();
    }
}
```

### Handler

```csharp
// Features/Auth/Login/LoginHandler.cs
namespace {Project}.Api.Features.Auth.Login;

public sealed class LoginHandler : IRequestHandler<LoginCommand, LoginResponse>
{
    private readonly ApplicationDbContext _dbContext;
    private readonly ITokenService _tokenService;
    private readonly JwtSettings _jwtSettings;

    public LoginHandler(
        ApplicationDbContext dbContext,
        ITokenService tokenService,
        IOptions<JwtSettings> jwtSettings)
    {
        _dbContext = dbContext;
        _tokenService = tokenService;
        _jwtSettings = jwtSettings.Value;
    }

    public async Task<LoginResponse> Handle(LoginCommand request, CancellationToken cancellationToken)
    {
        var user = await _dbContext.Users
            .FirstOrDefaultAsync(u => u.Email == request.Email.ToLowerInvariant(), cancellationToken)
            ?? throw new UserNotFoundException(request.Email);

        if (!BCrypt.Net.BCrypt.Verify(request.Password, user.PasswordHash))
            throw new UnauthorizedAccessException("Invalid credentials.");

        var accessToken = _tokenService.GenerateAccessToken(user);
        var refreshToken = _tokenService.GenerateRefreshToken();
        var expiry = DateTime.UtcNow.AddDays(_jwtSettings.RefreshTokenExpiryDays);

        user.RefreshToken = refreshToken;
        user.RefreshTokenExpiry = expiry;
        user.UpdatedAt = DateTime.UtcNow;

        await _dbContext.SaveChangesAsync(cancellationToken);

        return new LoginResponse(
            accessToken,
            refreshToken,
            DateTime.UtcNow.AddMinutes(_jwtSettings.ExpiryMinutes),
            user.Email,
            $"{user.FirstName} {user.LastName}");
    }
}
```

---

## 11. MediatR Pipeline Behaviours

### Validation Behaviour

```csharp
// Common/Behaviours/ValidationBehaviour.cs
namespace {Project}.Api.Common.Behaviours;

public sealed class ValidationBehaviour<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehaviour(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(e => e is not null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next();
    }
}
```

### Logging Behaviour

```csharp
// Common/Behaviours/LoggingBehaviour.cs
namespace {Project}.Api.Common.Behaviours;

public sealed class LoggingBehaviour<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<LoggingBehaviour<TRequest, TResponse>> _logger;

    public LoggingBehaviour(ILogger<LoggingBehaviour<TRequest, TResponse>> logger)
        => _logger = logger;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Handling {RequestName}: {@Request}", requestName, request);

        var response = await next();

        _logger.LogInformation("Handled {RequestName}", requestName);
        return response;
    }
}
```

---

## 12. Controllers

### UsersController

```csharp
// Controllers/UsersController.cs
namespace {Project}.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public sealed class UsersController : ControllerBase
{
    private readonly ISender _mediator;

    public UsersController(ISender mediator) => _mediator = mediator;

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<GetUserByIdResponse>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(new GetUserByIdQuery(id), cancellationToken);
        return Ok(ApiResponse<GetUserByIdResponse>.Ok(result));
    }

    [HttpGet]
    [ProducesResponseType(typeof(ApiResponse<IReadOnlyList<GetAllUsersResponse>>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll(CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(new GetAllUsersQuery(), cancellationToken);
        return Ok(ApiResponse<IReadOnlyList<GetAllUsersResponse>>.Ok(result));
    }

    [HttpPost]
    [AllowAnonymous]
    [ProducesResponseType(typeof(ApiResponse<CreateUserResponse>), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> Create(
        [FromBody] CreateUserCommand command,
        CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(command, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id = result.Id },
            ApiResponse<CreateUserResponse>.Ok(result, "User created successfully."));
    }

    [HttpPut("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<UpdateUserResponse>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(
        Guid id,
        [FromBody] UpdateUserCommandBody body,
        CancellationToken cancellationToken)
    {
        var command = new UpdateUserCommand(id, body.FirstName, body.LastName);
        var result = await _mediator.Send(command, cancellationToken);
        return Ok(ApiResponse<UpdateUserResponse>.Ok(result));
    }

    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(Guid id, CancellationToken cancellationToken)
    {
        await _mediator.Send(new DeleteUserCommand(id), cancellationToken);
        return NoContent();
    }
}
```

### AuthController

```csharp
// Controllers/AuthController.cs
namespace {Project}.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public sealed class AuthController : ControllerBase
{
    private readonly ISender _mediator;

    public AuthController(ISender mediator) => _mediator = mediator;

    [HttpPost("register")]
    [ProducesResponseType(typeof(ApiResponse<RegisterResponse>), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Register(
        [FromBody] RegisterCommand command, CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(command, cancellationToken);
        return StatusCode(StatusCodes.Status201Created,
            ApiResponse<RegisterResponse>.Ok(result, "Registration successful."));
    }

    [HttpPost("login")]
    [ProducesResponseType(typeof(ApiResponse<LoginResponse>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> Login(
        [FromBody] LoginCommand command, CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(command, cancellationToken);
        return Ok(ApiResponse<LoginResponse>.Ok(result));
    }

    [HttpPost("refresh-token")]
    [ProducesResponseType(typeof(ApiResponse<RefreshTokenResponse>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> RefreshToken(
        [FromBody] RefreshTokenCommand command, CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(command, cancellationToken);
        return Ok(ApiResponse<RefreshTokenResponse>.Ok(result));
    }
}
```

---

## 13. Infrastructure Layer — Implementation Details

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
        foreach (var entry in ChangeTracker.Entries<BaseEntity>()
            .Where(e => e.State == EntityState.Modified))
        {
            entry.Entity.UpdatedAt = DateTime.UtcNow;
        }
        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

### Entity Configuration

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

        services.AddScoped<ITokenService, TokenService>();
        services.Configure<JwtSettings>(configuration.GetSection(nameof(JwtSettings)));

        return services;
    }
}
```

---

## 14. Program.cs

```csharp
using {Project}.Infrastructure;
using Serilog;
using SharpGrip.FluentValidation.AutoValidation.Mvc.Extensions;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day));

builder.Services.AddInfrastructure(builder.Configuration);

// MediatR — registers all handlers, behaviours automatically
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehaviour<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehaviour<,>));
});

builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "{Project} API", Version = "v1" });
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "Bearer",
        BearerFormat = "JWT"
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
            },
            Array.Empty<string>()
        }
    });
});

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

---

## 15. appsettings.json Template

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

## 16. NuGet Packages Per Project

### `{Project}.Api`

```bash
dotnet add package MediatR
dotnet add package FluentValidation
dotnet add package FluentValidation.AspNetCore
dotnet add package SharpGrip.FluentValidation.AutoValidation.Mvc
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Swashbuckle.AspNetCore
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
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

---

## 17. CLI Commands to Scaffold the Solution

```bash
dotnet new sln -n {Project}

dotnet new webapi -o src/{Project}.Api --framework net10.0
dotnet new classlib -o src/{Project}.Infrastructure --framework net10.0

dotnet sln add src/{Project}.Api/{Project}.Api.csproj
dotnet sln add src/{Project}.Infrastructure/{Project}.Infrastructure.csproj

dotnet add src/{Project}.Api reference src/{Project}.Infrastructure

# Test projects (if selected)
dotnet new xunit -o tests/{Project}.Tests.Unit --framework net10.0
dotnet new xunit -o tests/{Project}.Tests.Integration --framework net10.0
dotnet sln add tests/{Project}.Tests.Unit/{Project}.Tests.Unit.csproj
dotnet sln add tests/{Project}.Tests.Integration/{Project}.Tests.Integration.csproj
dotnet add tests/{Project}.Tests.Unit reference src/{Project}.Api
dotnet add tests/{Project}.Tests.Integration reference src/{Project}.Api src/{Project}.Infrastructure
```

---

## 18. EF Core Migration Commands

```bash
dotnet ef migrations add InitialCreate --project src/{Project}.Infrastructure --startup-project src/{Project}.Api
dotnet ef database update --project src/{Project}.Infrastructure --startup-project src/{Project}.Api
```

---

## 19. Unit Testing Rules for Vertical Slice Architecture

- Test **Handlers** in isolation — inject a real `ApplicationDbContext` backed by EF Core InMemory.
- This is the preferred approach for Vertical Slice unit tests as handlers directly access DbContext.
- Alternatively, wrap DbContext behind an interface `IApplicationDbContext` and mock it with Moq.
- Test **Validators** independently by calling `validator.Validate(command)`.
- Use `AutoFixture` for generating test commands/queries.

**Example Handler Unit Test (with InMemory DB):**

```csharp
public class CreateUserHandlerTests
{
    private readonly ApplicationDbContext _dbContext;
    private readonly CreateUserHandler _sut;
    private readonly Mock<ILogger<CreateUserHandler>> _loggerMock = new();

    public CreateUserHandlerTests()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase("UnitTest_" + Guid.NewGuid())
            .Options;

        _dbContext = new ApplicationDbContext(options);
        _sut = new CreateUserHandler(_dbContext, _loggerMock.Object);
    }

    [Fact]
    public async Task Handle_WithValidCommand_CreatesUserInDatabase()
    {
        var command = new CreateUserCommand("Jane", "Smith", "jane@example.com", "SecurePass1!");

        var response = await _sut.Handle(command, CancellationToken.None);

        response.Should().NotBeNull();
        response.Email.Should().Be("jane@example.com");

        var userInDb = await _dbContext.Users.FindAsync(response.Id);
        userInDb.Should().NotBeNull();
        userInDb!.Email.Should().Be("jane@example.com");
    }

    [Fact]
    public async Task Handle_WithDuplicateEmail_ThrowsEmailAlreadyExistsException()
    {
        _dbContext.Users.Add(new User
        {
            Email = "existing@example.com",
            PasswordHash = "hash",
            FirstName = "Existing",
            LastName = "User"
        });
        await _dbContext.SaveChangesAsync();

        var command = new CreateUserCommand("New", "User", "existing@example.com", "Password1!");

        var act = () => _sut.Handle(command, CancellationToken.None);

        await act.Should().ThrowAsync<EmailAlreadyExistsException>();
    }
}
```

---

## 20. Integration Testing Rules for Vertical Slice Architecture

- Use `WebApplicationFactory<Program>` and replace the DB with InMemory.
- Test the full HTTP request → Controller → MediatR → Handler → InMemory DB pipeline.
- Group tests by feature folder matching the production `Features/` structure.

```csharp
public class CreateUserTests : IClassFixture<TestWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly IFixture _fixture = new Fixture();

    public CreateUserTests(TestWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task CreateUser_WithValidRequest_Returns201AndUserData()
    {
        var command = new CreateUserCommand(
            "Integration", "Test", "integration@test.com", "Password1!");

        var response = await _client.PostAsJsonAsync("/api/users", command);

        response.StatusCode.Should().Be(HttpStatusCode.Created);

        var body = await response.Content.ReadFromJsonAsync<ApiResponse<CreateUserResponse>>();
        body!.Success.Should().BeTrue();
        body.Data!.Email.Should().Be("integration@test.com");
    }
}
```

---

## 21. Multi-Tenant Support for Vertical Slice (If Selected)

Add `TenantId` to `BaseEntity`. Resolve the tenant in `TenantMiddleware` and store it in a scoped `ITenantContext` service. In each handler, filter queries by `tenantContext.TenantId`. Apply global query filter on `ApplicationDbContext.OnModelCreating`.

```csharp
// Each handler resolves tenant from injected ITenantContext
var users = await _dbContext.Users
    .Where(u => u.TenantId == _tenantContext.TenantId)
    .ToListAsync(cancellationToken);
```

---

## 22. Cloud Provider Services (Conditional — apply only if cloud provider selected)

> Generate this section only when the user selected **Azure** or **AWS** in Question 1.8 of `scaffold.md`.

In Vertical Slice Architecture, cloud service **implementations** live in `{Project}.Infrastructure` and **interfaces** live inside `{Project}.Api/Common/Interfaces/`.

### 22.1 — Folder Structure

```
{Project}.Api/
├── Common/
│   └── Interfaces/
│       ├── IStorageService.cs    ← if storage selected
│       └── ISecretService.cs     ← if Key Vault / Secrets Manager selected

{Project}.Infrastructure/
├── CloudServices/
│   ├── Storage/
│   │   ├── AzureBlobStorageService.cs    ← Azure only
│   │   └── AwsS3StorageService.cs        ← AWS only
│   └── Secrets/
│       ├── AzureKeyVaultSecretService.cs ← Azure only
│       └── AwsSecretsManagerSecretService.cs ← AWS only
```

### 22.2 — Interfaces

```csharp
// {Project}.Api/Common/Interfaces/IStorageService.cs
/// <summary>Abstraction for cloud blob/object storage operations.</summary>
public interface IStorageService
{
    Task<string> UploadAsync(string containerName, string blobName, Stream content,
        string contentType, CancellationToken cancellationToken = default);
    Task<Stream> DownloadAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
    Task DeleteAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
}

// {Project}.Api/Common/Interfaces/ISecretService.cs
/// <summary>Abstraction for runtime secret retrieval.</summary>
public interface ISecretService
{
    Task<string?> GetSecretAsync(string secretName, CancellationToken cancellationToken = default);
}
```

### 22.3 — Registration (`{Project}.Infrastructure/DependencyInjection.cs`)

```csharp
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

// ── AWS S3 (replace Azure Blob Storage above if AWS selected) ─────────────
services.AddAWSService<IAmazonS3>();
services.AddScoped<IStorageService, AwsS3StorageService>();

// ── AWS Secrets Manager (replace Azure Key Vault above if AWS selected) ───
services.AddAWSService<IAmazonSecretsManager>();
services.AddScoped<ISecretService, AwsSecretsManagerSecretService>();
```

### 22.4 — Usage Inside Feature Slice Handlers

Inject `IStorageService` or `ISecretService` directly into MediatR handlers. Interfaces are resolved from the DI container — the handler never references the Azure/AWS SDK.

```csharp
// Features/Documents/UploadDocument/UploadDocumentHandler.cs
public sealed class UploadDocumentHandler : IRequestHandler<UploadDocumentCommand, string>
{
    private readonly IStorageService _storageService;

    public UploadDocumentHandler(IStorageService storageService)
        => _storageService = storageService;

    public async Task<string> Handle(UploadDocumentCommand request, CancellationToken cancellationToken)
    {
        var blobName = $"docs/{request.UserId}/{Guid.NewGuid()}_{request.FileName}";
        return await _storageService.UploadAsync("app-documents", blobName,
            request.FileStream, request.ContentType, cancellationToken);
    }
}
```

---

## 23. Third-Party HTTP Client Integration with Polly (Conditional)

> Generate this section only when the user selected **Yes** in Question 1.9 of `scaffold.md`.

### 23.1 — Folder Structure

```
{Project}.Api/
├── Common/
│   └── Interfaces/
│       └── HttpClients/
│           └── I{ServiceName}Client.cs      ← interface per service

{Project}.Infrastructure/
├── HttpClients/
│   ├── {ServiceName}Client.cs               ← typed HttpClient implementation
│   └── HttpClientServiceExtensions.cs       ← Polly registration
```

### 23.2 — Interface Pattern

```csharp
// {Project}.Api/Common/Interfaces/HttpClients/IPaymentServiceClient.cs
/// <summary>Typed HTTP client contract for the Payment external service.</summary>
public interface IPaymentServiceClient
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request, CancellationToken cancellationToken = default);
}
```

### 23.3 — Implementation Pattern

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

### 23.4 — Polly Registration

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

Call `services.AddHttpClients(configuration)` from `{Project}.Infrastructure/DependencyInjection.cs`.

### 23.5 — Usage Inside Feature Slice Handlers

```csharp
// Features/Orders/PlaceOrder/PlaceOrderHandler.cs
public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, PlaceOrderResponse>
{
    private readonly ApplicationDbContext _dbContext;
    private readonly IPaymentServiceClient _paymentClient;

    public PlaceOrderHandler(ApplicationDbContext dbContext, IPaymentServiceClient paymentClient)
    {
        _dbContext = dbContext;
        _paymentClient = paymentClient;
    }

    public async Task<PlaceOrderResponse> Handle(PlaceOrderCommand request, CancellationToken cancellationToken)
    {
        var order = new Order { UserId = request.UserId, Total = request.Total };
        _dbContext.Orders.Add(order);

        var paymentResult = await _paymentClient.ProcessPaymentAsync(
            new PaymentRequest { Amount = request.Total, OrderId = order.Id }, cancellationToken);

        order.TransactionId = paymentResult.TransactionId;
        await _dbContext.SaveChangesAsync(cancellationToken);

        return new PlaceOrderResponse(order.Id, paymentResult.TransactionId);
    }
}
```
